#redis内部数据结构——dict

用过redis命令的都知道，redis也支持键值对的缓存形式，那么redis是怎么实现的呢？

与Java中的HashMap一样，redis采用<a href="https://en.wikipedia.org/wiki/Hash_table">hash table</a>来做映射。通过哈希函数(hash function)计算key在数组中的位置，然后采用拉链法(separate chaining)解决冲突，当装载因子(load factor)达到临界值时进行扩容，同时引发重哈希(rehashing)，与Java的hashMap所不一样的是，redis扩容时采用的重哈希策略是一种称之为增量式的重哈希策略，通过将所需的重哈希动作分散到增删查的操作中，避免重哈希的耗时对单次的操作产生太大的影响，加快redis操作的响应速度

##dict结构定义

dict的定义在dict.h文件中，如下

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
	
	typedef struct dictType {
	    unsigned int (*hashFunction)(const void *key);
	    void *(*keyDup)(void *privdata, const void *key);
	    void *(*valDup)(void *privdata, const void *obj);
	    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
	    void (*keyDestructor)(void *privdata, void *key);
	    void (*valDestructor)(void *privdata, void *obj);
	} dictType;
	
	/* This is our hash table structure. Every dictionary has two of this as we
	 * implement incremental rehashing, for the old to the new table. */
	typedef struct dictht {
	    dictEntry **table;
	    unsigned long size;
	    unsigned long sizemask;
	    unsigned long used;
	} dictht;
	
	typedef struct dict {
	    dictType *type;
	    void *privdata;
	    dictht ht[2];
	    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
	    int iterators; /* number of iterators currently running */
	} dict;
	
由于采用的是增量式的重哈希策略，所以redis的dict包含两个哈希表，当处于重哈希阶段时，两个哈希表都有效，否则，只会存在ht[0]。rehashidx表示当前重哈希的进度，当值为-1表示不在重哈希过程。

dictType结构包含若干函数指针，用于dict的调用者对涉及key和value的各种操作进行自定义。这些操作包含：

- hashFunction，对key进行哈希值计算的哈希函数。
- keyDup和valDup，key和value的拷贝函数，用于在需要的时候对key和value进行深拷贝。
- keyCompare，定义两个key的比较操作，在根据key进行查找时会用到，类似Java的equals方法。
- keyDestructor和valDestructor，分别定义对key和value的析构函数。

哈希表的具体定义在dictht，结构如下：

- dictEntry指针数组 table。k-v对最终会存储在对应的数组位置(bucket)上。
- size：dictEntry指针数组的长度。它总是2的指数。
- sizemask，它的值等于(size-1)。用于将哈希函数得出的值映射到table上(哈希值 & sizemask，即取余操作)。
- used：table中现有的数据个数。used / size就是装载因子(load factor)。比值越大，哈希值冲突概率越高。
	
##dict重哈希

增量式重哈希进度推进的函数：

	static void _dictRehashStep(dict *d) {
	    if (d->iterators == 0) dictRehash(d,1);
	}
	
	int dictRehash(dict *d, int n) {
	    int empty_visits = n*10; /* Max number of empty buckets to visit. */
	    if (!dictIsRehashing(d)) return 0;
	
	    while(n-- && d->ht[0].used != 0) {
	        dictEntry *de, *nextde;
	
	        /* Note that rehashidx can't overflow as we are sure there are more
	         * elements because ht[0].used != 0 */
	        assert(d->ht[0].size > (unsigned long)d->rehashidx);
	        while(d->ht[0].table[d->rehashidx] == NULL) {
	            d->rehashidx++;
	            if (--empty_visits == 0) return 1;
	        }
	        de = d->ht[0].table[d->rehashidx];
	        /* Move all the keys in this bucket from the old to the new hash HT */
	        while(de) {
	            unsigned int h;
	
	            nextde = de->next;
	            /* Get the index in the new hash table */
	            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
	            de->next = d->ht[1].table[h];
	            d->ht[1].table[h] = de;
	            d->ht[0].used--;
	            d->ht[1].used++;
	            de = nextde;
	        }
	        d->ht[0].table[d->rehashidx] = NULL;
	        d->rehashidx++;
	    }
	
	    /* Check if we already rehashed the whole table... */
	    if (d->ht[0].used == 0) {
	        zfree(d->ht[0].table);
	        d->ht[0] = d->ht[1];
	        _dictReset(&d->ht[1]);
	        d->rehashidx = -1;
	        return 0;
	    }
	
	    /* More to rehash... */
	    return 1;
	}
查找、新增、删除调用的都是_dictRehashStep函数。dictRehash函数每次会从ht[0]上转移n条链到ht[1]上，如果对应的位置上不存在数据，则往后顺延查找不为空的位置，至多查找empty_visits个位置。数据迁移完毕，就将ht[1]设置为ht[0]，重哈希结束。
	
## 总结
哈希表设计的主要点在于：一个有好的散列的哈希函数、哈希冲突的解决方法、装载因子达到上限后的扩容重哈希策略。redis实现的dict基本和Java的差不多，主要在于重哈希策略采用的是增量式，以及其他一些数据结构上的差别。
	