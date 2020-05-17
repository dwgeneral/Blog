#### 什么是分布式锁

对于分布式服务来说，我们需要保证在同一时间内，只能有一个进程修改共享变量或执行代码块，即保证共享资源的原子性。比如并发领取优惠券的场景，我们为了不出现超发的情况，就需要进行上述逻辑的保证。而分布式锁就是实现上述逻辑的一种机制。

实现分布式锁的方式有很多，可以基于数据库实现，基于分布式协调服务实现，基于缓存服务实现等。前几天项目中恰好使用了Redis分布式锁，所以想跟大家说说 Redis 分布式锁的那些事。

#### Redis实现分布式锁

**基于SETNX**
Redis 可以通过 SETNX 方法配合 ExpireTime(过期时间参数) 实现锁操作

```ruby
SET lock1 my_random_value NX PX 2000  # 获取锁
if redis.call('get', KEYS[1] === ARGV[1]) then
  return redis.call('del', KEYS[1])
else
  return 0
end

```
- 上述命令的设置值和过期时间操作是原子性操作。
- 一定要在设置值的时候设置过期时间。如果锁没有过期时间的话，服务器可能会在删除锁之前宕机，会造成死锁的问题。
- 旧版的 Redis(<2.6.12) 不支持 set 过期时间参数，如果采用先设置了值，再设置过期时间，有可能在程序执行了第一条命令后就宕机了，会造成死锁的问题。导致其他进程将一直获取不到该锁。
- 在解锁的时候，需要验证value是和加锁的一致才删除key，这是为了防止客户端A误删了客户端B的锁。假设A获取了锁，过期时间2s，此时3s后，锁已经自动释放了，并且客户端B获取到了同一资源的新锁，此时A去释放锁，就可能会删除B的锁，所以需要判断value一致。
- 释放锁的操作必须使用Lua脚本来实现。因为使用Lua脚本可以保证GET和DEL操作的原子性。

**部署方式问题**
分布式锁的实现与Redis部署方式也有很大关系，Redis有3中部署方式：单机模式，主从哨兵模式，集群模式。

- 如果采用单机模式，会存在单点故障问题，此时加锁就无效了。
- 如果采用主从模式，一般来说主服务器负责写操作，从服务器负责读操作，加锁的时候，如果master节点故障了，通过哨兵发生主从切换，此时有可能出现锁丢失的问题。
- 如果采用集群模式，同样会有上述问题，加锁会先对集群中的某个主节点加锁，然后异步复制到其备节点。如果还没复制到备节点时，主节点宕机，还是可能会发生所丢失问题。

#### RedLock 算法
基于以上的问题，Redis官方提出了一个 RedLock 算法。假设Redis的部署模式是集群模式，共有5个master节点，加锁过程如下：
  - 获取当前时间戳，单位毫秒
  - 尝试按顺序在所有5个实例中获取锁，在所有实例中使用相同的键名和随机值，并设置一个锁的过期时间。和每个实例获取锁的时间一定要比锁的过期时间小，如果超时则跳过，获取下一个，这可以防止某个Redis节点通信故障造成延时的问题。
  - 当客户端获取到了超过半数的锁（至少3个），且锁的获取时间小雨锁的过期时间，才认为锁获取成功。（锁的获取时间 = 当前时间戳 - 步骤1中获得的时间戳）
  - 如果获得了锁，则其有效时间被认为是锁的过期时间 - 锁的获取时间
  - 如果未能获得锁，则将刚才获得的锁进行解锁
释放锁的过程比较简单：客户端向所有Redis节点发起释放锁的操作，不管这些节点当时在获取锁的时候是否成功