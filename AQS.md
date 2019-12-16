# AQS 同步器、reentrantlock、fork join、countdownlatch

## countdownlatch闭锁
* 所有子线程共用一个 countdownlatch对象，
* 每个子线程执行完了调用countdownlatch的countDown方法，
* 主线程调用await方法，如果子线程没有执行完，则调用countdownlatch对象的await方法是阻塞的，如果所有子线程都执行完了，执行countdownlatch对象的await方法是顺畅的，接下来就是处理主线程的业务逻辑。

# reentrantlock
* reentrantlock是通过内部内sync来管理锁，而sync是aqs类的一个子类
* reentrantlock里面包含2种锁，分别是公平锁和非公平锁，NonfairSync和FairSync是Sync的子类
* reentrantlock默认采用非公平锁，通过构造函数来初始化
  
  ```
    public ReentrantLock() {
    // 默认非公平锁
    sync = new NonfairSync();
    }
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

  ```
* 在reentrantlock中，阻塞队列是一个链表，把每个抢占锁的线程封装成一个Node，数据结构如下
  ```
  static final class Node {
    // 标识节点当前在共享模式下
    static final Node SHARED = new Node();
    // 标识节点当前在独占模式下
    static final Node EXCLUSIVE = null;

    // ======== 下面的几个int常量是给waitStatus用的 ===========
    /** waitStatus value to indicate thread has cancelled */
    // 代码此线程取消了争抢这个锁
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking */
    // 官方的描述是，其表示当前node的后继节点对应的线程需要被唤醒（也就是说waitStatus并不是表示当前节点的状态，而是表明后续节点需要被唤醒）
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition */
    static final int CONDITION = -2;
    /**
     * waitStatus value to indicate the next acquireShared should
     * unconditionally propagate
     */
    static final int PROPAGATE = -3;
    // =====================================================


    // 取值为上面的1、-1、-2、-3，或者0
    // 这么理解，暂时只需要知道如果这个值 大于0 代表此线程取消了等待，
    //    ps: 半天抢不到锁，不抢了，ReentrantLock是可以指定timeouot的。。。
    volatile int waitStatus;
    // 前驱节点的引用
    volatile Node prev;
    // 后继节点的引用
    volatile Node next;
    // 这个就是线程本尊
    volatile Thread thread;

    }
  ```    
