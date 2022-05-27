### BlockingQueue

在所有的并发容器中，BlockingQueue是最常见的一种。BlockingQueue是一个带阻塞功能的队列，当入队列时，若队列已满，则阻塞调用者；当出队列时，若队列为空，则阻塞调用者。

### ArrayBlockingQueue

ArrayBlockingQueue是一个用数组实现的环形队列。在构造函数中，会要求传入数组的容量。

```java
public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
```

### Condition的实现

Condition控制的核心逻辑是通过AQS的ConditionObject类进行实现，并且使用一种相对独立的ConditionNode（控制节点）类进行描述。


### Condition的内部核心实现 -- await()

调用该方法的线程成功获取了锁的线程，也就是同步队列中的首节点，该方法会将当前线程构造成节点并加入等待队列中，然后释放同步状态，唤醒同步队列中的后继节点，然后当前线程会进入等待状态。

```java
public final void await() throws InterruptedException {
    // 检查当前线程是否中断，若中断，则抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 添加一个新的线程到条件队列中并将其线程的生命周期置为：waiting，然后返回该线程对应的节点
    Node node = addConditionWaiter();
    // 释放锁，唤醒AQS同步队列的后续节点继续获取锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 判断该节点是否在AQS的同步队列里, 初始的时候，Node只在Condition的条件队列里，
    // 而不在AQS的同步队列中，但执行notify操作的时候，会放进AQS的同步队列中
    // 若该节点已经在同步队列里，则说明什么？？？
    // 则表示该节点对应的线程已经被唤醒，现在需要重新抢锁
    // 注意：在AQS中，只有条件队列里的节点且该节点的前驱节点状态为SIGNAL，才能去获取锁，条件队列里的节点是不能获取锁的
    while (!isOnSyncQueue(node)) {
        // 阻塞线程
        LockSupport.park(this);
        // 检查park期间是否收到过中断信号，若收到中断信号，则退出await()
        // 若 interruptMode = 0, 则说明该线程并未收到中断的消息

        // 若 interruptMode = REINTERRUPT = 1, 表明该线程从条件队列中唤醒后需要重新中断
        // 若 interruptMode = THROW_IE = -1,表明该线程从条件队列中唤醒后需要抛出中断异常
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 再次获取锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        // 从条件队列中删除已取消的服务员节点
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}


 private Node addConditionWaiter() {
    // 条件队列里的最后一个节点 
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    // 如果 last waiter 节点对应的线程已经被取消，则从队列中清除。
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    // 新建一个节点，节点的状态为：Node.CONDITION（等待被唤醒）
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}


final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        // 获取当前锁的状态
        int savedState = getState();
        // 若释放成功，则返回当前锁的状态
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        //  若在执行该方法期间产生异常，则将node状态置为CANCELLED
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}

final boolean isOnSyncQueue(Node node) {
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    if (node.next != null) // If has successor, it must be on queue
        return true;
    // 从链表尾部查找    
    return findNodeFromTail(node);
}

// 判断当前线程是否中断，若有中断，进入 transferAfterCancelledWait  并返回中断模式，否则则返回0
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
            (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) : 0;
}


final boolean transferAfterCancelledWait(Node node) {
    // 再次修改节点状态为waiting，若修改成功，则进入AQS同步队列，然后返回true
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        enq(node);
        return true;
    }
    /*
    * If we lost out to a signal(), then we can't proceed
    * until it finishes its enq().  Cancelling during an
    * incomplete transfer is both rare and transient, so just
    * spin.
    */
    while (!isOnSyncQueue(node))
        // 让出当前CPU的调度权(使其阻塞)，直到当前线程对应的节点在同步队列中
        Thread.yield();
    return false;
}
```

> yield():它让掉当前线程 CPU 的时间片，使正在运行中的线程重新变成就绪状态，并重新竞争 CPU 的调度权。它可能会获取到，也有可能被其他线程获取到


```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}


private void reportInterruptAfterWait(int interruptMode)
    throws InterruptedException {
    if (interruptMode == THROW_IE)
        throw new InterruptedException();
    else if (interruptMode == REINTERRUPT)
        selfInterrupt();
}
```

```java
private void unlinkCancelledWaiters() {
    // 条件队列里的第一个节点 
    Node t = firstWaiter;
    Node trail = null;

    // 从条件队列里的第一个节点开始遍历 
    while (t != null) {
        Node next = t.nextWaiter;
        // 若条件队列里的第一个节点的状态waitStatus 不等于 Node.CONDITION 时
        if (t.waitStatus != Node.CONDITION) {
            // 
            t.nextWaiter = null;
            if (trail == null)
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            if (next == null)
                lastWaiter = trail;
        }
        else
            trail = t;
        t = next;
    }
}
```

### Condition的内部核心实现 -- signal()

```java
public final void signal() {
    // 判断当前线程是否持有锁
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    // 获取条件队列里的第一个节点    
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}

private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
                (first = firstWaiter) != null);
}


final boolean transferForSignal(Node node) {

    // 如果无法更改节点状态为 Node.CONDITION ，则取消该节点。    
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    // 唤醒之前先进入AQS同步队列
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}

```

> [wait/notifyAll与Condition的区别)(https://juejin.cn/post/6906295656520876039)





