# 总结

## 新特性
- 支持read-change-write操作，使用merge operator
- 三种不同的memtable类型，skiplist，vector，prefix-hash
    - 顾名思义，prefix-hash是对key的前缀做hash，然后在对应的bucket里完全匹配key
- memtable的flush是可以pipeline的方式进行，排队进行
- StackableDB接口
- 可以执行非阻塞操作的读。只读取在blockcache里的数据，没有就报错？
- 支持事务，通过transaction log
- 多线程compaction

## 名词
- tablecache：打开的sstfile文件描述符的cache