* 非公平锁，在调用（acquire）cas进行一次抢锁后，如果这个时候恰好锁没有被占用，那就直接获取锁。如果cas后被锁占用之后，就去调用tryAcquire(AQS)去获取锁，在tryAcquire中又先通过通过cas去抢锁，如果此时正好，如果还是没有抢到，就会把当前线程挂起，放到阻塞队列中等待被唤醒
* 公平锁，直接去调用tryAcquire(AQS)去获取锁，先判断堵塞队列中是否有等待的，没有等待则直接去用cas抢锁，如果没有抢到锁，则进入acquireQueued(AQS)
  ```
  在调用次方法前获取调用addwaiter方法，将当前线程节点添加了阻塞队列链表的末尾，addwaiter的时候也需要通过cas进行竞争入队列，如果发现正有线程在入队列，则通过自旋（一直循环去入队列，直到入队尾成功） --以下几个方法都是CQS的方法
  final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    //如果当前节点node的前一个节点p是head节点，head节点初始化时是个空对象，head节点一般是占有锁的线程，
                    //如果head节点是初始化进来的，则他是不占锁的，并且是个空对象，这个时候可以再去tryAcquire获取一下锁
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //一直遍历前区节点（从后到前）第一个遍历到前区的waitstatus要挂起，这返回true,如果都不要挂起则返回false
                if (shouldParkAfterFailedAcquire(p, node) &&
                //挂起线程
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }


    // 此方法的作用是把线程包装成node，同时进入到队列中
    // 参数mode此时是Node.EXCLUSIVE，代表独占模式
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        // 以下几行代码想把当前node加到链表的最后面去，也就是进到阻塞队列的最后
        Node pred = tail;

        // tail!=null => 队列不为空(tail==head的时候，其实队列是空的，不过不管这个吧)
        if (pred != null) { 
            // 将当前的队尾节点，设置为自己的前驱 
            node.prev = pred; 
            // 用CAS把自己设置为队尾, 如果成功后，tail == node 了，这个节点成为阻塞队列新的尾巴
            if (compareAndSetTail(pred, node)) { 
                // 进到这里说明设置成功，当前node==tail, 将自己与之前的队尾相连，
                // 上面已经有 node.prev = pred，加上下面这句，也就实现了和之前的尾节点双向连接了
                pred.next = node;
                // 线程入队了，可以返回了
                return node;
            }
        }
        // 如果会到这里，
        // 说明 pred==null(队列是空的) 或者 CAS失败(有线程在竞争入队)
        enq(node);
        return node;
    }


    // 采用自旋的方式入队
    // 之前说过，到这个方法只有两种可能：等待队列为空，或者有线程竞争入队，
    // 自旋在这边的语义是：CAS设置tail过程中，竞争一次竞争不到，我就多次竞争，总会排到的
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            // 之前说过，队列为空也会进来这里
            if (t == null) { // Must initialize
                // 初始化head节点
                // 原来 head 和 tail 初始化的时候都是 null 的
                // 还是一步CAS，你懂的，现在可能是很多线程同时进来呢
                if (compareAndSetHead(new Node()))
                    // 给后面用：这个时候head节点的waitStatus==0, 看new Node()构造方法就知道了

                    // 这个时候有了head，但是tail还是null，设置一下，
                    // 把tail指向head，放心，马上就有线程要来了，到时候tail就要被抢了
                    // 注意：这里只是设置了tail=head，这里可没return哦，没有return，没有return
                    // 所以，设置完了以后，继续for循环，下次就到下面的else分支了
                    tail = head;
            } else {
                // 下面几行，和上一个方法 addWaiter 是一样的，
                // 只是这个套在无限循环里，反正就是将当前线程排到队尾，有线程竞争的话排不上重复排
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
  ```
* 对于在阻塞队列中的线程，什么时候被唤醒呢，就是在对调用reentrantlock调用unlock，紧接着sync调用release，也就是说在释放锁的时候唤醒
    ```
    public void unlock() {
        sync.release(1);
    }

    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                //这里就是唤醒阻塞队列里的线程
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

    protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    // 是否完全释放锁
    boolean free = false;
    // 其实就是重入的问题，如果c==0，也就是说没有嵌套锁了，可以释放了，否则还不能释放掉
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
    }

    CQS的方法unparkSuccessor

    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        //如果head节点当前waitStatus<0, 将其修改为0
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            //说明head节点的next节点是空或者取消了等待，
            //这种情况则是从队尾开始寻找，排在最前面的waitstatus小于等于0的节点
            //为什么要从后往前找，因为
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            //唤醒next节点的线程
            LockSupport.unpark(s.thread);
    }

    ```
* AQS中，取消占用锁排队：包括三步node不关联任何线程，node的waitstatus为取消状态，node出阻塞队列。
* reentranklock 加锁解锁的需要三部分的协作
  * 1、锁状态，state=0 代表没有线程没有占有锁，可以通过cas去占用锁，占用成功后state设置为1，如果是同一个线程再次进来占锁，则state继续+1，解锁则state-1,这个state是volatile来修饰的，知道变成0，才可以继续被别的线程占用
  * 2、线程的阻塞和唤醒，阻塞线程LockSupport.park(this);，唤醒线程LockSupport.unpark(node.thread);
  * 3、阻塞队列，AQS通过一个fifo队列，也就是一个链表，每一个线程节点node,都有后续节点和前置节点的引用。

