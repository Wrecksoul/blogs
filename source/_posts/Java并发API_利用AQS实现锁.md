---
title: java并发个人总结——使用AQS实现简单锁
categories:
- java基础
tags:
- 并发
- J.U.C
---

理解j.u.c的突破口就是理解AQS，然而理解AQS必须从具体实现类入手，比如ReetrantLock，否则无从下手。但是ReetrantLock又有点复杂，所以不如自行实现了一个能多简单就多简单的类，依据呢，就是[java并发个人总结——locks API](https://wrecksoul.github.io/blogs/2018/01/09/Java%E5%B9%B6%E5%8F%91API_locks/) 中AQS实现类的一些说明以及仿照ReetrantLock的源码。以后再从它入手一点点看AQS的源码。

## 实现一个简单锁
废话少说，上代码：
```java
/**
* 非公平不可重入锁
*/
public class MyLock extends AbstractQueuedSynchronizer implements Lock {

	private static final long serialVersionUID = -6691612847559022610L;

	/*
	 * 按照API的要求，重新AQS的tryAcquire方法，我们使用的是非公平的模式
	 * 
	 * @see
	 * java.util.concurrent.locks.AbstractQueuedSynchronizer#tryAcquire(int)
	 */
	@Override
	protected boolean tryAcquire(int arg) {
		if (compareAndSetState(0, 1)) {
			setExclusiveOwnerThread(Thread.currentThread());
			return true;
		}
		return false;
	}

	/*
	 * 按照API的要求，重新AQS的tryRelease方法
	 * 
	 * @see
	 * java.util.concurrent.locks.AbstractQueuedSynchronizer#tryRelease(int)
	 */
	@Override
	protected boolean tryRelease(int arg) {
		setExclusiveOwnerThread(null);
		setState(0);
		return true;
	}
	/*
	 * 只有需要用到Condition的时候才需要实现它
	 * @see java.util.concurrent.locks.AbstractQueuedSynchronizer#isHeldExclusively()
	 */
	@Override
	protected boolean isHeldExclusively() {
		return getExclusiveOwnerThread() == Thread.currentThread();
	}

	/**
	 * 以下方法来自接口Lock
	 */

	@Override
	public void lock() {
		acquire(1);
	}

	@Override
	public void lockInterruptibly() throws InterruptedException {
		acquireInterruptibly(1);
	}

	@Override
	public boolean tryLock() {
		if (compareAndSetState(0, 1)) {
			setExclusiveOwnerThread(Thread.currentThread());
			return true;
		}
		return false;
	}

	@Override
	public boolean tryLock(long time, TimeUnit unit)
			throws InterruptedException {
		return tryAcquireNanos(1, unit.toNanos(time));
	}

	@Override
	public void unlock() {
		release(1);
	}

	@Override
	public Condition newCondition() {
		return new ConditionObject();
	}
}
```
MyLock是我参考ReetrantLock并结合AQS的API，然后删除了很多代码之后得到的，以后还会用到这个类，逐步完善它，我的最终目标是把它变成ReetrantLock的可重入非公平锁。

这个类和ReetrantLock有些不同——MyLock继承了AQS，而ReetrantLock没有。这是因为ReetrantLock为了实现两种锁，使用了设计模式，我的MyLock类相当于一个暴露在外的、阉割版的ReetrantLock内部类NonFairSync，只实现了一种锁——非公平锁。

## 测试我的锁

测试一下就知道能不能用了：
```java
/**
* 启动1000个线程，每个线程将r变量执行10次+1操作，最终结果应该是10000
*/
class Task implements Runnable{
	private int r = 0;
	private Lock  lock= new MyLock();
	
	public int getR(){
		return r;
	}
	@Override
	public void run() {
		try{
			lock.lock();
			for (int i = 0; i < 10; i++) {
				r ++;
			}
		} finally {
			lock.unlock();
		}
	}
	
}
```
```
public static void main(String[] args) throws InterruptedException {
	Task task = new Task();
	ExecutorService pool = Executors.newFixedThreadPool(4);
	for (int i = 0; i < 100; i++) {
		pool.execute(task);
	}
	pool.shutdown();
	TimeUnit.SECONDS.sleep(3);//等待各个线程完成任务，这个方式虽然有点low但还是管用的
	System.out.println(task.getR());
}
```

测试结果：
> 10000

在做这个测试的时候我发现一个问题，在并发量不是很高的时候，即使不加锁也总是可以得到正确的结果，比如线程数100的时候，在我的4核i5cpu的机器上总是可以得到正确结果1000。

这里需要正视的一个问题就是：
<div class="note danger"><p>线程安全，是保证在多线程情况下一定可以得到正确结果。
线程不安全，不能保证多线程情况下得到正确结果。注意，仅仅是**不保证**，并不是一定得到错误结果。在并发量比较小的情况下很可能得到的结果是正确的，但是在高并发下存在难以排查甚至后果严重的重大隐患。</p></div>

### 阶段终版

由于API中不是这样推荐的，它希望我们以内部类的形式使用AQS的子类，所以我们改动一下：

```
public class MyLock implements Lock{

	private final Sync sync = new Sync();

	@Override
	public void lock() {
		sync.acquire(1);
	}

	@Override
	public void lockInterruptibly() throws InterruptedException {
		sync.acquireInterruptibly(1);
	}

	@Override
	public boolean tryLock() {
		return sync.tryAcquire(1);
	}

	@Override
	public boolean tryLock(long time, TimeUnit unit)
			throws InterruptedException {
		return sync.tryAcquireNanos(1, unit.toNanos(time));
	}

	@Override
	public void unlock() {
		sync.release(1);
	}

	@Override
	public Condition newCondition() {
		return sync.newCondition();
	}
	
	private static class Sync extends AbstractQueuedSynchronizer{
		private static final long serialVersionUID = -7157758664440239362L;

		/*
		 * 按照API的要求，重新AQS的tryAcquire方法，我们使用的是非公平的模式
		 * 
		 * @see
		 * java.util.concurrent.locks.AbstractQueuedSynchronizer#tryAcquire(int)
		 */
		@Override
		protected final boolean tryAcquire(int arg) {
			if (compareAndSetState(0, 1)) {
				setExclusiveOwnerThread(Thread.currentThread());
				return true;
			}
			return false;
		}

		/*
		 * 按照API的要求，重新AQS的tryRelease方法
		 * 
		 * @see
		 * java.util.concurrent.locks.AbstractQueuedSynchronizer#tryRelease(int)
		 */
		@Override
		protected final boolean tryRelease(int arg) {
			setExclusiveOwnerThread(null);
			setState(0);
			return true;
		}
		/*
		 * 只有需要用到Condition的时候才需要实现它
		 * @see java.util.concurrent.locks.AbstractQueuedSynchronizer#isHeldExclusively()
		 */
		@Override
		protected final boolean isHeldExclusively() {
			return getExclusiveOwnerThread() == Thread.currentThread();
		}
		
		final ConditionObject newCondition() {
            return new ConditionObject();
        }
	}
	
}
```

1. 把主要的逻辑写在一个内部类 `Sync` 中，并且把 `Sync` 的方法都声明成final的防止被重写。
2. 然后MyLock实现Lock接口，利用内部类完成工作。

## 结论

在j.u.c.locks包中AQS以及几个接口的指导下，我只写了几十行代码顺利完成了一个简单的锁，这是AQS给我们提供的遍历。

MyLock类完全可以被称为锁，它继承了AQS的线程排队机制并且支持Condition协调线程，而且它还实现了Lock接口，但是它不支持重入，有死锁风险。