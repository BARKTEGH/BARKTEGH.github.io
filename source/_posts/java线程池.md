---
title: Java线程池
date: 2019-05-20 19:08:16
tags: 
- java多线程
- 线程池
categories:
- java

---



---
###  线程池的基本使用
java中实现线程池的类为java.uitl.concurrent.ThreadPoolExecutor继承了AbstractExecutorService类。
AbstractExecutorService类基本方法：

![](https://raw.githubusercontent.com/BARKTEGH/MarkDownPhotos/master/multiThread/AbstrascexecuteService.png)

AbstractExecutorService是一个抽象类，它实现了ExecutorService接口。
ExecutorService接口基本方法：

![](https://raw.githubusercontent.com/BARKTEGH/MarkDownPhotos/master/multiThread/executeService.png)

ExecutorService又是继承了Executor接口。我们看一下Executor接口的实现：
	
	public interface Executor {
	    void execute(Runnable command);
	}
![](https://raw.githubusercontent.com/BARKTEGH/MarkDownPhotos/master/multiThread/ThreadPoolExecutor.png)

---
#### 线程池的创建
ThreadPoolExecutor的四个构造函数：

	public class ThreadPoolExecutor extends AbstractExecutorService {
	    .....
	    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
	            BlockingQueue<Runnable> workQueue);
	 
	    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
	            BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory);
	 
	    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
	            BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);
	 
	    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
	        BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler);
	    ...
	}
每个参数的含义：

 * **corePoolSize**（线程池的基本大小）：当提交一个任务到线程池时，线程池会创建一个线
程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程，等到需要执行的任
务数大于线程池基本大小时就不再创建。如果调用了线程池的prestartAllCoreThreads()方法，
线程池会提前创建并启动所有基本线程。

* **runnableTaskQueue（任务队列）**：用于保存等待执行的任务的阻塞队列。可以选择以下几
个阻塞队列。
	* ArrayBlockingQueue：是一个基于数组结构的有界阻塞队列，此队列按FIFO（先进先出）原
则对元素进行排序。
	* LinkedBlockingQueue：一个基于链表结构的阻塞队列，此队列按FIFO排序元素，吞吐量通
常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队列。
	* SynchronousQueue：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用
移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于Linked-BlockingQueue，静态工
厂方法Executors.newCachedThreadPool使用了这个队列。
	* PriorityBlockingQueue：一个具有优先级的无限阻塞队列。

* **maximumPoolSize（线程池最大数量）：**线程池允许创建的最大线程数。如果队列满了，并
且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。值得注意的是，如
果使用了无界的任务队列这个参数就没什么效果。

* **ThreadFactory**：用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设
置更有意义的名字。使用开源框架guava提供的ThreadFactoryBuilder可以快速给线程池里的线
程设置有意义的名字，代码如下。

		new ThreadFactoryBuilder().setNameFormat("XX-task-%d").build();
* **RejectedExecutionHandler（饱和策略）**：当队列和线程池都满了，说明线程池处于饱和状
态，那么必须采取一种策略处理提交的新任务。这个策略默认情况下是AbortPolicy，表示无法
处理新任务时抛出异常。在JDK 1.5中Java线程池框架提供了以下4种策略。
	* AbortPolicy：直接抛出异常。
	* CallerRunsPolicy：只用调用者所在线程来运行任务
	* DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务。
	* DiscardPolicy：不处理，丢弃掉。
	
* **keepAliveTime（线程活动保持时间）**：线程池的工作线程空闲后，保持存活的时间。所以，
如果任务很多，并且每个任务执行的时间比较短，可以调大时间，提高线程的利用率。

* **TimeUnit（线程活动保持时间的单位）**：可选的单位有天（DAYS）、小时（HOURS）、分钟
（MINUTES）、毫秒（MILLISECONDS）、微秒（MICROSECONDS，千分之一毫秒）和纳秒（NANOSECONDS，千分之一微秒）。

在java doc中，并不提倡我们直接使用ThreadPoolExecutor，而是使用Executors类中提供的几个静态方法来创建线程池：
>* Executors.newCachedThreadPool();        //创建一个缓冲池，缓冲池容量大小Integer.MAX_VALUE
>
>* Executors.newSingleThreadExecutor();   //创建容量为1的缓冲池
> 
>* Executors.newFixedThreadPool(int);    //创建固定容量大小的缓冲池

