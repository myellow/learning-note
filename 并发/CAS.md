#### 1.什么是CAS
* 比较并交换(compare and swap, CAS)，是原子操作的一种，可用于在多线程编程中实现不被打断的数据交换操作，从而避免多线程同时改写某一数据时由于执行顺序不确定性以及中断的不可预知性产生的数据不一致问题。 该操作通过将内存中的值与指定数据进行比较，当数值一样时将内存中的数据替换为新的值。 -- https://zh.wikipedia.org/wiki/比较并交换
#### 2.Java中的CAS
* 在Java中通常使用```atomic```类来解决并发数据的问题，如```AtomicInteger```，这一系列类是在JDK1.5的时候出现的，在我们常用的```java.util.concurrent.atomic```包下。
* 底层是使用Unsafe来实现的。Unsafe这个类是JDK提供的一个比较底层的类，它不让我们程序员直接使用（其实可以通过反射的方式获取到这个类的实例）。它又很多能力：
    * 内存管理：包括分配内存、释放内存
    * 操作类、对象、变量：通过获取对象和变量偏移量直接修改数据
    * 挂起与恢复：将线程阻塞或者恢复阻塞状态
    * CAS：调用 CPU 的 CAS 指令进行比较和交换
    * 内存屏障：定义内存屏障，避免指令重排序
* AQS（AbstractQueuedSynchronizer）也大量使用了CAS来实现。
#### 3.ABA问题
* 什么是ABA问题：
    * 进程P1读取了一个数值A
    * P1被挂起(时间片耗尽、中断等)，进程P2开始执行
    * P2修改数值A为数值B，然后又修改回A
    * P1被唤醒，比较后发现数值A没有变化，程序继续执行。
    * 对于P1来说，数值A未发生过改变，但实际上A已经被变化过了，继续使用可能会出现问题。
* Java引入了新类来解决ABA问题：```AtomicStampedReference```。对比普通原子类，它多加了一个版本号的参数。
    ```Java
    int stamp = 10001;
    AtomicStampedReference<Integer> stampedReference = new AtomicStampedReference<>(0, stamp);
    stampedReference.compareAndSet(0, 10, stamp, stamp + 1);
    ```
#### 4.JDK8的CAS优化
* 大量的线程同时并发修改一个AtomicInteger，可能有很多线程会不停的自旋，进入一个无限重复的循环中。这些线程不停地获取值，然后发起CAS操作，但是发现这个值被别人改过了，于是再次进入下一个循环，获取值，发起CAS操作又失败了，再次进入下一个循环。
* Java 8推出了一个新的类，LongAdder，他就是尝试使用分段CAS以及自动分段迁移的方式来大幅度提升多线程高并发执行CAS操作的性能！
![image](https://note.youdao.com/yws/api/personal/file/WEB52c9a284e8b1c89f4b92269f7e690ca6?method=download&shareKey=3f8432e80df4a7f6d76a103ee609cf97)    
    * 在LongAdder的底层实现中，首先有一个base值，刚开始多线程来不停的累加数值，都是对base进行累加的，比如刚开始累加成了base = 5。
    * 接着如果发现并发更新的线程数量过多，就会开始施行分段CAS的机制，也就是内部会搞一个Cell数组，每个数组是一个数值分段。这时，让大量的线程分别去对不同Cell内部的value值进行CAS累加操作，这样就把CAS计算压力分散到了不同的Cell分段数值中了！这样就可以大幅度的降低多线程并发更新同一个数值时出现的无限循环的问题，大幅度提升了多线程并发更新数值的性能和效率！
    * 而且他内部实现了自动分段迁移的机制，也就是如果某个Cell的value执行CAS失败了，那么就会自动去找另外一个Cell分段内的value值进行CAS操作。这样也解决了线程空旋转、自旋不停等待执行CAS操作的问题，让一个线程过来执行CAS时可以尽快的完成这个操作。
    * 最后，如果你要从LongAdder中获取当前累加的总值，就会把base值和所有Cell分段数值加起来返回给你。
        ```Java
        public long sum() {
            Cell[] as = cells; Cell a;
            long sum = base;
            if (as != null) {
                for (int i = 0; i < as.length; ++i) {
                    if ((a = as[i]) != null)
                        sum += a.value;
                }
            }
            return sum;
        }
        ```
#### 5.参考资料
* https://juejin.im/post/5c062c87e51d451dbc21801b