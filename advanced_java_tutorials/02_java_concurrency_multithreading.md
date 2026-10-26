# Advanced Java Concurrency and Multithreading

## Overview
Java's concurrency model is built on threads, synchronization mechanisms, and high-level concurrency utilities. Understanding how to properly manage concurrent execution is essential for building scalable, high-performance applications that can take advantage of modern multi-core processors.

## Thread Lifecycle and States

Java threads can be in one of six states:
1. **NEW**: Thread created but not started
2. **RUNNABLE**: Thread executing or ready to execute
3. **BLOCKED**: Thread blocked waiting for monitor lock
4. **WAITING**: Thread waiting indefinitely for another thread
5. **TIMED_WAITING**: Thread waiting for specified time
6. **TERMINATED**: Thread completed execution

## Synchronization Mechanisms

### Intrinsic Locks (synchronized)
The `synchronized` keyword provides mutual exclusion through monitor locks:

```java
public class SynchronizationDemo {
    private int count = 0;
    private final Object lock = new Object();
    
    // Method-level synchronization
    public synchronized void incrementSyncMethod() {
        count++;
    }
    
    // Block-level synchronization
    public void incrementSyncBlock() {
        synchronized(lock) {
            count++;
        }
    }
    
    // Class-level synchronization for static methods
    public static synchronized void staticSyncMethod() {
        // Static synchronized method
    }
}
```

### Volatile Keyword
The `volatile` keyword ensures visibility of changes across threads:

```java
public class VisibilityDemo {
    private volatile boolean flag = false;
    private volatile int value = 0;
    
    public void writer() {
        value = 42;  // Write to volatile field
        flag = true; // Write to volatile field (happens-after relationship)
    }
    
    public void reader() {
        if (flag) {  // Read from volatile field
            int temp = value; // Guaranteed to see 42
            System.out.println(temp);
        }
    }
}
```

## java.util.concurrent Package

### Thread Pools
Thread pools manage a group of worker threads efficiently:

```java
import java.util.concurrent.*;

public class ThreadPoolDemo {
    public static void main(String[] args) {
        // Fixed thread pool
        ExecutorService fixedPool = Executors.newFixedThreadPool(4);
        
        // Cached thread pool
        ExecutorService cachedPool = Executors.newCachedThreadPool();
        
        // Scheduled thread pool
        ScheduledExecutorService scheduledPool = Executors.newScheduledThreadPool(2);
        
        // Submit tasks
        for (int i = 0; i < 10; i++) {
            final int taskId = i;
            fixedPool.submit(() -> {
                System.out.println("Task " + taskId + " executed by " + 
                                 Thread.currentThread().getName());
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        
        // Proper shutdown
        fixedPool.shutdown();
        try {
            if (!fixedPool.awaitTermination(60, TimeUnit.SECONDS)) {
                fixedPool.shutdownNow();
            }
        } catch (InterruptedException e) {
            fixedPool.shutdownNow();
        }
    }
}
```

### Locks Interface
ReentrantLock provides more flexibility than synchronized:

```java
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.Condition;

public class ReentrantLockDemo {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition condition = lock.newCondition();
    private int value = 0;
    
    public void increment() {
        lock.lock();
        try {
            value++;
            condition.signalAll(); // Notify waiting threads
        } finally {
            lock.unlock();
        }
    }
    
    public void waitForValue(int target) throws InterruptedException {
        lock.lock();
        try {
            while (value < target) {
                condition.await(); // Release lock and wait
            }
        } finally {
            lock.unlock();
        }
    }
    
    // Try lock with timeout
    public boolean tryIncrement(long timeout, TimeUnit unit) 
            throws InterruptedException {
        if (lock.tryLock(timeout, unit)) {
            try {
                value++;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;
    }
}
```

## Advanced Concurrency Utilities

### CompletableFuture
For asynchronous programming and composition:

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;

