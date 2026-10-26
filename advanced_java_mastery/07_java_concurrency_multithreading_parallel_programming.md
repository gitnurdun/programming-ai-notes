# Java Concurrency: Mastering Multithreading and Parallel Programming

## Introduction

Java concurrency is a comprehensive framework that provides high-level abstractions for managing multiple threads of execution. It includes the `java.util.concurrent` package, which offers thread pools, concurrent collections, synchronization utilities, and atomic variables that make concurrent programming more manageable and efficient.

This guide will explore Java concurrency in depth, covering everything from basic threading concepts to advanced concurrent programming techniques.

## Fundamentals of Java Threading

### Creating and Managing Threads

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class ThreadCreationBasics {
    public static void main(String[] args) throws InterruptedException {
        // Method 1: Extending Thread class
        Thread thread1 = new Thread() {
            @Override
            public void run() {
                System.out.println("Thread 1 running: " + Thread.currentThread().getName());
                for (int i = 0; i < 5; i++) {
                    System.out.println("Thread 1: " + i);
                    try {
                        Thread.sleep(100);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            }
        };
        
        // Method 2: Implementing Runnable interface
        Runnable task = () -> {
            System.out.println("Runnable task running: " + Thread.currentThread().getName());
            for (int i = 0; i < 5; i++) {
                System.out.println("Runnable: " + i);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        };
        
        Thread thread2 = new Thread(task, "CustomThread");
        
        // Method 3: Using Lambda expressions (Java 8+)
        Thread thread3 = new Thread(() -> {
            System.out.println("Lambda thread running: " + Thread.currentThread().getName());
            for (int i = 0; i < 5; i++) {
                System.out.println("Lambda: " + i);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }, "LambdaThread");
        
        // Starting threads
        thread1.start();
        thread2.start();
        thread3.start();
        
        // Wait for all threads to complete
        thread1.join();
        thread2.join();
        thread3.join();
        
        System.out.println("All threads completed");
        
        // Thread information and management
        Thread currentThread = Thread.currentThread();
        System.out.println("Current thread: " + currentThread.getName());
        System.out.println("Thread ID: " + currentThread.getId());
        System.out.println("Thread priority: " + currentThread.getPriority());
        System.out.println("Thread state: " + currentThread.getState());
        System.out.println("Thread group: " + currentThread.getThreadGroup().getName());
        
        // Thread priority example
        Thread highPriority = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                System.out.println("High priority: " + i);
                Thread.yield(); // Suggest scheduler to switch threads
            }
        });
        
        Thread lowPriority = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                System.out.println("Low priority: " + i);
            }
        });
        
        highPriority.setPriority(Thread.MAX_PRIORITY);
        lowPriority.setPriority(Thread.MIN_PRIORITY);
        
        highPriority.start();
        lowPriority.start();
        
        highPriority.join();
        lowPriority.join();
    }
}
```

### Callable and Future

```java
import java.util.concurrent.*;
import java.util.*;

