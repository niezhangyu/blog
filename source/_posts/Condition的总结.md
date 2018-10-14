---
title: Condition的总结
date: 2018-10-13 22:30:09
tags:
---

### Condition简单使用案例

~~~java
public class ConditionDemo {

 public static void main(String[] args) {
     ReentrantLock lock = new ReentrantLock();
     Condition condition = lock.newCondition();
     new Thread("Thread1"){
            @Override
            public void run() {
                lock.lock();
                try {
                    System.out.println("我是线程1，我开始了，小兄弟");
                    System.out.println(Thread.currentThread().getName()
                                       +" 开始工作,当前时间是:"+System.currentTimeMillis());
                    Thread.currentThread().sleep(1000);
                    System.out.println(Thread.currentThread().getName()
                                       +" 暂停工作,当前时间是:"+System.currentTimeMillis());
                    condition.await();
                    System.out.println(Thread.currentThread().getName()
                                       +" 继续工作,当前时间是:"+System.currentTimeMillis());
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        }.start();

     new Thread("Thread2"){
         @Override
         public void run() {
             lock.lock();
             try {
                 System.out.println("我是线程2，我开始了，小老弟");
                 System.out.println(Thread.currentThread().getName()
                                    +" 开始工作,当前时间是:"+System.currentTimeMillis());
                 Thread.currentThread().sleep(2000);
                 condition.signal();
                 System.out.println(Thread.currentThread().getName()
                                    +" 完成工作,唤醒小老弟继续工作,当前时间是:"
                                    +System.currentTimeMillis());
             } catch (Exception e) {
                 e.printStackTrace();
             } finally {
                 lock.unlock();
             }
         }
     }.start();
    }
}

~~~



### Condition的await()方法

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
	// 1. 将当前线程包装成Node，尾插入到等待队列中
    Node node = addConditionWaiter();
	// 2. 释放当前线程所占用的lock，在释放的过程中会唤醒同步队列中的下一个节点
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
		// 3. 当前线程进入到等待状态
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
	// 4. 自旋等待获取到同步状态（即获取到lock）
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
	// 5. 处理被中断的情况
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```



1.先将当前线程添加到Condition内部维系的链表内，即加入等待队列

2.释放当前线程占用的锁

3.释放完毕后，while循环遍历AQS队列查看当前节点是否在队列中，如果不处于队列中，则调用LockSupport.park()阻塞当前线程。

4.自旋挂起后，直到被唤醒或者超时，将自己从等待队列中释放



### Condition的signal()方法

```ja
public final void signal() {
    //1. 先检测当前线程是否已经获取lock
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    //2. 获取等待队列中第一个节点，之后的操作都是针对这个节点
	Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}

private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
		//1. 将头结点从等待队列中移除
        first.nextWaiter = null;
		//2. while中transferForSignal方法对头结点做真正的处理
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
	//1. 更新状态为0
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     */
	//2.将该节点移入到同步队列中去
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}

```



取出Condition内部维护的等待队列的头节点（等待时间最长），开始唤醒操作

### Condition内部维护队列

Condition内部维护了等待队列的头节点和尾节点，该队列的作用是存放等待Signal信号的线程，该线程被封装为Node节点后存放在等待队列中。

1. 线程1调用reentrantLock.lock时，线程被加入到AQS的等待队列中
2. 线程1的await方法被调用时，该线程从AQS中移除，对应操作时锁的释放
3. 接着线程1马上被加入到Condition的等待队列中，等待signal信号
4. 线程2因为线程1释放锁的关系被唤醒，判断到可以获取锁，于是线程2获取锁，并被加入到AQS的等待队列中
5. 线程2执行完操作后，调用signal方法，这个时候由于Condition的等待队列只有线程1一个节点，于是该节点被取出并加入到AQS的等待队列中。这个时候线程1没有获取锁，也没有被唤醒
6. signal方法执行完毕后，线程2调用了reetrantLock.unLock()方法释放锁，这个时候因为AQS中只有线程1，于是AQS释放锁后按从头到尾的顺序唤醒线程时，线程1被唤醒，于是线程1回复执行。
7. 直到释放整个过程执行完毕。

