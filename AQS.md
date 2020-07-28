#### 1.概述
1. AQS，指的是java.util.concurrent.locks.AbstractQueuedSynchronizer，是一个用于构建锁和同步器的框架。
2. AQS支出两种资源共享方式：Exclusive（独占，只有一个线程可以执行，如ReentrantLock）、Share（共享，可以有多个线程同时执行，如Semaphore/CountDownLatch）。其次它又可以分为公平锁和非公平锁。
3. 维护了一个volatile int state(代表资源)和一个CLH(Craig,Landin,and Hagersten)队列。
![CLH队列](https://note.youdao.com/yws/api/personal/file/WEB783306383f995efd46591e22cee70596?method=download&shareKey=3f8432e80df4a7f6d76a103ee609cf97)
4. 由于底层使用了模板方法模式，所以我们实际需要去实现一个同步器的时候，只需要继承AQS并重写指定方法即可。至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了。一般来说，自定义同步器只需要实现```tryAcquire-tryRelease```、```tryAcquireShared-tryReleaseShared```其中一种，但也支持同时都实现如ReentrantReadWriteLock。
    ```
    isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。
    tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false。
    tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败则返回false。
    tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
    tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。
    ```
#### 2.源码简析
1. Node节点
    * Node节点是AbstractQueuedSynchronizer里的一个内部静态类。当线程获取资源失败（比如tryAcquire时试图设置state状态失败），会被构造成一个结点加入CLH队列中，同时当前线程会被阻塞在队列中（通过LockSupport.park实现，其实是等待态）。当持有同步状态的线程释放同步状态时，会唤醒后继结点，然后此结点线程继续加入到对同步状态的争夺中。
        ```Java
        static final class Node {
            /** 表示线程已经被取消（超时/中断） */
            static final int CANCELLED =  1;
            /** 表示后继（successor）线程需要被唤醒（unpaking） */
            static final int SIGNAL    = -1;
            /** 表示结点线程等待在condition上，当被signal后，会从等待队列转移到同步到队列中 */
            static final int CONDITION = -2;
            /** 表示下一次共享式同步状态会被无条件地传播下去 */
            static final int PROPAGATE = -3;
    
            /**
             * 等待状态
             */
            volatile int waitStatus;
    
            /**
             * 前驱节点
             */
            volatile Node prev;
    
            /**
             * 后继节点
             */
            volatile Node next;
    
            /**
             * node对应的线程
             */
            volatile Thread thread;
        }
        ```
    * waitStatus状态值。AQS在判断状态时，通过用waitStatus<0表示有效状态，而waitStatus>0表示取消状态。值为0，代表初始化状态。
        * CANCELLED：值为1，在同步队列中等待的线程等待超时或被中断，需要从同步队列中取消该Node的结点，其结点的waitStatus为CANCELLED，即结束状态，进入该状态后的结点将不会再变化。
        * SIGNAL：值为-1，被标识为该等待唤醒状态的后继结点，当其前继结点的线程释放了同步锁或被取消，将会通知该后继结点的线程执行。说白了，就是处于唤醒状态，只要前继结点释放锁，就会通知标识为SIGNAL状态的后继结点的线程执行。
        * CONDITION：值为-2，与Condition相关，该标识的结点处于等待队列中，结点的线程等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将从等待队列转移到同步队列中，等待获取同步锁。
        * PROPAGATE：值为-3，与共享模式相关，在共享模式中，该状态标识结点的线程处于可运行状态。

2. 独占式的acquire()
    * acquire()
        ```Java
        public final void acquire(int arg) {
            if (!tryAcquire(arg) &&
                acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
                selfInterrupt();
        }
        ```
        * tryAcquire()尝试直接去获取资源，如果成功则直接返回；
        * addWaiter()将该线程构造成独占结点加入等待队列的尾部；
        * acquireQueued()使线程在等待队列中获取资源，一直获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
        * 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。
    * addWaiter()
        ```Java
        private Node addWaiter(Node mode) {
            Node node = new Node(Thread.currentThread(), mode); // 构造结点
            // 记录尾部结点
            Node pred = tail;
            if (pred != null) {
                node.prev = pred; // 如果尾部结点不为空，则把当前结点的前驱结点指向尾部结点
                if (compareAndSetTail(pred, node)) { // 通过CAS尝试把结点添加到尾部
                    pred.next = node;
                    return node;
                }
            }
            // 尾部结点为空或者添加失败则进入enq()方法
            enq(node);
            return node;
        }
        ```
    * enq()
        ```Java
        private Node enq(final Node node) {
            // CAS自旋，不断尝试把自己添加到队尾
            for (;;) {
                Node t = tail; // 记录尾部结点
                if (t == null) { // 如果尾部为null则初始化
                    if (compareAndSetHead(new Node()))
                        tail = head;
                } else {
                    node.prev = t; // 把当前结点前驱指向尾部结点
                    if (compareAndSetTail(t, node)) { // 通过CAS设置尾部结点
                        t.next = node;
                        return t; // 返回当前结点的前驱（原尾部结点）
                    }
                }
            }
        }
        ```
    * acquireQueued()
        ```Java
        final boolean acquireQueued(final Node node, int arg) {
            boolean failed = true; // 标记资源是否获取失败
            try {
                boolean interrupted = false; // 标记等待过程是否被中断
                for (;;) {
                    final Node p = node.predecessor(); // 获取node的前驱结点
                    if (p == head && tryAcquire(arg)) { // 只有等待队列的第二个结点才允许去调用tryAcquire()获取资源
                        setHead(node); // 获取同步状态成功，将当前结点设置为头结点。
                        p.next = null; // 帮助GC
                        failed = false;
                        return interrupted;
                    }
                    // 判断当前是否可以进入waiting状态
                    if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                        interrupted = true; // 等待过程被中断，则将interrupted标记为true
                }
            } finally {
                if (failed)
                    cancelAcquire(node);
            }
        }
        ```
        * 通过tryAcquire()和addWaiter()，该线程获取资源失败，已经被放入等待队列尾部了。如果不能尝试获取资源或者获取资源失败，则进入shouldParkAfterFailedAcquire去判断判断当前线程是否应该阻塞，若可以，调用parkAndCheckInterrupt阻塞当前线程，直到被中断或者被前驱结点唤醒。若还不能休息，继续循环。
    * shouldParkAfterFailedAcquire()
        ```Java
        private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
            int ws = pred.waitStatus; // 获取前驱结点的等待状态
            if (ws == Node.SIGNAL)
                // 如果前驱结点的等待状态是SIGNAL，则意味着当前结点可以安全地park
                return true;
            if (ws > 0) {
                // 如果前驱结点已经取消，则前驱结点无效。从后往前遍历，找到一个非CANCEL状态的结点，将自己设置为它的后继结点
                // 这里意味着自己被前面的无效结点加塞，所以直接跳过前面的无效结点，直接排在最近一个有效结点后面
                do {
                    node.prev = pred = pred.prev;
                } while (pred.waitStatus > 0);
                pred.next = node;
            } else {
                // waitStatus must be 0 or PROPAGATE
                // 把前置结点的状态置为SIGNAL，让它完事之后通知后继结点(当前的node)。在park之前再尝试获取资源
                compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
            }
            return false;
        }
        ```
        * 若shouldParkAfterFailedAcquire返回true，也就是当前结点的前驱结点为SIGNAL状态，则意味着当前结点可以放心休息，进入parking状态了。parkAncCheckInterrupt阻塞线程并处理中断。
    * parkAndCheckInterrupt()
        ```Java
        private final boolean parkAndCheckInterrupt() {
            LockSupport.park(this); // 使用LockSupport使线程进入阻塞状态
            return Thread.interrupted(); // 如果被唤醒，查看自己是不是被中断的
        }
        ```
        * park()会让当前线程进入waiting状态。在此状态下，有两种途径可以唤醒该线程：被unpark()、被interrupt()。需要注意的是，Thread.interrupted()会清除当前线程的中断标记位。 
        * 线程状态转换
        ![线程状态转换](https://note.youdao.com/yws/api/personal/file/WEB4e980fa883c45ae1e9fd68b9936cc42d?method=download&shareKey=3f8432e80df4a7f6d76a103ee609cf97)
    * 小结
        * 首先tryAcquire获取同步状态，成功则直接返回；否则，进入下一环节；
        * 线程获取同步状态失败，就构造一个结点，加入同步队列中，这个过程要保证线程安全；
        * 加入队列中的结点线程进入自旋状态，若是老二结点（即前驱结点为头结点），才有机会尝试去获取同步状态；否则，当其前驱结点的状态为SIGNAL，线程便可安心休息，进入阻塞状态，直到被中断或者被前驱结点唤醒。
        * 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。
3. 独占式release()
    * release()
        ```Java
        public final boolean release(int arg) {
            if (tryRelease(arg)) { // 调用重写的方法去释放资源
                Node h = head;
                if (h != null && h.waitStatus != 0)
                    unparkSuccessor(h); // 唤醒后继结点
                return true;
            }
            return false;
        }
        ```
    * unparkSuccessor()
        ```Java
        private void unparkSuccessor(Node node) {
            int ws = node.waitStatus;
            if (ws < 0) // 置零当前线程所在的结点状态，允许失败
                compareAndSetWaitStatus(node, ws, 0);
    
            Node s = node.next; // 寻找下一个需要被唤醒的节点
            if (s == null || s.waitStatus > 0) { // 如果节点为空或者被取消
                s = null;
                for (Node t = tail; t != null && t != node; t = t.prev) // 从后往前遍历，找到最前面的处于正常阻塞状态的节点，进行唤醒
                    if (t.waitStatus <= 0)
                        s = t;
            }
            if (s != null)
                LockSupport.unpark(s.thread); // 调用unpark唤醒线程
        }
        ```
        * 一句话概括：用unpark()唤醒等待队列中最前边的那个未放弃线程，这里我们也用s来表示吧。此时，再和acquireQueued()联系起来，s被唤醒后，进入if (p == head && tryAcquire(arg))的判断（即使p!=head也没关系，它会再进入shouldParkAfterFailedAcquire()寻找一个安全点。这里既然s已经是等待队列中最前边的那个未放弃线程了，那么通过shouldParkAfterFailedAcquire()的调整，s也必然会跑到head的next结点，下一次自旋p==head就成立啦），然后s把自己设置成head标杆结点，表示自己已经获取到资源了，acquire()也返回了！
    * 小结
        * release()是独占模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果彻底释放了（即state=0）,它会唤醒等待队列里的其他线程来获取资源。
4. 共享式acquireShared()
    * acquireShared()
        ```Java
        public final void acquireShared(int arg) {
            if (tryAcquireShared(arg) < 0) // 尝试去获取资源，获取失败返回负数，非负数则表示获取成功
                doAcquireShared(arg);
        }
        ```
5. 共享式releaseShared()