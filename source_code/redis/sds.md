# String内部数据结构

字符串在各大编程语言中可以说是遇到最多的数据结构。类似地，在redis中同样也有字符串的存储。不过，redis的字符串是一种称之为简单动态字符串(Simple Dynamic String, SDS)的实现。同传统的c语言字符串相比，有如下特点：

- 动态扩展。sds实现为多种长度的结构，支持动态扩展追加。
- 二进制安全(<a href="https://en.wikipedia.org/wiki/Binary-safe">Binary Safe</a>)。sds不仅能存储可打印字符，同时也支持存储二进制数据。
- 兼容传统的c语言字符串(<a href="https://en.wikipedia.org/wiki/Null-terminated_string">Null-terminated String</a>)。能使用部分c标准库函数。

## SDS结构定义
sds的结构定义在文件sds.h中,如下:

	typedef char *sds;

与传统的c字符串类型一致,都是字符指针(char *)。然而，与传统c字符串不同的，sds还有一个header，以此来包含字符长度信息。

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

除sdshdr5外，各类型header内均包含下面3个字段：

- len: 表示字符串的真正长度（不包含NULL结束符在内）。
- alloc: 表示字符串的最大容量（不包含最后多余的那个字节）。
- flags: 总是占用一个字节。其中的最低3个bit用来表示header的类型。

sds共有5种类型的header，创建时会根据长度选择适合的类型，以便尽可能地节省内存空间。在sds.h中有定义如下：

	#define SDS_TYPE_5  0
	#define SDS_TYPE_8  1
	#define SDS_TYPE_16 2
	#define SDS_TYPE_32 3
	#define SDS_TYPE_64 4
	
在各个header的类型定义中，有几个需要注意的地方：

- header的结构体使用了\_\_attribute__ ((packed))。这是为了让编译器以紧凑模式来分配内存。如果没有这个属性，编译器可能会为struct的字段做优化对齐，在其中填充空字节。那样的话，就不能保证header和sds的数据部分紧紧前后相邻，也不能按照固定向低地址方向偏移1个字节的方式来获取flags字段了。
- 无长度的字符数组char buf[]。这是C语言中定义字符数组的一种特殊写法，称为柔性数组（<a href="https://en.wikipedia.org/wiki/Flexible_array_member">flexible array member</a>），只能定义在一个结构体的最后一个字段上。它在这里只是起到一个标记的作用，表示在flags字段后面就是一个字符数组。它并不占用内存空间，如果计算sizeof(struct sdshdr16)的值，那么结果是5个字节，其中没有buf字段。
- sdshdr5与其它几个header结构不同，它不包含alloc字段，而长度使用flags的高5位来存储。因此，它不能为字符串分配空余空间。如果字符串需要动态增长，那么它就必然要重新分配内存才行。所以说，这种类型的sds字符串更适合存储静态的短字符串（长度小于32）。

## SDS创建和销毁
分析创建函数之前，先看sds.h中定义的几个宏和sds.c中几个辅助函数。

宏定义：

	#define SDS_TYPE_MASK 7
	#define SDS_TYPE_BITS 3
	#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));
	#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))
	#define SDS_TYPE_5_LEN(f) ((f)>>SDS_TYPE_BITS)

- sds的header里的flags标记，低三位记录的是类型，所以MASK是7，BITS是3
- SDS\_HDR_VAR会设置sh指针指向特定header类型
- SDS_HDR会返回特定的header类型
- SDS\_TYPE\_5_LEN则是sdshdr5的flags高5位是记录长度的


根据类型获取header大小的函数，源码如下：

	static inline int sdsHdrSize(char type) {
	    switch(type&SDS_TYPE_MASK) {
	        case SDS_TYPE_5:
	            return sizeof(struct sdshdr5);
	        case SDS_TYPE_8:
	            return sizeof(struct sdshdr8);
	        case SDS_TYPE_16:
	            return sizeof(struct sdshdr16);
	        case SDS_TYPE_32:
	            return sizeof(struct sdshdr32);
	        case SDS_TYPE_64:
	            return sizeof(struct sdshdr64);
	    }
	    return 0;
	}

根据字符串长度获取sds类型的函数，源码如下：

	static inline char sdsReqType(size_t string_size) {
		 //3.2.0版本后修改了边界写法
		 //详见https://github.com/antirez/redis/commit/603234076f4e59967f331bc97de3c0db9947c8ef
	    if (string_size < 1<<5)
	        return SDS_TYPE_5;
	    if (string_size < 1<<8)
	        return SDS_TYPE_8;
	    if (string_size < 1<<16)
	        return SDS_TYPE_16;
	    if (string_size < 1ll<<32)
	        return SDS_TYPE_32;
	    return SDS_TYPE_64;
	}

好了，到现在为止，已经准备充分了，开始分析创建函数。

