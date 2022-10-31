### Plan
Plan:
1. Multithreading in Java
2. Threads
3. Executors
4. Error handling
5. Blocking
    - Synchronize and Lock
    - Volatile and Atomic
    - Wait(), Notify() and NotifyAll()
    - Scheduled Executor
    - Blocking queues
    - Concurrency

# Multithreading in Java

## What is multithreading?

## Threads
A Thread is a very light-weighted process, or we can say the smallest part of the process. It can be used to implement some tasks in parallel.

All the programs in Java works in at least one thread. When the `main()` method is called the thread called "main" is started.

Lets review the example of program that uses threads:
```java
public class Main {
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
```

To create a custom thread you need to create new instance of `Thread` class and pass the `Runnable` interface to it. Now use the start() method to execute the thread in parallel.
```java
Thread myCustomThread = new Thread(...);
myCustomThread.start();
```

The `sleep(1000)` method is used to make the thread to wait for 1000 milliseconds.

The `join()` method is used to force the main thread to wait until `myCustomThread` has finished it's execution.

`InterruptedException` is the exception that is thrown when the developer interrupts the thread or the program is stopped.

In order to stop thread execution the one can use `interrupt()` method. In this case, the `InterruptionException` is going to be thrown.

## Executors
You can use `Executors` to work with thread in more convenient way. It allows you to execute a single thread or to create pools of thread. To start the thread using `Executor` you need to call `execute()` method and pass the Runnable interface in it;

There are several built in executors:
1. `Executors.newSingleThreadExecutor()` - is used to create on thread and run it
2. `Executors.newCachedThreadPool()` - is used to create additional threads as they are required for tasks or to use the existing ones.
3. `Executors.newFixedThreadPool(numberOfThreadsInPool)` - is used to create a fixed-sized pool of threads and run the tasks.

In case the task cannot be executed because there are no available threads in the bool, the task is added to a queue.

Also, you can create a custom thread pools by creating a new `ThreadPoolExecutor` instance and configuring it.

Following example illustrates the usage of "CachedThreadPool":
```java
public class Main {
	public static void main(String[] args) {
		System.out.println("Starting the program");

		ExecutorService executor = Executors.newCachedThreadPool();
		executor.execute(createTask(500, "Task 1"));
		executor.execute(createTask(1000, "Task 2"));
		executor.execute(createTask(1500, "Task 3"));
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
```

The threads in cached thread pool are created dynamically if needed.

You can use `shutdown()` method to stop the execution of the threads in the pool.

Also, the usage of executor allows you to perform the task that returns some response. Here is the example:

```java
public class Main {

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
```

Instead of `Runnable` interface we need to pass `Callable`. In this case you need to use `submit()` method instead of `execute()` to run the thread. `submit()` returns the `Future`, that can be used to get response from the thread. `get()` method blocks the execution of the thread it is called in until the thread is completed.

If there is a risk of infinite loop in the thread or the thread can never finish the execution it can be better to use the overload of `get()` method to pass the timeout `future.get(1, TimeUnit.MINUTES)`. In this case, if the thread is still not done in this time period, the thread will be interrupted.


## Error Handling in threads

You can observe, that it is not possible to catch the exception inside the thread in a usual way.

For example:
```java
public class ErrorHandling {

	public static void main(String[] args) {
		try {
			Executors.newSingleThreadExecutor(new HandlerThreadFactory())
				.execute(ErrorHandling::riskyTask);
		} catch(RuntimeException e) {
			// will not be called
			System.out.println("Exception occurred inside the thread");
		}
		
	}

	public static void riskyTask() {
		System.out.println("Inside riskyTask");
		throw new RuntimeException("Exception inside the thread");
	}
```

This code doesn't work in a way we expect. To catch the `RuntimeException` we need to provide the `Thread` with a `Thread.UncaughtExceptionHandler`.

```java
public class ErrorHandling {

	public static void main(String[] args) {
		Executors.newSingleThreadExecutor(new HandlerThreadFactory())
				.execute(ErrorHandling::riskyTask);
	}

	public static void riskyTask() {
		System.out.println("Inside riskyTask");
		throw new RuntimeException("Exception inside the thread");
	}

	static class UncaughtExceptionHandler implements Thread.UncaughtExceptionHandler {
		@Override
		public void uncaughtException(Thread t, Throwable e) {
			System.out.printf("Exception occurred in thread %s, message [%s]\n", t.getName(), e.getMessage());
		}
	}

	static class HandlerThreadFactory implements ThreadFactory {
		@Override
		public Thread newThread(Runnable r) {
			Thread t = new Thread(r);
			t.setUncaughtExceptionHandler(new UncaughtExceptionHandler());
			return t;
		}
	}
}
```

We created the `UncaughtExceptionHandler` class (but you can use lambda instead) to declare what to do when the exception occurred in the thread. After that we created a `HandlerThreadFactory` and are passing it to `executor`. It is made to force the `executor` to create only the thread with `UncaughtExceptionHandler` set as we needed.

This way, the exception is caught by our handler and processed.

## Blocking

