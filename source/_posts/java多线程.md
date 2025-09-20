---
title: java多线程
date: 2019-01-21 17:04:50
tags: 
- java多线程
categories:
- java
---


##  Java多线程简单使用

### 继承Thread类
通过继承Thread，重写run方法

	public class MyThread extends Thread{

	    @Override
	    public void run() {
	        super.run();
	        System.out.println("MyThread");
	    }
	}
测试代码：
	
	@Test
    public void test1(){
        MyThread myThread = new MyThread();
        myThread.start();
        System.out.println("运行结束");

    }
> 因为java是单根继承，不支持多继承，在程序设计上是有局限的。

### 实现Runnable接口（推荐）

实现Runnale接口，重写run方法

	public class MyRunnable implements Runnable {

	    @Override
	    public void run() {
	        System.out.println("running");
	    }
	}
测试代码：

	@Test
    public void test2(){
        MyRunnable myRunnable = new MyRunnable();
        Thread thread = new Thread(myRunnable);
        thread.start();
        System.out.println("ending");

    }

> * 注意，这里测试代码是实现Runnable接口后，通过Thread来接受Runnable接口对象，这样可以让Thread的start方法来调用Run方法，起到多线程的作用。

> *  如果直接在主程序中调用run方法，就仅仅是程序的调用，无法起到多线程的作用。


---
## java多线程的同步

### synchronize实现同步

java中的每一个对象都可以作为锁，具体表现形式为下面三种形式：

>* 对于普通同步方法，锁是当前实例对象
>* 对于静态同步方法，锁是当前类的class对象
>* 对于同步方法块，锁是Synchronize括号里配置的对象

上面三种形式默认synchronized相当于synchronized（this）锁this对象。

对于synchronized(非this对象x)这种情况，有下属三种结论：

>* 当多个线程同时执行synchronized（x）{}同步代码块呈现同步效果。
>* 当其它线程执行x对象中synchronized同步方法呈现同步效果
>* 当其它线程执行x对象方法里面的synchronized（this）代码块也呈现同步效果。

### volatile关键字
关键字volatile主要作用是使变量在多个线程间可见

----
## Java线程间通信

### 等待/通知机制

![]()