public class CompletableFutureDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        // Creating and chaining CompletableFutures
        CompletableFuture<String> future = CompletableFuture
            .supplyAsync(() -> {
                // Simulate long-running task
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                return "Hello";
            })
            .thenApply(result -> result + " World")
            .thenCompose(result -> CompletableFuture.supplyAsync(() -> result + "!"))
            .thenApply(String::toUpperCase);
        
        // Handle exceptions
        CompletableFuture<String> futureWithException = CompletableFuture
            .supplyAsync(() -> {
                if (Math.random() > 0.5) {
                    throw new RuntimeException("Random error");
                }
                return "Success";
            })
            .exceptionally(throwable -> "Default Value");
        
        System.out.println(future.get()); // HELLO WORLD!
        System.out.println(futureWithException.get());
        
        // Combining multiple futures
        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "First");
        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "Second");
        
        CompletableFuture<String> combined = future1
            .thenCombine(future2, (result1, result2) -> result1 + " " + result2);
        
        System.out.println(combined.get()); // First Second
    }
}
```

### Fork/Join Framework
For divide-and-conquer algorithms:

```java
import java.util.concurrent.RecursiveTask;
import java.util.concurrent.ForkJoinPool;

public class ForkJoinDemo extends RecursiveTask<Long> {
    private final int[] array;
    private final int start;
    private final int end;
    private static final int THRESHOLD = 1000;
    
    public ForkJoinDemo(int[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }
    
    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            // Direct computation for small arrays
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += array[i];
            }
            return sum;
        } else {
            // Divide and conquer
            int mid = (start + end) / 2;
            ForkJoinDemo leftTask = new ForkJoinDemo(array, start, mid);
            ForkJoinDemo rightTask = new ForkJoinDemo(array, mid, end);
            
            // Fork subtasks
            leftTask.fork();
            rightTask.fork();
            
            // Join results
            long rightResult = rightTask.join();
            long leftResult = leftTask.join();
            
            return leftResult + rightResult;
        }
    }
    
    public static void main(String[] args) {
        int[] array = new int[1000000];
        for (int i = 0; i < array.length; i++) {
            array[i] = i;
        }
        
        ForkJoinPool pool = new ForkJoinPool();
        ForkJoinDemo task = new ForkJoinDemo(array, 0, array.length);
        
        long result = pool.invoke(task);
        System.out.println("Sum: " + result);
    }
}
```

## Atomic Variables

Atomic operations provide lock-free thread safety:

```java
import java.util.concurrent.atomic.*;

public class AtomicDemo {
    private AtomicInteger atomicInt = new AtomicInteger(0);
    private AtomicReference<String> atomicRef = new AtomicReference<>("Initial");
    private AtomicLongArray atomicArray = new AtomicLongArray(10);
    
    public void demonstrateAtomicOperations() {
        // Atomic increment
        int newValue = atomicInt.incrementAndGet();
        
        // Compare and set
        boolean updated = atomicInt.compareAndSet(1, 10);
        
        // Atomic reference operations
        String oldValue = atomicRef.getAndSet("New Value");
        
        // Atomic array operations
        atomicArray.set(0, 100L);
        long arrayValue = atomicArray.incrementAndGet(0);
        
        System.out.println("Atomic int: " + newValue);
        System.out.println("CAS success: " + updated);
        System.out.println("Old reference: " + oldValue);
        System.out.println("Array value: " + arrayValue);
    }
    
    // Lazy initialization with atomic reference
    private AtomicReference<ExpensiveObject> instance = new AtomicReference<>();
    
    public ExpensiveObject getInstance() {
        ExpensiveObject current = instance.get();
        if (current == null) {
            current = new ExpensiveObject();
            if (!instance.compareAndSet(null, current)) {
                // Another thread created the instance
                current = instance.get();
            }
        }
        return current;
    }
    
    private static class ExpensiveObject {
        // Expensive to create object
    }
}
```

## Thread Safety Best Practices

1. **Immutability**: Create immutable objects when possible
2. **Thread Confinement**: Keep data within a single thread
3. **Synchronization**: Use appropriate synchronization mechanisms
4. **Lock Granularity**: Minimize lock contention
5. **Avoid Busy Waiting**: Use proper waiting mechanisms

## Performance Considerations

1. **False Sharing**: CPU cache line conflicts between threads
2. **Lock Contention**: Reduce synchronized block scope
3. **Thread Pool Sizing**: Optimal number of threads based on workload
4. **Amdahl's Law**: Understand parallelization limits

## Conclusion

Advanced Java concurrency requires deep understanding of:
- Thread lifecycle and synchronization primitives
- High-level concurrency utilities
- Performance implications of different approaches
- Best practices for thread safety

Mastering these concepts is essential for building scalable, high-performance enterprise applications that can fully utilize modern hardware capabilities.