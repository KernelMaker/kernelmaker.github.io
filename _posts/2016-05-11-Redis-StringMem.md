---
layout: post
title: redis 一组kv实际内存占用计算
---

之前帮公司DBA同事调研Redis内存占用的问题，比如redis在执行一条"set aaa bbb"命令后，这组键值对"aaa" -> "bbb"自身占用6个字节，那为了存储它们redis实际需要占用多少字节呢？本文以redis（tag 2.8.20）和64位机器为参考来一探究竟

## SDS

redis对字符串做了自己的封装，叫sds，定义如下：

```cpp
typedef char *sds;

struct sdshdr {
    unsigned int len;
    unsigned int free;
    char buf[];
};
```
其实就是给字符串最前面多加两个`unsigned int`来保存字符串信息，len是总长度，free是当前可用长度，所以假设当前有一个字符串"aaa"，那么通过sds来存它最少需要多少个字节呢，很简单4(len)+4(free)+3("aaa")+1('\0') = 12，也就是说通过sds来保存一个字符串，会在字符串实际占用之上再多占用9个字节

## Object

sds仅是对字符串的封装，在其之上，redis还会封装一层RedisObject，也很简单，定义如下：

```cpp

typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */
    int refcount;
    void *ptr;
} robj;
```
每个robj占用((4b(type)+4b(encoding)+24b(lru))/8)+4(refcount)+8(ptr) = 16字节，ptr指向所存内容，以"aaa"为例，包成sds占用12字节，再包成robj，实际占用16+12 = 28字节，其中ptr指向sds，所以对字符串的robj的内存占用公式可以总结为"N+9+16"，N为字符串长度

_**注:**_ robj的ptr并不总是保存实际内容，假设字符为"123"，当然type为`REDIS_STRING`，redis会在tryObjectEncoding函数中判断robj的ptr指向的sds能不能转化成整数，假如可以，那么而直接将ptr的值赋为123，并释放之前的sds，此时encoding为`REDIS_ENCODING_INT`表明这个ptr保存的是一个整数，所以实际占用仅为一个robj的大小，即16字节

## 计算

对sds有了初步了解之后，我们就可以开始计算了，当redis接收到客户端"set aaa bbb"命令之后，

第一步就是参数解析，直接看关键代码：

首先是协议解析，调用栈是readQueryFromClient -> processInputBuffer -> processMultibulkBuffer，在processMultibulkBuffer中会对每个参数调用createStringObject生成robj，先来看createStringObject的实现：

```cpp
robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));
    o->type = type;
    o->encoding = REDIS_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;

    /* Set the LRU to the current lruclock (minutes resolution). */
    o->lru = server.lruclock;
    return o;
}
```
那么对于"set"，"aaa"，"bbb"会生成3个robj，各占28个字节，不过对于set这个robj仅仅是用来查找命令lookupCommand使用，后来会释放，对"aaa"这个robj也是会用新的"aaa"的sds来存储，robj也会释放，前两个不计算在内，所以目前仅有"bbb"总共占56字节


第二步，命令的执行，调用栈是processCommand -> call -> proc(此处为setcommand) -> setGenericCommand -> setKey，最终数据是在setKey中被存在db中，这里有一点特殊说明一下，在setcommand调用setGenericCommand之前会调用

```cpp
c->argv[2] = tryObjectEncoding(c->argv[2]);
```
这里会对命令中的value进行上面说的tryObjectEncoding，此处argv[2]是包含"bbb"的robj，所以这里tryObjectEncoding后，这个robj不会变小，但如果此处是包含"123"的robj，那么经过tryObjectEncoding后，大小会从28变为16(具体原因参考Object一节 _**注**_ 部分)

接着往下看，setKey的定义如下：

```cpp
void setKey(redisDb *db, robj *key, robj *val) {
    if (lookupKeyWrite(db,key) == NULL) {
        dbAdd(db,key,val);
    } else {
        dbOverwrite(db,key,val);
    }
    incrRefCount(val);
    removeExpire(db,key);
    signalModifiedKey(db,key);
}
```

