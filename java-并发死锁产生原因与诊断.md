---
title: java 并发死锁产生原因与诊断
date: 2015-09-03 22:53:36
tags:
---
**死锁的产生原因**

并发死锁产生的原因一般是由于加锁顺序不一致引起的，假设一个事物需要获得连续获得两个资源的锁，如果执行事物的两个线程获得这两个锁的顺序不一致，就有可能产生死锁。

下面是我画的一个简单的说明图：
![enter description here][1]

在Dead lock point， 第一个线程拥有A锁，想获得B锁， 第二个线程拥有B锁，想获得A锁，两个线程相互拥有另一个线程所等待的资源。
造成这种局面的原因就是两个线程加锁的顺序不一样，一旦程序中产生死锁，唯一能做的就是杀死进程，重启应用。

**用java代码实现死锁**

运行以下代码，程序会一直运行，不退出。
```java
public class TestDeadLock {  
    private static Lock lockA = new ReentrantLock();  
    private static Lock lockB = new ReentrantLock();  
  
    public static void main(String args[]){  
        new Thread(new Runnable() {  
            @Override  
            public void run() {  
                try{  
                    lockA.lock();  
                    TimeUnit.SECONDS.sleep(2);  
                    try{  
                        lockB.lock();  
                    }finally {  
                        lockB.unlock();  
                    }  
                }catch (InterruptedException e) {  
  
                }finally {  
                    lockA.unlock();  
                }  
            }  
        }).start();  
        new Thread(new Runnable() {  
            @Override  
            public void run() {  
                try{  
                    lockB.lock();  
                    TimeUnit.SECONDS.sleep(2);  
                    try{  
                        lockA.lock();  
                    }finally {  
                        lockA.unlock();  
                    }  
                }catch (InterruptedException e) {  
  
                }finally {  
                    lockB.unlock();  
                }  
            }  
        }).start();  
    }  
} 
```
 代码链接： https://github.com/HongkaiWen/study/blob/master/src/main/java/concurrent/deadlock/TestDeadLock.java

**程序产生死锁后的排查**

产生死锁的现象就是程序停在那里，不会对调用者产生反馈。以我上边写的例子来说，就是程序无法退出。
下面以我这个例子来说明如何确定程序是因为死锁才停在那里。（这个程序是我故意写的死锁，一般的应用中是没有人会故意写死锁的，所以程序发生阻塞时，需要定位问题的原因）

**windows平台：**

我在我本机windows平台下运行我的死锁程序， 程序停在那里不动。
这时我打开jvisualvm.exe （jdk的bin目录下），这是一个图形界面程序，截图如下：

![enter description here][2]

关于此界面的说明：
首先左侧是一个JVM的进程列表，在列表中我可以找到我的应用的进程，单击目标进程即可查看进程详细信息。
右边很明显了，红色字体提示检测到死锁，点击线程Dump就可以查看进一步的详细信息了。
下面是一条一条的时间线是所有线程的状态了，Thread-1 和 Thread-2是我们的用户线程，黄色代表等待状态。
 
下面是线程dump的信息，信息比较多，我贴出关键部分：

```
Found one Java-level deadlock:  
=============================  
"Thread-1":  
  waiting for ownable synchronizer 0x09df1d10, (a java.util.concurrent.locks.ReentrantLock$NonfairSync),  
  which is held by "Thread-0"  
"Thread-0":  
  waiting for ownable synchronizer 0x09df1ef8, (a java.util.concurrent.locks.ReentrantLock$NonfairSync),  
  which is held by "Thread-1"  
```

意思就是线程1等待线程2持有的资源，线程2等待线程1持有的资源，再下面的信息可以定位到具体代码的哪一行：

```
"Thread-1":  
    at sun.misc.Unsafe.park(Native Method)  
    - parking to wait for  <0x09df1d10> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)  
    at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)  
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)  
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:870)  
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1199)  
    at java.util.concurrent.locks.ReentrantLock$NonfairSync.lock(ReentrantLock.java:209)  
    at java.util.concurrent.locks.ReentrantLock.lock(ReentrantLock.java:285)  
    at concurrent.TestDeadLock$2.run(TestDeadLock.java:40)  
    at java.lang.Thread.run(Thread.java:745)  
"Thread-0":  
    at sun.misc.Unsafe.park(Native Method)  
    - parking to wait for  <0x09df1ef8> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)  
    at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)  
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)  
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:870)  
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1199)  
    at java.util.concurrent.locks.ReentrantLock$NonfairSync.lock(ReentrantLock.java:209)  
    at java.util.concurrent.locks.ReentrantLock.lock(ReentrantLock.java:285)  
    at concurrent.TestDeadLock$1.run(TestDeadLock.java:22)  
    at java.lang.Thread.run(Thread.java:745)  
```

**linux平台：**

 一般linux平台不太方便用图形界面的工具，而一般java的程序都是部署在linux平台上的，这种环境程序发生阻塞时要怎么办呢，和windows思路是一样的，只不过是工具不同而已。
首先我在linux上运行我的java程序，程序如期卡住。


![enter description here][3]

这时，我运行进程查看命令，找到我的进程：

![enter description here][4]

