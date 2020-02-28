---
title: ReentrantLock认识
date: 2020-02-28 23:17:34
tags:
---

ReentrantLock和synchronized都是独占锁，在并发场景时用来确保某块代码区域只被一个线程访问。

ReentrantLock提供了更灵活的方式来实现并发控制。synchronized是自动加锁和解锁，方法开始执行则加锁，执行完自动解锁，ReentrantLock则使用condition显式地控制加锁和解锁。就像java和c++一样，一个自动管理内存，一个是需要开发手动地申请和释放内存。<!--more-->



ReentrantLock与synchronized不同的一个地方，可以实现公平锁机制，所谓公平锁就是谁等待的时间更长，就可以优先获取锁。



java中的ArrayBlockingQueue的实现，使用ReentrantLock实现FIFO先进先出堵塞队列

```java
public class ArrayBlockingQueue {

    final Lock lock = new ReentrantLock();
    //非空条件
    final Condition notEmpty = lock.newCondition();
    //非满条件
    final Condition notFull = lock.newCondition();

    //队列长度
    final Object[] items = new Object[100];

    int putIdx/*写索引*/, takeIdx/*读索引*/, count/*队列的数量*/;

    /**
     * 如果队列已满则线程就会被堵塞
     * @param x
     * @throws InterruptedException
     */
    public void put(Object x) throws InterruptedException {
        lock.lock();
        try {
            while (count == 100) {
                //写线程堵塞
                notFull.await();
            }
            items[putIdx++] = x;
            count++;
            //唤醒读线程
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public Object take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                //读线程堵塞
                notEmpty.await();
            }
            Object x = items[takeIdx++];
            count--;
            notFull.signal();
            return x;
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        final ArrayBlockingQueue blockingQueue = new ArrayBlockingQueue();
        Thread t1 = new Thread(() -> {
            System.out.println("t1 run");
            for (int i=0;i<100;i++) {
                try {
                    blockingQueue.put(i);
                    System.out.println("putting.."+i);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        Thread t2 = new Thread(() -> {
            System.out.println("t2 run");
            for (int i=0;i<100;i++) {
                try {
                    Object val = blockingQueue.take();
                    System.out.println("taking.."+val);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }) ;

        t1.start();
        t2.start();
    }
}

```



