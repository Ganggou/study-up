## memory
* 一般程序的内存 （由高地址至低地址）
  * 环境变量和命令行参数
  * 栈区 分配内存时向地址减小的方向增长
  * 堆区 分配内存时向地址增大的方向增长
  * 全局区
  * 程序代码区
* golang内存分配核心思想
  * 每次从操作系统申请一大块儿的内存，由Go来对这块儿内存做分配，减少系统调用
  * 内存分配算法采用Google的TCMalloc算法。算法比较复杂，究其原理可自行查阅。其核心思想就是把内存切分的非常的细小，分为多级管理，以降低锁的粒度
  * 回收对象内存时，并没有将其真正释放掉，只是放回预先分配的大块内存中，以便复用。只有内存闲置过多的时候，才会尝试归还部分内存给操作系统，降低整体开销
* Go的内存结构
  * spans
    * 此区域存放了mspan的指针，spans区域用于表示arena区中的某一页(page)属于哪个mspan
    * mspan可以说是go内存管理的最基本单元，page是内存存储的基本单元(8kb)
    * go将内存块分为大小不同的67种，然后再把这67种大内存块，逐个分为小块称之为span(连续的page)，在go语言中就是上文提及的mspan。
  * bitmap
    * 标记arena(即heap)中的对象。一是的标记对应地址中是否存在对象，另外是标记此对象是否被gc标记过。一个功能一个bit位，所以，heap bitmaps用两个bit位
    * bitmap的地址是由高地址向低地址增长的
    * arena中包含基本的管理单元和程序运行时候生成的对象或实体，这两部分分别被spans和bitmap这两块非heap区域的内存所对应着
  * arena  
    * arena区域就是我们通常所说的heap
    * 从管理分配角度，由多个连续的页(page)组成的大块内存
    * 从使用角度出发，就是平时咱们所了解的:heap中存在很多”对象”
* 内存管理组件
  * mspan
    * 内存管理的基础单元，直接存储数据的地方
    * 双向链表，结构包括pre、next指针，一共多少span，span类型，管理多少页，哪些块使用了哪些没有
  * mcache
    * 为了避免多线程申请内存时不断的加锁，goroutine为每个线程分配了span内存块的缓存，这个缓存即是mcache，每个goroutine都会绑定的一个mcache，各个goroutine申请内存时不存在锁竞争的情况
    * mcache存在与processor上
    * mcache中的span链表分为两组，一组是包含指针类型的对象，另一组是不包含指针类型的对象，方便GC
    * 内存分配
      * tiny allocations的分配，有一个微型分配器tiny allocator来分配，分配的对象都是不包含指针的，例如一些小的字符串和不包含指针的独立的逃逸变量等（size<16bytes，无指针）
      * small allocations的分配，就是mcache根据对象的大小来找自身存在的大小相匹配mspan来分配（16bytes< size <=32kb）
      * 当mcach没有可用空间时，会从mcentral的 mspans 列表获取一个新的所需大小规格的mspan
  * mcentral
    * 每个mcentral保存一种特定类型的全局mspan列表，包括已分配出去的和未分配出去的
    * 有多少种类型的mspan就有多少个mcentral
    * 由于mspan是全局的，会被所有的mcache访问，所以会出现并发性问题，因而mcentral会存在一个锁
    * 假如需要分配内存时，mcentral没有空闲的mspan列表了，此时需要向mheap去获取
  * mheap
    * Go程序持有的整个堆空间，mheap全局唯一
    * 大于32K的对象被定义为大对象，直接通过mheap 分配。这些大对象的申请是由mcache发出的，而mcache在P上，程序运行的时候往往会存在多个P，因此，这个内存申请是并发的；所以为了保证线程安全，必须有一个全局锁。large allocations (size > 32k)
    * 假如需要分配的内存时，mheap中也没有了，则向操作系统申请一系列新的页（最小 1MB）
* 总结分配顺序
  * 首先通过计算使用的大小规格。
  * 然后使用mcache中对应大小规格的块分配。
  * 当mcache没有可用空间时，会从mcentral的 mspans 列表获取一个新的所需大小规格的mspan
  * 如果mcentral中没有可用的块，则向mheap申请，并根据算法找到最合适的mspan。
  * 如果申请到的mspan 超出申请大小，将会根据需求进行切分，以返回用户所需的页数。剩余的页构成一个新的 mspan 放回 mheap 的空闲列表。
  * 如果 mheap 中没有可用 span，则向操作系统申请一系列新的页（最小 1MB）。