等待/通知机制，是指一个线程A调用了对象O的wait()方法进入等待状态，而另一个线程B
调用了对象O的notify()或者notifyAll()方法，线程A收到通知后从对象O的wait()方法返回，进而
执行后续操作。上述两个线程通过对象O来完成交互，而对象上的wait()和notify/notifyAll()的
关系就如同开关信号一样，用来完成等待方和通知方之间的交互工作。

	public class WaitNotify {
	    static boolean flag = true;
	    static Object  lock = new Object();
	
	    public static void main(String[] args) throws Exception {
	        Thread waitThread = new Thread(new Wait(), "WaitThread");
	        waitThread.start();
	        TimeUnit.SECONDS.sleep(1);
	
	        Thread notifyThread = new Thread(new Notify(), "NotifyThread");
	        notifyThread.start();
	    }
	
	    static class Wait implements Runnable {
	        public void run() {
	            // 加锁，拥有lock的Monitor
	            synchronized (lock) {
	                // 当条件不满足时，继续wait，同时释放了lock的锁
	                while (flag) {
	                    try {
	                        System.out.println(Thread.currentThread() + " flag is true. wait @ "
	                                           + new SimpleDateFormat("HH:mm:ss").format(new Date()));
	                        lock.wait();
	                    } catch (InterruptedException e) {
	                    }
	                }
	                // 条件满足时，完成工作
	                System.out.println(Thread.currentThread() + " flag is false. running @ "
	                                   + new SimpleDateFormat("HH:mm:ss").format(new Date()));
	            }
	        }
	    }
	
	    static class Notify implements Runnable {
	        public void run() {
	            // 加锁，拥有lock的Monitor
	            synchronized (lock) {
	                // 获取lock的锁，然后进行通知，通知时不会释放lock的锁，
	                // 直到当前线程释放了lock后，WaitThread才能从wait方法中返回
	                System.out.println(Thread.currentThread() + " hold lock. notify @ " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
	                lock.notifyAll();
	                flag = false;
	                SleepUtils.second(5);
	            }
	            // 再次加锁
	            synchronized (lock) {
	                System.out.println(Thread.currentThread() + " hold lock again. sleep @ "
	                                   + new SimpleDateFormat("HH:mm:ss").format(new Date()));
	                SleepUtils.second(5);
	            }
	        }
	    }
	}
我们在使用这些方法需要注意下面几点：

* 1）使用wait()、notify()和notifyAll()时需要先对调用对象加锁。

		即这些方法必须位于synchronize同步块内。
* 2）调用wait()方法后，线程状态由RUNNING变为WAITING，并将当前线程放置到对象的
等待队列。
* 3）notify()或notifyAll()方法调用后，等待线程依旧不会从wait()返回，需要调用notify()或
notifAll()的线程释放锁之后，等待线程才有机会从wait()返回。
	
		即会将notify()或者notifAll()所在的同步块执行完才会调转到wait()所在的同步块。
* 4）notify()方法将等待队列中的一个等待线程从等待队列中移到同步队列中，而notifyAll()
方法则是将等待队列中所有的线程全部移到同步队列，被移动的线程状态由WAITING变为
BLOCKED。
* 5）从wait()方法返回的前提是获得了调用对象的锁。


### 管道输入/输出流
管道流主要用于线程之间的数据传输，传输的媒介是内存。java中主要包括了4种具体的实现：PipedOutputStream、PipedInputStream、PipedReader和PipedWriter，前两种面向字节，而后两种面向字符。

	public class Piped {

	    public static void main(String[] args) throws Exception {
	        PipedWriter out = new PipedWriter();
	        PipedReader in = new PipedReader();
	        // 将输出流和输入流进行连接，否则在使用时会抛出IOException
	        out.connect(in);
	
	        Thread printThread = new Thread(new Print(in), "PrintThread");
	        printThread.start();
	        int receive = 0;
	        try {
	            while ((receive = System.in.read()) != -1) {
	                out.write(receive);
	            }
	        } finally {
	            out.close();
	        }
	    }
	
	    static class Print implements Runnable {
	        private PipedReader in;
	
	        public Print(PipedReader in) {
	            this.in = in;
	        }
	
	        public void run() {
	            int receive = 0;
	            try {
	                while ((receive = in.read()) != -1) {
	                    System.out.print((char) receive);
	                }
	            } catch (IOException ex) {
	            }
	        }
	    }
	}
### join方法
* thread.join()当前线程等待thread线程终止后才继续执行。
* thread.join(long millis)具备超时特性，在上者情况下，如果在指定时间内进程没终止，将从超时方法返回。
* thread.join(long millis,int nanos) 

		public class Join {
		    public static void main(String[] args) throws Exception {
		        Thread previous = Thread.currentThread();
		        for (int i = 0; i < 10; i++) {
		            // 每个线程拥有前一个线程的引用，需要等待前一个线程终止，才能从等待中返回
		            Thread thread = new Thread(new Domino(previous), String.valueOf(i));
		            thread.start();
		            previous = thread;
		        }
		
		        TimeUnit.SECONDS.sleep(5);
		        System.out.println(Thread.currentThread().getName() + " terminate.");
		    }
		
		    static class Domino implements Runnable {
		        private Thread thread;
		
		        public Domino(Thread thread) {
		            this.thread = thread;
		        }
		
		        public void run() {
		            try {
		                thread.join();
		            } catch (InterruptedException e) {
		            }
		            System.out.println(Thread.currentThread().getName() + " terminate.");
		        }
		    }
		}

结果如下：
	
	main terminate.
	0 terminate.
	1 terminate.
	2 terminate.
	3 terminate.
	4 terminate.
	5 terminate.
	6 terminate.
	7 terminate.
	8 terminate.
	9 terminate.

### ThreadLocal的使用

ThreadLocal，即线程变量，是一个以ThreadLocal对象为键、任意对象为值的存储结构。这
个结构被附带在线程上，也就是说一个线程可以根据一个ThreadLocal对象查询到绑定在这个
线程上的一个值。

可以通过set(T)方法来设置一个值，在当前线程下再通过get()方法获取到原先设置的值。

	public class Profiler {
	    // 第一次get()方法调用时会进行初始化（如果set方法没有调用），每个线程会调用一次
	    private static final ThreadLocal<Long> TIME_THREADLOCAL = new ThreadLocal<Long>() {
	         protected Long initialValue() {
	            return System.currentTimeMillis();}
	    };
	
	    public static final void begin() {
	        TIME_THREADLOCAL.set(System.currentTimeMillis());
	    }
	
	    public static final long end() {
	        return System.currentTimeMillis() - TIME_THREADLOCAL.get();
	    }
	
	    public static void main(String[] args) throws Exception {
	        Profiler.begin();
	        TimeUnit.SECONDS.sleep(1);
	        System.out.println("Cost: " + Profiler.end() + " mills");
	    }
	}

---
## java多线程基础

### 线程优先级
	
在Java线程中，通过一个整型成员变量priority来控制优先级，优先级的范围从1~10，在线
程构建的时候可以通过setPriority(int)方法来修改优先级，默认优先级是5，优先级高的线程分
配时间片的数量要多于优先级低的线程。设置线程优先级时，针对频繁阻塞（休眠或者I/O操
作）的线程需要设置较高优先级，而偏重计算（需要较多CPU时间或者偏运算）的线程则设置较
低的优先级，确保处理器不会被独占。在不同的JVM以及操作系统上，线程规划会存在差异，
有些操作系统甚至会忽略对线程优先级的设定。
	
	setPriority(int)
### 线程的状态
java线程在运行的生命周期可能处于下表的6中不同的状态，在给定的一个时刻，
线程只能处于其中的一个状态。

![](https://raw.githubusercontent.com/BARKTEGH/MarkDownPhotos/master/multiThread/TIM%E6%88%AA%E5%9B%BE20181122170533.png)

![](https://raw.githubusercontent.com/BARKTEGH/MarkDownPhotos/master/multiThread/TIM%E6%88%AA%E5%9B%BE20181122170707.png)

### 守护线程Daemon

Daemon线程是一种支持型线程，因为它主要被用作程序中后台调度以及支持性工作。这
意味着，当一个Java虚拟机中不存在非Daemon线程的时候，Java虚拟机将会退出。可以通过调
用Thread.setDaemon(true)将线程设置为Daemon线程。
Daemon线程被用作完成支持性工作，但是在Java虚拟机退出时Daemon线程中的finally块
并不一定会执行。

* *Daemon属性需要在启动线程之前设置，不能在启动线程之后设置*
* *在构建Daemon线程时，不能依靠finally块中的内容来确保执行关闭或清理资源的逻辑。*

### 线程中断
使用interrupt()方法来停止线程，但interrupt()方法仅仅是在当前线程中刚打了一个断点。


java提供两种方法判断线程是否中断：
> * this.interrupted()测试当前进程是否已经中断，执行后将具有状态标志清除为flase功能
> * this.isInterrupted()测试线程是否已经中断，不清除状态标志

推荐使用异常法来停止线程：

	public class MyThreadInterrupt extends Thread {

	    @Override
	    public void run() {
	        super.run();
	        try{
	            for(int i=0;i<500000;i++){
	                if(this.isInterrupted()){
	                    System.out.println("线程终止，即将退出");
	                    throw new InterruptedException();
	                }
	                System.out.println("i="+(i+1));
	            }
	            System.out.println("for循环完了，继续执行程序。。。");
	        }catch (InterruptedException e){
	            System.out.println("进入catch捕捉异常");
	            e.printStackTrace();
	        }
	    }
	}
测试代码：
	
	@Test
    public void test3(){
        try{
            MyThreadInterrupt myThreadInterrupt = new MyThreadInterrupt();
            myThreadInterrupt.start();
            Thread.sleep(2000);
            myThreadInterrupt.interrupt();
        } catch (InterruptedException e){
            System.out.println("mian catch");
            e.printStackTrace();
        }

    }

**如果不使用抛异常来终止线程，那么线程即使中断还是会for循环继续运行**

或者使用标志位来控制是否需要停止任务并终止线程

	public class Shutdown {
	    public static void main(String[] args) throws Exception {
	        Runner one = new Runner();
	        Thread countThread = new Thread(one, "CountThread");
	        countThread.start();
	        // 睡眠1秒，main线程对CountThread进行中断，使CountThread能够感知中断而结束
	        TimeUnit.SECONDS.sleep(1);
	        countThread.interrupt();
	        Runner two = new Runner();
	        countThread = new Thread(two, "CountThread");
	        countThread.start();
	        // 睡眠1秒，main线程对Runner two进行取消，使CountThread能够感知on为false而结束
	        TimeUnit.SECONDS.sleep(1);
	        two.cancel();
	    }
	
	    private static class Runner implements Runnable {
	        private long             i;
	
	        private volatile boolean on = true;
	
	        @Override
	        public void run() {
	            while (on && !Thread.currentThread().isInterrupted()) {
	                i++;
	            }
	            System.out.println("Count i = " + i);
	        }
	
	        public void cancel() {
	            on = false;
	        }
	    }
	}

> 注意： suspend(),resume(),stop()已过期


    
> 全文来自 java并发编程的艺术 和 java多线程编程核心技术