从它们的具体实现来看，它们实际上也是调用了ThreadPoolExecutor，只不过参数都已配置好了。

	//newFixedThreadPool创建的线程池corePoolSize和maximumPoolSize值是相等的，它使用的LinkedBlockingQueue；
	public static ExecutorService newFixedThreadPool(int nThreads) {
	    return new ThreadPoolExecutor(nThreads, nThreads,
	                                  0L, TimeUnit.MILLISECONDS,
	                                  new LinkedBlockingQueue<Runnable>());
	}
	//newSingleThreadExecutor将corePoolSize和maximumPoolSize都设置为1，也使用的LinkedBlockingQueue；
	public static ExecutorService newSingleThreadExecutor() {
	    return new FinalizableDelegatedExecutorService
	        (new ThreadPoolExecutor(1, 1,
	                                0L, TimeUnit.MILLISECONDS,
	                                new LinkedBlockingQueue<Runnable>()));
	}
	//newCachedThreadPool将corePoolSize设置为0，将maximumPoolSize设置为Integer.MAX_VALUE，使用的SynchronousQueue，也就是说来了任务就创建线程运行，当线程空闲超过60秒，就销毁线程。
	public static ExecutorService newCachedThreadPool() {
	    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
	                                  60L, TimeUnit.SECONDS,
	                                  new SynchronousQueue<Runnable>());
	}

---
#### 线程的初始化、提交任务、容量的动态调整和关闭线程池
默认情况下，创建线程池之后，线程池中是没有线程的，需要提交任务之后才会创建线程。
在实际中如果需要线程池创建之后立即创建线程，可以通过以下两个方法办到：

* prestartCoreThread()：初始化一个核心线程；
* prestartAllCoreThreads()：初始化所有核心线程

向线程池提交任务有两个方法：execute和submit。

* execute()方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功。execute()方法实际上是Executor中声明的方法，在ThreadPoolExecutor进行了具体的实现。

* submit()方法用于提交需要返回值的任务。线程池会返回一个future类型的对象，通过这个
future对象可以判断任务是否执行成功，并且可以通过future的get()方法来获取返回值，get()方
法会阻塞当前线程直到任务完成，而使用get（long timeout，TimeUnit unit）方法则会阻塞当前线
程一段时间后立即返回，这时候有可能任务没有执行完。submit()方法是在ExecutorService中声明的方法，在AbstractExecutorService就已经有了具体的实现，在ThreadPoolExecutor中并没有对其进行重写。

ThreadPoolExecutor提供了动态调整线程池容量大小的方法：

* setCorePoolSize()设置核心池大小
* setMaximumPoolSize()，设置线程池最大能创建的线程数目大小


 　当上述参数从小变大时，ThreadPoolExecutor进行线程赋值，还可能立即创建新的线程来执行任务。
关闭线程池有shutdown和shutdownNow两个方法。

简单使用范例：
	
	public class Test {
	     public static void main(String[] args) {   
	         ThreadPoolExecutor executor = new ThreadPoolExecutor(5, 10, 200, TimeUnit.MILLISECONDS,
	                 new ArrayBlockingQueue<Runnable>(5));
	          
	         for(int i=0;i<15;i++){
	             MyTask myTask = new MyTask(i);
	             executor.execute(myTask);
	             System.out.println("线程池中线程数目："+executor.getPoolSize()+"，队列中等待执行的任务数目："+
	             executor.getQueue().size()+"，已执行玩别的任务数目："+executor.getCompletedTaskCount());
	         }
	         executor.shutdown();
	     }
	}
	 
	 
	class MyTask implements Runnable {
	    private int taskNum;
	     
	    public MyTask(int num) {
	        this.taskNum = num;
	    }
	     
	    @Override
	    public void run() {
	        System.out.println("正在执行task "+taskNum);
	        try {
	            Thread.currentThread().sleep(4000);
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        }
	        System.out.println("task "+taskNum+"执行完毕");
	    }
	}

---
#### 线程池原理分析

当提交一个新任务到线程池时，线程池的处理流程如下。

* 1）线程池判断核心线程池里的线程是否都在执行任务。如果不是，则创建一个新的工作
线程来执行任务。如果核心线程池里的线程都在执行任务，则进入下个流程。

* 2）线程池判断工作队列是否已经满。如果工作队列没有满，则将新提交的任务存储在这
个工作队列里。如果工作队列满了，则进入下个流程。
* 3）线程池判断线程池的线程是否都处于工作状态。如果没有，则创建一个新的工作线程
来执行任务。如果已经满了，则交给饱和策略来处理这个任务
![]()

![]()