此时可以通过jstack命令导出jvm的栈调用信息：

![enter description here][5]

下面是stack.txt中导出的信息（部分）：
 找到了和之前windows一样的信息。

```
Found one Java-level deadlock:  
=============================  
"Thread-1":  
  waiting for ownable synchronizer 0x00000000d6c68848, (a java.util.concurrent.locks.ReentrantLock$NonfairSync),  
  which is held by "Thread-0"  
"Thread-0":  
  waiting for ownable synchronizer 0x00000000d6c68878, (a java.util.concurrent.locks.ReentrantLock$NonfairSync),  
  which is held by "Thread-1"  
  
Java stack information for the threads listed above:  
===================================================  
"Thread-1":  
    at sun.misc.Unsafe.park(Native Method)  
    - parking to wait for  <0x00000000d6c68848> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)  
    at java.util.concurrent.locks.LockSupport.park(LockSupport.java:186)  
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:834)  
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:867)  
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1197)  
    at java.util.concurrent.locks.ReentrantLock$NonfairSync.lock(ReentrantLock.java:214)  
    at java.util.concurrent.locks.ReentrantLock.lock(ReentrantLock.java:290)  
    at concurrent.deadlock.TestDeadLock$2.run(TestDeadLock.java:40)  
    at java.lang.Thread.run(Thread.java:745)  
"Thread-0":  
    at sun.misc.Unsafe.park(Native Method)  
    - parking to wait for  <0x00000000d6c68878> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)  
    at java.util.concurrent.locks.LockSupport.park(LockSupport.java:186)  
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:834)  
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:867)  
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1197)  
    at java.util.concurrent.locks.ReentrantLock$NonfairSync.lock(ReentrantLock.java:214)  
    at java.util.concurrent.locks.ReentrantLock.lock(ReentrantLock.java:290)  
    at concurrent.deadlock.TestDeadLock$1.run(TestDeadLock.java:22)  
    at java.lang.Thread.run(Thread.java:745)  
  
Found 1 deadlock.  
```

**死锁程序的预防与处理**

避免 死锁的方法就是不要写 顺序不一致加锁 的代码
 如果 必须要写顺序不一致的加锁的代码，可以采用 trylock 方法，避免程序一直等待。
 预防 死锁可以采用finddebug插件去扫描代码发现死锁。
 处理 死锁，如果运行中的程序已经发生死锁，据我所知没有别的办法，只能停止程序，代价非常大，所以要尽量避免死锁。
 
关于死锁的避免与预防有时间我再整理一下分享一下我的心得。
关于并发编程，想要写出正确可靠高性能的程序实在比较难，我知道的也是一些皮毛，欢迎大家批评指正，互相学习。

-----------------------------分割线------------------------------

上面是差不多两年前写的文章，如果从死锁的角度出发去学习死锁产生的原因以及分析方法可以按照上面的过程来了解，当然死锁产生的原因也不仅上面一种，还有资源死锁等。

如果从另外一个角度出发，解决问题的工具箱，通过上面的文章，我们介绍了一个工具**jstack**，他可以打印出程序运行中的调用栈信息，在发生程序卡住，响应缓慢等问题时这也是分析问题原因的一个工具。

再之前的工作中遇见过一段很简单的代码，执行时间超长，当时分析不出原因，正是通过jstack来分析出的，定位到一个正则性能低下。

总之，jvm卡在那里不动了，日志也没有错误输出，jstack一把，可以定位程序死在哪一行。


  [1]: https://raw.githubusercontent.com/HongkaiWen/images/master/blog/java%20%E5%B9%B6%E5%8F%91%E6%AD%BB%E9%94%81%E4%BA%A7%E7%94%9F%E5%8E%9F%E5%9B%A0%E4%B8%8E%E8%AF%8A%E6%96%AD/image1.png
  [2]: https://raw.githubusercontent.com/HongkaiWen/images/master/blog/java%20%E5%B9%B6%E5%8F%91%E6%AD%BB%E9%94%81%E4%BA%A7%E7%94%9F%E5%8E%9F%E5%9B%A0%E4%B8%8E%E8%AF%8A%E6%96%AD/image2.png
  [3]: https://raw.githubusercontent.com/HongkaiWen/images/master/blog/java%20%E5%B9%B6%E5%8F%91%E6%AD%BB%E9%94%81%E4%BA%A7%E7%94%9F%E5%8E%9F%E5%9B%A0%E4%B8%8E%E8%AF%8A%E6%96%AD/image3.png
  [4]: https://raw.githubusercontent.com/HongkaiWen/images/master/blog/java%20%E5%B9%B6%E5%8F%91%E6%AD%BB%E9%94%81%E4%BA%A7%E7%94%9F%E5%8E%9F%E5%9B%A0%E4%B8%8E%E8%AF%8A%E6%96%AD/image4.png
  [5]: https://raw.githubusercontent.com/HongkaiWen/images/master/blog/java%20%E5%B9%B6%E5%8F%91%E6%AD%BB%E9%94%81%E4%BA%A7%E7%94%9F%E5%8E%9F%E5%9B%A0%E4%B8%8E%E8%AF%8A%E6%96%AD/image5.png
