# ZSET

## 主要结构 
db.dict.ht[].table[]=dictEntry
dictEntry.v.val=robj
robj.ptr=zset
zset{dict {key=member,score=distEntry.v.val},zsl{跳表，支持范围查询}}

## zscore
zscoreCommand->lookupKeyReadOrReply->dictFind{dictEntry=he = d->ht[table].table[idx];//dictEntry.v.val=robj}->robj{ptr=zset}->zsetScore{dictEntry *de = dictFind(zs->dict, member); score=dictEntry.v.val}

zset.dict.ht[].table[]=dictEntry
score=dictEntry.v.val

## zadd
ZADD page_rank 9 baidu.com 8 bing.com

## zset zadd 概要源码
```
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;

typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;



typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;


server.c 
 {"zadd",zaddCommand,-4,
     "write use-memory fast @sortedset",
     0,NULL,1,1,1,0,0,0}


zaddCommand->zaddGenericCommand->createZsetObject[或者createZsetZiplistObject]

/* This generic command implements both ZADD and ZINCRBY. */
void zaddGenericCommand(client *c, int flags) {
   ...

    /* Start parsing all the scores, we need to emit any syntax error
     * before executing additions to the sorted set, as the command should
     * either execute fully or nothing at all. */
    scores = zmalloc(sizeof(double)*elements);
    for (j = 0; j < elements; j++) {
        if (getDoubleFromObjectOrReply(c,c->argv[scoreidx+j*2],&scores[j],NULL)
            != C_OK) goto cleanup;
    }

    /* Lookup the key and create the sorted set if does not exist. */
    //或查找，或新建zset结构 zobj.ptr指向zset
    zobj = lookupKeyWrite(c->db,key);
    if (checkType(c,zobj,OBJ_ZSET)) goto cleanup;
    if (zobj == NULL) {
        if (xx) goto reply_to_client; /* No key + XX option: nothing to do. */
        if (server.zset_max_ziplist_entries == 0 ||
            server.zset_max_ziplist_value < sdslen(c->argv[scoreidx+1]->ptr))
        {
            zobj = createZsetObject();
        } else {
            zobj = createZsetZiplistObject();
        }
        dbAdd(c->db,key,zobj);  redisDb.dict
    }
    ...

    //或新建或找到zset结构之后，再往这个结构里添加数据
    int retval = zsetAdd(zobj, score, ele, flags, &retflags, &newscore);
   ...
}


robj *createZsetObject(void) {
    zset *zs = zmalloc(sizeof(*zs));
    robj *o;

    zs->dict = dictCreate(&zsetDictType,NULL);
    zs->zsl = zslCreate();
    o = createObject(OBJ_ZSET,zs);
    o->encoding = OBJ_ENCODING_SKIPLIST;
    return o;
}

int zsetAdd(robj *zobj, double score, sds ele, int in_flags, int *out_flags, double *newscore) {
...
//把数据{member=ele，score}转成zskiplistnode，添加进入跳表之中
            ele = sdsdup(ele);
            znode = zslInsert(zs->zsl,score,ele);

...
}




```

## dict hash的方式

```
#define dictHashKey(d, key) (d)->type->hashFunction(key)
uint64_t h = dictHashKey(d, key);
uint64_t idx = h & d->ht[table].sizemask;
dictEntry he = d->ht[table].table[idx];
```