看一下execute的源码：

	public void execute(Runnable command) {
		if (command == null)
			throw new NullPointerException();
		// 如果线程数小于基本线程数，则进入addIfUnderCorePoolSize(command)，试图创建线程并执行，当正常是add..返回true，结束，无法执行返回false。
		//如果线程数大于基本线程数，进入if或者上面的add返回false
		if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command)) {
			// 如线程数大于等于基本线程数或线程创建失败，则将当前任务放到工作队列中。
			if (runState == RUNNING && workQueue.offer(command)) {
				if (runState != RUNNING || poolSize == 0)
					ensureQueuedTaskHandled(command);
		}
		
		// 如果线程池不处于运行中或任务无法放入队列，并且当前线程数量小于最大允许的线程数量，
		// 则创建一个线程执行任务。
		else if (!addIfUnderMaximumPoolSize(command))
			// 抛出RejectedExecutionException异常
			reject(command); // is shutdown or saturated
		}
	}

	//当线程数低于核心池大小时执行的方法
	private boolean addIfUnderCorePoolSize(Runnable firstTask) {
	    Thread t = null;
	    final ReentrantLock mainLock = this.mainLock;
	    mainLock.lock();
	    try {
	        if (poolSize < corePoolSize && runState == RUNNING)
	            t = addThread(firstTask);        //创建线程去执行firstTask任务   
	        } finally {
	        mainLock.unlock();
	    }
	    if (t == null)
	        return false;
	    t.start();
	    return true;
	}

	//用提交的任务创建了一个Worker对象，然后调用线程工厂threadFactory创建了一个新的线程t，
	//然后将线程t的引用赋值给了Worker对象的成员变量thread，
	//接着通过workers.add(w)将Worker对象添加到工作集当中。
	private Thread addThread(Runnable firstTask) {
	    Worker w = new Worker(firstTask);
	    Thread t = threadFactory.newThread(w);  //创建一个线程，执行任务   
	    if (t != null) {
	        w.thread = t;            //将创建的线程的引用赋值为w的成员变量       
	        workers.add(w);
	        int nt = ++poolSize;     //当前线程数加1       
	        if (nt > largestPoolSize)
	            largestPoolSize = nt;
	    }
	    return t;
	}

	//work类的run方法，
	//首先执行的是通过构造器传进来的任务firstTask，
	//在调用runTask()执行完firstTask之后，在while循环里面不断通过getTask()去取新的任务来执行，那么去哪里取呢？自然是从任务缓存队列里面去取
	public void run() {
		try {
			Runnable task = firstTask;
			firstTask = null;
			while (task != null || (task = getTask()) != null) {
				runTask(task);
				task = null;
			}
		} finally {
			workerDone(this);
		}
	}

	//getTask是ThreadPoolExecutor类中的方法，并不是Worker类中的方法，下面是getTask方法的实现
	//在getTask中，先判断当前线程池状态，如果runState大于SHUTDOWN（即为STOP或者TERMINATED），则直接返回null
	 //如果runState为SHUTDOWN或者RUNNING，则从任务缓存队列取任务。
	
	//如果当前线程池的线程数大于核心池大小corePoolSize或者允许为核心池中的线程设置空闲存活时间，则调用poll(time,timeUnit)来取任务，这个方法会等待一定的时间，如果取不到任务就返回null。

	//然后判断取到的任务r是否为null，为null则通过调用workerCanExit()方法来判断当前worker是否可以退出，
	Runnable getTask() {
	    for (;;) {
	        try {
	            int state = runState;
	            if (state > SHUTDOWN)
	                return null;
	            Runnable r;
	            if (state == SHUTDOWN)  // Help drain queue
	                r = workQueue.poll();
	            else if (poolSize > corePoolSize || allowCoreThreadTimeOut) //如果线程数大于核心池大小或者允许为核心池线程设置空闲时间，
	                //则通过poll取任务，若等待一定的时间取不到任务，则返回null
	                r = workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS);
	            else
	                r = workQueue.take();
	            if (r != null)
	                return r;
	            if (workerCanExit()) {    //如果没取到任务，即r为null，则判断当前的worker是否可以退出
	                if (runState >= SHUTDOWN) // Wake up others
	                    interruptIdleWorkers();   //中断处于空闲状态的worker
	                return null;
	            }
	            // Else retry
	        } catch (InterruptedException ie) {
	            // On interruption, re-check runState
	        }
	    }
	}
	//workerCanExit()的实现
	//如果线程池处于STOP状态、或者任务队列已为空或者允许为核心池线程设置空闲存活时间并且线程数大于1时，允许worker退出。如果允许worker退出，则调用interruptIdleWorkers()中断处于空闲状态的worker

	private boolean workerCanExit() {
	    final ReentrantLock mainLock = this.mainLock;
	    mainLock.lock();
	    boolean canExit;
	    //如果runState大于等于STOP，或者任务缓存队列为空了
	    //或者  允许为核心池线程设置空闲存活时间并且线程池中的线程数目大于1
	    try {
	        canExit = runState >= STOP ||
	            workQueue.isEmpty() ||
	            (allowCoreThreadTimeOut &&
	             poolSize > Math.max(1, corePoolSize));
	    } finally {
	        mainLock.unlock();
	    }
	    return canExit;
	}

	//interruptIdleWorkers()的实现
	//从实现可以看出，它实际上调用的是worker的interruptIfIdle()方法
	void interruptIdleWorkers() {
	    final ReentrantLock mainLock = this.mainLock;
	    mainLock.lock();
	    try {
	        for (Worker w : workers)  //实际上调用的是worker的interruptIfIdle()方法
	            w.interruptIfIdle();
	    } finally {
	        mainLock.unlock();
	    }
	}
	
	//在worker的interruptIfIdle()方法
	void interruptIfIdle() {
	    final ReentrantLock runLock = this.runLock;
	    if (runLock.tryLock()) {    //注意这里，是调用tryLock()来获取锁的，因为如果当前worker正在执行任务，锁已经被获取了，是无法获取到锁的
	                                //如果成功获取了锁，说明当前worker处于空闲状态
	        try {
	    		if (thread != Thread.currentThread())  
	    ·			thread.interrupt();
	        } finally {
	            runLock.unlock();
	        }
	    }
	}
	
	//addIfUnderMaximumPoolSize方法的实现
	//这个方法的实现思想和addIfUnderCorePoolSize方法的实现思想非常相似，唯一的区别在于addIfUnderMaximumPoolSize方法是在线程池中的线程数达到了核心池大小并且往任务队列中添加任务失败的情况下执行的：
	private boolean addIfUnderMaximumPoolSize(Runnable firstTask) {
	    Thread t = null;
	    final ReentrantLock mainLock = this.mainLock;
	    mainLock.lock();
	    try {
	        if (poolSize < maximumPoolSize && runState == RUNNING)
	            t = addThread(firstTask);
	    } finally {
	        mainLock.unlock();
	    }
	    if (t == null)
	        return false;
	    t.start();
	    return true;
	}

		

	