* 线程1获取锁后，还没是否锁，此时线程2调用lock来占用锁，阻塞队列的情况是，第一个节点是Head节点，Head节点的所属线程是null，Head节点的waitstatus=-1，而这个节点的下一个节点是线程2创建的node1,node1的所属线程是线程2，node1节点的waitstatus=0,node1的前置节点是Head节点。

* reentrantlock的lock，trylock、lockInterruptibly区别
   * lock是去获取锁，如果获取失败则进入阻塞队列，知道获取成功
   * trylock 一般是说获取一段时间，入参可以设置，在定义时间内获取成功返回true,获取失败返回false
   * lockInterruptibly（如果当前线程在获取锁过程中被中断，则抛异常，不再获取锁，阻塞队列中也不会有该线程，因为在获取锁抛异常之后，会将当前线程对队列cancelAcquire）， 
  ```
  //该方法是
  public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
        //获取锁失败，直接将当期线程中断
            doAcquireInterruptibly(arg);
    }

    //该方法是特定时间内获取锁
    public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }
  ```
  ```
      static class Resource{
        ReentrantLock lock = new ReentrantLock();
    public static void main(String[] args){
        final Resource resource = new Resource();
        Thread t1 = new Thread(()->{
            resource.test();
        },"t1");
        t1.start();
        try{
            Thread.sleep(1000);
        }catch (Exception e){
            e.printStackTrace();
        }
        System.out.println("a2 "+System.currentTimeMillis());
        Thread t2 = new Thread(()->{
            resource.test1();
        },"t2");
        t2.start();

    }

        public void test(){
            lock.lock();
            try{
                System.out.println("a "+System.currentTimeMillis());
            }catch (Exception e){
                e.printStackTrace();
            }
            //故意不释放锁
            //lock.unlock();
        }

        public void test1(){
            try{
                System.out.println("b "+System.currentTimeMillis());
                final Thread t = Thread.currentThread();
                new Thread(()->{
                    try{
                        System.out.println("e "+System.currentTimeMillis());
                        Thread.sleep(1000);
                        //中断线程，
                        t.interrupt();
                        System.out.println("f "+System.currentTimeMillis());
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                },"t3").start();
                System.out.println("c "+System.currentTimeMillis());
                //因为上面t3线程中断了外面的线程，所以这里再获取中断锁的过程中会抛出线程中断异常。
                lock.lockInterruptibly();
                System.out.println(lock.tryLock());
            }catch (Exception e){
                e.printStackTrace();
                //获取中断锁的线程不会在阻塞队列中
                System.out.println("aaa"+lock.hasQueuedThread(Thread.currentThread()));
            }
        }

    }
  ```

## 参考博客
https://www.jianshu.com/p/01f2046aab64

https://www.jianshu.com/p/89132109d49d


## Condition

* 使用场景：生产者和消费者模型
* condition是依赖reentrantlock产生，reentrantlock.newCondition();
* 在使用 condition 时，必须先持有相应的锁。这个和 Object 类中的方法有相似的语义，需要先持有某个对象的监视器锁才可以执行 wait(), notify() 或 notifyAll() 方法
* 我们常用 obj.wait()，obj.notify() 或 obj.notifyAll() 来实现相似的功能，但是，它们是基于对象的监视器锁的. 这里说的 Condition 是基于 ReentrantLock 实现的，而 ReentrantLock 是依赖于 AbstractQueuedSynchronizer 实现的。
* 上面介绍AQS的时候，有一个阻塞队列（也叫同步队列sync queue）存放等待锁线程节点用的，而ConditionObject(Condition的子类)有两个属性
  ```
  public class ConditionObject implements Condition, java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
        // 条件队列的第一个节点 transient 是不需要系列化的意思
        /** First node of condition queue. */
        private transient Node firstWaiter;
        // 条件队列的最后一个节点
        /** Last node of condition queue. */
        private transient Node lastWaiter;
  ```