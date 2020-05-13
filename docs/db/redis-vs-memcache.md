> 由于历史遗留问题，发现项目中既有Redis，也有Memcached，而且都是做缓存，所以就想着统一为一个

#### 主要特征

- 支持的数据类型
  - Memcached 只支持基础的 key-value 键值对类型数据存储
  - Redis 支持更丰富的数据类型，String，List，Set，Hash，Sorted Set，Transactions 

- 持久化
  - Memcached 不支持数据落盘，只是基于内存存储数据
  - Redis 支持数据落盘，支持RDB与AOF两种及混合持久化方式（详细请看 [Redis 持久化机制](db/redis-persistence.md)

- 分布式
  - Memcached 通过在客户端使用 一致性hash算法 来实现分布式
    - 这种分布式的特点是 Memcached 节点之间不互相通信（由客户端程序负责分布式）
  - Redis 支持分布式集群部署，主从复制等工作模式（详细请看[Redis 集群](db/redis-cluster.md)

- 事件模型
  - Memcached 使用 libevent(epoll模型) 作为其底层的网络库，在高并发场景下有不错的表现
  - Redis 自己实现了处理事件的模块aeProcessEvents，支持多种I/O多路复用模型，程序会在编译时自动选择性能最高的I/O多路复用模型（select, epoll, kqueue, evport)

- 线程模型
  - Memcached 是多线程模型，使用 CAS 保证数据一致性（Check and Set）；CAS 是一个确保并发一致性的机制，属于“乐观锁”范畴，原理很简单：拿版本号，操作，对比版本号，如果一致就操作，不一致就放弃操作
  - Redis 的工作线程是单线程模型（目前已支持IO多线程模型），保证了数据按顺序提交

- 内存管理
  - Memcached 使用 Slab allocation 来进行内存管理。预先分配一系列大小固定的组，然后根据数据大小选择最合适的块存储。避免了内存碎片，缺点是扩展性差，浪费了一定的内存空间
    - Memcached 默认情况下一个 slab 的最大值为前一个的 1.25 倍
  - Redis 通过定义一个数组来记录所有的内存分配情况，采用的是包装的 malloc/free, 相较于 Memcached 的内存管理方法来说，要简单很多。malloc 首先以链表的方式搜索已管理的内存中可用的空间分配，导致内存碎片比较多

注意：这里说的谁性能高，谁性能低，是相较而言的，并不是真的性能差，恰恰相反，二者都是高性能缓存解决方案的佼佼者。

#### 总结一下

Memcached 是一个高性能的分布式缓存服务，搭建和操作都比较简单，基于内存，不需要持久化，不需要主从通信，多线程模型等特征让其在做缓存数据服务方面性能优越，基于 epoll 的事件IO模型让其在高并发场景下也有不错的表现
Redis 同样也是一个高性能的分布式缓存服务，同时Redis也可以作为持久化数据库使用。Redis基于内存同时支持丰富的动态数据结构让其在面对多种场景下都能保持性能稳定高效，其支持的分布式集群，能快速实现服务的横向扩展

很显然，Redis丰富的功能特性，让其基本已成为新一代缓存服务的最佳解决方案，虽然，Redis的单线程、需要持久化等特性仍然是一个导致服务不可用的很大隐患，但它在不断成长，I/O多线程，fork子进程混合RDB与AOF进行持久化，各种方案在逐渐完善，相信Redis未来可期

最后，不管你选择哪个，缓存服务不是数据库，你的系统同时需要缓存和数据库

#### 参考资料

- https://aws.amazon.com/cn/elasticache/redis-vs-memcached/
- https://www.infoworld.com/article/3063161/why-redis-beats-memcached-for-caching.html
- https://redis.io/documentation
- https://github.com/memcached/memcached/wiki

