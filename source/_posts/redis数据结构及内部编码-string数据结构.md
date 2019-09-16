---
title: redis数据结构及内部编码-string数据结构
tags:
  - redis
toc: true
originContent: >-
  在redis中，当我们想要知道一个key的类型的时候，我们可以使用type命令

  eg

  ```sh

  127.0.0.1:6379> set a "123"

  OK

  127.0.0.1:6379> type a

  string

  ```

  如果这个key不存在的话，会返回none

  eg:

  ```sh

  127.0.0.1:6379> type abcd

  none

  ```


  type命令实际返回的就是当前键的数据结构类型，它们分别是：

  - string（字符串）

  - hash（哈希）

  - list（列表）

  - set（集合）

  - zset（有序集合）

  但这些只是Redis对外的数据结构。每种数据结构都有自己底层的内部实现，并且每个都有多种实现，这样方便redis在合适的场景选择适合当前的编码方式。

  下图是redis每种数据结构对应的内部编码

  ![image.png](/images/2019/09/16/c192b310-d879-11e9-8076-0fae4f4f9b65.png)


  redis数据结构内部编码

  我们 可以通过 `object encoding`命令查询

  eg:

  ```sh

  127.0.0.1:6379> set hello "sss"

  OK

  127.0.0.1:6379> object encoding hello

  "embstr"

  127.0.0.1:6379> set hel "123"

  OK

  127.0.0.1:6379> object encoding hel

  "int"

  127.0.0.1:6379> set bigstr
  "dddddddddddfffffffffffdddddddddddddddddddddddddddddddddddddddddddsssssss"

  OK

  127.0.0.1:6379> object encoding bigstr

  "raw"

  ```

  从上面查询的结果我们可以看到，redis的string数据结构会根据输入的value不同使用不同的数据结构。

  下面我们从源码（基于redis 5.0.5）来分析下

  在redis中，的每个键值内部都是使用一个名字叫做 redisObject 这个 C语言结构体保存的，其代码如下：

  ```c

  typedef struct redisObject {
      unsigned type:4;
      unsigned encoding:4;
      unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                              * LFU data (least significant 8 bits frequency
                              * and most significant 16 bits access time). */
      int refcount;
      void *ptr;
  } robj;

  ```


  - type：表示键值的数据类型，包括 String、List、Set、ZSet、Hash

  - encoding：表示键值的内部编码方式，从 Redis源码看目前取值有如下几种：

  ```c

  /* Objects encoding. Some kind of objects like Strings and Hashes can be
   * internally represented in multiple ways. The 'encoding' field of the object
   * is set to one of this fields for this object. */
  #define OBJ_ENCODING_RAW 0     /* Raw representation */

  #define OBJ_ENCODING_INT 1     /* Encoded as integer */

  #define OBJ_ENCODING_HT 2      /* Encoded as hash table */

  #define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */

  #define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */

  #define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */

  #define OBJ_ENCODING_INTSET 6  /* Encoded as intset */

  #define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */

  #define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */

  #define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */

  #define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */

  ```

  - refcount：表示该键值被引用的数量，即一个键值可被多个键引用。


  String类型的内部编码

  在了解string类型的内部编码之前，我们先看下SDS:


  SDS（简单动态字符串）: 当你在阅读源码的时候，你可以很容易见到这个这个词。在代码里定义了5种SDS(源码在sds.h)

  ```c


  /* Note: sdshdr5 is never used, we just access the flags byte directly.
   * However is here to document the layout of type 5 SDS strings. */
  struct __attribute__ ((__packed__)) sdshdr5 {
      unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
      char buf[];
  };

  struct __attribute__ ((__packed__)) sdshdr8 {
      uint8_t len; /* used */
      uint8_t alloc; /* excluding the header and null terminator */
      unsigned char flags; /* 3 lsb of type, 5 unused bits */
      char buf[];
  };

  struct __attribute__ ((__packed__)) sdshdr16 {
      uint16_t len; /* used */
      uint16_t alloc; /* excluding the header and null terminator */
      unsigned char flags; /* 3 lsb of type, 5 unused bits */
      char buf[];
  };

  struct __attribute__ ((__packed__)) sdshdr32 {
      uint32_t len; /* used */
      uint32_t alloc; /* excluding the header and null terminator */
      unsigned char flags; /* 3 lsb of type, 5 unused bits */
      char buf[];
  };

  struct __attribute__ ((__packed__)) sdshdr64 {
      uint64_t len; /* used */
      uint64_t alloc; /* excluding the header and null terminator */
      unsigned char flags; /* 3 lsb of type, 5 unused bits */
      char buf[];
  };

  ```

  从上面的代码片段中，我们可以看出每个struct内的变量都差不多

  - len：字符串的长度（实际使用的长度）

  - alloc：分配内存的大小

  - flags：标志位，低三位表示类型，其余五位未使用

  - buf：字符数组



  通过上面的一系列枯燥的铺垫，我们开始切入正题


  ## 1. INT 编码方式

  当字符串键值的内容可以用一个64位有符号整型表示的时候，redis会将键值转换为long类型来存储，其对应的编码类型为:OBJ_ENCODING_INT


  对于`set hel "123"`命令，内存结构如下


  ![image.png](/images/2019/09/16/db30b020-d882-11e9-8076-0fae4f4f9b65.png)



  Redis 启动时会预先建立 10000 个分别存储 0~9999 的 redisObject 变量作为共享对象，这就意味着如果 set字符串的键值在
  0~10000 之间的话，则可以 直接指向共享对象 而不需要再建立新对象。


  ```c
    /* Check if we can represent this string as a long integer.
       * Note that we are sure that a string larger than 20 chars is not
       * representable as a 32 nor 64 bit integer. */
      len = sdslen(s);
      // 长度小于20 (64位有符号整型)
      if (len <= 20 && string2l(s,len,&value)) {
          /* This object is encodable as a long. Try to use a shared object.
           * Note that we avoid using shared integers when maxmemory is used
           * because every object needs to have a private LRU field for the LRU
           * algorithm to work well. */
          // 当value在[0,1000)的时候，使用字符串的共享策略
          if ((server.maxmemory == 0 ||
              !(server.maxmemory_policy & MAXMEMORY_FLAG_NO_SHARED_INTEGERS)) &&
              value >= 0 &&
              value < OBJ_SHARED_INTEGERS)
          {
              decrRefCount(o);
              incrRefCount(shared.integers[value]);
              return shared.integers[value];
          } else {
              if (o->encoding == OBJ_ENCODING_RAW) sdsfree(o->ptr);
              o->encoding = OBJ_ENCODING_INT;
              o->ptr = (void*) value;
              return o;
          }
      }
  ```


  ## 2. EMBSTR编码格式

  Redis 在保存长度小于 44 字节的字符串时会采用 OBJ_ENCODING_EMBSTR 编码方式,源码如下(`object.c`)：

  ```c

  /* Create a string object with EMBSTR encoding if it is smaller than
   * OBJ_ENCODING_EMBSTR_SIZE_LIMIT, otherwise the RAW encoding is
   * used.
   *
   * The current limit of 44 is chosen so that the biggest string object
   * we allocate as EMBSTR will still fit into the 64 byte arena of jemalloc. */
  #define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44

  robj *createStringObject(const char *ptr, size_t len) {
      //字符串长度小于等于44的时候使用embstr编码格式,大于44的时候使用raw编码格式
      if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
          return createEmbeddedStringObject(ptr,len);
      else
          return createRawStringObject(ptr,len);
  }


  * Create a string object with encoding OBJ_ENCODING_EMBSTR, that is
   * an object where the sds string is actually an unmodifiable string
   * allocated in the same chunk as the object itself. */
  robj *createEmbeddedStringObject(const char *ptr, size_t len) {
      robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);
      struct sdshdr8 *sh = (void*)(o+1);

      o->type = OBJ_STRING;
      o->encoding = OBJ_ENCODING_EMBSTR;
      o->ptr = sh+1;
      o->refcount = 1;
      if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
          o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
      } else {
          o->lru = LRU_CLOCK();
      }

      sh->len = len;
      sh->alloc = len;
      sh->flags = SDS_TYPE_8;
      if (ptr == SDS_NOINIT)
          sh->buf[len] = '\0';
      else if (ptr) {
          memcpy(sh->buf,ptr,len);
          sh->buf[len] = '\0';
      } else {
          memset(sh->buf,0,len+1);
      }
      return o;
  }


  ```


  指令 set hello "sss" 所设置的键值，其内存结构示意图如下:


  ![image.png](/images/2019/09/16/7c771620-d885-11e9-8076-0fae4f4f9b65.png)


  ## 3. RAW 编码格式

  通过上面的源码分析，当字符串键值的长度大于44的时候，redis会将键值的内部编码方式改为OBJ_ENCODING_RAW格式


  ```c

  /* Create a string object with encoding OBJ_ENCODING_RAW, that is a plain
   * string object where o->ptr points to a proper sds string. */
  robj *createRawStringObject(const char *ptr, size_t len) {
      return createObject(OBJ_STRING, sdsnewlen(ptr,len));
  }



  /* ===================== Creation and parsing of objects ====================
  */


  robj *createObject(int type, void *ptr) {
      robj *o = zmalloc(sizeof(*o));
      o->type = type;
      o->encoding = OBJ_ENCODING_RAW;
      o->ptr = ptr;
      o->refcount = 1;

      /* Set the LRU to the current lruclock (minutes resolution), or
       * alternatively the LFU counter. */
      if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
          o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
      } else {
          o->lru = LRU_CLOCK();
      }
      return o;
  }


  ```

  与上面的 OBJ_ENCODING_EMBSTR 编码方式的不同之处在于 此时动态字符串 sds 的内存与其依赖的 redisObject 的
  内存不再连续了


  ![image.png](/images/2019/09/16/72aa2000-d886-11e9-8076-0fae4f4f9b65.png)
