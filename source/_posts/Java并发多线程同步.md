---
title: Java并发多线程同步
date: 2018-03-20 10:43:39
tags: 
- JAVA
---
最近遇到了多线程并发同步问题，找到了`java.util.concurrent`包下的`CountDownLatch`、`CyclicBarrier`、`Semaphore`这三个类。
`CountDownLatch`可以实现类似计数器的功能，例如线程A需要等待B、C、D三个线程执行完成之后才可以执行。
`CyclicBarrier`可以实现让一组(多个)线程等待至某个状态之后再全部同时执行，当所有线程都被释放以后，CyclicBarrier可以被重用。
`Semaphore`可以控制同时访问的线程个数，通过`acquire()`获取一个许可，如果没有就等待，而`release()`释放一个许可。
<!--more-->
#### CountDownLatch
CountDownLatch类只有一个构造方法：
``` java
public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
```
这里的count是一个计数值，表示要等待多少任务，每次调用该对象示例的`countDown()`方法，该值都会减1，当count为0时表示没有需要等待的任务。常用的方法如下：
``` java
public void await() throws InterruptedException { };   //调用await()方法的线程会被挂起，它会等待直到count值为0才继续执行
public boolean await(long timeout, TimeUnit unit) throws InterruptedException { };  //和await()类似，只不过等待一定的时间后count值还没变为0的话就会继续执行
public void countDown() { };  //将count值减1
```
示例如下：
``` java
public static void testCountDownLatch() {
	final CountDownLatch latch = new CountDownLatch(2);
	new Thread("one") {
		public void run() {
			try {
				System.out.println("子线程" + Thread.currentThread().getName() + "正在执行");
				Thread.sleep(3000);
				System.out.println("子线程" + Thread.currentThread().getName() + "执行完毕");
				latch.countDown();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		};
	}.start();
	new Thread("two") {
		public void run() {
			try {
				System.out.println("子线程" + Thread.currentThread().getName() + "正在执行");
				Thread.sleep(3000);
				System.out.println("子线程" + Thread.currentThread().getName() + "执行完毕");
				latch.countDown();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		};
	}.start();

	try {
		System.out.println("等待2个线程执行完成");
		latch.await();
		System.out.println("子线程已经执行完毕");
	} catch (InterruptedException e) {
		e.printStackTrace();
	}
}
```
#### CyclicBarrier
该类有两个构造方法
``` java
 public CyclicBarrier(int parties, Runnable barrierAction)
 public CyclicBarrier(int parties)
```
参数parties是指让多少个线程或者任务等待至barrier状态，参数barrierAction是当这些线程都到达barrier状态后会执行的内容。
该类里面有两个比较重要的方法：
``` java
public int await() throws InterruptedException, BrokenBarrierException { };
public int await(long timeout, TimeUnit unit)throws InterruptedException,BrokenBarrierException,TimeoutException { };
```
无参的方法比较常用，用来挂起当前线程，直到所有线程都达到barrier状态再同时执行后续任务。
有参的方法是让线程等待一定时间，如果线程还没有达到barrier状态，就让到达barrier状态的线程执行后续任务。
示例如下：
``` java

static void testCyclicBarrier() {
//	CyclicBarrier barrier = new CyclicBarrier(5);
	CyclicBarrier barrier = new CyclicBarrier(5,new Runnable() {
		
		@Override
	public void run() {
			System.out.println("所有线程执行完毕，随机挑选一个线程来执行打印");
			System.out.println("挑选的线程为" + Thread.currentThread().getName());
				
		}
	});
	for (int i = 0; i < 5; i++) {
		new Writer(barrier, "thread:" + i).start();
	}

}

static class Writer extends Thread {
	private CyclicBarrier cyclicBarrier;

	public Writer(CyclicBarrier cyclicBarrier, String threadName) {
		this.cyclicBarrier = cyclicBarrier;
		if (threadName != null) {
			this.setName(threadName);
		}

	}

	@Override
	public void run() {

		try {
			System.out.println("线程" + Thread.currentThread().getName() + "正在作业中");
			Thread.sleep(5000);
			System.out.println("线程" + Thread.currentThread().getName() + "作业完成");
			cyclicBarrier.await();
			System.out.println("所有线程作业完毕，线程" + Thread.currentThread().getName() + "继续理其他任务");

		} catch (InterruptedException e) {
			e.printStackTrace();
		} catch (BrokenBarrierException e) {
			
			e.printStackTrace();
		}

	}

}
```
值得注意的是，CyclicBarrier是可以**重用**的。
#### Semaphore
该类提供了两个构造器：
``` java
public Semaphore(int permits) {          //参数permits表示许可数目，即同时可以允许多少线程进行访问
    sync = new NonfairSync(permits);
}
public Semaphore(int permits, boolean fair) {    //这个多了一个参数fair表示是否是公平的，即等待时间越久的越先获取许可
    sync = (fair)? new FairSync(permits) : new NonfairSync(permits);
}
```
下面是该类中比较重要的几个方法，首先是acquire()、release()：
``` java
public void acquire() throws InterruptedException {  }     //获取一个许可
public void acquire(int permits) throws InterruptedException { }    //获取permits个许可
public void release() { }          //释放一个许可
public void release(int permits) { }    //释放permits个许可
```
acquire()用来获取一个许可，若无许可能够获得，则会一直等待，直到获得许可。
release()用来释放许可。注意，在释放许可之前，必须先获获得许可。
这4个方法都会被阻塞，如果想立即得到执行结果，可以使用下面几个方法：
``` java
public boolean tryAcquire() { };    //尝试获取一个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
public boolean tryAcquire(long timeout, TimeUnit unit) throws InterruptedException { };  //尝试获取一个许可，若在指定的时间内获取成功，则立即返回true，否则则立即返回false
public boolean tryAcquire(int permits) { }; //尝试获取permits个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
public boolean tryAcquire(int permits, long timeout, TimeUnit unit) throws InterruptedException { }; //尝试获取permits个许可，若在指定的时间内获取成功，则立即返回true，否则则立即返回false
```
另外还可以通过availablePermits()方法得到可用的许可数目。
假如5个线程要使用3个资源，示例如下：
``` java
static void testSemaphore() {
		int N = 5;            //线程数
        Semaphore semaphore = new Semaphore(3); //资源数目
        for(int i=0;i<N;i++)
            new Worker("线程" +i,semaphore).start();
	}
	
	static class Worker extends Thread {

		private Semaphore semaphore;
		
		public Worker(String name, Semaphore semaphore) {
			super();
			this.setName(name);
			this.semaphore = semaphore;
		}

		@Override
		public void run() {
			try {
				semaphore.acquire();
				System.out.println("线程：" + Thread.currentThread().getName() + "占用一个资源");
				Thread.sleep(3000);
				System.out.println("线程：" + Thread.currentThread().getName() + "释放一个资源");
				semaphore.release();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}

	}
```
CountDownLatch和CyclicBarrier都能够实现线程之间的等待，只不过它们侧重点不同：
CountDownLatch一般用于某个线程等待若干个其他线程执行完任务之后，它才执行；而CyclicBarrier一般用于一组线程互相等待至某个状态，然后这一组线程再同时执行；另外，CountDownLatch是不能够重用的，而CyclicBarrier是可以重用的。Semaphore其实和锁有点类似，它一般用于控制对某组资源的访问权限。

----
以上