可以看到我们会将"aaa"和"bbb"的robj指针作为第二、三参数传给它，这里假设这个key之前不存在，那么会调用dbAdd把他插入到db中(db实际是个hash表）

```cpp
void dbAdd(redisDb *db, robj *key, robj *val) {
    sds copy = sdsdup(key->ptr); // 将key的robj转换为对应的sds，在dict中的key用
    //sds的形式存
    int retval = dictAdd(db->dict, copy, val);

    redisAssertWithInfo(NULL,key,retval == REDIS_OK);
    if (val->type == REDIS_LIST) signalListAsReady(db, key);
}

int dictAdd(dict *d, void *key, void *val)
{
    dictEntry *entry = dictAddRaw(d,key); //生成新的dictEntry并赋值key字段

    if (!entry) return DICT_ERR;
    dictSetVal(d, entry, val); //给dictEntry的v字段赋值，指向包含"bbb"的robj
    return DICT_OK;
}

dictEntry *dictAddRaw(dict *d, void *key)
{
    int index;
    dictEntry *entry;
    dictht *ht;

    if (dictIsRehashing(d)) _dictRehashStep(d);

    /* Get the index of the new element, or -1 if
     * the element already exists. */
    if ((index = _dictKeyIndex(d, key)) == -1)
        return NULL;

    /* Allocate the memory and store the new entry */
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    entry = zmalloc(sizeof(*entry)); //生成新的dictEntry
    entry->next = ht->table[index]; //插入到index位置的桶中
    ht->table[index] = entry;
    ht->used++;

    /* Set the hash entry fields. */
    dictSetKey(d, entry, key); //给dictEntry的key字段赋值，指向"aaa"的sds
    return entry;
}
```
dbAdd调用dictAdd，最终由dictAdd将一个sds和一个object插入到db中，说是插入，其实就是对这组键值对调用dictAddRaw生成一个dictEntry，并把他插入到按key求hash值索引到的桶中，说到这里已经明确了，这组键值对最终实际保存的位置就是在dictEntry中，它的大小就是最终实际大小，来看定义

```cpp
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
```
一个dictEntry的大小是8(key)+8(v)+8(next) = 24字节，key是一个"aaa"的sds指针，v是一个指向包含"bbb"的robj的指针，next是指向对应桶中第二个dictEntry的指针

## 结论
* 在执行"set aaa bbb"命令后，redis会用24(dictEntry)+12(sds("aaa"))+28(robj("bbb")) = 64字节来存储
* 在执行"set aaa 10000"命令后，redis会用24(dictEntry)+12(sds("aaa"))+16(robj("10000")) = 52字节来存储（redis对整数10000之内robj创建了shared object，也就是说如果这里不是10000而是123的话，不会为123新创建robj而是直接增加shared object中已有123的robj的计数，这样空间占用更小）
* 上面说的64和52只是redis申请的字节数，实际占用还要根据具体allocator来看，感兴趣的话可以参考我的"TCMalloc"系列来进一步阅读^^

## 补充
在第一步协议解析之后，redis会对每个参数生成对应的robj并且存储在redisClient的argv中，执行完命令之后会调用resetClient来释放这个argv中的每个robj，那岂不是那些被存在dictEntry的robj都被释放了？其实不会，先看resetClient实现：

```cpp
void resetClient(redisClient *c) {
    freeClientArgv(c);
    c->reqtype = 0;
    c->multibulklen = 0;
    c->bulklen = -1;
    /* We clear the ASKING flag as well if we are not inside a MULTI. */
    if (!(c->flags & REDIS_MULTI)) c->flags &= (~REDIS_ASKING);
}

static void freeClientArgv(redisClient *c) {
    int j;
    for (j = 0; j < c->argc; j++)
        decrRefCount(c->argv[j]);
    c->argc = 0;
    c->cmd = NULL;
}
```

resetClient调用freeClientArgv来释放argv中的每个robj，真正的操作只是减引用计数，只有为0时才真正释放，所以为了保证dictEntry中的robj不被释放，肯定有地方把robj的引用计数加1了，具体在哪里呢，value的引用计数是在setKey函数里加的：

```cpp
void setKey(redisDb *db, robj *key, robj *val) {
    if (lookupKeyWrite(db,key) == NULL) {
        dbAdd(db,key,val);
    } else {
        dbOverwrite(db,key,val);
    }
    incrRefCount(val); //对val的引用计数加1
    removeExpire(db,key);
    signalModifiedKey(db,key);
}
```
那么key的呢？它在哪里加？事实上，它没有加，最终会在resetClient中释放，不过在dbAdd函数中，会对key生成一个新的sds并把它存入dictEntry中，所以释放argv里的robj没有影响，具体如下：

```cpp
void dbAdd(redisDb *db, robj *key, robj *val) {
    sds copy = sdsdup(key->ptr); //对key生成一个新的sds
    int retval = dictAdd(db->dict, copy, val);

    redisAssertWithInfo(NULL,key,retval == REDIS_OK);
    if (val->type == REDIS_LIST) signalListAsReady(db, key);
 }
```

好了，这些细节其实对本篇的主题理解无影响，不过细扣这些可以让我们更加深刻的理解redis，还是很有必要的，也许这是在给我的强迫症找借口，O(∩_∩)O哈哈~


## 总结
这篇讲的东西切入点很小，平时大伙有可能都忽略它的存在，不过学习这些细节也会很有收获的，至少我觉得是个好习惯^^