categories:
  - 微服务
date: 2019-09-16 19:49:43
---

在redis中，当我们想要知道一个key的类型的时候，我们可以使用type命令
eg
```sh
127.0.0.1:6379> set a "123"
OK
127.0.0.1:6379> type a
string
```
如果这个key不存在的话，会返回none
eg:
```sh
127.0.0.1:6379> type abcd
none
```

type命令实际返回的就是当前键的数据结构类型，它们分别是：
- string（字符串）
- hash（哈希）
- list（列表）
- set（集合）
- zset（有序集合）
<!--more-->
但这些只是Redis对外的数据结构。每种数据结构都有自己底层的内部实现，并且每个都有多种实现，这样方便redis在合适的场景选择适合当前的编码方式。
下图是redis每种数据结构对应的内部编码
![image.png](/images/2019/09/16/c192b310-d879-11e9-8076-0fae4f4f9b65.png)

redis数据结构内部编码
我们 可以通过 `object encoding`命令查询
eg:
```sh
127.0.0.1:6379> set hello "sss"
OK
127.0.0.1:6379> object encoding hello
"embstr"
127.0.0.1:6379> set hel "123"
OK
127.0.0.1:6379> object encoding hel
"int"
127.0.0.1:6379> set bigstr "dddddddddddfffffffffffdddddddddddddddddddddddddddddddddddddddddddsssssss"
OK
127.0.0.1:6379> object encoding bigstr
"raw"
```
从上面查询的结果我们可以看到，redis的string数据结构会根据输入的value不同使用不同的数据结构。
下面我们从源码（基于redis 5.0.5）来分析下
在redis中，的每个键值内部都是使用一个名字叫做 redisObject 这个 C语言结构体保存的，其代码如下：
```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

- type：表示键值的数据类型，包括 String、List、Set、ZSet、Hash
- encoding：表示键值的内部编码方式，从 Redis源码看目前取值有如下几种：
```c
/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
```
- refcount：表示该键值被引用的数量，即一个键值可被多个键引用。

String类型的内部编码
在了解string类型的内部编码之前，我们先看下SDS:

SDS（简单动态字符串）: 当你在阅读源码的时候，你可以很容易见到这个这个词。在代码里定义了5种SDS(源码在sds.h)
```c