Work类实现：

	private final class Worker implements Runnable {
	    private final ReentrantLock runLock = new ReentrantLock();
	    private Runnable firstTask;
	    volatile long completedTasks;
	    Thread thread;
	    Worker(Runnable firstTask) {
	        this.firstTask = firstTask;
	    }
	    boolean isActive() {
	        return runLock.isLocked();
	    }
	    void interruptIfIdle() {
	        final ReentrantLock runLock = this.runLock;
	        if (runLock.tryLock()) {
	            try {
	        if (thread != Thread.currentThread())
	        thread.interrupt();
	            } finally {
	                runLock.unlock();
	            }
	        }
	    }
	    void interruptNow() {
	        thread.interrupt();
	    }
	 
	    private void runTask(Runnable task) {
	        final ReentrantLock runLock = this.runLock;
	        runLock.lock();
	        try {
	            if (runState < STOP &&
	                Thread.interrupted() &&
	                runState >= STOP)
	            boolean ran = false;
	            beforeExecute(thread, task);   //beforeExecute方法是ThreadPoolExecutor类的一个方法，没有具体实现，用户可以根据
	            //自己需要重载这个方法和后面的afterExecute方法来进行一些统计信息，比如某个任务的执行时间等           
	            try {
	                task.run();
	                ran = true;
	                afterExecute(task, null);
	                ++completedTasks;
	            } catch (RuntimeException ex) {
	                if (!ran)
	                    afterExecute(task, ex);
	                throw ex;
	            }
	        } finally {
	            runLock.unlock();
	        }
	    }
	 
	    public void run() {
	        try {
	            Runnable task = firstTask;
	            firstTask = null;
	            while (task != null || (task = getTask()) != null) {
	                runTask(task);
	                task = null;
	            }
	        } finally {
	            workerDone(this);   //当任务队列中没有任务时，进行清理工作       
	        }
	    }
	}

	
	
> 来源自：http://www.cnblogs.com/dolphin0520/p/3932921.html
> 
> 来源自：java并发编程的艺术