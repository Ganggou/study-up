## channel
不要通过共享内存来通信，而要通过通信来实现内存共享。
* CSP模型
  * CSP 是 Communicating Sequential Process 的简称，中文可以叫做通信顺序进程，是一种并发编程模型，是一个很强大的并发数据模型，是上个世纪七十年代提出的，用于描述两个独立的并发实体通过共享的通讯 channel(管道)进行通信的并发模型。
  * go语言并没有完全实现CSP模型的所有理论，仅仅是借用了 process和channel这两个概念。process是在go语言上的表现就是 goroutine， 是实际并发执行的实体，每个实体之间是通过channel通讯来实现数据共享。
* channel底层结构
  * hchan
  ```
  type hchan struct {
    // chan 里元素数量
    qcount   uint
    // chan 底层循环数组的长度
    dataqsiz uint
    // 指向底层循环数组的指针
    // 只针对有缓冲的 channel
    buf      unsafe.Pointer
    // chan 中元素大小
    elemsize uint16
    // chan 是否被关闭的标志
    closed   uint32
    // chan 中元素类型
    elemtype *_type // element type
    // 已发送元素在循环数组中的索引
    sendx    uint   // send index
    // 已接收元素在循环数组中的索引
    recvx    uint   // receive index
    // 等待接收的 goroutine 队列
    recvq    waitq  // list of recv waiters
    // 等待发送的 goroutine 队列
    sendq    waitq  // list of send waiters

    // 保护 hchan 中所有字段
    lock mutex
  }
  
  type waitq struct {
    first *sudog
    last  *sudog
  }
  ```

* channel收发数据（假设G1 send，G2 recv）
  * 缓存链表中每一步操作都需要加锁，细化为：
      * 加锁
      * 把数据从goroutine中copy到“队列”中(或者从队列中copy到goroutine中）
      * 释放锁 
  * channel缓存满时
      * G1执行，会主动调用Go的调度器,让G1等待，并从让出M，让其他G去使用
      * 同时G1也会被抽象成含有G1指针和send元素的sudog结构体保存到hchan的sendq中等待被唤醒
      * 假设G2从缓存队列中取出数据，channel会将等待队列中的G1推出，将G1当时send的数据推到缓存中，然后调用Go的scheduler，唤醒G1，并把G1放到可运行的Goroutine队列中
  * 如果接收队列里有 goroutine，直接将要发送的数据拷贝到接收 goroutine
      * G2执行，会主动调用Go的调度器,让G2等待，并从让出M，让其他G去使用
      * G2还会被抽象成含有G2指针和recv空元素的sudog结构体保存到hchan的recvq中等待被唤醒
      * 当G1再执行的时候，并没有锁住channel，然后将数据放到缓存中，而是直接把数据从G1直接copy到了G2的栈中。在唤醒过程中，G2无需再获得channel的锁，然后从缓存中取数据。减少了内存的copy，提高了效率
       
* channel的关闭
  * close 函数先上一把大锁，接着把所有挂在这个 channel 上的 sender 和 receiver 全都连成一个 sudog 链表，再解锁。最后，再将所有的 sudog 全都唤醒
* channel操作panic的情况
  * 向一个关闭的 channel 进行写操作；关闭一个 nil 的 channel；重复关闭一个 channel(读、写一个 nil channel 都会被阻塞) 