/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```
从上面的代码片段中，我们可以看出每个struct内的变量都差不多
- len：字符串的长度（实际使用的长度）
- alloc：分配内存的大小
- flags：标志位，低三位表示类型，其余五位未使用
- buf：字符数组


通过上面的一系列枯燥的铺垫，我们开始切入正题

## 1. INT 编码方式
当字符串键值的内容可以用一个64位有符号整型表示的时候，redis会将键值转换为long类型来存储，其对应的编码类型为:OBJ_ENCODING_INT

对于`set hel "123"`命令，内存结构如下

![image.png](/images/2019/09/16/db30b020-d882-11e9-8076-0fae4f4f9b65.png)


Redis 启动时会预先建立 10000 个分别存储 0~9999 的 redisObject 变量作为共享对象，这就意味着如果 set字符串的键值在 0~10000 之间的话，则可以 直接指向共享对象 而不需要再建立新对象。

```c
  /* Check if we can represent this string as a long integer.
     * Note that we are sure that a string larger than 20 chars is not
     * representable as a 32 nor 64 bit integer. */
    len = sdslen(s);
    // 长度小于20 (64位有符号整型)
    if (len <= 20 && string2l(s,len,&value)) {
        /* This object is encodable as a long. Try to use a shared object.
         * Note that we avoid using shared integers when maxmemory is used
         * because every object needs to have a private LRU field for the LRU
         * algorithm to work well. */
        // 当value在[0,1000)的时候，使用字符串的共享策略
        if ((server.maxmemory == 0 ||
            !(server.maxmemory_policy & MAXMEMORY_FLAG_NO_SHARED_INTEGERS)) &&
            value >= 0 &&
            value < OBJ_SHARED_INTEGERS)
        {
            decrRefCount(o);
            incrRefCount(shared.integers[value]);
            return shared.integers[value];
        } else {
            if (o->encoding == OBJ_ENCODING_RAW) sdsfree(o->ptr);
            o->encoding = OBJ_ENCODING_INT;
            o->ptr = (void*) value;
            return o;
        }
    }