public class CallableFutureExample {
    public static void main(String[] args) {
        // Create ExecutorService
        ExecutorService executor = Executors.newFixedThreadPool(3);
        
        try {
            // Submit Callable tasks
            List<Future<Integer>> futures = new ArrayList<>();
            
            // Callable that returns a value
            Callable<Integer> task1 = () -> {
                System.out.println("Task 1 starting: " + Thread.currentThread().getName());
                Thread.sleep(2000); // Simulate work
                System.out.println("Task 1 completed");
                return 42;
            };
            
            Callable<Integer> task2 = () -> {
                System.out.println("Task 2 starting: " + Thread.currentThread().getName());
                Thread.sleep(1000);
                System.out.println("Task 2 completed");
                return 100;
            };
            
            Callable<Integer> task3 = () -> {
                System.out.println("Task 3 starting: " + Thread.currentThread().getName());
                Thread.sleep(1500);
                System.out.println("Task 3 completed");
                return 200;
            };
            
            // Submit all tasks
            futures.add(executor.submit(task1));
            futures.add(executor.submit(task2));
            futures.add(executor.submit(task3));
            
            // Get results from futures
            System.out.println("Getting results:");
            for (Future<Integer> future : futures) {
                try {
                    // get() blocks until result is available
                    Integer result = future.get();
                    System.out.println("Result: " + result);
                } catch (ExecutionException e) {
                    System.out.println("Task execution failed: " + e.getCause().getMessage());
                }
            }
            
            // Example with timeout
            System.out.println("\nTesting timeout:");
            Future<String> timeoutTask = executor.submit(() -> {
                Thread.sleep(3000); // This will exceed our timeout
                return "This won't be returned";
            });
            
            try {
                String result = timeoutTask.get(1, TimeUnit.SECONDS);
                System.out.println("Timeout result: " + result);
            } catch (TimeoutException e) {
                System.out.println("Task timed out: " + e.getMessage());
                timeoutTask.cancel(true); // Cancel the task
            }
            
            // Check if task is done
            Future<Integer> checkTask = executor.submit(() -> {
                Thread.sleep(2000);
                return 999;
            });
            
            System.out.println("Task done: " + checkTask.isDone());
            System.out.println("Task cancelled: " + checkTask.isCancelled());
            
            // Wait a bit and check again
            Thread.sleep(2100);
            System.out.println("Task done after waiting: " + checkTask.isDone());
            
            // FutureTask example
            FutureTask<Integer> futureTask = new FutureTask<>(() -> {
                System.out.println("FutureTask executing: " + Thread.currentThread().getName());
                Thread.sleep(1000);
                return 777;
            });
            
            Thread futureThread = new Thread(futureTask);
            futureThread.start();
            
            // Do other work while the task is running
            System.out.println("Doing other work...");
            Thread.sleep(500);
            
            // Get the result when ready
            Integer futureResult = futureTask.get();
            System.out.println("FutureTask result: " + futureResult);
            
        } catch (InterruptedException | ExecutionException | TimeoutException e) {
            e.printStackTrace();
        } finally {
            executor.shutdown();
            try {
                if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
                    executor.shutdownNow();
                }
            } catch (InterruptedException e) {
                executor.shutdownNow();
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

### Thread Safety and Synchronization

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.*;

public class ThreadSafetyExamples {
    // Non-thread-safe counter
    static class UnsafeCounter {
        private int count = 0;
        
        public void increment() {
            count++; // Not thread-safe!
        }
        
        public int getCount() {
            return count;
        }
    }
    
    // Thread-safe counter using synchronized
    static class SynchronizedCounter {
        private int count = 0;
        
        public synchronized void increment() {
            count++;
        }
        
        public synchronized int getCount() {
            return count;
        }
    }
    
    // Thread-safe counter using Atomic
    static class AtomicCounter {
        private AtomicInteger count = new AtomicInteger(0);
        
        public void increment() {
            count.incrementAndGet();
        }
        
        public int getCount() {
            return count.get();
        }
    }
    
    // Race condition demonstration
    static class RaceConditionDemo {
        private int balance = 1000;
        
        public synchronized void withdraw(int amount) {
            if (balance >= amount) {
                // Simulate some processing time - creates race condition window
                try {
                    Thread.sleep(1);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                balance -= amount;
                System.out.println(Thread.currentThread().getName() + " withdrew " + amount + 
                                 ", balance: " + balance);
            } else {
                System.out.println(Thread.currentThread().getName() + " insufficient funds");
            }
        }
        
        public synchronized int getBalance() {
            return balance;
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        System.out.println("--- Comparing thread safety approaches ---");
        
        int numThreads = 10;
        int iterations = 1000;
        
        // Test unsafe counter
        UnsafeCounter unsafeCounter = new UnsafeCounter();
        ExecutorService executor = Executors.newFixedThreadPool(numThreads);
        
        for (int i = 0; i < numThreads; i++) {
            executor.submit(() -> {
                for (int j = 0; j < iterations; j++) {
                    unsafeCounter.increment();
                }
            });
        }
        
        executor.shutdown();
        executor.awaitTermination(10, TimeUnit.SECONDS);
        System.out.println("Unsafe counter result: " + unsafeCounter.getCount());
        System.out.println("Expected: " + (numThreads * iterations));
        
        // Test synchronized counter
        SynchronizedCounter syncCounter = new SynchronizedCounter();
        executor = Executors.newFixedThreadPool(numThreads);
        
        for (int i = 0; i < numThreads; i++) {
            executor.submit(() -> {
                for (int j = 0; j < iterations; j++) {
                    syncCounter.increment();
                }
            });
        }
        
        executor.shutdown();
        executor.awaitTermination(10, TimeUnit.SECONDS);
        System.out.println("Synchronized counter result: " + syncCounter.getCount());
        
        // Test atomic counter
        AtomicCounter atomicCounter = new AtomicCounter();
        executor = Executors.newFixedThreadPool(numThreads);
        
        for (int i = 0; i < numThreads; i++) {
            executor.submit(() -> {
                for (int j = 0; j < iterations; j++) {
                    atomicCounter.increment();
                }
            });
        }
        
        executor.shutdown();
        executor.awaitTermination(10, TimeUnit.SECONDS);
        System.out.println("Atomic counter result: " + atomicCounter.getCount());
        
        // Race condition demonstration
        System.out.println("\n--- Race condition demonstration ---");
        RaceConditionDemo account = new RaceConditionDemo();
        
        executor = Executors.newFixedThreadPool(5);
        
        for (int i = 0; i < 10; i++) {
            executor.submit(() -> account.withdraw(100));
        }
        
        executor.shutdown();
        executor.awaitTermination(10, TimeUnit.SECONDS);
        System.out.println("Final balance: " + account.getBalance());
        
        // Using ReentrantLock for more flexible synchronization
        System.out.println("\n--- ReentrantLock example ---");
        ReentrantLock lock = new ReentrantLock();
        Condition condition = lock.newCondition();
        int[] sharedData = {0};
        boolean[] isDataReady = {false};
        
        // Producer thread
        Thread producer = new Thread(() -> {
            lock.lock();
            try {
                System.out.println("Producer: Producing data");
                Thread.sleep(1000); // Simulate work
                sharedData[0] = 42;
                isDataReady[0] = true;
                System.out.println("Producer: Data ready, signaling");
                condition.signalAll(); // Signal waiting consumers
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                lock.unlock();
            }
        });
        
        // Consumer thread
        Thread consumer = new Thread(() -> {
            lock.lock();
            try {
                while (!isDataReady[0]) {
                    System.out.println("Consumer: Waiting for data...");
                    condition.await(); // Wait until signaled
                }
                System.out.println("Consumer: Received data: " + sharedData[0]);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                lock.unlock();
            }
        });
        
        consumer.start();
        Thread.sleep(100); // Give consumer time to wait
        producer.start();
        
        consumer.join();
        producer.join();
        
        System.out.println("Lock fairness: " + lock.isFair());
        System.out.println("Lock held by current thread: " + lock.isHeldByCurrentThread());
        System.out.println("Number of waiting threads: " + lock.getQueueLength());
    }
}
```

## Executor Framework

### ThreadPool Management

```java
import java.util.concurrent.*;
import java.util.*;

public class ExecutorFrameworkDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("--- Different types of ThreadPools ---");
        
        // 1. FixedThreadPool - Fixed number of threads
        System.out.println("\n=== FixedThreadPool ===");
        ExecutorService fixedPool = Executors.newFixedThreadPool(3);
        
        List<Future<String>> fixedFutures = new ArrayList<>();
        for (int i = 1; i <= 6; i++) {
            final int taskId = i;
            Future<String> future = fixedPool.submit(() -> {
                System.out.println("FixedPool Task " + taskId + " started by " + 
                                 Thread.currentThread().getName());
                Thread.sleep(2000); // Simulate work
                System.out.println("FixedPool Task " + taskId + " completed");
                return "Result from task " + taskId;
            });
            fixedFutures.add(future);
        }
        
        // Get results
        for (Future<String> future : fixedFutures) {
            try {
                System.out.println("Fixed result: " + future.get());
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }
        
        fixedPool.shutdown();
        
        // 2. CachedThreadPool - Creates new threads as needed
        System.out.println("\n=== CachedThreadPool ===");
        ExecutorService cachedPool = Executors.newCachedThreadPool();
        
        for (int i = 1; i <= 4; i++) {
            final int taskId = i;
            cachedPool.submit(() -> {
                System.out.println("CachedPool Task " + taskId + " started by " + 
                                 Thread.currentThread().getName());
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                System.out.println("CachedPool Task " + taskId + " completed");
            });
        }
        
        cachedPool.shutdown();
        cachedPool.awaitTermination(10, TimeUnit.SECONDS);
        
        // 3. SingleThreadExecutor - Single worker thread
        System.out.println("\n=== SingleThreadExecutor ===");
        ExecutorService singlePool = Executors.newSingleThreadExecutor();
        
        for (int i = 1; i <= 3; i++) {
            final int taskId = i;
            singlePool.submit(() -> {
                System.out.println("SinglePool Task " + taskId + " started by " + 
                                 Thread.currentThread().getName());
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                System.out.println("SinglePool Task " + taskId + " completed");
            });
        }
        
        singlePool.shutdown();
        singlePool.awaitTermination(10, TimeUnit.SECONDS);
        
        // 4. ScheduledExecutorService - For scheduled tasks
        System.out.println("\n=== ScheduledExecutorService ===");
        ScheduledExecutorService scheduledPool = Executors.newScheduledThreadPool(2);
        
        // Schedule single execution
        ScheduledFuture<String> scheduledFuture = scheduledPool.schedule(() -> {
            System.out.println("Scheduled task executed by " + Thread.currentThread().getName());
            return "Scheduled result";
        }, 2, TimeUnit.SECONDS);
        
        // Schedule with fixed delay
        ScheduledFuture<?> fixedDelayFuture = scheduledPool.scheduleWithFixedDelay(() -> {
            System.out.println("Fixed delay task executed at " + 
                             System.currentTimeMillis() + " by " + 
                             Thread.currentThread().getName());
        }, 1, 2, TimeUnit.SECONDS);
        
        // Schedule with fixed rate
        ScheduledFuture<?> fixedRateFuture = scheduledPool.scheduleAtFixedRate(() -> {
            System.out.println("Fixed rate task executed at " + 
                             System.currentTimeMillis() + " by " + 
                             Thread.currentThread().getName());
        }, 1, 3, TimeUnit.SECONDS);
        
        // Get scheduled result
        try {
            String result = scheduledFuture.get(5, TimeUnit.SECONDS);
            System.out.println("Scheduled result: " + result);
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
        
        // Let scheduled tasks run for a while, then cancel
        Thread.sleep(10000);
        fixedDelayFuture.cancel(false);
        fixedRateFuture.cancel(false);
        
        scheduledPool.shutdown();
        scheduledPool.awaitTermination(5, TimeUnit.SECONDS);
        
        // 5. Custom ThreadPool with specific configuration
        System.out.println("\n=== Custom ThreadPool ===");
        ThreadPoolExecutor customPool = new ThreadPoolExecutor(
            2,  // core pool size
            4,  // maximum pool size
            60L, TimeUnit.SECONDS,  // keep alive time
            new LinkedBlockingQueue<>(10),  // work queue
            new ThreadFactory() {
                private AtomicInteger counter = new AtomicInteger(0);
                @Override
                public Thread newThread(Runnable r) {
                    Thread t = new Thread(r, "CustomPool-" + counter.incrementAndGet());
                    t.setDaemon(false);
                    return t;
                }
            },
            new ThreadPoolExecutor.CallerRunsPolicy()  // rejection policy
        );
        
        // Submit more tasks than core pool size
        List<Future<?>> customFutures = new ArrayList<>();
        for (int i = 1; i <= 15; i++) {
            final int taskId = i;
            Future<?> future = customPool.submit(() -> {
                System.out.println("CustomPool Task " + taskId + " executed by " + 
                                 Thread.currentThread().getName() + 
                                 " at pool size: " + customPool.getPoolSize());
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
            customFutures.add(future);
        }
        
        // Monitor pool statistics
        Thread monitor = new Thread(() -> {
            while (!customPool.isTerminated()) {
                System.out.println("Pool size: " + customPool.getPoolSize() + 
                                 ", Active: " + customPool.getActiveCount() + 
                                 ", Completed: " + customPool.getCompletedTaskCount() + 
                                 ", Queue size: " + customPool.getQueue().size());
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        });
        
        monitor.start();
        
        // Wait for all tasks to complete
        for (Future<?> future : customFutures) {
            try {
                future.get();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }
        
        monitor.interrupt();
        customPool.shutdown();
        customPool.awaitTermination(10, TimeUnit.SECONDS);
        
        System.out.println("Custom pool stats - Tasks completed: " + customPool.getCompletedTaskCount() +
                          ", Total tasks: " + customPool.getTaskCount());
    }
}
```

### ForkJoinPool and Parallel Streams

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.ForkJoinPool;
import java.util.stream.*;

public class ForkJoinPoolDemo {
    public static void main(String[] args) {
        System.out.println("--- ForkJoinPool and Parallel Processing ---");
        
        // 1. Basic ForkJoinPool usage
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        
        // Recursive task example (parallel computation)
        class ParallelSumTask extends RecursiveTask<Long> {
            private final long[] array;
            private final int start;
            private final int end;
            private static final int THRESHOLD = 1000;
            
            public ParallelSumTask(long[] array, int start, int end) {
                this.array = array;
                this.start = start;
                this.end = end;
            }
            
            @Override
            protected Long compute() {
                if (end - start <= THRESHOLD) {
                    // Sequential computation for small segments
                    long sum = 0;
                    for (int i = start; i < end; i++) {
                        sum += array[i];
                    }
                    return sum;
                } else {
                    // Split the task into two subtasks
                    int mid = start + (end - start) / 2;
                    ParallelSumTask leftTask = new ParallelSumTask(array, start, mid);
                    ParallelSumTask rightTask = new ParallelSumTask(array, mid, end);
                    
                    // Fork the left task and compute the right task directly
                    leftTask.fork();
                    long rightResult = rightTask.compute();
                    long leftResult = leftTask.join();
                    
                    return leftResult + rightResult;
                }
            }
        }
        
        // Create a large array for testing
        long[] largeArray = new long[10000000];
        for (int i = 0; i < largeArray.length; i++) {
            largeArray[i] = i + 1;
        }
        
        System.out.println("Array size: " + largeArray.length);
        
        // Sequential sum for comparison
        long startTime = System.currentTimeMillis();
        long sequentialSum = 0;
        for (long value : largeArray) {
            sequentialSum += value;
        }
        long sequentialTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Sequential sum: " + sequentialSum + " in " + sequentialTime + "ms");
        
        // ForkJoinPool sum
        startTime = System.currentTimeMillis();
        ParallelSumTask task = new ParallelSumTask(largeArray, 0, largeArray.length);
        long forkJoinSum = forkJoinPool.invoke(task);
        long forkJoinTime = System.currentTimeMillis() - startTime;
        
        System.out.println("ForkJoin sum: " + forkJoinSum + " in " + forkJoinTime + "ms");
        
        // 2. Parallel Streams
        System.out.println("\n--- Parallel Streams ---");
        
        List<Integer> numbers = IntStream.range(1, 1000000)
            .boxed()
            .collect(Collectors.toList());
        
        // Sequential stream
        startTime = System.currentTimeMillis();
        long sequentialProduct = numbers.stream()
            .mapToLong(Integer::longValue)
            .reduce(1L, (a, b) -> a * b > 1_000_000 ? 1_000_000 : a * b);
        long sequentialStreamTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Sequential stream time: " + sequentialStreamTime + "ms");
        
        // Parallel stream
        startTime = System.currentTimeMillis();
        long parallelProduct = numbers.parallelStream()
            .mapToLong(Integer::longValue)
            .reduce(1L, (a, b) -> a * b > 1_000_000 ? 1_000_000 : a * b);
        long parallelStreamTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Parallel stream time: " + parallelStreamTime + "ms");
        
        // Parallel stream with custom ForkJoinPool
        ForkJoinPool customPool = new ForkJoinPool(2); // Use only 2 threads
        try {
            startTime = System.currentTimeMillis();
            long customPoolResult = customPool.submit(() -> 
                numbers.parallelStream()
                    .mapToLong(Integer::longValue)
                    .reduce(1L, (a, b) -> a * b > 1_000_000 ? 1_000_000 : a * b)
            ).get();
            long customPoolTime = System.currentTimeMillis() - startTime;
            System.out.println("Custom ForkJoinPool (2 threads) time: " + customPoolTime + "ms");
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
        
        // 3. Parallel processing examples
        System.out.println("\n--- Parallel Processing Examples ---");
        
        List<String> words = Arrays.asList(
            "parallel", "stream", "processing", "concurrency", "java", 
            "multithreading", "performance", "optimization", "scalability"
        );
        
        // Parallel filtering and transformation
        startTime = System.currentTimeMillis();
        List<String> processedWords = words.parallelStream()
            .filter(word -> word.length() > 5)
            .map(String::toUpperCase)
            .sorted()
            .collect(Collectors.toList());
        long parallelProcessingTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Processed words: " + processedWords);
        System.out.println("Parallel processing time: " + parallelProcessingTime + "ms");
        
        // Parallel grouping
        startTime = System.currentTimeMillis();
        Map<Integer, List<String>> groupedByLength = words.parallelStream()
            .collect(Collectors.groupingByConcurrent(String::length));
        long groupingTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Grouped by length: " + groupedByLength);
        System.out.println("Parallel grouping time: " + groupingTime + "ms");
        
        // 4. ForkJoinPool common pool
        System.out.println("\n--- Common ForkJoinPool ---");
        
        ForkJoinPool commonPool = ForkJoinPool.commonPool();
        System.out.println("Common pool parallelism: " + commonPool.getParallelism());
        System.out.println("Common pool thread count: " + commonPool.getPoolSize());
        
        // Use common pool for parallel operations
        startTime = System.currentTimeMillis();
        List<Integer> results = IntStream.range(0, 1000000)
            .parallel()  // Uses common ForkJoinPool
            .map(i -> i * i)
            .filter(i -> i % 1000 == 0)
            .boxed()
            .collect(Collectors.toList());
        long commonPoolTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Results with common pool: " + results.size() + " items");
        System.out.println("Common pool time: " + commonPoolTime + "ms");
        
        forkJoinPool.shutdown();
        customPool.shutdown();
    }
}
```

## Concurrent Collections

### Blocking Collections

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class BlockingCollectionsDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("--- Blocking Collections ---");
        
        // 1. BlockingQueue examples
        System.out.println("\n=== BlockingQueue Examples ===");
        
        // ArrayBlockingQueue - bounded blocking queue
        BlockingQueue<String> boundedQueue = new ArrayBlockingQueue<>(3);
        
        // Producer thread
        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) {
                    String item = "Item-" + i;
                    boundedQueue.put(item); // Blocks if queue is full
                    System.out.println("Produced: " + item + " (Queue size: " + boundedQueue.size() + ")");
                    Thread.sleep(500); // Simulate production time
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        // Consumer thread
        Thread consumer = new Thread(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    String item = boundedQueue.take(); // Blocks if queue is empty
                    System.out.println("Consumed: " + item + " (Queue size: " + boundedQueue.size() + ")");
                    Thread.sleep(700); // Simulate consumption time
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        producer.start();
        Thread.sleep(100); // Start consumer slightly after producer
        consumer.start();
        
        producer.join();
        consumer.join();
        
        // 2. LinkedBlockingQueue - optionally bounded
        System.out.println("\n=== LinkedBlockingQueue ===");
        BlockingQueue<Integer> unboundedQueue = new LinkedBlockingQueue<>();
        
        // Multiple producers and consumers
        ExecutorService executor = Executors.newFixedThreadPool(6);
        
        AtomicInteger producedCount = new AtomicInteger(0);
        AtomicInteger consumedCount = new AtomicInteger(0);
        
        // Producer tasks
        for (int p = 0; p < 3; p++) {
            final int producerId = p;
            executor.submit(() -> {
                try {
                    for (int i = 0; i < 10; i++) {
                        int value = producerId * 100 + i;
                        unboundedQueue.put(value);
                        System.out.println("Producer-" + producerId + " produced: " + value);
                        producedCount.incrementAndGet();
                        Thread.sleep(100);
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        
        // Consumer tasks
        for (int c = 0; c < 3; c++) {
            final int consumerId = c;
            executor.submit(() -> {
                try {
                    for (int i = 0; i < 10; i++) {
                        Integer value = unboundedQueue.take();
                        System.out.println("Consumer-" + consumerId + " consumed: " + value);
                        consumedCount.incrementAndGet();
                        Thread.sleep(150);
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        
        executor.shutdown();
        executor.awaitTermination(30, TimeUnit.SECONDS);
        
        System.out.println("Total produced: " + producedCount.get() + 
                          ", Total consumed: " + consumedCount.get());
        
        // 3. PriorityBlockingQueue
        System.out.println("\n=== PriorityBlockingQueue ===");
        BlockingQueue<Task> priorityQueue = new PriorityBlockingQueue<>();
        
        // Add tasks with different priorities
        priorityQueue.offer(new Task("Low priority", 3));
        priorityQueue.offer(new Task("High priority", 1));
        priorityQueue.offer(new Task("Medium priority", 2));
        priorityQueue.offer(new Task("Critical priority", 0));
        priorityQueue.offer(new Task("Normal priority", 4));
        
        // Consume in priority order
        System.out.println("Consuming from PriorityBlockingQueue:");
        for (int i = 0; i < 5; i++) {
            Task task = priorityQueue.poll(); // Doesn't block if empty
            if (task != null) {
                System.out.println("Consumed: " + task);
            }
        }
        
        // 4. DelayQueue - holds elements until their delay has expired
        System.out.println("\n=== DelayQueue ===");
        DelayQueue<DelayedTask> delayQueue = new DelayQueue<>();
        
        // Add delayed tasks
        delayQueue.put(new DelayedTask("Task 1", 3000)); // 3 seconds delay
        delayQueue.put(new DelayedTask("Task 2", 1000)); // 1 second delay
        delayQueue.put(new DelayedTask("Task 3", 5000)); // 5 seconds delay
        
        System.out.println("Processing DelayQueue:");
        long startTime = System.currentTimeMillis();
        while (System.currentTimeMillis() - startTime < 8000) { // Run for 8 seconds
            DelayedTask task = delayQueue.poll(); // Non-blocking poll
            if (task != null) {
                System.out.println("Executed: " + task.getName() + 
                                 " at " + (System.currentTimeMillis() - startTime) + "ms");
            }
            Thread.sleep(100);
        }
        
        // 5. SynchronousQueue - each insert waits for a corresponding remove
        System.out.println("\n=== SynchronousQueue ===");
        BlockingQueue<String> synchronousQueue = new SynchronousQueue<>();
        
        Thread syncProducer = new Thread(() -> {
            try {
                for (int i = 1; i <= 3; i++) {
                    String item = "SyncItem-" + i;
                    System.out.println("SyncProducer about to offer: " + item);
                    synchronousQueue.put(item); // Will block until consumer takes
                    System.out.println("SyncProducer: " + item + " taken by consumer");
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        Thread syncConsumer = new Thread(() -> {
            try {
                for (int i = 0; i < 3; i++) {
                    System.out.println("SyncConsumer waiting to take item...");
                    String item = synchronousQueue.take(); // Will block until producer offers
                    System.out.println("SyncConsumer took: " + item);
                    Thread.sleep(1000); // Simulate processing time
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        syncProducer.start();
        Thread.sleep(500); // Start consumer slightly after producer
        syncConsumer.start();
        
        syncProducer.join();
        syncConsumer.join();
        
        System.out.println("All blocking queue examples completed.");
    }
}

// Task class for priority queue
class Task implements Comparable<Task> {
    private String name;
    private int priority;
    
    public Task(String name, int priority) {
        this.name = name;
        this.priority = priority;
    }
    
    @Override
    public int compareTo(Task other) {
        return Integer.compare(this.priority, other.priority);
    }
    
    @Override
    public String toString() {
        return name + "(priority: " + priority + ")";
    }
}

// DelayedTask class for DelayQueue
class DelayedTask implements Delayed {
    private String name;
    private long startTime;
    private long delayMs;
    
    public DelayedTask(String name, long delayMs) {
        this.name = name;
        this.delayMs = delayMs;
        this.startTime = System.currentTimeMillis() + delayMs;
    }
    
    @Override
    public long getDelay(TimeUnit unit) {
        long diff = startTime - System.currentTimeMillis();
        return unit.convert(diff, TimeUnit.MILLISECONDS);
    }
    
    @Override
    public int compareTo(Delayed other) {
        if (this.startTime < ((DelayedTask) other).startTime) {
            return -1;
        }
        if (this.startTime > ((DelayedTask) other).startTime) {
            return 1;
        }
        return 0;
    }
    
    public String getName() {
        return name;
    }
}
```

### Concurrent Collections

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class ConcurrentCollectionsDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("--- Concurrent Collections ---");
        
        // 1. ConcurrentHashMap
        System.out.println("\n=== ConcurrentHashMap ===");
        ConcurrentHashMap<String, Integer> concurrentMap = new ConcurrentHashMap<>();
        
        // KeySet view that supports removal
        Set<String> keySet = concurrentMap.keySet();
        concurrentMap.put("A", 1);
        concurrentMap.put("B", 2);
        concurrentMap.put("C", 3);
        
        System.out.println("Initial map: " + concurrentMap);
        System.out.println("KeySet: " + keySet);
        
        // Atomic operations
        concurrentMap.putIfAbsent("D", 4);
        concurrentMap.replace("B", 2, 20);
        concurrentMap.compute("A", (key, value) -> value * 2);
        
        System.out.println("After atomic operations: " + concurrentMap);
        
        // Atomic operations with functions that may be evaluated
        concurrentMap.computeIfPresent("C", (key, value) -> value + 10);
        concurrentMap.merge("E", 100, Integer::sum);
        
        System.out.println("After compute/merge: " + concurrentMap);
        
        // Safe iteration without external synchronization
        System.out.println("Safe iteration:");
        concurrentMap.forEach((key, value) -> 
            System.out.println("  " + key + " = " + value));
        
        // Parallel operations
        AtomicInteger sum = new AtomicInteger(0);
        concurrentMap.forEach((key, value) -> sum.addAndGet(value));
        System.out.println("Sum of values: " + sum.get());
        
        // 2. ConcurrentSkipListMap - thread-safe sorted map
        System.out.println("\n=== ConcurrentSkipListMap ===");
        ConcurrentSkipListMap<String, Integer> skipListMap = new ConcurrentSkipListMap<>();
        
        skipListMap.put("Charlie", 85);
        skipListMap.put("Alice", 92);
        skipListMap.put("Bob", 78);
        skipListMap.put("David", 95);
        
        System.out.println("SkipListMap: " + skipListMap);
        
        // Navigable map operations
        System.out.println("First entry: " + skipListMap.firstEntry());
        System.out.println("Last entry: " + skipListMap.lastEntry());
        System.out.println("HeadMap (Bob): " + skipListMap.headMap("Bob"));
        System.out.println("TailMap (Bob): " + skipListMap.tailMap("Bob"));
        System.out.println("SubMap [Alice, David): " + skipListMap.subMap("Alice", "David"));
        
        // 3. CopyOnWriteArrayList - thread-safe for read-heavy operations
        System.out.println("\n=== CopyOnWriteArrayList ===");
        CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>();
        
        cowList.add("A");
        cowList.add("B");
        cowList.add("C");
        
        System.out.println("Initial CoW list: " + cowList);
        
        // Safe to iterate while modifying
        Thread modifier = new Thread(() -> {
            try {
                Thread.sleep(500); // Let iterator start
                cowList.add("D");
                cowList.add("E");
                cowList.remove(0); // Remove "A"
                System.out.println("Modified CoW list: " + cowList);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        // Iterator sees snapshot at time of creation
        System.out.println("Iterating over CoW list:");
        Iterator<String> iter = cowList.iterator();
        modifier.start();
        
        while (iter.hasNext()) {
            String item = iter.next();
            System.out.println("Iterator sees: " + item);
            Thread.sleep(300); // Slow iteration to see effect
        }
        
        modifier.join();
        System.out.println("Final CoW list: " + cowList);
        
        // 4. CopyOnWriteArraySet - thread-safe set based on CoWArrayList
        System.out.println("\n=== CopyOnWriteArraySet ===");
        CopyOnWriteArraySet<String> cowSet = new CopyOnWriteArraySet<>();
        
        cowSet.add("Apple");
        cowSet.add("Banana");
        cowSet.add("Cherry");
        cowSet.add("Apple"); // Duplicate - will not be added
        
        System.out.println("CoW Set: " + cowSet);
        
        // 5. ConcurrentLinkedQueue - high-performance queue
        System.out.println("\n=== ConcurrentLinkedQueue ===");
        ConcurrentLinkedQueue<String> clq = new ConcurrentLinkedQueue<>();
        
        clq.add("First");
        clq.add("Second");
        clq.add("Third");
        
        System.out.println("Initial queue: " + clq);
        System.out.println("Poll: " + clq.poll());
        System.out.println("Peek: " + clq.peek());
        System.out.println("After operations: " + clq);
        
        // 6. Performance comparison
        System.out.println("\n=== Performance Comparison ===");
        int operations = 100000;
        
        // Synchronized HashMap
        Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
        testMapPerformance(syncMap, "Synchronized HashMap", operations);
        
        // ConcurrentHashMap
        Map<String, Integer> concurrentHashMap = new ConcurrentHashMap<>();
        testMapPerformance(concurrentHashMap, "ConcurrentHashMap", operations);
        
        // ConcurrentLinkedQueue vs synchronized Queue
        Queue<String> syncQueue = Collections.synchronizedQueue(new LinkedList<>());
        testQueuePerformance(syncQueue, "Synchronized Queue", operations);
        
        ConcurrentLinkedQueue<String> clQueue = new ConcurrentLinkedQueue<>();
        testQueuePerformance(clQueue, "ConcurrentLinkedQueue", operations);
    }
    
    private static void testMapPerformance(Map<String, Integer> map, String name, int operations) {
        ExecutorService executor = Executors.newFixedThreadPool(10);
        CountDownLatch latch = new CountDownLatch(10);
        
        long startTime = System.currentTimeMillis();
        
        for (int i = 0; i < 10; i++) {
            final int threadId = i;
            executor.submit(() -> {
                try {
                    for (int j = 0; j < operations / 10; j++) {
                        String key = "key-" + threadId + "-" + j;
                        map.put(key, j);
                        map.get(key);
                    }
                } finally {
                    latch.countDown();
                }
            });
        }
        
        try {
            latch.await();
            long endTime = System.currentTimeMillis();
            System.out.println(name + " time: " + (endTime - startTime) + "ms");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            executor.shutdown();
        }
    }
    
    private static void testQueuePerformance(Queue<String> queue, String name, int operations) {
        ExecutorService executor = Executors.newFixedThreadPool(10);
        CountDownLatch latch = new CountDownLatch(10);
        
        long startTime = System.currentTimeMillis();
        
        for (int i = 0; i < 10; i++) {
            final int threadId = i;
            executor.submit(() -> {
                try {
                    for (int j = 0; j < operations / 10; j++) {
                        String item = "item-" + threadId + "-" + j;
                        queue.offer(item);
                        queue.poll();
                    }
                } finally {
                    latch.countDown();
                }
            });
        }
        
        try {
            latch.await();
            long endTime = System.currentTimeMillis();
            System.out.println(name + " time: " + (endTime - startTime) + "ms");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            executor.shutdown();
        }
    }
}
```

## Advanced Synchronization Utilities

### Semaphores, CountDownLatch, and CyclicBarrier

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class AdvancedSynchronizationDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("--- Advanced Synchronization Utilities ---");
        
        // 1. Semaphore - controlling access to a resource
        System.out.println("\n=== Semaphore Example ===");
        // Limit access to 3 database connections
        Semaphore dbConnections = new Semaphore(3);
        AtomicInteger successfulConnections = new AtomicInteger(0);
        AtomicInteger failedConnections = new AtomicInteger(0);
        
        ExecutorService semaphoreExecutor = Executors.newFixedThreadPool(10);
        
        for (int i = 0; i < 15; i++) {
            final int clientId = i;
            semaphoreExecutor.submit(() -> {
                try {
                    System.out.println("Client " + clientId + " attempting to acquire database connection");
                    
                    // Try to acquire a permit with timeout
                    if (dbConnections.tryAcquire(2, TimeUnit.SECONDS)) {
                        System.out.println("Client " + clientId + " acquired database connection");
                        successfulConnections.incrementAndGet();
                        
                        // Simulate database work
                        Thread.sleep(3000);
                        
                        dbConnections.release();
                        System.out.println("Client " + clientId + " released database connection");
                    } else {
                        System.out.println("Client " + clientId + " failed to acquire connection (timeout)");
                        failedConnections.incrementAndGet();
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        
        semaphoreExecutor.shutdown();
        semaphoreExecutor.awaitTermination(30, TimeUnit.SECONDS);
        
        System.out.println("Successful connections: " + successfulConnections.get());
        System.out.println("Failed connections: " + failedConnections.get());
        
        // 2. CountDownLatch - countdown to release waiting threads
        System.out.println("\n=== CountDownLatch Example ===");
        CountDownLatch startSignal = new CountDownLatch(1);
        CountDownLatch doneSignal = new CountDownLatch(5); // 5 workers
        
        ExecutorService latchExecutor = Executors.newFixedThreadPool(6);
        
        // Worker threads
        for (int i = 0; i < 5; i++) {
            final int workerId = i;
            latchExecutor.submit(() -> {
                try {
                    System.out.println("Worker " + workerId + " waiting for start signal");
                    startSignal.await(); // Wait for start signal
                    
                    System.out.println("Worker " + workerId + " starting work");
                    Thread.sleep(1000 + (int)(Math.random() * 2000)); // Simulate work
                    System.out.println("Worker " + workerId + " finished work");
                    
                    doneSignal.countDown(); // Signal completion
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        
        // Coordinator thread
        latchExecutor.submit(() -> {
            try {
                Thread.sleep(2000); // Prepare for 2 seconds
                System.out.println("Coordinator: All prepared, sending start signal!");
                startSignal.countDown(); // Send start signal
                
                System.out.println("Coordinator: Waiting for all workers to finish...");
                doneSignal.await(); // Wait for all workers to complete
                System.out.println("Coordinator: All workers finished!");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        latchExecutor.shutdown();
        latchExecutor.awaitTermination(30, TimeUnit.SECONDS);
        
        // 3. CyclicBarrier - waits for a number of threads to reach a barrier
        System.out.println("\n=== CyclicBarrier Example ===");
        CyclicBarrier barrier = new CyclicBarrier(4, () -> {
            System.out.println("*** All runners reached the checkpoint! ***");
        });
        
        ExecutorService barrierExecutor = Executors.newFixedThreadPool(4);
        
        for (int i = 0; i < 4; i++) {
            final int runnerId = i;
            barrierExecutor.submit(() -> {
                try {
                    String[] phases = {"Warm-up", "Sprint", "Recovery", "Final Sprint"};
                    
                    for (int phase = 0; phase < phases.length; phase++) {
                        System.out.println("Runner " + runnerId + " in phase: " + phases[phase]);
                        Thread.sleep(1000 + (int)(Math.random() * 1000)); // Simulate phase duration
                        
                        System.out.println("Runner " + runnerId + " approaching checkpoint " + (phase + 1));
                        barrier.await(); // Wait for all runners to reach this checkpoint
                    }
                    
                    System.out.println("Runner " + runnerId + " finished the race!");
                } catch (InterruptedException | BrokenBarrierException e) {
                    Thread.currentThread().interrupt();
                    System.out.println("Runner " + runnerId + " was interrupted: " + e.getMessage());
                }
            });
        }
        
        barrierExecutor.shutdown();
        barrierExecutor.awaitTermination(30, TimeUnit.SECONDS);
        
        // 4. Phaser - more flexible barrier than CyclicBarrier
        System.out.println("\n=== Phaser Example ===");
        Phaser phaser = new Phaser(3) { // 3 parties initially
            @Override
            protected boolean onAdvance(int phase, int registeredParties) {
                System.out.println("Phase " + phase + " completed, " + registeredParties + " parties registered");
                return phase >= 3 || registeredParties == 0; // Terminate after phase 3 or if no parties left
            }
        };
        
        ExecutorService phaserExecutor = Executors.newFixedThreadPool(3);
        
        for (int i = 0; i < 3; i++) {
            final int participantId = i;
            phaserExecutor.submit(() -> {
                try {
                    for (int phase = 0; phase < 4; phase++) {
                        if (phaser.isTerminated()) {
                            break;
                        }
                        
                        System.out.println("Participant " + participantId + " in phase " + phase);
                        Thread.sleep(1000 + (int)(Math.random() * 1000));
                        
                        // Arrive and await advance to next phase
                        int currentPhase = phaser.arriveAndAwaitAdvance();
                        System.out.println("Participant " + participantId + " advanced to phase " + currentPhase);
                        
                        // Unregister one participant after phase 1
                        if (participantId == 0 && phase == 1) {
                            phaser.arriveAndDeregister();
                            System.out.println("Participant 0 deregistered");
                            break;
                        }
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        
        phaserExecutor.shutdown();
        phaserExecutor.awaitTermination(30, TimeUnit.SECONDS);
        
        // 5. Exchanger - exchanging objects between two threads
        System.out.println("\n=== Exchanger Example ===");
        Exchanger<String> exchanger = new Exchanger<>();
        
        Thread producer1 = new Thread(() -> {
            try {
                String data = "Data from Producer 1";
                System.out.println("Producer 1 has: " + data);
                
                String result = exchanger.exchange(data, 3, TimeUnit.SECONDS);
                System.out.println("Producer 1 received: " + result);
            } catch (InterruptedException | TimeoutException e) {
                Thread.currentThread().interrupt();
                System.out.println("Producer 1 interrupted: " + e.getMessage());
            }
        });
        
        Thread producer2 = new Thread(() -> {
            try {
                String data = "Data from Producer 2";
                System.out.println("Producer 2 has: " + data);
                
                Thread.sleep(500); // Slight delay to ensure exchange order
                String result = exchanger.exchange(data, 3, TimeUnit.SECONDS);
                System.out.println("Producer 2 received: " + result);
            } catch (InterruptedException | TimeoutException e) {
                Thread.currentThread().interrupt();
                System.out.println("Producer 2 interrupted: " + e.getMessage());
            }
        });
        
        producer1.start();
        producer2.start();
        
        producer1.join();
        producer2.join();
        
        System.out.println("\nAll synchronization examples completed!");
    }
}
```

## Atomic Variables and Lock-Free Programming

### Atomic Variables

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.*;

public class AtomicVariablesDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("--- Atomic Variables and Lock-Free Programming ---");
        
        // 1. Basic Atomic Variables
        System.out.println("\n=== Basic Atomic Variables ===");
        
        AtomicInteger atomicInt = new AtomicInteger(0);
        AtomicLong atomicLong = new AtomicLong(0);
        AtomicBoolean atomicBoolean = new AtomicBoolean(false);
        AtomicReference<String> atomicRef = new AtomicReference<>("Initial");
        
        System.out.println("Initial values:");
        System.out.println("AtomicInt: " + atomicInt.get());
        System.out.println("AtomicLong: " + atomicLong.get());
        System.out.println("AtomicBoolean: " + atomicBoolean.get());
        System.out.println("AtomicReference: " + atomicRef.get());
        
        // Perform atomic operations
        atomicInt.set(42);
        atomicLong.incrementAndGet();
        atomicBoolean.compareAndSet(false, true);
        atomicRef.compareAndSet("Initial", "Modified");
        
        System.out.println("\nAfter atomic operations:");
        System.out.println("AtomicInt: " + atomicInt.get());
        System.out.println("AtomicLong: " + atomicLong.get());
        System.out.println("AtomicBoolean: " + atomicBoolean.get());
        System.out.println("AtomicReference: " + atomicRef.get());
        
        // 2. Compare-And-Swap (CAS) operations
        System.out.println("\n=== Compare-And-Swap Operations ===");
        
        AtomicInteger casCounter = new AtomicInteger(0);
        
        for (int i = 0; i < 10; i++) {
            int expected;
            int newValue;
            boolean success;
            
            do {
                expected = casCounter.get();
                newValue = expected + 1;
                success = casCounter.compareAndSet(expected, newValue);
            } while (!success);
            
            System.out.println("Updated counter to: " + casCounter.get());
        }
        
        // 3. Atomic Arrays
        System.out.println("\n=== Atomic Arrays ===");
        
        AtomicIntegerArray intArray = new AtomicIntegerArray(5);
        
        // Initialize array
        for (int i = 0; i < 5; i++) {
            intArray.set(i, i * 10);
        }
        
        System.out.println("Initial array: " + Arrays.toString(getArray(intArray)));
        
        // Perform atomic operations on array elements
        intArray.getAndAdd(0, 5);  // Add 5 to element at index 0
        intArray.incrementAndGet(1); // Increment element at index 1
        intArray.compareAndSet(2, 20, 999); // If element 2 is 20, set to 999
        
        System.out.println("After atomic operations: " + Arrays.toString(getArray(intArray)));
        
        AtomicLongArray longArray = new AtomicLongArray(3);
        AtomicReferenceArray<String> refArray = new AtomicReferenceArray<>(3);
        
        longArray.set(0, 100L);
        refArray.set(0, "First");
        
        System.out.println("LongArray: [" + longArray.get(0) + "]");
        System.out.println("RefArray: [" + refArray.get(0) + "]");
        
        // 4. Performance comparison: Atomic vs Synchronized
        System.out.println("\n=== Performance Comparison ===");
        int iterations = 1000000;
        
        // Synchronized counter
        Object syncLock = new Object();
        int[] syncCounter = {0};
        
        long startTime = System.currentTimeMillis();
        Thread[] syncThreads = new Thread[4];
        for (int i = 0; i < 4; i++) {
            syncThreads[i] = new Thread(() -> {
                for (int j = 0; j < iterations / 4; j++) {
                    synchronized (syncLock) {
                        syncCounter[0]++;
                    }
                }
            });
        }
        
        for (Thread t : syncThreads) t.start();
        for (Thread t : syncThreads) t.join();
        long syncTime = System.currentTimeMillis() - startTime;
        
        // Atomic counter
        AtomicInteger atomicCounter = new AtomicInteger(0);
        startTime = System.currentTimeMillis();
        Thread[] atomicThreads = new Thread[4];
        for (int i = 0; i < 4; i++) {
            atomicThreads[i] = new Thread(() -> {
                for (int j = 0; j < iterations / 4; j++) {
                    atomicCounter.incrementAndGet();
                }
            });
        }
        
        for (Thread t : atomicThreads) t.start();
        for (Thread t : atomicThreads) t.join();
        long atomicTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Synchronized counter result: " + syncCounter[0] + 
                          " in " + syncTime + "ms");
        System.out.println("Atomic counter result: " + atomicCounter.get() + 
                          " in " + atomicTime + "ms");
        System.out.println("Atomic is " + (double)syncTime/atomicTime + "x faster");
        
        // 5. Atomic Markable Reference
        System.out.println("\n=== Atomic Markable Reference ===");
        
        AtomicMarkableReference<String> markableRef = new AtomicMarkableReference<>("Initial", false);
        
        System.out.println("Initial: " + markableRef.getReference() + 
                          ", Marked: " + markableRef.isMarked());
        
        // Update with marking
        String[] currentRef = new String[1];
        boolean[] currentMark = new boolean[1];
        markableRef.get(currentRef, currentMark);
        
        boolean success = markableRef.compareAndSet(currentRef[0], "Updated", currentMark[0], true);
        System.out.println("CAS successful: " + success);
        System.out.println("After CAS: " + markableRef.getReference() + 
                          ", Marked: " + markableRef.isMarked());
        
        // 6. Atomic Stamped Reference (for ABA problem prevention)
        System.out.println("\n=== Atomic Stamped Reference ===");
        
        AtomicStampedReference<String> stampedRef = new AtomicStampedReference<>("A", 0);
        
        System.out.println("Initial: " + stampedRef.getReference() + 
                          ", Stamp: " + stampedRef.getStamp());
        
        int[] stamp = new int[1];
        String current = stampedRef.get(stamp);
        int currentStamp = stamp[0];
        
        // Update with stamp increment
        boolean stampedSuccess = stampedRef.compareAndSet(
            current, "B", currentStamp, currentStamp + 1);
        
        System.out.println("Stamped CAS successful: " + stampedSuccess);
        stampedRef.get(stamp);
        System.out.println("After CAS: " + stampedRef.getReference() + 
                          ", Stamp: " + stamp[0]);
        
        // 7. Accumulator and Adder classes (Java 8+)
        System.out.println("\n=== Accumulator and Adder Classes ===");
        
        // LongAdder - better performance under high contention
        LongAdder longAdder = new LongAdder();
        ExecutorService adderExecutor = Executors.newFixedThreadPool(10);
        
        for (int i = 0; i < 10; i++) {
            adderExecutor.submit(() -> {
                for (int j = 0; j < 10000; j++) {
                    longAdder.increment();
                }
            });
        }
        
        adderExecutor.shutdown();
        adderExecutor.awaitTermination(10, TimeUnit.SECONDS);
        
        System.out.println("LongAdder result: " + longAdder.sum());
        
        // DoubleAdder
        DoubleAdder doubleAdder = new DoubleAdder();
        doubleAdder.add(3.14);
        doubleAdder.add(2.71);
        doubleAdder.add(1.41);
        
        System.out.println("DoubleAdder result: " + doubleAdder.sum());
        
        // LongAccumulator
        LongAccumulator productAccumulator = new LongAccumulator((x, y) -> x * y, 1);
        productAccumulator.accumulate(2);
        productAccumulator.accumulate(3);
        productAccumulator.accumulate(4);
        
        System.out.println("Product accumulator: " + productAccumulator.get());
        
        // Custom accumulator for minimum
        LongAccumulator minAccumulator = new LongAccumulator(Long::min, Long.MAX_VALUE);
        int[] randomNumbers = {15, 3, 9, 1, 22, 7, 13};
        for (int num : randomNumbers) {
            minAccumulator.accumulate(num);
        }
        
        System.out.println("Minimum accumulator: " + minAccumulator.get());
        
        System.out.println("\nAtomic variables demonstration completed!");
    }
    
    // Helper method to convert AtomicIntegerArray to int[]
    private static int[] getArray(AtomicIntegerArray array) {
        int[] result = new int[array.length()];
        for (int i = 0; i < array.length(); i++) {
            result[i] = array.get(i);
        }
        return result;
    }
}
```

## CompletableFuture and Asynchronous Programming

### CompletableFuture Fundamentals

```java
import java.util.concurrent.*;
import java.util.*;
import java.util.function.*;

public class CompletableFutureDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        System.out.println("--- CompletableFuture and Asynchronous Programming ---");
        
        // 1. Basic CompletableFuture creation
        System.out.println("\n=== Basic CompletableFuture Creation ===");
        
        // Completed future
        CompletableFuture<String> completedFuture = CompletableFuture.completedFuture("Already done!");
        System.out.println("Completed future: " + completedFuture.get());
        
        // Run Async (no return value)
        CompletableFuture<Void> runFuture = CompletableFuture.runAsync(() -> {
            System.out.println("Running asynchronously: " + Thread.currentThread().getName());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            System.out.println("RunAsync completed");
        });
        
        // Supply Async (with return value)
        CompletableFuture<String> supplyFuture = CompletableFuture.supplyAsync(() -> {
            System.out.println("Supplying asynchronously: " + Thread.currentThread().getName());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "SupplyAsync result";
        });
        
        runFuture.get();
        String result = supplyFuture.get();
        System.out.println("SupplyAsync result: " + result);
        
        // 2. Chaining operations (thenApply, thenCompose)
        System.out.println("\n=== Chaining Operations ===");
        
        CompletableFuture<String> chainedFuture = CompletableFuture
            .supplyAsync(() -> {
                System.out.println("Step 1: Getting user ID");
                return "user-123";
            })
            .thenApply(userId -> {
                System.out.println("Step 2: Getting user details for " + userId);
                return "UserDetails{name='John Doe', id=" + userId + "}";
            })
            .thenApply(userDetails -> {
                System.out.println("Step 3: Formatting user details: " + userDetails);
                return userDetails.toUpperCase();
            });
        
        System.out.println("Chained result: " + chainedFuture.get());
        
        // thenCompose for flat mapping
        CompletableFuture<String> composedFuture = CompletableFuture
            .supplyAsync(() -> {
                System.out.println("Getting order ID");
                return "order-456";
            })
            .thenCompose(orderId -> {
                // Returns another CompletableFuture
                return CompletableFuture.supplyAsync(() -> {
                    System.out.println("Getting order details for " + orderId);
                    return "Order{id=" + orderId + ", status=SHIPPED}";
                });
            });
        
        System.out.println("Composed result: " + composedFuture.get());
        
        // 3. Combining futures (thenCombine, allOf, anyOf)
        System.out.println("\n=== Combining Futures ===");
        
        CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(1000); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            System.out.println("Future 1 completed with 10");
            return 10;
        });
        
        CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(1500); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            System.out.println("Future 2 completed with 20");
            return 20;
        });
        
        // thenCombine - combines results of two futures
        CompletableFuture<Integer> combinedFuture = future1.thenCombine(future2, Integer::sum);
        System.out.println("Combined result (10 + 20): " + combinedFuture.get());
        
        // allOf - waits for all futures to complete
        CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(1000); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            return "Result 1";
        });
        
        CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(2000); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            return "Result 2";
        });
        
        CompletableFuture<String> f3 = CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(500); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            return "Result 3";
        });
        
        CompletableFuture<Void> allOfFuture = CompletableFuture.allOf(f1, f2, f3);
        allOfFuture.join(); // Wait for all to complete
        
        List<String> allResults = Arrays.asList(f1.get(), f2.get(), f3.get());
        System.out.println("All results: " + allResults);
        
        // anyOf - completes when any future completes
        CompletableFuture<Object> anyOfFuture = CompletableFuture.anyOf(f1, f2, f3);
        System.out.println("Any of result: " + anyOfFuture.get());
        
        // 4. Exception handling
        System.out.println("\n=== Exception Handling ===");
        
        CompletableFuture<String> exceptionFuture = CompletableFuture
            .supplyAsync(() -> {
                throw new RuntimeException("Something went wrong!");
            })
            .thenApply(result -> "This won't execute")
            .exceptionally(throwable -> {
                System.out.println("Exception caught: " + throwable.getMessage());
                return "Default Value";
            });
        
        System.out.println("Exception handling result: " + exceptionFuture.get());
        
        // Handle with - provides more control over exception handling
        CompletableFuture<String> handleFuture = CompletableFuture
            .supplyAsync(() -> {
                throw new RuntimeException("Handle exception!");
            })
            .handle((result, throwable) -> {
                if (throwable != null) {
                    System.out.println("Handled exception: " + throwable.getMessage());
                    return "Handled Result";
                } else {
                    return result;
                }
            });
        
        System.out.println("Handle result: " + handleFuture.get());
        
        // 5. Async execution with custom executor
        System.out.println("\n=== Custom Executor ===");
        
        ExecutorService customExecutor = Executors.newFixedThreadPool(2);
        
        CompletableFuture<String> customExecutorFuture = CompletableFuture
            .supplyAsync(() -> {
                System.out.println("Running on custom executor: " + Thread.currentThread().getName());
                return "Custom Executor Result";
            }, customExecutor)
            .thenApplyAsync(result -> {
                System.out.println("Continuing on custom executor: " + Thread.currentThread().getName());
                return result + " - Processed";
            }, customExecutor);
        
        System.out.println("Custom executor result: " + customExecutorFuture.get());
        customExecutor.shutdown();
        
        // 6. Timeout handling
        System.out.println("\n=== Timeout Handling ===");
        
        CompletableFuture<String> timeoutFuture = CompletableFuture
            .supplyAsync(() -> {
                try {
                    Thread.sleep(5000); // This will timeout
                    return "This won't return";
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return "Interrupted";
                }
            })
            .orTimeout(2, TimeUnit.SECONDS) // Timeout after 2 seconds
            .exceptionally(throwable -> {
                System.out.println("Timed out: " + throwable.getMessage());
                return "Timed Out Result";
            });
        
        System.out.println("Timeout result: " + timeoutFuture.get());
        
        // 7. Real-world example: Simulating an API call
        System.out.println("\n=== Real-World Example ===");
        
        // Simulate fetching user data
        CompletableFuture<User> userFuture = fetchUserAsync("user-123");
        
        // Simulate fetching user's orders
        CompletableFuture<List<Order>> ordersFuture = fetchUserOrdersAsync("user-123");
        
        // Combine user and orders data
        CompletableFuture<UserProfile> profileFuture = userFuture
            .thenCombine(ordersFuture, UserProfile::new);
        
        UserProfile profile = profileFuture.get();
        System.out.println("User Profile: " + profile);
        
        // 8. Advanced composition patterns
        System.out.println("\n=== Advanced Composition Patterns ===");
        
        // Sequential execution pattern
        CompletableFuture<String> sequential = CompletableFuture
            .supplyAsync(() -> "Step 1")
            .thenCompose(result1 -> CompletableFuture.supplyAsync(() -> result1 + " -> Step 2"))
            .thenCompose(result2 -> CompletableFuture.supplyAsync(() -> result2 + " -> Step 3"));
        
        System.out.println("Sequential: " + sequential.get());
        
        // Parallel execution with collection
        List<String> items = Arrays.asList("A", "B", "C", "D", "E");
        
        List<CompletableFuture<String>> futures = items.stream()
            .map(item -> CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep((long) (Math.random() * 1000));
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                return "Processed: " + item;
            }))
            .collect(ArrayList::new, ArrayList::add, ArrayList::addAll);
        
        CompletableFuture<Void> allItems = CompletableFuture.allOf(
            futures.toArray(new CompletableFuture[0]));
        
        CompletableFuture<List<String>> allResultsFuture = allItems.thenApply(v -> 
            futures.stream()
                .map(CompletableFuture::join)
                .collect(ArrayList::new, ArrayList::add, ArrayList::addAll));
        
        List<String> allResults = allResultsFuture.get();
        System.out.println("All items processed: " + allResults);
        
        System.out.println("\nCompletableFuture demonstration completed!");
    }
    
    // Helper methods for real-world example
    private static CompletableFuture<User> fetchUserAsync(String userId) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000); // Simulate network delay
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return new User(userId, "John Doe", "john@example.com");
        });
    }
    
    private static CompletableFuture<List<Order>> fetchUserOrdersAsync(String userId) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1500); // Simulate network delay
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return Arrays.asList(
                new Order("order-001", 99.99, "SHIPPED"),
                new Order("order-002", 149.50, "DELIVERED")
            );
        });
    }
}

// Supporting classes for the example
class User {
    private String id;
    private String name;
    private String email;
    
    public User(String id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }
    
    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    
    @Override
    public String toString() {
        return "User{id='" + id + "', name='" + name + "', email='" + email + "'}";
    }
}

class Order {
    private String id;
    private double amount;
    private String status;
    
    public Order(String id, double amount, String status) {
        this.id = id;
        this.amount = amount;
        this.status = status;
    }
    
    public String getId() { return id; }
    public double getAmount() { return amount; }
    public String getStatus() { return status; }
    
    @Override
    public String toString() {
        return "Order{id='" + id + "', amount=" + amount + ", status='" + status + "'}";
    }
}

class UserProfile {
    private User user;
    private List<Order> orders;
    
    public UserProfile(User user, List<Order> orders) {
        this.user = user;
        this.orders = orders;
    }
    
    @Override
    public String toString() {
        return "UserProfile{user=" + user + ", orders=" + orders + "}";
    }
}
```

## Best Practices and Performance Considerations

### Concurrency Best Practices

```java
import java.util.concurrent.*;
import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;

public class ConcurrencyBestPractices {
    public static void main(String[] args) {
        System.out.println("--- Concurrency Best Practices ---");
        
        // 1. Proper resource management
        System.out.println("\n=== Proper Resource Management ===");
        
        // Always properly shut down executors
        ExecutorService executor = Executors.newFixedThreadPool(4);
        try {
            for (int i = 0; i < 10; i++) {
                final int taskId = i;
                executor.submit(() -> {
                    System.out.println("Task " + taskId + " running on " + 
                                     Thread.currentThread().getName());
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                });
            }
        } finally {
            executor.shutdown();
            try {
                if (!executor.awaitTermination(10, TimeUnit.SECONDS)) {
                    System.out.println("Executor didn't terminate in time, forcing shutdown");
                    executor.shutdownNow();
                }
            } catch (InterruptedException e) {
                executor.shutdownNow();
                Thread.currentThread().interrupt();
            }
        }
        
        // 2. Avoiding common pitfalls
        System.out.println("\n=== Avoiding Common Pitfalls ===");
        
        // Pitfall 1: Not handling InterruptedException properly
        Thread problematicThread = new Thread(() -> {
            try {
                for (int i = 0; i < 1000000; i++) {
                    // Simulate work
                    if (i % 1000 == 0) {
                        Thread.sleep(1); // This can throw InterruptedException
                    }
                }
            } catch (InterruptedException e) {
                // Wrong: not restoring the interrupted status
                System.out.println("Thread was interrupted (status not restored)");
            }
        });
        
        Thread correctThread = new Thread(() -> {
            try {
                for (int i = 0; i < 1000000; i++) {
                    if (i % 1000 == 0) {
                        Thread.sleep(1);
                    }
                }
            } catch (InterruptedException e) {
                // Correct: restore the interrupted status
                System.out.println("Thread was interrupted (status restored)");
                Thread.currentThread().interrupt(); // Restore status
            }
        });
        
        // Pitfall 2: Using synchronized collections in concurrent environments
        List<String> badList = Collections.synchronizedList(new ArrayList<>());
        // Better: Use concurrent collections
        List<String> goodList = new CopyOnWriteArrayList<>();
        
        // Pitfall 3: Deadlock example
        Object lock1 = new Object();
        Object lock2 = new Object();
        
        Thread deadlock1 = new Thread(() -> {
            synchronized (lock1) {
                System.out.println("Thread 1: Holding lock 1...");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                System.out.println("Thread 1: Waiting for lock 2...");
                synchronized (lock2) {
                    System.out.println("Thread 1: Holding lock 1 & 2...");
                }
            }
        });
        
        Thread deadlock2 = new Thread(() -> {
            synchronized (lock2) {
                System.out.println("Thread 2: Holding lock 2...");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                System.out.println("Thread 2: Waiting for lock 1...");
                synchronized (lock1) {
                    System.out.println("Thread 2: Holding lock 1 & 2...");
                }
            }
        });
        
        // Don't start these threads as they will cause deadlock
        // deadlock1.start();
        // deadlock2.start();
        
        // 3. Performance considerations
        System.out.println("\n=== Performance Considerations ===");
        
        int operations = 100000;
        
        // Test atomic operations vs synchronized
        testAtomicVsSynchronized(operations);
        
        // Test different executor types
        testExecutorPerformance(operations);
        
        // 4. Choosing the right concurrent construct
        System.out.println("\n=== Choosing the Right Construct ===");
        
        // For thread-safe maps: Use ConcurrentHashMap
        Map<String, Integer> concurrentMap = new ConcurrentHashMap<>();
        
        // For read-heavy operations: Use CopyOnWriteArrayList
        List<String> cowList = new CopyOnWriteArrayList<>();
        
        // For high-throughput queues: Use ConcurrentLinkedQueue
        Queue<String> clq = new ConcurrentLinkedQueue<>();
        
        // For bounded queues: Use ArrayBlockingQueue
        BlockingQueue<String> boundedQueue = new ArrayBlockingQueue<>(100);
        
        // For atomic counters: Use LongAdder (not AtomicInteger) under high contention
        LongAdder highContentionCounter = new LongAdder();
        
        // 5. Thread-local variables for thread-specific data
        System.out.println("\n=== Thread-Local Variables ===");
        
        ThreadLocal<SimpleDateFormat> dateFormat = ThreadLocal.withInitial(
            () -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss")
        );
        
        ThreadLocal<AtomicInteger> threadCounter = ThreadLocal.withInitial(AtomicInteger::new);
        
        ExecutorService threadLocalExecutor = Executors.newFixedThreadPool(3);
        
        for (int i = 0; i < 3; i++) {
            final int threadNum = i;
            threadLocalExecutor.submit(() -> {
                for (int j = 0; j < 3; j++) {
                    String formattedDate = dateFormat.get().format(new Date());
                    int count = threadCounter.get().incrementAndGet();
                    System.out.println("Thread " + threadNum + 
                                     " - Date: " + formattedDate + 
                                     ", Count: " + count);
                }
            });
        }
        
        threadLocalExecutor.shutdown();
        try {
            threadLocalExecutor.awaitTermination(10, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        // 6. Avoiding false sharing in concurrent applications
        System.out.println("\n=== Avoiding False Sharing ===");
        
        Counter counter = new Counter();
        ExecutorService falseSharingExecutor = Executors.newFixedThreadPool(4);
        
        for (int i = 0; i < 4; i++) {
            final int threadId = i;
            falseSharingExecutor.submit(() -> {
                for (int j = 0; j < 1000000; j++) {
                    if (threadId == 0) counter.c1.incrementAndGet();
                    else if (threadId == 1) counter.c2.incrementAndGet();
                    else if (threadId == 2) counter.c3.incrementAndGet();
                    else counter.c4.incrementAndGet();
                }
            });
        }
        
        falseSharingExecutor.shutdown();
        try {
            falseSharingExecutor.awaitTermination(30, TimeUnit.SECONDS);
            System.out.println("Counters - c1: " + counter.c1.get() + 
                             ", c2: " + counter.c2.get() + 
                             ", c3: " + counter.c3.get() + 
                             ", c4: " + counter.c4.get());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        System.out.println("\nBest practices demonstration completed!");
    }
    
    private static void testAtomicVsSynchronized(int operations) {
        // Synchronized counter
        Object syncLock = new Object();
        int[] syncCounter = {0};
        
        long startTime = System.currentTimeMillis();
        Thread[] syncThreads = new Thread[4];
        for (int i = 0; i < 4; i++) {
            syncThreads[i] = new Thread(() -> {
                for (int j = 0; j < operations / 4; j++) {
                    synchronized (syncLock) {
                        syncCounter[0]++;
                    }
                }
            });
        }
        
        for (Thread t : syncThreads) t.start();
        for (Thread t : syncThreads) try { t.join(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        long syncTime = System.currentTimeMillis() - startTime;
        
        // Atomic counter
        AtomicInteger atomicCounter = new AtomicInteger(0);
        startTime = System.currentTimeMillis();
        Thread[] atomicThreads = new Thread[4];
        for (int i = 0; i < 4; i++) {
            atomicThreads[i] = new Thread(() -> {
                for (int j = 0; j < operations / 4; j++) {
                    atomicCounter.incrementAndGet();
                }
            });
        }
        
        for (Thread t : atomicThreads) t.start();
        for (Thread t : atomicThreads) try { t.join(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        long atomicTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Synchronized performance: " + syncTime + "ms");
        System.out.println("Atomic performance: " + atomicTime + "ms");
    }
    
    private static void testExecutorPerformance(int operations) {
        // Cached thread pool
        ExecutorService cachedPool = Executors.newCachedThreadPool();
        long startTime = System.currentTimeMillis();
        
        CountDownLatch latch = new CountDownLatch(4);
        for (int i = 0; i < 4; i++) {
            cachedPool.submit(() -> {
                for (int j = 0; j < operations / 4; j++) {
                    // Simulate work
                    Math.random();
                }
                latch.countDown();
            });
        }
        
        try {
            latch.await();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        long cachedTime = System.currentTimeMillis() - startTime;
        cachedPool.shutdown();
        
        // Fixed thread pool
        ExecutorService fixedPool = Executors.newFixedThreadPool(4);
        startTime = System.currentTimeMillis();
        
        latch = new CountDownLatch(4);
        for (int i = 0; i < 4; i++) {
            fixedPool.submit(() -> {
                for (int j = 0; j < operations / 4; j++) {
                    // Simulate work
                    Math.random();
                }
                latch.countDown();
            });
        }
        
        try {
            latch.await();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        long fixedTime = System.currentTimeMillis() - startTime;
        fixedPool.shutdown();
        
        System.out.println("Cached pool performance: " + cachedTime + "ms");
        System.out.println("Fixed pool performance: " + fixedTime + "ms");
    }
}

// Counter class to demonstrate false sharing
class Counter {
    public AtomicInteger c1 = new AtomicInteger(0);
    public AtomicInteger c2 = new AtomicInteger(0);
    public AtomicInteger c3 = new AtomicInteger(0);
    public AtomicInteger c4 = new AtomicInteger(0);
}
```

## Summary

Java concurrency provides a rich set of tools and utilities for building high-performance, thread-safe applications. Key takeaways include:

1. **Thread Fundamentals**: Understand different ways to create and manage threads using Thread class, Runnable interface, and modern ExecutorService
2. **Synchronization**: Master synchronization techniques using synchronized blocks/methods, ReentrantLock, Semaphores, and other utilities
3. **Executor Framework**: Leverage thread pools for efficient thread management and resource utilization
4. **Concurrent Collections**: Use thread-safe collection implementations designed for concurrent access
5. **Synchronization Utilities**: Utilize advanced synchronization primitives like CountDownLatch, CyclicBarrier, and Phaser
6. **Atomic Variables**: Implement lock-free programming patterns using atomic variables and operations
7. **Asynchronous Programming**: Master CompletableFuture for complex asynchronous operations and composition
8. **Best Practices**: Follow concurrency best practices for performance, safety, and maintainability

Mastering Java concurrency is essential for building scalable, efficient applications that can take advantage of modern multi-core processors while maintaining thread safety and data integrity.