When the threads are performing some tasks, it is important to guarantee some resources used by it to not be changed by other thread, because it can cause the code to behave unexpectedly. Java provides mechanisms of blocking the part of code so that only one Thread can access some particular method or resource at time.

### Synchronize and Lock
Lets review the following example:

```java
public class Utils {

	public Utils() {
	}

	public void utilsMethod(int n) {
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

public class Blocking {

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
}

// Output 
/*
100 10 20 200 30 300 40 400 50 500
*/
```

As you can see, the threads are using the same method of the same object. As a result, the output numbers are in chaotic sequence, that can produce unexpected behavior in more complex examples. There are two ways to fix it: using `synchronized` or by using `Lock` object.

Using synchronized:
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
		synchronized(this) { // it means that thread needs to be synchronized with "this" to get synchronized with following code 
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
// output
// 10 20 30 40 50 100 200 300 400 500 
```

Using `Lock`:
We need to change main method and the Utils class a bit
```java
public class Main {
	public static void main(String[] args) {
		Lock lock = new ReentrantLock();
		Utils utils = new Utils(lock);
		Thread t1 = new Thread(createTask(utils, 10));
		Thread t2 = new Thread(createTask(utils, 100));
		t1.start();
		t2.start();
	}

    // ...
}

public class Utils {

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
```

It is `lock()` method blocks the part of code from other threads and `unlock()` - unlocks. It is required to use `try-finally` when working in `Lock`. Otherwise, the code will remain blocked and no threads will be able to execute it.

In most cases, it is recommended to use `synchronized` over `Lock`, as it is more secure and is easier to read.

### Volatile and Atomic

### Wait(), Notify() and NotifyAll()

In some cases you'd like to implement one task using multiple threads. For example, you are modelling the process of creating a tea. The steps of creating a tea are:
1. Pick up a cup
2. Add some sugar
3. Add a tea-bag
4. Pour some boiled water
5. Mix all

We will create a separate thread for each procedure.

Java allows to hold execution of some thread using `wait()`, `notify()`, and `notifyAll()` methods. `wait()` is used to stop thread execution until the `notify()` or `notifyAll()` methods are called. These methods are placed in the `Object` class, so all objects in Java contain them. `notifyAll()` should be used, when there are more `wait()` calls in the class (`notify()` resumes the thread only on one of the wait method calls in the class).

So now, lets review the example of using these methods to make some tea:
```java
public class TeaProcess {

	public static void main(String[] args) throws InterruptedException {
		ExecutorService executor = Executors.newFixedThreadPool(5);

		Tea tea = new Tea();
		executor.execute(createTask("Mix", tea::waitUntilHasWater, tea::mix));
		executor.execute(createTask("Pour water", tea::waitUntilHasTeaBag, tea::pourWater));
		executor.execute(createTask("Add tea bag", tea::waitUntilHasSugar, tea::addTeaBag));
		executor.execute(createTask("Add sugar", tea::waitUntilHasCup, tea::addSugar));
		executor.execute(createTask("Take a cup", () -> {}, tea::takeACup));

		Thread.sleep(6000);
		System.out.println("Tea is ready: " + tea.isReady());
		executor.shutdown();
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
		private boolean hasCup;
		private boolean hasSugar;
		private boolean hasTeaBag;
		private boolean hasWater;
		private boolean isMixed;

		public Tea() {
		}

		public synchronized void takeACup() {
			hasCup = true;
			notifyAll();
		}
		public synchronized void addSugar() {
			hasSugar = true;
			notifyAll();
		}
		public synchronized void addTeaBag() {
			hasTeaBag = true;
			notifyAll();
		}
		public synchronized void pourWater() {
			hasWater = true;
			notifyAll();
		}
		public synchronized void mix() {
			isMixed = true;
		}

		public synchronized void waitUntilHasCup() {
			try {
				while (!hasCup) wait();
			} catch (InterruptedException e) {
				System.out.println("Wait unit has cup was interrupted");
			}
		}

		public synchronized void waitUntilHasSugar() {
			try {
				while (!hasSugar) wait();
			} catch (InterruptedException e) {
				System.out.println("Wait unit has sugar was interrupted");
			}
		}

		public synchronized void waitUntilHasTeaBag() {
			try {
				while (!hasTeaBag) wait();
			} catch (InterruptedException e) {
				System.out.println("Wait unit has tea bag was interrupted");
			}
		}

		public synchronized void waitUntilHasWater() {
			try {
				while (!hasWater) wait();
			} catch (InterruptedException e) {
				System.out.println("Wait unit has water was interrupted");
			}
		}

		public synchronized boolean isReady() {
			return hasCup && hasSugar && hasTeaBag && hasWater && isMixed;
		}
	}
}
// output
/*
Take a cup
Add sugar
Add tea bag
Pour water
Mix
Tea is ready: true
*
```

It is important to add `wait()` into the while loop, as `notifyAll()` resumes all the threads and we need to stop them again.

Remember to call `wait()`, `notify()`, and `notifyAll()` inside the synchronized block. For example, if you want to call `notifyAll` method for some object x you can do it this way.
```java
synchronized(x) {
	x.notifyAll()
}
```
In the example with tea creating these methods where called inside the synchronized inside the method, that automatically synchronizes code by `this`.
