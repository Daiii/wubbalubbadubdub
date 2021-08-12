## SDS

[toc]

SDS：简单动态字符串(simple dynamic string)

1. SDS是Redis默认的字符标识，比如包含字符串值的键值对都是底层由SDS实现的
2. SDS用来保存数据库中的字符串值
3. SDS被作用缓冲区：比如AOF模块的AOF缓冲区，以及客户端状态中输入缓冲区

### SDS的结构

```c++
struct sdshdr {
  // buf中已经占用的长度
  int len;
  
  // buf中剩余可用空间的长度
  int free;
  
  // 字符数组
  char buf[];
}
```



### SDS与C字符串的区别

#### 常熟复杂度获取字符串长度

因为C语言字符串并不记录自身的长度信息，所以为了获取一个C字符串的长度，程序必须遍历整个字符串，对遇到的每个字符进行计数，知道遇到代表字符串结尾的空字符串'\0'位置，这个操作的复杂度为O(N)。

和C字符串不同，因为SDS在len属性中记录了SDS本身的长度，所以获取一个SDS长度的复杂度为O(1)。

#### 杜绝缓冲区溢出

除了获取字符串长度的复杂度高之外，C字符串不记录自身长度带来的另一个问题是容易造成缓冲区溢出(buffer overflow)。

与C字符串不同，SDS的空间分配策略完全杜绝了发生缓冲区溢出的可能性：当SDS API需要怼SDS进行修改时，API会先检查SDS的空间是否满足修改所需要的要就，如果不满足的话，API会自动将SDS的空间扩展至执行修改所需的大小，然后才执行实际的修改操作。所以使用SDS既不需要手动修改SDS空间大小，也不会出现缓冲区溢出的问题。

#### 减少修改字符串时带来的内存重分配次数

##### 空间预分配

空间预分配用于优化SDS的字符串增长操作：当SDS的API对一个SDS进行修改，并且需要怼SDS进行空间扩展的时候，程序不仅会为SDS分配修改所必须要的空间，还会为SDS分配额外的未使用空间。

**加倍扩容**：(len + addlen) * 2

```c++
/* Enlarge the free space at the end of the sds string so that the caller
 * is sure that after calling this function can overwrite up to addlen
 * bytes after the end of the string, plus one more byte for nul term.
 *
 * Note: this does not change the *length* of the sds string as returned
 * by sdslen(), but only the free buffer space we have. */
sds sdsMakeRoomFor(sds s, size_t addlen) {
    void *sh, *newsh;
    size_t avail = sdsavail(s);
    size_t len, newlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;

    /* Return ASAP if there is enough space left. */
    if (avail >= addlen) return s;

    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);
    newlen = (len+addlen);
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;
}
```

**注意：当大小超过1M之后扩容不会成倍增加，每次扩容增加1M**

##### 惰性空间释放

惰性空间释放用于优化SDS的字符串缩短操作：当SDS的API需要缩短SDS保存的字符串时，程序并不立即使用内存重分配来回收速断后多出来的字节，而是使用free属性将这些字节的数量记录起来，并等待将来使用。

**惰性缩容**：

```c++
/* Grow the sds to have the specified length. Bytes that were not part of
 * the original length of the sds will be set to zero.
 *
 * if the specified length is smaller than the current length, no operation
 * is performed. */
sds sdsgrowzero(sds s, size_t len) {
    size_t curlen = sdslen(s);

    if (len <= curlen) return s;
    s = sdsMakeRoomFor(s,len-curlen);
    if (s == NULL) return NULL;

    /* Make sure added region doesn't contain garbage */
    memset(s+curlen,0,(len-curlen+1)); /* also set trailing \0 byte */
    sdssetlen(s, len);
    return s;
}
```

##### 二进制安全

C字符串中的字符必须符合某种编码(比如ASCII)，并且除了字符串的末尾之外，字符串里面不能包含空字符，否则最先被程序读入的空字符将被误认为是字符串结尾，这些限制使得C字符串只能保存文本数据，而不能保存像图片、音频、视频、压缩文件这样的二进制数据。

##### 兼容部分C字符串函数

虽然SDS的API都是二进制安全的，但它们一样遵循C字符串以空字符结尾的惯例：这些API总会将SDS保存的数据的末尾设置为空字符串，并且总会在为buf数组分配空间时多分配一个字节来容纳这个空字符，这事为了让那些保存文本数据的SDS可以重用一部分<string.h>库定义的函数。



### 总结

| C字符串                                  | SDS                                        |
| ---------------------------------------- | ------------------------------------------ |
| 获取字符串长度的复杂度O(N)               | 获取字符串长度的复杂度O(1)                 |
| API是不安全的，可能会造成缓冲区溢出      | API是安全的，不会造成缓冲区溢出            |
| 修改字符串长度N次必然会执行N次内存重分配 | 修改字符串长度N次最多需要执行N次内存重分配 |
| 只能保存文本数据                         | 可以保存文本或者二进制数据                 |
| 可以使用所有<string.h>库中的函数         | 可以使用一部分<string.h>库中的函数         |