sds的创建函数在文件sds.c中,如下:

	sds sdsnewlen(const void *init, size_t initlen) {
	    void *sh;
	    sds s;
	    char type = sdsReqType(initlen);
	    /* Empty strings are usually created in order to append. Use type 8
	     * since type 5 is not good at this. */
	    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8; //5
	    int hdrlen = sdsHdrSize(type);
	    unsigned char *fp; /* flags pointer. */
	    sh = s_malloc(hdrlen+initlen+1); //1
	    if (!init) //4
	        memset(sh, 0, hdrlen+initlen+1);
	    if (sh == NULL) return NULL;
	    s = (char*)sh+hdrlen; //2
	    fp = ((unsigned char*)s)-1; //3
	    switch(type) {
	        case SDS_TYPE_5: {
	            *fp = type | (initlen << SDS_TYPE_BITS);
	            break;
	        }
	        case SDS_TYPE_8: {
	            SDS_HDR_VAR(8,s);
	            sh->len = initlen;
	            sh->alloc = initlen;
	            *fp = type;
	            break;
	        }
	        case SDS_TYPE_16: {
	            SDS_HDR_VAR(16,s);
	            sh->len = initlen;
	            sh->alloc = initlen;
	            *fp = type;
	            break;
	        }
	        case SDS_TYPE_32: {
	            SDS_HDR_VAR(32,s);
	            sh->len = initlen;
	            sh->alloc = initlen;
	            *fp = type;
	            break;
	        }
	        case SDS_TYPE_64: {
	            SDS_HDR_VAR(64,s);
	            sh->len = initlen;
	            sh->alloc = initlen;
	            *fp = type;
	            break;
	        }
	    }
	    if (initlen && init)
	        memcpy(s, init, initlen);
	    s[initlen] = '\0'; //6
	    return s;
	}
	
	/* Create a new sds string starting from a null terminated C string. */
	sds sdsnew(const char *init) {
	    size_t initlen = (init == NULL) ? 0 : strlen(init);
	    return sdsnewlen(init, initlen);
	}

	/* Free an sds string. No operation is performed if 's' is NULL. */
	void sdsfree(sds s) {
	    if (s == NULL) return;
	    s_free((char*)s-sdsHdrSize(s[-1])); //7
	}
分析源码需注意的点：

- 语句1分配内存空间，返回的是内存空间的起始地址。
- 语句2跳过header将指针指到字符数据起始处。
- 语句3在字符数据起始处往header方向退一个char长度，即flags。
- 语句4init指向的字符数组是用来初始化数据的，如果init为NULL，就全初始化为0。
- 语句5如果创建的是长度为0的空串，那么header的类型不是SDS\_TYPE\_5，而是SDS\_TYPE\_8。根据注释可知，这是因为大多数时候创建空串是为了接下来的追加数据的操作，但SDS\_TYPE_5类型的sds字符串是不适合追加数据是(会引发内存重新分配)。
- 语句6初始化的sds字符串数据最后会追加一个NULL结束符(s[initlen] = ‘\0’)。
- 语句7sds内存释放时是从header起始开始。


## SDS数据拼接
追加字符数据时设计的辅助函数主要是获取长度len和设置len，此处略过不讲。直接看追加源码：

	sds sdscatlen(sds s, const void *t, size_t len) {
	    size_t curlen = sdslen(s);
	    s = sdsMakeRoomFor(s,len);
	    if (s == NULL) return NULL;
	    memcpy(s+curlen, t, len);
	    sdssetlen(s, curlen+len);
	    s[curlen+len] = '\0';
	    return s;
	}
	
	//根据指定的长度给已有的sds分配足够的空间
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
	
	    type = sdsReqType(newlen);
	
	    /* Don't use type 5: the user is appending to the string and type 5 is
	     * not able to remember empty space, so sdsMakeRoomFor() must be called
	     * at every appending operation. */
	    if (type == SDS_TYPE_5) type = SDS_TYPE_8;
	
	    hdrlen = sdsHdrSize(type);
	    if (oldtype==type) {
	        newsh = s_realloc(sh, hdrlen+newlen+1);
	        if (newsh == NULL) return NULL;
	        s = (char*)newsh+hdrlen;
	    } else {
	        /* Since the header size changes, need to move the string forward,
	         * and can't use realloc */
	        newsh = s_malloc(hdrlen+newlen+1);
	        if (newsh == NULL) return NULL;
	        memcpy((char*)newsh+hdrlen, s, len+1);
	        s_free(sh);
	        s = (char*)newsh+hdrlen;
	        s[-1] = type;
	        sdssetlen(s, len);
	    }
	    sdssetalloc(s, newlen);
	    return s;
	}
	
在sds追加实现中，首先是要通过sdsMakeRoomFor函数确保sds存储空间够用的，之后才是拷贝拼接字符串。关于它的实现代码，我们需要注意的是：

- 原字符串剩余(avail)空间大于要追加的长度(addlen)，直接返回。
- 宏SDS\_MAX_PREALLOC定义在sds.h中，值为1024*1024，即1MB。分配空间时，redis采用预分配原则，会分配额外的未使用空间。
- 如需更换sds类型，那么整个字符串空间都需要重新分配，并拷贝原来的数据到新的位置,释放原来的内存空间。
- 如不需要更换类型，那么调用s_realloc，在原地址上重新分配空间。realloc的含义：如果原地址有足够的空间完成重新分配，返回旧地址；否则，分配新的地址，并进行数据搬迁。参见http://man.cx/realloc。

## 总结
一个完整的sds空间结构，由前后相邻的两部分组成：

- header。一般包括字符串长度(len)、最大容量(alloc)和flags。sdshdr5的长度为flags的高5位，没有最大容量。
- 字符数组。数组长度等于最大容量+1。字符串数据填充在数组开头，其后为未用的剩余空间，允许字符串在不重新分配内存的情况下继续扩展，即有效字符串数据为最大容量长度。在真正的字符串数据之后，跟着一个ASCII码为0的'\0'字符，即上述兼容传统c字符串。

与传统的c字符串相比，sds有如下优势特点如下：

1. 获取字符串长度时间为O(1)，而传统c字符串需要遍历数组，为O(n)。
2. 拼接字符串首先是保证空间足够，因此不会造成缓冲区(buf数组)溢出，而传统c字符串在拼接时是有溢出的可能性的。
3. 减少字符串修改时带来的频繁的内存重分配问题。通过预分配和惰性释放，在字符串频繁操作时也能保证性能。
4. 二进制安全。sds不仅能保存文本字符，也能存储音频、图片等二进制数据。
