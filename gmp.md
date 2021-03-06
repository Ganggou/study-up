#### 协程和线程的对应关系
* N:1 
  * 优 协程在用户态线程即完成切换，不会陷入到内核态，这种切换非常的轻量快速。
  * 劣 某个程序用不了硬件的多核加速能力，一旦某协程阻塞，造成线程阻塞，本进程的其他协程都无法执行
* 1:N
  * 优 容易实现
  * 劣 协程的创建、删除和切换的代价都由 CPU 完成
* M:N
  * 优 N:1和1:1的结合避免两者的缺点
  * 缺 实现复杂
###### 线程由 CPU 调度是抢占式的，协程由用户态调度是协作式的，一个协程让出 CPU 后，才执行下一个协程。goroutine最多占用 CPU 10ms，防止其他 goroutine 被饿死

#### GMP
G goroutine

M thread

P processor
![GMP 图示](https://cdn.learnku.com/uploads/images/202003/11/58489/Ugu3C2WSpM.jpeg!large)

* 全局队列（Global Queue）：存放等待运行的 G。
* P 的本地队列：同全局队列类似，存放的也是等待运行的 G，存的数量有限，不超过 256 个。新建 G’时，G’优先加入到 P 的本地队列，如果队列满了，则会把本地队列中一半的 G 移动到全局队列。
* P 列表：所有的 P 都在程序启动时创建，并保存在数组中，最多有 GOMAXPROCS(可配置) 个。
* M：线程想运行任务就得获取 P，从 P 的本地队列获取 G，P 队列为空时，M 也会尝试从全局队列拿一批 G 放到 P 的本地队列，或从其他 P 的本地队列偷一半放到自己 P 的本地队列，而不是销毁线程。M 运行 G，G 执行之后，M 会从 P 获取下一个 G，不断重复下去。当本线程因为 G 进行系统调用阻塞时，线程释放绑定的 P，把 P 转移给其他空闲的线程或创建新线程执行。
