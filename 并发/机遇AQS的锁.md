#### 1.Semaphore
* Semaphore是一个共享式不可重入锁。默认是非公平式的Semaphore，可通过```Semaphore(int permits, boolean fair)```构造公平式的Semaphore。
* 常用方法：
    * ```void acquire()```：尝试获取资源，如果一直获取不到则阻塞直到获取到资源。如果线程被中断了则抛出InterruptedException，参考```void acquireUninterruptibly()```。 
    * ```boolean tryAcquire()```：尝试获取资源，获取失败则直接返回false，不会阻塞。这个操作是不公平的（即使构造的是公平Semaphore）。
    * ```boolean tryAcquire(long, TimeUnit)```：尝试获取资源，如果规定时间内没有获取到资源则返回false。
    * ```int drainPermits()```：尝试去获取剩余的所有资源，返回获取到的资源。
    * ```void release()```：尝试释放资源，阻塞线程直到资源释放成功。
#### 2.ReentrantLock
* ReentrantLock是一个独享式可重入锁。默认是非公平的ReentrantLock，可通过```ReentrantLock(boolean fair)```构造公平的ReentrantLock。
* 常用方法：
    * ```void lock()```：尝试获取lock，如果一直获取不到则阻塞到获取到lock。如果线程被中断了也不会立刻响应，参考```void lockInterruptibly()```。
    * ```boolean tryLock()```：尝试获取lock，获取失败则直接返回false，不会阻塞。。这个操作是不公平的（即使构造的是公平ReentrantLock）。
    * ```boolean tryLock(long, TimeUnit)```：尝试获取lock，如果规定时间内没有获取到资源则返回false。
    * ```void unlock()```：释放lock，state-1。如果重入多次，则需要unlock多次。尽量在finally里处理unlock。
    * ```int getHoldCount```：返回当前线程获取lock的次数。
    * ```int getQueueLength()```：返回当前正在等待获取lock的线程的估计数。
    * ```boolean isHeldByCurrentThread()```：如果当前线程是获取到lock的线程则返回true否则false。
* Condition，可以通过调用ReentrantLock的newCondition()获取。可以借助Condition来实现自定义的等待和唤醒功能。
    * ```void await()```：保存当前线程的lock状态state并park线程，等待被signal()唤醒或被interrupt()中断。
    * ```void signal()```：唤醒Condition等待最长的线程。
#### 3.CountDownLatch
* CountDownLatch有一个计数器字段，可以根据实际需要减少它。然后可以用它来阻塞一个调用线程，直到它被计数到零。计数器不可复用。
* CountDownLatch的tryAcquireShared只有当state=0的时候才会返回1（获取成功）。
* 常用方法：
    * ```void countDown()```：计数器减1，通过CAS不断尝试减1。
    * ```void await()```：阻塞调用的线程，直到计数器减少到0。如果本来计数器就已经为0，则直接返回。
    * ```boolean await(long, TimeUnit)```：阻塞调用的线程，直到计数器减少到0，返回true。如果计数器没在规定时间内达到0，则返回false。
#### 4.CyclicBarrier
* CyclicBarrier是一个同步工具类，它允许一组线程互相等待，直到到达某个公共屏障点。
* 对比CountDownLatch
    * CountDownLatch只能用一次；CyclicBarrier可以多次使用。
    * 执行countDown()的线程不会等待其它线程；CyclicBarrier各个协作线程会等待，直到达到屏障点。
    * CyclicBarrier提供了额外的处理，可以在达到屏障时，释放线程前，执行给定的runnable处理（由最后到达的线程执行）。
* 常用方法：
    * ```int await()```：等待所有线程达到屏障点，返回到达屏障的index，最后到达的线程返回0。如果有任意等待线程被中断，则中断的等待线程会抛出InterruptedException其它线程抛出BrokenBarrierException；如果有任意线程在有等待线程的时候调用了reset()方法，则所有等待线程会抛出BrokenBarrierException。
    * ```void reset()```：重置当前屏障，所有等待线程会抛出则所有等待线程会抛出BrokenBarrierException。
    * ```int getNumberWaiting()```：返回当前正在等待屏障点的线程数量。
#### 5.ReentrantReadWriteLock
* ReentrantReadWriteLock是一个可重入的读写锁，适用于读多写少的情况。读写锁维护了一对相关的锁，一个用于只读操作，一个用于写入操作。只要没有writer，读取锁可以由多个reader线程同时保持。写入锁是独占的。默认构造是非公平的。
* 锁状态：因为AQS只有一个状态位，所以ReentrantReadWriteLock通过把状态拆成两份来标识读锁和写锁，高16位为读锁，低16位为写锁。写锁的重入计数直接在低位进行累计；读锁可以同时有多个，可以把线程重入读锁的次数作为值存在ThreadLocal里。
* 锁降级：重入还允许从写入锁降级为读取锁，实现方式是：先获取写入锁，然后获取读取锁，最后释放写入锁。但是，不允许从读取锁升级到写入锁。
* Condition：只有写入锁才支持，读取锁获取Condition会抛出UnsupportedOperationException。
* 非公平式：写入锁优先级是最高的。如果写入锁的获取没有被阻塞，会一直自旋获取；如果有线程正在尝试获取写入锁，则读取锁继续等待，不尝试获取。
* 公平式：只要等待队列不为空且第一个线程不是当前线程（```boolean hasQueuedPredecessors()```返回true），则继续等待，不尝试获取。
* 常用方法：
    * ```ReadLock readLock()```：获取当前读写锁的读锁对象（其它方法与```ReentrantLock```一致）。
    * ```WriteLock writeLock()```：获取当前读写锁的写锁对象（其它方法与```ReentrantLock```一致）。
* 参考资料
    * https://www.cnblogs.com/xiaoxi/p/9140541.html（//todo 后续需要参考并补充内容）
    * http://ifeve.com/juc-reentrantreadwritelock/
#### 6.相关概念：
* 公平锁能保证：老的线程排队使用锁，新线程仍然排队使用锁。 
* 非公平锁保证：老的线程排队使用锁；但是无法保证新线程抢占已经在排队的线程的锁。 