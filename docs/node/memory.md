#### V8的内存限制

Node在Javascript的执行上直接受益于V8，可以随着V8的升级就能享受到更好的性能和新的语言特性（ES6，ES7等），同时也受到V8的限制，内存限制就是其中之一。

Node的内存构成主要由通过V8进行分配的部分和Node自行分配的部分。受V8的垃圾回收限制的主要是V8的堆内存。

在一般的后端开发语言中，在基本的内存使用上没有什么限制，然而在Node中通过Javascript使用内存时就会发现只能使用部分内存。64位操作系统下约为1.4GB，32位操作系统下约为0.7GB。 

这个限制可以从V8的源码中找到，下面的 Page::kPageSize 默认为 1MB。可以看到，老生代内存(后面会说到)在64位系统下为1400MB，在32为系统下为700MB，而新生代内存(后面会说到)是由2个reversed_semispace_size_构成，reversed_semispace_size_在64位系统和32位系统上分别为16MB和8MB，所以新生代内存在64位系统和32位系统上的最大值分别为32MB和16MB。
```C
// semispace_size_ should be a power of 2 and old_generation_size_ should be
// a multiple of Page::kPageSize
#if defined(V8_TARGET_ARCH_X64)
#define LUMP_OF_MEMORY (2 * MB)
      code_range_size_(512*MB),
#else
#define LUMP_OF_MEMORY MB
      code_range_size_(0),
#endif
#if defined(ANDROID)
      reserved_semispace_size_(4 * Max(LUMP_OF_MEMORY, Page::kPageSize)),
      max_semispace_size_(4 * Max(LUMP_OF_MEMORY, Page::kPageSize)),
      initial_semispace_size_(Page::kPageSize),
      max_old_generation_size_(192*MB),
      max_executable_size_(max_old_generation_size_),
#else
      reserved_semispace_size_(8 * Max(LUMP_OF_MEMORY, Page::kPageSize)),
      max_semispace_size_(8 * Max(LUMP_OF_MEMORY, Page::kPageSize)),
      initial_semispace_size_(Page::kPageSize),
      max_old_generation_size_(700ul * LUMP_OF_MEMORY),
      max_executable_size_(256l * LUMP_OF_MEMORY),
#endif

// Returns the maximum amount of memory reserved for the heap.  For
// the young generation, we reserve 4 times the amount needed for a
// semi space.  The young generation consists of two semi spaces and
// we reserve twice the amount needed for those in order to ensure
// that new space can be aligned to its size
intptr_t MaxReserved() {
  return 4 * reserved_semispace_size_ + max_old_generation_size_;
}
```
根据上面 MaxReserved 方法的计算公司，V8堆空间的最大值在64位操作系统上为1464MB，在32位操作系统上为732MB。这个值可以解释文章开头的1.4GB和0.7GB


那你一定很好奇了，V8为什么要有这个限制呢？要弄清除这个问题，我们需要了解JS对象从出生到被回收的全过程。

#### V8的对象分配与垃圾回收算法

在V8中所有的Javascript对象都是通过堆来进行分配的。我们可以通过 process.memoryUsage() 来查看目前V8的内存使用情况。
```javascript
{
  rss: 4935680,       // resident set size 进程的常驻内存部分
  heapTotal: 1826816, // 已申请到的堆内存
  heapUsed: 650472,   // 已使用的堆内存
  external: 49879     // 绑定到Javascript的C++对象的内存使用情况
}
```
V8的垃圾回收算法的主要策略是基于分代式垃圾回收策略。将内存空间分为新生代内存和老生代内存。

根据在内存中存活时间的不同，将存活时间短的对象分配在新生代内存空间，将存活时间较长或常驻内存的对象分配在老生代内存空间。

新生代内存空间的垃圾回收机制
  - 基于 Scavenge 算法进行垃圾回收，内部主要采用 Cheney 算法
  - 将内存空间分为From空间和To空间，From空间为正在使用状态的空间，To空间为处于闲置状态的空间
  - 分配对象内存空间时，先在From空间进行分配
  - 垃圾回收时，检查From空间的存活对象，把存活对象复制到To空间，非存活空间占用的空间会被释放
  - 完成复制后，将From空间与To空间角色互换
  - 当一个对象经过多次复制依然存在，会被移动到老生代空间，称为晋升
  - 重点：只复制存活的对象，对于存活周期短的对象只占少部分，所以时间上高效
  - 思想：典型的空间换时间

老生代内存空间的垃圾回收机制
  - 采用 Mark-Sweep 和 Mark-Compact 结合的方式进行垃圾回收
  - Mark-Sweep 标记清除
    - 标记阶段遍历内存空间中所有对象，并标记活着的对象
    - 清除阶段，只清除没有标记的对象
    - 在进行一次标记清除回收后，内存空间会出现不连续的状态，也就是内存碎片
  - Mark-Compact 标记整理
    - 为了解决内存碎片问题被提出来
    - 在标记死亡后，将活着的对象往一端移动，移动完成后，直接清理掉边界外的内存
    - 它的问题就是需要整理，所以会慢
  - V8 主要采用 Mark-Sweep，当空间不足时，使用 Mark-Compact

为了避免Javascript应用与垃圾回收看到的对象状态不一致。V8在垃圾回收时，会将整个应用逻辑暂停，待执行完垃圾回收后再恢复应用逻辑。称为全停顿

在V8的分代式垃圾回收中，一次小垃圾回收只回收新生代内存，因为新生代内存本身配置的就很小，全停顿影响不大。但老生代内存空间大，对象多，全停顿就会时间很长。

为了降低在对老生代内存空间进行回收时的全停顿影响，V8从标记阶段入手，采用增量标记的方式，每标记一小段内存后，就让应用逻辑执行一会。垃圾回收与应用逻辑执行交替执行，直到标记阶段完成。

#### 回答开篇

回到开篇的问题，V8为什么会有堆内存最大1.4GB或0.7GB的限制。正是因为如果堆内存空间过大的话，在进行垃圾回收时的全停顿会给应用逻辑造成很大的麻烦。导致服务不可用，这是万万不能接受的。

根据官方给的说法，以1.5GB的堆内存垃圾回收为例，V8在做一次小垃圾回收时需要的时间为50ms，在做一次非增量式的全堆垃圾回收时，需要耗时1s,也就是说服务需要暂停1s。V8怕你被的服务慢的要死，所以做了如上的堆内存限制。