```

## 2. EMBSTR编码格式
Redis 在保存长度小于 44 字节的字符串时会采用 OBJ_ENCODING_EMBSTR 编码方式,源码如下(`object.c`)：
```c
/* Create a string object with EMBSTR encoding if it is smaller than
 * OBJ_ENCODING_EMBSTR_SIZE_LIMIT, otherwise the RAW encoding is
 * used.
 *
 * The current limit of 44 is chosen so that the biggest string object
 * we allocate as EMBSTR will still fit into the 64 byte arena of jemalloc. */
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
robj *createStringObject(const char *ptr, size_t len) {
    //字符串长度小于等于44的时候使用embstr编码格式,大于44的时候使用raw编码格式
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}

* Create a string object with encoding OBJ_ENCODING_EMBSTR, that is
 * an object where the sds string is actually an unmodifiable string
 * allocated in the same chunk as the object itself. */
robj *createEmbeddedStringObject(const char *ptr, size_t len) {
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);
    struct sdshdr8 *sh = (void*)(o+1);

    o->type = OBJ_STRING;
    o->encoding = OBJ_ENCODING_EMBSTR;
    o->ptr = sh+1;
    o->refcount = 1;
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }

    sh->len = len;
    sh->alloc = len;
    sh->flags = SDS_TYPE_8;
    if (ptr == SDS_NOINIT)
        sh->buf[len] = '\0';
    else if (ptr) {
        memcpy(sh->buf,ptr,len);
        sh->buf[len] = '\0';
    } else {
        memset(sh->buf,0,len+1);
    }
    return o;
}

```

指令 set hello "sss" 所设置的键值，其内存结构示意图如下:

![image.png](/images/2019/09/16/7c771620-d885-11e9-8076-0fae4f4f9b65.png)

## 3. RAW 编码格式
通过上面的源码分析，当字符串键值的长度大于44的时候，redis会将键值的内部编码方式改为OBJ_ENCODING_RAW格式

```c
/* Create a string object with encoding OBJ_ENCODING_RAW, that is a plain
 * string object where o->ptr points to a proper sds string. */
robj *createRawStringObject(const char *ptr, size_t len) {
    return createObject(OBJ_STRING, sdsnewlen(ptr,len));
}


/* ===================== Creation and parsing of objects ==================== */

robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));
    o->type = type;
    o->encoding = OBJ_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;

    /* Set the LRU to the current lruclock (minutes resolution), or
     * alternatively the LFU counter. */
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }
    return o;
}

```
与上面的 OBJ_ENCODING_EMBSTR 编码方式的不同之处在于 此时动态字符串 sds 的内存与其依赖的 redisObject 的 内存不再连续了

![image.png](/images/2019/09/16/72aa2000-d886-11e9-8076-0fae4f4f9b65.png)