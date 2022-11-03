# Java Multithreading

Multithreading is a Java feature that allows concurrent execution of two or more parts of a program for maximum utilization of CPU. Each part of such program is called a thread. This document describes basic concepts of working with threads in Java.

## Plan
1. [Threads](#threads)
2. [Executors](#executors)
3. [Synchronize and Lock](#locks)
4. [wait(), notify() and notifyAll()](#wait-notify)
5. [Blocking queues](#queues)
6. [Condition](#condition)
7. [CountDownLatch](#countdownlatch)
8. [CyclicBarrier](#cyclicbarrier)
9. [Semaphore](#semaphore)
10. [Exchanger](#exchanger)

<h2 id="threads">Threads</h2>

A Thread is a very light-weight process, or we can say the smallest part of the process. It can be used to implement some tasks in parallel.

All the programs in Java works in at least one thread. For example, when the `main()` method is called, the thread called "main" is started.

Lets review the example of program that uses threads:
```java
public class ThreadsUsageExample {
	public static void main(String[] args) {
		System.out.println("Starting the program");
		Thread myCustomThread = new Thread(() -> {
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				throw new RuntimeException(e);
			}
			System.out.println("Hello from custom thread");
		});
		System.out.println("Starting myCustomThread");
		myCustomThread.start();
		System.out.println("Waiting for myCustomThread to finish");
		try {
			myCustomThread.join();
		} catch (InterruptedException e) {
			System.out.println("myCustomThread was interrupted!");
		}
		System.out.println("End of the program");
	}
}
/* Output
Starting the program
Starting myCustomThread
Waiting for myCustomThread to finish
Hello from custom thread
End of the program
*/
```

To create a custom thread you need to create new instance of `Thread` class and pass the `Runnable` interface to it. Now use the `start()` method to execute the thread in parallel.
```java
Thread myCustomThread = new Thread(...);
myCustomThread.start();
```

The `sleep()` method is used to make the thread wait for some time.

The `join()` method is used to force the main thread to wait until `myCustomThread` is going to finish it's execution.

`InterruptedException` is an exception that is thrown when the developer interrupts the thread using `interrupt()` method.

Java Thread has 4 main states:
1. New - The thread is newly created.
2. Runnable - The thread is running.
3. Blocked - The thread is waiting for other thread to take action.
4. Dead - The thread is terminated.

<h2 id="executors">Executors</h2>

You can use `ExecutorService` interface to work with threads in more convenient way. It allows you to create a single thread or a pool of threads. To start the thread using `ExecutorService` you need to call `execute()` method and pass the `Runnable` to it.

Standard java library provides the developer with following executors:
1. `Executors.newSingleThreadExecutor()` - creates a pool of one thread.
2. `Executors.newCachedThreadPool()` - creates a pool of threads that can expand the more tasks you are providing to executor.
3. `Executors.newFixedThreadPool(numberOfThreadsInPool)` - creates a pool of fixed size.

In case the task cannot be executed because there are no available threads in the pool, the task will be stacked in a queue and executed lately (when the threads will free).

Also, you can define a custom thread pool by creating a new `ThreadPoolExecutor` instance and configuring it.

Following example illustrates the usage of "CachedThreadPool":
```java
public class ExecutorsUsageExample {
	public static void main(String[] args) throws InterruptedException {
		System.out.println("Starting the program");

		ExecutorService executor = Executors.newCachedThreadPool();
		executor.execute(createTask(500, "Task 1"));
		executor.execute(createTask(1000, "Task 2"));
		executor.execute(createTask(1500, "Task 3"));

		Thread.sleep(4000);
		executor.shutdownNow();
		System.out.println("End of the program");
	}

	private static Runnable createTask(long delay, String taskName) {
		return () -> {
			System.out.println(taskName + " is running in " + Thread.currentThread().getName());
			try {
				Thread.sleep(delay);
				System.out.println("Finished executing " + taskName);
			} catch (InterruptedException e) {
				System.out.println(taskName + " thread was interrupted");
			}
		};
	}
}
/* Output
Starting the program
Task 1 is running in pool-1-thread-1
Task 2 is running in pool-1-thread-2
Task 3 is running in pool-1-thread-3
Finished executing Task 1
Finished executing Task 2
Finished executing Task 3
End of the program
*/
```

The threads in cached thread pools are creating dynamically if needed.

You can use `shutdownNow()` method to stop all threads in a pool (this method calls `interrupt()` for all threads).

Also, the usage of executor allows you to perform a task that returns a response. In order to create such task, you need to implement a `Callable` interface and provide it to the `submit()` method. This method returns a `Future` object, that can be used to get response from the thread using `get()` method. If the thread is still executing, `get()` method will pause the current thread until the response will be formed.

Here is the example:

```java
public class ExecutorsWithResponseUsageExample {

	public static void main(String[] args) {
		System.out.println("Starting the program");
		ExecutorService executor = Executors.newFixedThreadPool(3);
		List<Future<Integer>> futures = new LinkedList<>();
		for (int i = 0; i < 8; i++) {
			Callable<Integer> taskWithResponse = createTaskWithResponse();
			futures.add(executor.submit(taskWithResponse));
		}
		List<Integer> responses = new LinkedList<>();
		for (Future<Integer> future : futures) {
			try {
				Integer response = future.get();
				responses.add(response);
			} catch (InterruptedException e) {
				System.out.println("Thread was interrupted");
			} catch (ExecutionException e) {
				System.out.println("Exception occurred inside the thread");
			}
		}
		System.out.println(responses);
		System.out.println("End of the program");
	}

	private static Callable<Integer> createTaskWithResponse() {
		return () -> {
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			return new Random().nextInt(100);
		};
	}
}
/* Output
Starting the program
[81, 22, 78, 48, 26, 43, 93, 9]
End of the program
*/
```

<h2 id="locks">Synchronize and Lock</h2>

When the threads are performing some tasks, it is important to allow the access to some resource only to one thread at time. Otherwise, the one thread can interfere into the work of other thread and cause unpredictable behavior. Java provides mechanisms to block the part of the code so that only one Thread can access some particular resource at time.

Lets review the following example:

```java
public class Example {

	public static void main(String[] args) {
		Utils utils = new Utils();
		Thread t1 = new Thread(createTask(utils, 10));
		Thread t2 = new Thread(createTask(utils, 100));
		t1.start();
		t2.start();
	}

	private static Runnable createTask(Utils utils, int startValue) {
		return () -> utils.utilsMethod(startValue);
	}

	static class Utils {

		public Utils() {}

		public void utilsMethod(int n) {
			for (int i = 1; i < 6; i++) {
				try {
					Thread.sleep(500);
					System.out.printf("%d ", n * i);
				} catch (InterruptedException e) {
					throw new RuntimeException(e);
				}
			}
		}
	}
}

/* Output 
100 10 20 200 30 300 40 400 50 500
*/
```

As you can see, the threads are using the same method of the same object. As a result, threads are changing the same value simultaneously, that can produce unexpected behavior in more complex situations. There are two ways to fix it: using `synchronized` or by using `Lock` object. We will review both approaches:

Using `synchronized`:

There are two ways we can rewrite the `utilsMethod` to
limit the number of threads that can execute it at time:
```java
public class Utils {

	public Utils() {
	}

	public synchronized void utilsMethod(int n) {
		for (int i = 1; i < 11; i++) {
			try {
				Thread.sleep(500);
				System.out.println(n * i);
			} catch (InterruptedException e) {
				throw new RuntimeException(e);
			}
		}
	}
}
```

or

```java
public class Utils {

	public Utils() {
	}

	public void utilsMethod(int n) {
		synchronized(this) {
			for (int i = 1; i < 11; i++) {
				try {
					Thread.sleep(500);
					System.out.println(n * i);
				} catch (InterruptedException e) {
					throw new RuntimeException(e);
				}
			}
		}
	}
}
```

`synchronized` prevents other threads to access this part of code. So the output will differ.

```java
/* Output
10 20 30 40 50 100 200 300 400 500 
*/
```

Using `Lock`:

We need to change main method and the Utils class a bit
```java
public class SynchronizedWithLock {
	public static void main(String[] args) {
		Lock lock = new ReentrantLock();
		Utils utils = new Utils(lock);
		Thread t1 = new Thread(createTask(utils, 10));
		Thread t2 = new Thread(createTask(utils, 100));
		t1.start();
		t2.start();
	}

	private static Runnable createTask(Utils utils, int startValue) {
		return () -> utils.utilsMethod(startValue);
	}

	static class Utils {

		private final Lock lock;

		public Utils(Lock lock) {
			this.lock = lock;
		}

		public void utilsMethod(int n) {
			lock.lock();
			try {
				for (int i = 1; i < 6; i++) {
					try {
						Thread.sleep(500);
						System.out.print((n * i) + " ");
					} catch (InterruptedException e) {
						throw new RuntimeException(e);
					}
				}
			} finally {
				lock.unlock();
			}
		}
	}
}

/* output
10 20 30 40 50 100 200 300 400 500 
*/
```

The part of code, that is surrounded by `lock()` and `unlock()` methods, defines the code that can be executed only by one thread at time. Remember to use `try-finally` when working in `Lock`. Otherwise, there is a risk your method will be permanently blocked if an exception will occur.

In most cases, it is recommended to use `synchronized` over `Lock`, as it is more secure and is improves the code readability. But `Lock` allows more flexibility and can be used in more complex situations.

<h2 id="wait-notify">wait(), notify() and notifyAll()</h2>

In some cases you'd like to implement one task using multiple threads. For example, you are modelling the process of creating a tea. The steps of creating a tea are:
1. Pick up a cup
2. Add some sugar
3. Add a tea-bag
4. Pour some boiled water
5. Mix all

We will create a separate thread for each procedure.

Java allows to stop and resume the execution of some thread using `wait()`, `notify()`, and `notifyAll()` methods. `wait()` is used to stop thread execution until `notify()` or `notifyAll()` methods are called. These methods are placed in the `Object` class, so all objects in Java contain them. `notifyAll()` should be used, when there are more `wait()` calls in the class.

So now, lets review the example of using these methods to make some tea:
```java
public class TeaProcess {
	public static void main(String[] args) throws InterruptedException {
		ExecutorService executor = Executors.newFixedThreadPool(5);

		Tea tea = new Tea();
		List<Future<?>> tasks = new LinkedList<>();
		tasks.add(executor.submit(createTask("Mix", tea::waitUntilHasWater, tea::mix)));
		tasks.add(executor.submit(createTask("Pour water", tea::waitUntilHasTeaBag, tea::pourWater)));
		tasks.add(executor.submit(createTask("Add tea bag", tea::waitUntilHasSugar, tea::addTeaBag)));
		tasks.add(executor.submit(createTask("Add sugar", tea::waitUntilHasCup, tea::addSugar)));
		tasks.add(executor.submit(createTask("Take a cup", () -> {
		}, tea::takeACup)));

		while (true) {
			if (tasks.stream().allMatch(Future::isDone)) {
				System.out.println("Tea is ready: " + tea.isReady());
				executor.shutdown();
				break;
			}
		}
	}

	private static Runnable createTask(String message, Runnable waitMethod, Runnable taskMethod) {
		return () -> {
			try {
				Thread.sleep(1000);
				waitMethod.run();
				System.out.println(message);
				taskMethod.run();
			} catch (InterruptedException e) {
				throw new RuntimeException(e);
			}
		};
	}

	static class Tea {
		private enum Step {NONE, CUP, SUGAR, TEA_BAG, WATER, MIXED}
		private Step currentStep = Step.NONE;

		public Tea() {
		}

		public synchronized void takeACup() {
			currentStep = Step.CUP;
			notifyAll();
		}

		public synchronized void addSugar() {
			currentStep = Step.SUGAR;
			notifyAll();
		}

		public synchronized void addTeaBag() {
			currentStep = Step.TEA_BAG;
			notifyAll();
		}

		public synchronized void pourWater() {
			currentStep = Step.WATER;
			notifyAll();
		}

		public synchronized void mix() {
			currentStep = Step.MIXED;
		}

		public synchronized void waitUntilHasCup() {
			try {
				while (currentStep != Step.CUP) wait();
			} catch (InterruptedException e) {
				System.out.println("Wait unit has cup was interrupted");
			}
		}

		public synchronized void waitUntilHasSugar() {
			try {
				while (currentStep != Step.SUGAR) wait();
			} catch (InterruptedException e) {
				System.out.println("Wait unit has sugar was interrupted");
			}
		}

		public synchronized void waitUntilHasTeaBag() {
			try {
				while (currentStep != Step.TEA_BAG) wait();
			} catch (InterruptedException e) {
				System.out.println("Wait unit has tea bag was interrupted");
			}
		}

		public synchronized void waitUntilHasWater() {
			try {
				while (currentStep != Step.WATER) wait();
			} catch (InterruptedException e) {
				System.out.println("Wait unit has water was interrupted");
			}
		}

		public synchronized boolean isReady() {
			return currentStep == Step.MIXED;
		}
	}
}
/* Output
Take a cup
Add sugar
Add tea bag
Pour water
Mix
Tea is ready: true
*/
```

It is important to add `wait()` into the while loop, as `notifyAll()` resumes all the threads, event the ones we where not intended to and we need to stop them again.

In case we are using `notify()` method there is a risk that we will resume the thread we are not interested in, so it is more secure to use `notifyAll()`.

Remember to call `wait()`, `notify()`, and `notifyAll()` inside the synchronized block.

<h2 id="queues">Blocking queue</h2>

In some cases it can be easier to share data between the threads in form of a queue. You can think about this as of the microservice architecture, where the microservices implements some specific tasks, and the queue is used to send messages between them.
In java you can use `BlockingQueue` interface to archive this kind of effect. It has two main implementations: `ArrayBlockingQueue` and `LinkedBlockingQueue`. When you try to get the value from this queue using `take()` method and the queue is empty, the thread will be blocked until the element won't be added to the queue.
Lets review the example of toaster automaton program, that makes a toasts and feed it to the costumer:

```java
public class Toaster {

	public static void main(String[] args) {
		ExecutorService executor = Executors.newFixedThreadPool(3);
		BlockingQueue<Toast> queueToButter = new LinkedBlockingQueue<>();
		BlockingQueue<Toast> queueToJam = new LinkedBlockingQueue<>();
		BlockingQueue<Toast> queueToEater = new LinkedBlockingQueue<>();

		executor.execute(butterTask(queueToButter, queueToJam));
		executor.execute(jamTask(queueToJam, queueToEater));
		executor.execute(eaterTask(queueToEater));

		for (int i = 0; i < 10; i++) {
			try {
				queueToButter.put(new Toast(i));
			} catch (InterruptedException e) {
				System.out.println("Interrupted to insert toast.");
			}
		}

		try {
			Thread.sleep(3000);
			executor.shutdownNow();
		} catch (InterruptedException e) {
			throw new RuntimeException(e);
		}

	}

	private static Runnable butterTask(BlockingQueue<Toast> inQueue, BlockingQueue<Toast> outQueue) {
		return () -> {
			try {
				while (!Thread.interrupted()) {
					Toast toast = inQueue.take();
					toast.butter();
					outQueue.put(toast);
				}
			} catch (InterruptedException e) {
				System.out.println("Butter task is interrupted");
			}
		};
	}

	public static Runnable jamTask(BlockingQueue<Toast> inQueue, BlockingQueue<Toast> outQueue) {
		return () -> {
			try {
				while (!Thread.interrupted()) {
					Toast toast = inQueue.take();
					toast.jam();
					outQueue.put(toast);
				}
			} catch (InterruptedException e) {
				System.out.println("Jam task is interrupted");
			}
		};
	}

	public static Runnable eaterTask(BlockingQueue<Toast> toastQueue) {
		return () -> {
			try {
				while (!Thread.interrupted()) {
					Toast toast = toastQueue.take();
					System.out.printf("Toast %s is going to be eaten\n", toast.getId());
				}
			} catch (InterruptedException e) {
				System.out.println("Eater task is interrupted");
			}
		};
	}
}

class Toast {

	public enum Status {DRY, BUTTERED, JAMMED}

	private final Integer id;

	private Status status = Status.DRY;

	Toast(Integer id) {
		this.id = id;
	}

	public void butter() {
		status = Status.BUTTERED;
	}

	public void jam() {
		status = Status.JAMMED;
	}

	public Integer getId() {
		return id;
	}

	public Status getStatus() {
		return status;
	}
}
/* Output
Toast 0 is going to be eaten
Toast 1 is going to be eaten
Toast 2 is going to be eaten
Toast 3 is going to be eaten
Toast 4 is going to be eaten
Toast 5 is going to be eaten
Toast 6 is going to be eaten
Toast 7 is going to be eaten
Toast 8 is going to be eaten
Toast 9 is going to be eaten
Eater task is interrupted
Butter task is interrupted
Jam task is interrupted
*/
```

As you can see, there is no explicit synchronization in the code. The `BlockingQueue` handles it itself. 

<h2 id="condition">Condition</h3>

`Condition` is a class that contains `await()`, `signal()` and `signalAll()` methods. They are used as `wait()`, `notify()`, and `notifyAll()` methods from `Object` class respectively. In order to create an instance of this class you need to call `newCondition()` method of a `Lock` object. Lets rewrite `TeaProcess` example to work with `Condition` instead:


```java
public class TeaProcessWithCondition {

	public static void main(String[] args) {
		ExecutorService executor = Executors.newFixedThreadPool(5);

		Tea tea = new Tea();
		List<Future<?>> tasks = new LinkedList<>();
		tasks.add(executor.submit(createTask("Mix", tea::waitUntilHasWater, tea::mix)));
		tasks.add(executor.submit(createTask("Pour water", tea::waitUntilHasTeaBag, tea::pourWater)));
		tasks.add(executor.submit(createTask("Add tea bag", tea::waitUntilHasSugar, tea::addTeaBag)));
		tasks.add(executor.submit(createTask("Add sugar", tea::waitUntilHasCup, tea::addSugar)));
		tasks.add(executor.submit(createTask("Take a cup", () -> {}, tea::takeACup)));

		while (true) {
			if (tasks.stream().allMatch(Future::isDone)) {
				System.out.println("Tea is ready: " + tea.isReady());
				executor.shutdown();
				break;
			}
		}
	}

	private static Runnable createTask(String message, Runnable waitMethod, Runnable taskMethod) {
		return () -> {
			try {
				Thread.sleep(1000);
				waitMethod.run();
				System.out.println(message);
				taskMethod.run();
			} catch (InterruptedException e) {
				throw new RuntimeException(e);
			}
		};
	}

	static class Tea {

		private enum Step {NONE, CUP, SUGAR, TEA_BAG, WATER, MIXED}
		private Step currentStep = Step.NONE;

		private static final Lock lock = new ReentrantLock();
		private static final Condition cupIsReady = lock.newCondition();
		private static final Condition sugarIsReady = lock.newCondition();
		private static final Condition teaBagIsReady = lock.newCondition();

		private static final Condition waterIsReady = lock.newCondition();

		public Tea() {
		}

		public void takeACup() {
			lock.lock();
			try {
				currentStep = Step.CUP;
				cupIsReady.signalAll();
			} finally {
				lock.unlock();
			}
		}

		public void addSugar() {
			lock.lock();
			try {
				currentStep = Step.SUGAR;
				sugarIsReady.signalAll();
			} finally {
				lock.unlock();
			}
		}

		public void addTeaBag() {
			lock.lock();
			try {
				currentStep = Step.TEA_BAG;
				teaBagIsReady.signalAll();
			} finally {
				lock.unlock();
			}
		}

		public void pourWater() {
			lock.lock();
			try {
				currentStep = Step.WATER;
				waterIsReady.signalAll();

			} finally {
				lock.unlock();
			}
		}

		public void mix() {
			lock.lock();
			currentStep = Step.MIXED;
			lock.unlock();
		}

		public void waitUntilHasCup() {
			lock.lock();
			try {
				cupIsReady.await();
			} catch (InterruptedException e) {
				System.out.println("Wait unit has cup was interrupted");
			} finally {
				lock.unlock();
			}
		}

		public void waitUntilHasSugar() {
			lock.lock();
			try {
				sugarIsReady.await();
			} catch (InterruptedException e) {
				System.out.println("Wait unit has sugar was interrupted");
			} finally {
				lock.unlock();
			}
		}

		public void waitUntilHasTeaBag() {
			lock.lock();
			try {
				teaBagIsReady.await();
			} catch (InterruptedException e) {
				System.out.println("Wait unit has tea bag was interrupted");
			} finally {
				lock.unlock();
			}
		}

		public void waitUntilHasWater() {
			lock.lock();
			try {
				waterIsReady.await();
			} catch (InterruptedException e) {
				System.out.println("Wait unit has water was interrupted");
			} finally {
				lock.unlock();
			}
		}

		public boolean isReady() {
			lock.lock();
			try {
				return currentStep == Step.MIXED;
			} finally {
				lock.unlock();
			}
		}
	}
}
/* Output
Take a cup
Add sugar
Add tea bag
Pour water
Mix
Tea is ready: true
*/
```
By using `Condition` you can assign `await()` method calls to some specific condition that makes it easier to develop more complex multithreading applications. The drawback of using `Condition` is that you are forced to use `Lock` class instead of `synchronized`.

<h2 id="countdownlatch">CountDownLatch</h2>

`CountDownLatch` object can be used to stop a thread until the countdown reaches zero.

Lets review the code example:
```java
public class CountDownLatchExample {

	private static final Integer NUMBER_OF_THREADS_TO_REQUIRE = 5;

	public static void main(String[] args) {
		CountDownLatch countDownLatch = new CountDownLatch(NUMBER_OF_THREADS_TO_REQUIRE);
		ExecutorService executor = Executors.newFixedThreadPool(3);
		Future<?> lastTask = executor.submit(createTaskToDoAfterCountDown(countDownLatch));
		for (int i = 0; i < NUMBER_OF_THREADS_TO_REQUIRE; i++) {
			executor.execute(createTask(countDownLatch, i));
		}

		while (!lastTask.isDone()) {
			executor.shutdown();
		}
	}

	private static Runnable createTaskToDoAfterCountDown(CountDownLatch countDownLatch) {
		return () -> {
			try {
				countDownLatch.await();
				System.out.println("All tasks are completed");
			} catch (InterruptedException e) {
				throw new RuntimeException(e);
			}
		};
	}

	private static Runnable createTask(CountDownLatch countDownLatch, Integer id) {
		return () -> {
			try {
				Thread.sleep(1000);
				System.out.println("Task " + id + " has finished");
				countDownLatch.countDown();
			} catch (InterruptedException e) {
				throw new RuntimeException(e);
			}
		};
	}
}
/* Output
Task 1 has finished
Task 0 has finished
Task 3 has finished
Task 2 has finished
Task 4 has finished
All tasks are completed
*/
```

Here `CountDownLatch` is used to wait util the the specific number of tasks will be finished. `countDown()` method reduces the countdown by one. To reset the countdown you need to create new instance of the class.

<h2 id="cyclicbarrier">CyclicBarrier</h2>

`CyclicBarrier` is very similar to `CountDownLatch` but can be used multiple times. The constructor is very simple. It takes the number of threads that must call `await()` to resume the threads.
```java
public CyclicBarrier(int parties)
```
Optionally, you can pass the second argument to the constructor, which is a `Runnable` instance. This way, runnable will be executed when the last thread will call `await()` method.

Lets see the example:
```java
public class CyclicBarrierExample {

	private static final Integer NUMBER_OF_THREADS_TO_REQUIRE = 5;

	public static void main(String[] args) {
		CyclicBarrier cyclicBarrier = new CyclicBarrier(NUMBER_OF_THREADS_TO_REQUIRE, () -> {
			System.out.println("Time to unblock threads");
		});
		ExecutorService executor = Executors.newCachedThreadPool();
		List<Future<?>> tasks = new LinkedList<>();
		for (int i = 0; i < NUMBER_OF_THREADS_TO_REQUIRE; i++) {
			tasks.add(executor.submit(createTask(cyclicBarrier, i)));
		}
		while (true) {
			if (tasks.stream().allMatch(Future::isDone)) {
				executor.shutdown();
				break;
			}
		}
	}

	private static Runnable createTask(CyclicBarrier cyclicBarrier, Integer id) {
		return () -> {
			try {
				Thread.sleep(1000);
				System.out.println("Task " + id + " is on hold");
				cyclicBarrier.await();
				System.out.println("Task " + id + " finished");
			} catch (InterruptedException | BrokenBarrierException e) {
				System.out.println("Oops");
			}
		};
	}

}
/* Output
Task 0 is on hold
Task 4 is on hold
Task 1 is on hold
Task 2 is on hold
Task 3 is on hold
Time to unblock threads
Task 3 finished
Task 0 finished
Task 1 finished
Task 2 finished
Task 4 finished
*/
```

<h2 id="semaphore">Semaphore</h2>

When `Lock` and `synchronized` allows access to resource only to one thread at time, `Semaphore` can be used to do it for multiple threads.

Lets illustrate the work of `Semaphore` by developing a simple bicycle renting program.
```java
public class SemaphoreExample {

	private static final Integer NUMBER_OF_BICYCLES = 3;

	public static void main(String[] args) {
		Semaphore semaphore = new Semaphore(NUMBER_OF_BICYCLES);
		ExecutorService executor = Executors.newCachedThreadPool();
		List<Runnable> rents = Arrays.asList(
				createBicycleRent(semaphore, "Aragorn"),
				createBicycleRent(semaphore, "Legolas"),
				createBicycleRent(semaphore, "Gimli"),
				createBicycleRent(semaphore, "Boromir"),
				createBicycleRent(semaphore, "Frodo"),
				createBicycleRent(semaphore, "Sam"),
				createBicycleRent(semaphore, "Pippin"),
				createBicycleRent(semaphore, "Merry"),
				createBicycleRent(semaphore, "Gandalf")
		);
		List<Future<?>> finishedRents = rents.stream().map(executor::submit).collect(Collectors.toList());
		while (true) {
			if (finishedRents.stream().allMatch(Future::isDone)) {
				executor.shutdown();
				break;
			}
		}
	}

	private static Runnable createBicycleRent(Semaphore semaphore, String renterName) {
		return () -> {
			try {
				System.out.printf("%s waits for bicycle\n", renterName);
				semaphore.acquire();
				System.out.printf("%s acquired a bicycle\n", renterName);
				Thread.sleep(1000);
				System.out.printf("%s returns a bicycle\n", renterName);
			} catch (InterruptedException e) {
				throw new RuntimeException(e);
			} finally {
				semaphore.release();
			}
		};
	}
}
/* output
Gimli waits for bicycle
Gandalf waits for bicycle
Gandalf acquired a bicycle
Merry waits for bicycle
Pippin waits for bicycle
Legolas waits for bicycle
Boromir waits for bicycle
Sam waits for bicycle
Aragorn waits for bicycle
Frodo waits for bicycle
Merry acquired a bicycle
Gimli acquired a bicycle
Gandalf returns a bicycle
Merry returns a bicycle
Gimli returns a bicycle
Pippin acquired a bicycle
Boromir acquired a bicycle
Legolas acquired a bicycle
Boromir returns a bicycle
Legolas returns a bicycle
Aragorn acquired a bicycle
Pippin returns a bicycle
Sam acquired a bicycle
Frodo acquired a bicycle
Aragorn returns a bicycle
Sam returns a bicycle
Frodo returns a bicycle
*/
```

We can use `Semaphore` to control how many bicycles can be ranted at time. In the example above this number equals to 3.

<h2 id="exchanger">Exchanger</h2>

`Exchanger` is used to swap two objects between the threads. Most of all, it is used when the creating of an object takes many time, and we decided to run it in separate thread to speed up the calculation.

Lets see the example:

```java
public class ExchangerExample {

	public static void main(String[] args) {
		Exchanger<List<Integer>> exchanger = new Exchanger<>();
		ExecutorService executor = Executors.newSingleThreadExecutor();
		executor.execute(createRandomNumberListTask(exchanger, 5));

		for (int i = 0; i < 6; i++) {
			try {
				List<Integer> randomNumbers = exchanger.exchange(Collections.emptyList());
				System.out.println("List " + i + ": " + randomNumbers);
			} catch (InterruptedException e) {
				throw new RuntimeException(e);
			}
		}
		executor.shutdownNow();
	}

	private static Runnable createRandomNumberListTask(Exchanger<List<Integer>> exchanger, Integer n) {
		Random random = new Random();
		List<Integer> list = new LinkedList<>();
		return () -> {
			try {
				while (!Thread.interrupted()) {
					for (int i = 0; i < n; i++) {
						list.add(random.nextInt(99 - 10) + 10);
					}
					exchanger.exchange(new ArrayList<>(list));
					list.clear();
				}
			} catch (InterruptedException e) {
				System.out.println("Random list creator is interrupted");
			}
		};
	}
}
/* Output
List 0: [87, 98, 95, 40, 15]
List 1: [15, 30, 21, 59, 10]
List 2: [76, 66, 60, 98, 48]
List 3: [95, 64, 60, 40, 71]
List 4: [20, 30, 22, 65, 78]
List 5: [51, 56, 83, 66, 58]
Random list creator is interrupted
*/
```

The list of random integers is created in separate thread, so the main thread performs the required task faster.