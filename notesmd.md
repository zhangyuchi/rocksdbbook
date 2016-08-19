# 总结

## 新特性

* 支持read-change-write操作，使用merge operator
* 三种不同的memtable类型，skiplist，vector，prefix-hash

  * 顾名思义，prefix-hash是对key的前缀做hash，然后在对应的bucket里完全匹配key

* memtable的flush是可以pipeline的方式进行，排队进行

* StackableDB接口
* 可以执行非阻塞操作的读。只读取在blockcache里的数据，没有就报错？
* 支持事务，通过transaction log
* 多线程compaction
* 由prefix\_extractor计算每个key的key-prefix，针对每个key-prefix有对应的bloomfilter
* [feature not in leveldb](Features-Not-in-LevelDB.md)
- 

## 名词

* tablecache：打开的sstfile文件描述符的cache

