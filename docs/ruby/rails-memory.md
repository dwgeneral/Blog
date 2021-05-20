> Ruby 是个简洁优雅的语言，Rails 是个令人愉快的框架，但如果你不多关注它的性能，那它一定会让你不开心

内存问题是导致诸多 Ruby 应用变慢的首要原因。Rails 性能优化的二八法则：80%的提速是源自于对内存的优化，剩下的20%属于其他。

我们都知道一般一个功能比较齐全，优化良好，稳定运行了一段时间的全栈 Rails 应用，单个 Passenger 进程的内存占用基本在400MB左右。如果你不多注意内存的占用情况，你的程序内存超过1GB 是很有可能的。这么多需要回收的内存，GC肯定要忙死了，你的程序自然也就会响应越来越慢了。

**为什么会占这么多内存**
- Ruby 自身的小缺点
  - Ruby 的内存分配涉及三层级，分别为：Ruby解释器(管理Ruby对象); 操作系统的内存分配器库; 内核
  - 这三层级在进行内存分配与释放的时候，都会产生碎片。具体三层级的分配和释放过程，请参见参考资料[1]
  - Ruby 使用 glibc(C运行时) 的 malloc(size) API进行内存分配，这是一个比较古老的内存分配器，性能比较低，分配时会产生大量碎片。
  - Ruby 的GC机制的工作原理是将 Ruby堆页面槽标记为空闲，允许重用该槽。如果整个Ruby堆页面最终只包含空闲槽，则整个Ruby堆页面可以释放回内存分配器。但是，如果不是所有的插槽都是空闲的，假设你有很多Ruby堆页面，垃圾回收会释放不同位置的对象，这样即使你最终会有很多空闲插槽，但还有很多Ruby堆页面并不完全由空闲插槽组成。即使Ruby有空闲插槽来分配对象，就内存分配器和内核而言，它们仍然被分配了内存！
- 你的问题
  - 做了很多内存密集型的逻辑，比如序列化/反序列化，在内存中对数据进行排序，用Ruby进行大数据统计等
  - 不规范的使用方式，比如字符串回调等

**优化方案**
- 将内存分配器换为 jemalloc
  - jemalloc 是Facebook出品的， 最早用于FreeBSD中的内存分配器，后来像firefox从3.0也开始使用它，redis从2.4之后默认在linux上使用jemalloc。既然有这么多性能敏感型的软件都使用了jemalloc那它一定有过人之处。
  ```javascript
  > sudo apt-get install libjemalloc-dev
  > rvm reinstall 2.4.1 -C --with-jemalloc
  // or Gem https://github.com/kzk/jemalloc-rb
  ```
- 修改配置 MALLOC_ARENA_MAX = 2
  - 减少可用的最大内存区域(arenas)数量。简单说就是限制最大的OS堆产生数，以此减少碎片，同时也可能会带来一点性能上的下降。

**如何监控 Rails 内存使用情况**
- 在服务器端开发中，监控服务的执行性能是相对容易的，通过在K8S集群上搭建Grafana和ELK，可以清楚的看到CPU，内存，慢查询，接口响应等信息。
- 但要监控 Rails进程的内存使用情况，在出现内存泄漏之前，就能感知到哪些代码存在风险，而不是在服务器内存已经开始飙升了，我们再回头来查问题，就不是那么容易了
- 究其原因是应用日志中并没有记录进程的内存变化情况，Ruby也没有API可以直接查到进程使用的物理内存。
- 这里给大家分享一个从一篇文章中看到的解决方案，我在我司的项目中使用过，确实还是比较好用的。

- 在Linux环境下 /proc 文件系统是内核的映像， /proc/pid/status 文件记录了这个进程的状态信息。当然也存在常驻物理内存(Residence)信息(VmRSS)。
- 所以我们可以在Rails处理请求之前记录内存，等Rails处理完请求之后，再记录内存，计算内存的变化情况，写入到 production.log。
- 通过 ELK 的 Kibana 我们可以方便的看到这些日志信息，当然也可以写一个定时任务，定期将这些信息写入到 memory.log 文件，然后导出，根据内存占用情况进行具体的分析


**参考资料**
- https://winterwhisper.github.io/programming/2019-03-21-ruby-memory-allocation-and-fragmentation.html
- https://www.jianshu.com/p/6d8289382251
- https://www.speedshop.co/2017/12/04/malloc-doubles-ruby-memory.html
- https://blog.csdn.net/robbin/article/details/83328993