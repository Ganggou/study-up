## tips
* 并发与并行
  * 并发是指一个处理器同时处理多个任务，宏观上一段时间同时处理多个任务，微观上同一时刻只有一个任务在执行
  * 并行是指多个处理器或者是多核的处理器同时处理多个不同的任务。 
  * 并发是逻辑上的同时发生（simultaneous），而并行是物理上的同时发生。  
  * 多线程可能被分配到一个CPU内核中执行，也可能被分配到不同CPU执行，分配过程是操作系统所为，不可人为控制
* select原理
  * 使用多路复用的原因：大多数时候，大部分阻塞的线程或进程是处于等待状态，只有少部分会接收并处理响应，而其余的都在等待。系统为此还需要多做很多额外的线程或者进程的管理工作 
  * 每个case语句，单独抽象出以下结构体
  ```
  type scase struct {
    c           *hchan         // chan
    elem        unsafe.Pointer // 读或者写的缓冲区地址
    kind        uint16   //case语句的类型，是default、传值写数据(channel <-) 还是  取值读数据(<- channel)
    pc          uintptr // race pc (for race detector / msan)
    releasetime int64
  }
  ```
  * selectgo函数
    * 在一个select中，所有的case语句会构成一个scase结构体的数组
    * 打乱传入的case结构体顺序
    * 锁住其中的所有的channel
    * 遍历所有的channel，查看其是否可读或者可写
    * 如果其中的channel可读或者可写，则解锁所有channel，并返回对应的channel数据
    * 假如没有channel可读或者可写，但是有default语句，则同上:返回default语句对应的scase并解锁所有的channel
    * 假如既没有channel可读或者可写，也没有default语句，则将当前运行的groutine阻塞，并加入到当前所有channel的等待队列中去
    * 然后解锁所有channel，等待被唤醒
    * 此时如果有个channel可读或者可写ready了，则唤醒，并再次加锁所有channel
    * 遍历所有channel找到那个对应的channel和G，唤醒G，并将没有成功的G从所有channel的等待队列中移除
    * 如果对应的scase值不为空，则返回需要的值，并解锁所有channel
    * 如果对应的scase为空，则循环此过程
* slice增长
  * 如果老的长度小于 1024，直接翻倍
  * 如果老的长度大于 1024，如果老的长度小于需求的最小长度，增加 25% 容量
  ```
   for 0 < newcap && newcap < cap {
       newcap += newcap / 4
   }
  ```
