### 锁实现的基本原理

为了实现一把具有阻塞或唤醒功能的锁，需要具备几个核心要素：

- 需要一个state变量，标记该锁的状态。state变量至少有两个值：0、1。对state变量的操作，要确保线程安全，也就是会用到CAS。

- 需要记录当前是哪个线程持有锁。

- 需要底层支持对一个线程进行阻塞或唤醒操作。

- 需要有一个队列维护所有阻塞的线程。这个队列也必须是线程安全的无锁队列，也需要用到CAS。


> 在AQS中，state取值不仅可以是0、1，还可以大于1，就是为了支持锁的重入性。

|  字段和属性值   | 含义  |
|  ----  | ----  |
| status = 0  | 表示没有线程持有锁，exclusiveOwnerThread = null |
| status = 0  | 表示有一个线程持有锁，exclusiveOwnerThread = 该线程 |
| status > 1  | 表示该线程重入了该锁 |


### AQS 原理概览




### 源码分析

先来看下AQS中最基本的数据结构——Node，Node即为CLH变体队列中的节点。

解释一下Node中几个方法和属性值的含义：

|  方法和属性值   | 含义  |
|  ----  | ----  |
| waitStatus  | 当前节点在队列中的状态 |
| thread  | 表示处于该节点的线程 |
| prev  | 前驱指针 |
| predecessor  | 返回前驱节点，没有的话抛出npe |
| nextWaiter  | 指向下一个处于CONDITION状态的节点 |
| next  | 后继指针 |

waitStatus有下面几个枚举值：

|  字段和属性值   | 含义  |
|  ----  | ----  |
| SIGNAL = -1  | 当前节点的线程如果释放了或取消了同步状态，将会将当前节点的状态标志位SINGAL，用于通知当前节点的下一节点，准备获取同步状态。 |
| CANCELLED = 1 | 被中断或获取同步状态超时的线程将会被置为当前状态，且该状态下的线程不会再阻塞。 |
| CONDITION = -2  | 当前节点在Condition中的等待队列上，其他线程调用了Condition的singal()方法后，该节点会从等待队列转移到AQS的同步队列中，等待获取同步锁。 |
|CONDITION = -3|当前线程处在SHARED情况下，该字段才会使用|
| 0  |当一个Node被初始化的时候的默认值 |

### 通过ReentrantLock理解AQS

```
public class Test {
    private static final ReentrantLock LOCK = new ReentrantLock();

    public void run() {
        LOCK.lock();
        try {
            //dosomething
        }finally {
            LOCK.unlock();
        }
    }
}
```

> 默认情况下, ReentrantLock使用非公平锁的形式。

### 公平锁与非公平锁

```
// 非公平锁实现
static final class NonfairSync extends Sync {
	final void lock() {
        // 先尝试CAS获取锁
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            // 再排队
            acquire(1);
    }
    ...
}
// 公平锁实现
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;
	// 去排队
    final void lock() {
        acquire(1);
    }
    ...
}

```

### acquire

```
// 如果第一次获取锁失败，说明此时有其他线程持有锁，所以执行acquire
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

```

### tryAcquire

```
// 调用非公平锁的tryAcquire，再一次尝试去获取锁
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}

// 返回false表明没有获取到锁，true表明成功获取锁/重入锁
final boolean nonfairTryAcquire(int acquires) {
    // 获取当前线程
    final Thread current = Thread.currentThread();
    // 获取state状态
    int c = getState();
    // 如果state是0，表明当前没有线程获取锁
    if (c == 0) {
        // 尝试去获取锁，获取成功就设置独占线程为当前线程
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 如果当前线程已经是独占线程，此时说明锁重入了
    else if (current == getExclusiveOwnerThread()) {
        // 修改state的值
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        // 设置state值，因为此时的获取锁的线程就是当前线程
        setState(nextc);
        return true;
    }
    return false;
}
// 公平锁的tryAcquire实现
protected final boolean tryAcquire(int acquires) {
    ...
        if (c == 0) {
            // hasQueuedPredecessors是公平锁的主要体现
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
    ...
}

```

[从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)

[reentrantlock实现](https://leejay.top/post/reentrantlock/)

[The java.util.concurrent Synchronizer Framework](https://gonearewe.github.io/2021/04/10/AQS-%E6%A1%86%E6%9E%B6%E8%AE%BA%E6%96%87%E7%BF%BB%E8%AF%91-The-java.util.concurrent-Synchronizer-Framework/)