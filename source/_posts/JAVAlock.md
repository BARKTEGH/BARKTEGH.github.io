---
title: Lock
date: 2019-05-18 19:06:14
tags: 
- java多线程
- 队列同步器
- lock
categories:
- java
- lock
---




----
### LOCK接口

锁是用来控制多个线程访问共享资源的方式，一般来说，一个锁能够防止多个线程同时
访问共享资源（但是有些锁可以允许多个线程并发的访问共享资源，比如读写锁）。在Lock接
口出现之前，Java程序是靠synchronized关键字实现锁功能的，而Java SE 5之后，并发包中新增
了Lock接口（以及相关实现类）用来实现锁功能，它提供了与synchronized关键字类似的同步功
能，只是在使用时需要显式地获取和释放锁。虽然它缺少了（通过synchronized块或者方法所提
供的）隐式获取释放锁的便捷性，但是却拥有了锁获取与释放的可操作性、可中断的获取锁以
及超时获取锁等多种synchronized关键字所不具备的同步特性。


![](https://raw.githubusercontent.com/BARKTEGH/MarkDownPhotos/master/multiThread/TIM%E6%88%AA%E5%9B%BE20181123104801.png)

Lock接口有三个实现类，一个是ReentrantLock,另两个是ReentrantReadWriteLock类中的两个静态内部类ReadLock和WriteLock。

* 与互斥锁定相比，读-写锁定允许对共享数据进行更高级别的并发访问。虽然一次只有一个线程（writer 线程）可以修改共享数据，但在许多情况下，任何数量的线程可以同时读取共享数据（reader 线程）。从理论上讲，与互斥锁定相比，使用读-写锁定所允许的并发性增强将带来更大的性能提高。
* 在实践中，只有在多处理器上并且只在访问模式适用于共享数据时，才能完全实现并发性增强。——例如，某个最初用数据填充并且之后不经常对其进行
修改的 collection，因为经常对其进行搜索（比如搜索某种目录），所以这样的 collection 是使用读-写锁定的理想候选者。

---
###  队列同步器

队列同步器AbstractQueuedSynchronizer（以下简称同步器），是用来构建锁或者其他同步组
件的基础框架，它使用了一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获
取线程的排队工作。

同步器是实现锁（也可以是任意同步组件）的关键，在锁的实现中聚合同步器，利用同步
器实现锁的语义。可以这样理解二者之间的关系：**锁是面向使用者的，它定义了使用者与锁交
互的接口（比如可以允许两个线程并行访问），隐藏了实现细节；同步器面向的是锁的实现者，
它简化了锁的实现方式，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。**

同步器的主要使用方式是继承，子类通过继承同步器并实现它的抽象方法来管理同步状
态，在抽象方法的实现过程中免不了要对同步状态进行更改，这时就需要使用同步器提供的3
个方法（getState()、setState(int newState)和compareAndSetState(int expect,int update)）来进行操作，因为它们能够保证状态的改变是安全的。子类推荐被定义为自定义同步组件的静态内部
类，同步器自身没有实现任何同步接口，它仅仅是定义了若干同步状态获取和释放的方法来
供自定义同步组件使用，同步器既可以支持独占式地获取同步状态，也可以支持共享式地获
取同步状态，这样就可以方便实现不同类型的同步组件（ReentrantLock、
ReentrantReadWriteLock和CountDownLatch等）。

#### 实现一个lock--通过队列同步器

同步器AQS的设计是基于模板方法模式的，也就是说，**使用者需要继承同步器并重写指定的
方法，随后将同步器组合在自定义同步组件的实现中，并调用同步器提供的模板方法，而这些
模板方法将会调用使用者重写的方法。**


![](https://raw.githubusercontent.com/BARKTEGH/MarkDownPhotos/master/multiThread/TIM%E6%88%AA%E5%9B%BE20181123105140.png)

![](https://raw.githubusercontent.com/BARKTEGH/MarkDownPhotos/master/multiThread/TIM%E6%88%AA%E5%9B%BE20181123105200.png)

同步器提供的模板方法基本上分为3类：独占式获取与释放同步状态、共享式获取与释放
同步状态和查询同步队列中的等待线程情况。自定义同步组件将使用同步器提供的模板方法
来实现自己的同步语义。

下面为自定义的一个lock锁：

	import java.util.concurrent.TimeUnit;
	import java.util.concurrent.locks.AbstractQueuedSynchronizer;
	import java.util.concurrent.locks.Condition;
	import java.util.concurrent.locks.Lock;
	
	public class Mutex implements Lock {
	    // 静态内部类，自定义同步器，随后模板方法会调用这些方法
	    private static class Sync extends AbstractQueuedSynchronizer {
	        private static final long serialVersionUID = -4387327721959839431L;
	
	        // 是否处于占用状态
	        protected boolean isHeldExclusively() {
	            return getState() == 1;
	        }
	
	        // 当状态为0的时候获取锁
	        public boolean tryAcquire(int acquires) {
	            assert acquires == 1; // Otherwise unused
	            if (compareAndSetState(0, 1)) {
	                setExclusiveOwnerThread(Thread.currentThread());
	                return true;
	            }
	            return false;
	        }
	
	        // 释放锁，将状态设置为0
	        protected boolean tryRelease(int releases) {
	            assert releases == 1; // Otherwise unused
	            if (getState() == 0)
	                throw new IllegalMonitorStateException();
	            setExclusiveOwnerThread(null);
	            setState(0);
	            return true;
	        }
	
	        // 返回一个Condition，每个condition都包含了一个condition队列
	        Condition newCondition() {
	            return new ConditionObject();
	        }
	    }
	
	    // 仅需要将操作代理到Sync上即可
	    private final Sync sync = new Sync();
	
	    public void lock() {
	        sync.acquire(1);
	    }
	
	    public boolean tryLock() {
	        return sync.tryAcquire(1);
	    }
	
	    public void unlock() {
	        sync.release(1);
	    }
	
	    public Condition newCondition() {
	        return sync.newCondition();
	    }
	
	    public boolean isLocked() {
	        return sync.isHeldExclusively();
	    }
	
	    public boolean hasQueuedThreads() {
	        return sync.hasQueuedThreads();
	    }
	
	    public void lockInterruptibly() throws InterruptedException {
	        sync.acquireInterruptibly(1);
	    }
	
	    public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
	        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
	    }
	}

### 重入锁（ReentrantLock）
重入锁就是该锁支持一个线程对资源的重复可利用。除此之外，该锁还支持获取锁的公平与非公平选择。

ReentrantLock中有3个内部类，分别是Sync、FairSync和NonfairSync。

Sync是一个继承AQS的抽象类，使用独占锁，复写了tryRelease方法。tryAcquire方法由它的两个FairSync(公平锁)和NonfairSync(非公平锁)实现。

ReentrantLock的lock方法使用sync的lock方法，Sync的lock方法是个抽象方法，由公平锁和非公平锁去实现。unlock方法直接使用AQS的release方法。所以说公平锁和非公平锁的释放锁过程是一样的，不一样的是获取锁过程。

先看一下公平锁nonfair的lock方法：

	final void lock() {
    // acquire方法内部调用tryAcquire方法
    // 公平锁的获取锁方法，对于没有获取到的线程，会按照队列的方式挂起线程
    acquire(1);
	}
	
	protected final boolean tryAcquire(int acquires) {
	    final Thread current = Thread.currentThread();
	    int c = getState();
	    if (c == 0) {
	        // 公平锁这里多了一个!hasQueuedPredecessors()判断，表示是否有线程在队列里等待的时间比当前线程要长，如果有等待时间更长的线程，那么放弃获取锁
	        if (!hasQueuedPredecessors() &&
	            compareAndSetState(0, acquires)) {
	            setExclusiveOwnerThread(current);
	            return true;
	        }
	    }
	    else if (current == getExclusiveOwnerThread()) {
	        int nextc = c + acquires;
	        if (nextc < 0)
	            throw new Error("Maximum lock count exceeded");
	        setState(nextc);
	        return true;
	    }
	    return false;
	}
该方法与nonfairTryAcquire(int acquires)比较，唯一不同的位置为判断条件多了hasQueuedPredecessors()方法，即加入了同步队列中当前节点是否有前驱节点的判断，如果该
方法返回true，则表示有线程比当前线程更早地请求获取锁，因此需要等待前驱线程获取并释
放锁之后才能继续获取锁。

非公平锁lock方法：
	
	final void lock() {
	    // 非公平锁的获取锁
	    // 跟公平锁的区别就在这里。直接对状态位state进行cas操作，成功就获取锁，这是一种抢占式的方式。不成功跟公平锁一样进入队列挂起线程
	    if (compareAndSetState(0, 1))
	        setExclusiveOwnerThread(Thread.currentThread());
	    else
	        acquire(1);
	}
	// 调用Sync的nonfairTryAcquire方法
	protected final boolean tryAcquire(int acquires) {
	    return nonfairTryAcquire(acquires);
	}

	final boolean nonfairTryAcquire(int acquires) {
	    final Thread current = Thread.currentThread();
	    int c = getState();
	    if (c == 0) {
	        if (compareAndSetState(0, acquires)) {
	            setExclusiveOwnerThread(current);
	            return true;
	        }
	    }
	    else if (current == getExclusiveOwnerThread()) {
	        int nextc = c + acquires;
	        if (nextc < 0) // overflow
	            throw new Error("Maximum lock count exceeded");
	        setState(nextc);
	        return true;
	    }
	    return false;
	}

该方法增加了再次获取同步状态的处理逻辑：通过判断当前线程是否为获取锁的线程来
决定获取操作是否成功，如果是获取锁的线程再次请求，则将同步状态值进行增加并返回
true，表示获取同步状态成功。

对于unlock方法直接使用AQS的release方法。公平锁和非公平锁的释放锁过程是一样的，不一样的是获取锁过程。
	protected final boolean tryRelease(int releases) {
	    int c = getState() - releases; // 释放
	    if (Thread.currentThread() != getExclusiveOwnerThread()) // 如果当前线程不是独占线程，直接抛出异常
	        throw new IllegalMonitorStateException();
	    boolean free = false;
	    if (c == 0) { // 由于是可重入锁，需要判断是否全部释放了
	        free = true;
	        setExclusiveOwnerThread(null); // 全部释放的话直接把独占线程设置为null
	    }
	    setState(c);
	    return free;
	}
	
	// 恢复线程
	public final boolean release(int arg) {
	    if (tryRelease(arg)) {
	        Node h = head;  // 恢复第一个挂起的线程
	        if (h != null && h.waitStatus != 0)
	            unparkSuccessor(h);
	        return true;
	    }
	    return false;
	}

ReentrantLock的默认构造函数使用的是NonfairSync，如果想使用FairSync，使用带有boolean参数的构造函数，传入true表示FairSync，否则是NonfairSync。

###  读写锁 ReentrantReadWriteLock
读写锁在同一时刻可以允许多个读线程访问，但是在写线程访问时，所有的读
线程和其他写线程均被阻塞。读写锁维护了一对锁，一个读锁和一个写锁，通过分离读锁和写
锁，使得并发性相比一般的排他锁有了很大提升。

* 使用范例
	
		public class Cache {
			static Map<String, Object> map = new HashMap<String, Object>();
			static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
			static Lock r = rwl.readLock();
			static Lock w = rwl.writeLock();
			// 获取一个key对应的value
			public static final Object get(String key) {
				r.lock();
				try {
					return map.get(key);
				} finally {
					r.unlock();
				}
			}
			// 设置key对应的value，并返回旧的value
			public static final Object put(String key, Object value) {
				w.lock();
				try {
					return map.put(key, value);
				} finally {
					w.unlock();
				}
				}// 清空所有的内容
				public static final void clear() {
					w.lock();
					try {
						map.clear();
					} finally {
						w.unlock();
					}
			}
		}	
* 读写锁实现分析

> * 1 - 读写状态的设计

读写锁同样依赖自定义同步器来实现同步功能，而读写状态就是其同步器的同步状态。回想ReentrantLock中自定义同步器的实现，同步状态表示锁被一个线程重复获取的次数，而读写锁的自定义同步器需要在同步状态（一个整型变量）上维护多个读线程和一个写线程的状态，使得该状态的设计成为读写锁实现的关键。
如果在一个整型变量上维护多种状态，就一定需要“按位切割使用”这个变量，读写锁将变量切分成了两个部分，高16位表示读，低16位表示写。
> * 2 - 写锁的获取与释放

ReentrantReadWriteLock的tryAcquire方法

	protected final boolean tryAcquire(int acquires) {
		Thread current = Thread.currentThread();
		int c = getState();
		int w = exclusiveCount(c);
		if (c != 0) {
			// 存在读锁或者当前获取线程不是已经获取写锁的线程
			if (w == 0 || current != getExclusiveOwnerThread())
			return false;
			if (w + exclusiveCount(acquires) > MAX_COUNT)
			throw new Error("Maximum lock count exceeded");
			setState(c + acquires);
			return true;
		}
		if (writerShouldBlock() || !compareAndSetState(c, c + acquires)) {
			return false;
		}
		setExclusiveOwnerThread(current);
		return true;
	}
写锁的释放与ReentrantLock的释放过程基本类似，每次释放均减少写状态，当写状态为0
时表示写锁已被释放，从而等待的读写线程能够继续访问读写锁，同时前次写线程的修改对
后续读写线程可见。
> * 3- 读锁的获取的释放

ReentrantReadWriteLock的tryAcquireShared方法

	protected final int tryAcquireShared(int unused) {
		for (;;) {
			int c = getState();
			int nextc = c + (1 << 16);
			if (nextc < c)
				throw new Error("Maximum lock count exceeded");
			if (exclusiveCount(c) != 0 && owner != Thread.currentThread())
				return -1;
			if (compareAndSetState(c, nextc))
				return 1;
		}
	}

在tryAcquireShared(int unused)方法中，如果其他线程已经获取了写锁，则当前线程获取读
锁失败，进入等待状态。如果当前线程获取了写锁或者写锁未被获取，则当前线程（线程安全，
依靠CAS保证）增加读状态，成功获取读锁。
读锁的每次释放（线程安全的，可能有多个读线程同时释放读锁）均减少读状态，减少的
值是（1<<16）

###  Condition接口

![](https://raw.githubusercontent.com/BARKTEGH/MarkDownPhotos/master/multiThread/TIM%E6%88%AA%E5%9B%BE20181203164721.png)
简单使用：

	public class BoundedQueue<T> {
		private Object[] items;
		// 添加的下标，删除的下标和数组当前
		private int addIndex, removeI
		private Lock lock = new R
		private Condition notEmpty
		private Condition notFull
		public BoundedQueue(int size)
			items = new Object[si
		}
		// 添加一个元素，如果数组满，则添加线
		public void add(T t) throws I
			lock.lock();
			try {
				while (count == items.length)
				notFull.await();
				items[addIndex] = t;
				if (++addIndex == items.length)
				addIndex = 0;
				++count;
				notEmpty.signal();
			} finally {
				ock.unlock();
			}
		}
		// 由头部删除一个元素，如果数组空，则删除线程进入等待状态，直到有新添加
		@SuppressWarnings("unchecked")
		public T remove() throws InterruptedException {
			lock.lock();
			try {
				while (count == 0)
				notEmpty.await();
				Object x = items[removeIndex];
				if (++removeIndex == items.length)
				removeIndex = 0;
				--count;
				notFull.signal();
				return (T) x;
			} finally {
				lock.unlock();
			}
		}
	}

> 全文来自 java并发编程的艺术