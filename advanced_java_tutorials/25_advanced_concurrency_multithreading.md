# Advanced Java Concurrency and Multithreading

## Table of Contents
1. [Introduction to Java Concurrency](#introduction-to-java-concurrency)
2. [Thread Fundamentals and Lifecycle](#thread-fundamentals-and-lifecycle)
3. [Thread Synchronization Mechanisms](#thread-synchronization-mechanisms)
4. [Java Memory Model (JMM)](#java-memory-model-jmm)
5. [Advanced Synchronization Utilities](#advanced-synchronization-utilities)
6. [Thread Pools and Executor Framework](#thread-pools-and-executor-framework)
7. [Fork/Join Framework](#forkjoin-framework)
8. [CompletableFuture and Async Programming](#completablefuture-and-async-programming)
9. [Concurrent Collections](#concurrent-collections)
10. [Lock-Free Data Structures](#lock-free-data-structures)
11. [Performance Optimization and Best Practices](#performance-optimization-and-best-practices)
12. [Advanced Concurrency Patterns](#advanced-concurrency-patterns)

## Introduction to Java Concurrency

Java concurrency provides powerful mechanisms for developing high-performance, multithreaded applications. Understanding concurrency is crucial for building scalable systems that can effectively utilize modern multi-core processors. The Java platform offers extensive support for concurrent programming through the `java.util.concurrent` package, which provides high-level abstractions that make concurrent programming more accessible and less error-prone than low-level thread manipulation.

### Core Concurrency Concepts

```java
// Demonstrating basic concurrency concepts
public class ConcurrencyBasics {
    
    // Shared data between threads
    private static volatile int sharedCounter = 0;
    private static final Object lock = new Object();
    
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Java Concurrency Basics ===");
        
        // Example 1: Basic thread creation
        Thread basicThread = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                System.out.println("Basic thread: " + i);
                try {
                    Thread.sleep(100); // Simulate work
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        });
        
        basicThread.start();
        basicThread.join(); // Wait for thread to complete
        
        // Example 2: Race condition demonstration
        demonstrateRaceCondition();
        
        // Example 3: Synchronization solution
        demonstrateSynchronization();
    }
    
    // Race condition example
    private static void demonstrateRaceCondition() throws InterruptedException {
        System.out.println("\n--- Race Condition Example ---");
        sharedCounter = 0; // Reset counter
        
        Thread[] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    sharedCounter++; // Race condition here
                }
            });
            threads[i].start();
        }
        
        for (Thread thread : threads) {
            thread.join();
        }
        
        System.out.println("Expected: 10000, Actual: " + sharedCounter);
        System.out.println("Race condition caused incorrect result due to lack of synchronization");
    }
    
    // Synchronization solution
    private static void demonstrateSynchronization() throws InterruptedException {
        System.out.println("\n--- Synchronization Solution ---");
        sharedCounter = 0; // Reset counter
        
        Thread[] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    synchronized (lock) {
                        sharedCounter++; // Synchronized access
                    }
                }
            });
            threads[i].start();
        }
        
        for (Thread thread : threads) {
            thread.join();
        }
        
        System.out.println("Expected: 10000, Actual: " + sharedCounter);
        System.out.println("Synchronization fixed the race condition");
    }
}
```

### Concurrency Utilities Overview

```java
// Overview of java.util.concurrent utilities
public class ConcurrencyUtilitiesOverview {
    
    // Atomic variables for lock-free operations
    private static final AtomicInteger atomicCounter = new AtomicInteger(0);
    private static final AtomicBoolean atomicFlag = new AtomicBoolean(false);
    private static final AtomicReference<String> atomicRef = new AtomicReference<>();
    
    public static void demonstrateUtilities() throws InterruptedException {
        System.out.println("=== Concurrency Utilities Overview ===");
        
        // 1. Atomic variables
        demonstrateAtomicVariables();
        
        // 2. Locks
        demonstrateLocks();
        
        // 3. Blocking queues
        demonstrateBlockingQueues();
        
        // 4. Executor services
        demonstrateExecutorServices();
    }
    
    private static void demonstrateAtomicVariables() throws InterruptedException {
        System.out.println("\n--- Atomic Variables ---");
        atomicCounter.set(0);
        
        Thread[] threads = new Thread[5];
        for (int i = 0; i < 5; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    atomicCounter.incrementAndGet(); // Atomic operation
                }
            });
            threads[i].start();
        }
        
        for (Thread thread : threads) {
            thread.join();
        }
        
        System.out.println("Atomic counter result: " + atomicCounter.get());
        
        // Compare and swap operation
        atomicRef.set("Initial");
        boolean updated = atomicRef.compareAndSet("Initial", "Updated");
        System.out.println("Compare and swap result: " + updated);
        System.out.println("Atomic reference value: " + atomicRef.get());
    }
    
    private static void demonstrateLocks() throws InterruptedException {
        System.out.println("\n--- Locks ---");
        ReentrantLock lock = new ReentrantLock();
        Condition condition = lock.newCondition();
        
        ExecutorService executor = Executors.newFixedThreadPool(3);
        
        // Producer thread
        executor.submit(() -> {
            lock.lock();
            try {
                System.out.println("Producer acquired lock");
                Thread.sleep(1000); // Simulate work
                condition.signalAll(); // Notify waiting threads
                System.out.println("Producer signaled condition");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                lock.unlock();
            }
        });
        
        // Consumer threads
        for (int i = 0; i < 2; i++) {
            final int threadId = i;
            executor.submit(() -> {
                lock.lock();
                try {
                    System.out.println("Consumer " + threadId + " waiting for condition");
                    condition.await(); // Wait for signal
                    System.out.println("Consumer " + threadId + " received signal");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    lock.unlock();
                }
            });
        }
        
        Thread.sleep(2000); // Wait for demonstration
        executor.shutdown();
    }
    
    private static void demonstrateBlockingQueues() throws InterruptedException {
        System.out.println("\n--- Blocking Queues ---");
        BlockingQueue<String> queue = new LinkedBlockingQueue<>(5);
        
        // Producer thread
        Thread producer = new Thread(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    String item = "Item-" + i;
                    queue.put(item); // Blocks if queue is full
                    System.out.println("Produced: " + item);
                    Thread.sleep(200); // Simulate production time
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        // Consumer thread
        Thread consumer = new Thread(() -> {
            try {
                while (true) {
                    String item = queue.take(); // Blocks if queue is empty
                    System.out.println("Consumed: " + item);
                    Thread.sleep(300); // Simulate consumption time
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        producer.start();
        consumer.start();
        
        Thread.sleep(3000); // Run for 3 seconds
        producer.interrupt();
        consumer.interrupt();
    }
    
    private static void demonstrateExecutorServices() throws InterruptedException {
        System.out.println("\n--- Executor Services ---");
        ExecutorService executor = Executors.newFixedThreadPool(4);
        
        List<Future<String>> futures = new ArrayList<>();
        
        for (int i = 0; i < 5; i++) {
            final int taskId = i;
            Future<String> future = executor.submit(() -> {
                Thread.sleep(1000); // Simulate work
                return "Task " + taskId + " completed by " + 
                       Thread.currentThread().getName();
            });
            futures.add(future);
        }
        
        for (Future<String> future : futures) {
            try {
                System.out.println(future.get()); // Blocks until result is available
            } catch (ExecutionException e) {
                System.err.println("Task failed: " + e.getCause().getMessage());
            }
        }
        
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);
    }
    
    public static void main(String[] args) throws InterruptedException {
        demonstrateUtilities();
    }
}
```

## Thread Fundamentals and Lifecycle

Understanding the thread lifecycle is fundamental to effective concurrent programming.

### Thread States and Transitions

```java
// Thread lifecycle and states
public class ThreadLifecycleDemo {
    
    public static void demonstrateThreadStates() {
        System.out.println("=== Thread Lifecycle and States ===");
        
        Thread.State[] states = Thread.State.values();
        System.out.println("Available thread states:");
        for (Thread.State state : states) {
            System.out.println("  - " + state + ": " + getThreadStateDescription(state));
        }
    }
    
    private static String getThreadStateDescription(Thread.State state) {
        switch (state) {
            case NEW: return "Thread created but not started";
            case RUNNABLE: return "Thread executing in Java virtual machine";
            case BLOCKED: return "Thread blocked waiting for monitor lock";
            case WAITING: return "Thread waiting indefinitely for another thread";
            case TIMED_WAITING: return "Thread waiting for specified time";
            case TERMINATED: return "Thread execution completed";
            default: return "Unknown state";
        }
    }
    
    // Demonstrating different thread states
    public static void demonstrateThreadStateTransitions() throws InterruptedException {
        System.out.println("\n--- Thread State Transitions ---");
        
        final Object lock = new Object();
        Thread monitorThread = new Thread(() -> {
            synchronized (lock) {
                try {
                    System.out.println("Monitor thread acquired lock, current state: " + 
                                     Thread.currentThread().getState());
                    lock.wait(); // Transition to WAITING state
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        });
        
        Thread waitingThread = new Thread(() -> {
            synchronized (lock) {
                try {
                    System.out.println("Waiting thread waiting for notification");
                    lock.wait(5000); // Transition to TIMED_WAITING state
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        });
        
        // Start threads and monitor states
        System.out.println("Before starting - Monitor thread state: " + monitorThread.getState());
        System.out.println("Before starting - Waiting thread state: " + waitingThread.getState());
        
        monitorThread.start();
        waitingThread.start();
        
        // Give threads time to start
        Thread.sleep(100);
        
        System.out.println("After starting - Monitor thread state: " + monitorThread.getState());
        System.out.println("After starting - Waiting thread state: " + waitingThread.getState());
        
        // Notify the monitor thread
        synchronized (lock) {
            lock.notifyAll();
        }
        
        monitorThread.join();
        waitingThread.join();
        
        System.out.println("After completion - Monitor thread state: " + monitorThread.getState());
        System.out.println("After completion - Waiting thread state: " + waitingThread.getState());
    }
    
    // Thread interruption mechanism
    public static void demonstrateThreadInterruption() throws InterruptedException {
        System.out.println("\n--- Thread Interruption ---");
        
        Thread interruptibleThread = new Thread(() -> {
            try {
                // Simulate work that can be interrupted
                for (int i = 0; i < 100; i++) {
                    if (Thread.currentThread().isInterrupted()) {
                        System.out.println("Thread was interrupted, cleaning up and exiting");
                        return; // Exit gracefully
                    }
                    
                    // Simulate work
                    Thread.sleep(100);
                    System.out.println("Working... " + i);
                }
            } catch (InterruptedException e) {
                // Restore interrupted status
                Thread.currentThread().interrupt();
                System.out.println("Thread interrupted during sleep");
            }
        });
        
        interruptibleThread.start();
        
        // Interrupt the thread after 1 second
        Thread.sleep(1000);
        System.out.println("Interrupting thread...");
        interruptibleThread.interrupt();
        
        interruptibleThread.join();
        System.out.println("Thread interrupted successfully");
    }
    
    public static void main(String[] args) throws InterruptedException {
        demonstrateThreadStates();
        demonstrateThreadStateTransitions();
        demonstrateThreadInterruption();
    }
}
```

### Thread Creation and Management

```java
// Advanced thread creation and management techniques
public class ThreadManagementAdvanced {
    
    // Custom thread factory for better thread management
    public static class CustomThreadFactory implements ThreadFactory {
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String threadNamePrefix;
        private final boolean daemon;
        private final int priority;
        
        public CustomThreadFactory(String namePrefix, boolean daemon, int priority) {
            this.threadNamePrefix = namePrefix + "-thread-";
            this.daemon = daemon;
            this.priority = priority;
        }
        
        @Override
        public Thread newThread(Runnable r) {
            Thread thread = new Thread(r, threadNamePrefix + threadNumber.getAndIncrement());
            thread.setDaemon(daemon);
            thread.setPriority(priority);
            
            // Add uncaught exception handler
            thread.setUncaughtExceptionHandler((t, e) -> {
                System.err.println("Uncaught exception in thread " + t.getName() + ": " + e.getMessage());
                e.printStackTrace();
            });
            
            return thread;
        }
    }
    
    // Thread-local storage demonstration
    public static class ThreadLocalDemo {
        private static final ThreadLocal<String> threadLocalData = new ThreadLocal<>();
        private static final ThreadLocal<Integer> threadLocalCounter = ThreadLocal.withInitial(() -> 0);
        
        public static void demonstrateThreadLocal() {
            ExecutorService executor = Executors.newFixedThreadPool(3);
            
            for (int i = 0; i < 3; i++) {
                final int threadId = i;
                executor.submit(() -> {
                    // Set thread-local data
                    threadLocalData.set("Thread-" + threadId + "-data");
                    threadLocalCounter.set(threadId * 10);
                    
                    System.out.println("Thread " + Thread.currentThread().getName() + 
                                     " has data: " + threadLocalData.get() + 
                                     ", counter: " + threadLocalCounter.get());
                    
                    // Simulate work
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                    
                    // Verify data remains consistent
                    System.out.println("After sleep, thread " + Thread.currentThread().getName() + 
                                     " still has data: " + threadLocalData.get() + 
                                     ", counter: " + threadLocalCounter.get());
                });
            }
            
            try {
                executor.shutdown();
                executor.awaitTermination(5, TimeUnit.SECONDS);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
    
    // Daemon vs Non-daemon threads
    public static void demonstrateDaemonThreads() {
        System.out.println("\n--- Daemon vs Non-Daemon Threads ---");
        
        Thread daemonThread = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                System.out.println("Daemon thread running...");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
            System.out.println("Daemon thread exiting");
        });
        
        Thread nonDaemonThread = new Thread(() -> {
            try {
                Thread.sleep(3000);
                System.out.println("Non-daemon thread completed");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        daemonThread.setDaemon(true); // Must be set before starting
        daemonThread.start();
        nonDaemonThread.start();
        
        System.out.println("Main thread will wait for non-daemon thread, but not for daemon thread");
        
        try {
            nonDaemonThread.join(); // Main thread waits for non-daemon to finish
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        System.out.println("Non-daemon thread finished, main thread exiting");
        // Daemon thread will be terminated when all non-daemon threads finish
    }
    
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Advanced Thread Management ===");
        
        // Custom thread factory
        ThreadFactory customFactory = new CustomThreadFactory("Custom", false, Thread.NORM_PRIORITY);
        Thread customThread = customFactory.newThread(() -> {
            System.out.println("Running in custom thread: " + Thread.currentThread().getName());
        });
        customThread.start();
        customThread.join();
        
        // Thread-local demonstration
        ThreadLocalDemo.demonstrateThreadLocal();
        
        // Daemon threads
        demonstrateDaemonThreads();
    }
}
```

## Thread Synchronization Mechanisms

Proper synchronization is essential for preventing race conditions and ensuring data consistency.

### Synchronization Primitives

```java
// Synchronization mechanisms and their applications
public class SynchronizationMechanisms {
    
    // Synchronized methods and blocks
    public static class SynchronizedExample {
        private int counter = 0;
        private final Object lock = new Object();
        
        // Synchronized method - uses intrinsic lock of the object
        public synchronized void incrementMethod() {
            counter++;
        }
        
        // Synchronized block - more granular control
        public void incrementBlock() {
            synchronized (lock) {
                counter++;
            }
        }
        
        // Synchronized static method - uses class lock
        public static synchronized void staticMethod() {
            System.out.println("Static synchronized method called");
        }
        
        public int getCounter() {
            return counter;
        }
    }
    
    // ReentrantLock example with advanced features
    public static class ReentrantLockExample {
        private final ReentrantLock lock = new ReentrantLock();
        private final ReentrantLock fairLock = new ReentrantLock(true); // Fair lock
        private int value = 0;
        
        public void updateValue() {
            lock.lock();
            try {
                // Critical section
                value++;
                System.out.println("Updated value to: " + value + 
                                 " by thread: " + Thread.currentThread().getName());
            } finally {
                lock.unlock();
            }
        }
        
        public boolean tryUpdateValue() {
            if (lock.tryLock()) {
                try {
                    value++;
                    System.out.println("Tried update, value now: " + value);
                    return true;
                } finally {
                    lock.unlock();
                }
            } else {
                System.out.println("Could not acquire lock, thread: " + 
                                 Thread.currentThread().getName());
                return false;
            }
        }
        
        // Time-limited lock acquisition
        public boolean timedUpdateValue(long timeout, TimeUnit unit) {
            try {
                if (lock.tryLock(timeout, unit)) {
                    try {
                        value++;
                        System.out.println("Timed update, value now: " + value);
                        return true;
                    } finally {
                        lock.unlock();
                    }
                } else {
                    System.out.println("Timed update failed - could not acquire lock");
                    return false;
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return false;
            }
        }
        
        public int getValue() {
            lock.lock();
            try {
                return value;
            } finally {
                lock.unlock();
            }
        }
    }
    
    // ReadWriteLock for read-heavy scenarios
    public static class ReadWriteLockExample {
        private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
        private final Lock readLock = rwLock.readLock();
        private final Lock writeLock = rwLock.writeLock();
        private String data = "Initial data";
        
        public String readData() {
            readLock.lock();
            try {
                System.out.println("Reading data: " + data + 
                                 " by thread: " + Thread.currentThread().getName());
                Thread.sleep(100); // Simulate read operation
                return data;
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return null;
            } finally {
                readLock.unlock();
            }
        }
        
        public void writeData(String newData) {
            writeLock.lock();
            try {
                System.out.println("Writing data: " + newData + 
                                 " by thread: " + Thread.currentThread().getName());
                Thread.sleep(200); // Simulate write operation
                this.data = newData;
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                writeLock.unlock();
            }
        }
        
        public String getData() {
            readLock.lock();
            try {
                return data;
            } finally {
                readLock.unlock();
            }
        }
    }
    
    // Condition variables with locks
    public static class ConditionVariableExample {
        private final ReentrantLock lock = new ReentrantLock();
        private final Condition notFull = lock.newCondition();
        private final Condition notEmpty = lock.newCondition();
        private final Queue<String> queue = new ArrayDeque<>();
        private final int capacity;
        
        public ConditionVariableExample(int capacity) {
            this.capacity = capacity;
        }
        
        public void put(String item) throws InterruptedException {
            lock.lock();
            try {
                while (queue.size() == capacity) {
                    System.out.println("Queue full, producer waiting...");
                    notFull.await(); // Wait for space
                }
                queue.offer(item);
                System.out.println("Put item: " + item + ", queue size: " + queue.size());
                notEmpty.signal(); // Notify consumers
            } finally {
                lock.unlock();
            }
        }
        
        public String take() throws InterruptedException {
            lock.lock();
            try {
                while (queue.isEmpty()) {
                    System.out.println("Queue empty, consumer waiting...");
                    notEmpty.await(); // Wait for items
                }
                String item = queue.poll();
                System.out.println("Took item: " + item + ", queue size: " + queue.size());
                notFull.signal(); // Notify producers
                return item;
            } finally {
                lock.unlock();
            }
        }
        
        public int size() {
            lock.lock();
            try {
                return queue.size();
            } finally {
                lock.unlock();
            }
        }
    }
    
    public static void demonstrateSynchronization() throws InterruptedException {
        System.out.println("=== Synchronization Mechanisms ===");
        
        // Synchronized example
        SynchronizedExample syncExample = new SynchronizedExample();
        ExecutorService executor = Executors.newFixedThreadPool(5);
        
        for (int i = 0; i < 1000; i++) {
            executor.submit(syncExample::incrementMethod);
        }
        
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);
        
        System.out.println("Final counter value: " + syncExample.getCounter());
        
        // ReentrantLock example
        System.out.println("\n--- ReentrantLock Example ---");
        ReentrantLockExample lockExample = new ReentrantLockExample();
        
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                lockExample.updateValue();
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "Updater-1");
        
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                if (!lockExample.tryUpdateValue()) {
                    System.out.println("Failed to acquire lock in tryUpdateValue");
                }
                try {
                    Thread.sleep(150);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "Updater-2");
        
        t1.start();
        t2.start();
        
        t1.join();
        t2.join();
        
        System.out.println("Final value: " + lockExample.getValue());
        
        // ReadWriteLock example
        System.out.println("\n--- ReadWriteLock Example ---");
        ReadWriteLockExample rwExample = new ReadWriteLockExample();
        
        Thread writer = new Thread(() -> {
            for (int i = 0; i < 3; i++) {
                rwExample.writeData("Data-" + i);
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "Writer");
        
        for (int i = 0; i < 5; i++) {
            int readerId = i;
            new Thread(() -> {
                for (int j = 0; j < 2; j++) {
                    rwExample.readData();
                    try {
                        Thread.sleep(200);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
            }, "Reader-" + readerId).start();
        }
        
        writer.start();
        writer.join();
        
        // Condition variable example
        System.out.println("\n--- Condition Variables Example ---");
        ConditionVariableExample conditionExample = new ConditionVariableExample(3);
        
        Thread producer = new Thread(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    conditionExample.put("Item-" + i);
                    Thread.sleep(300);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "Producer");
        
        Thread consumer = new Thread(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    String item = conditionExample.take();
                    System.out.println("Consumed: " + item);
                    Thread.sleep(500);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "Consumer");
        
        producer.start();
        consumer.start();
        
        producer.join();
        consumer.join();
    }
    
    public static void main(String[] args) throws InterruptedException {
        demonstrateSynchronization();
    }
}
```

### Advanced Synchronization Patterns

```java
// Advanced synchronization patterns and techniques
public class AdvancedSynchronizationPatterns {
    
    // Semaphore for resource limiting
    public static class SemaphoreExample {
        private final Semaphore resourcePool;
        private final int poolSize;
        
        public SemaphoreExample(int poolSize) {
            this.poolSize = poolSize;
            this.resourcePool = new Semaphore(poolSize);
        }
        
        public void useResource(String resourceId) {
            try {
                resourcePool.acquire(); // Acquire permit
                System.out.println("Thread " + Thread.currentThread().getName() + 
                                 " acquired resource " + resourceId);
                
                // Simulate resource usage
                Thread.sleep(1000);
                
                System.out.println("Thread " + Thread.currentThread().getName() + 
                                 " releasing resource " + resourceId);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                resourcePool.release(); // Release permit
            }
        }
        
        public int getAvailablePermits() {
            return resourcePool.availablePermits();
        }
        
        public int getPoolSize() {
            return poolSize;
        }
    }
    
    // CountDownLatch for thread coordination
    public static class CountDownLatchExample {
        public void demonstrateCountDownLatch() throws InterruptedException {
            int numberOfTasks = 5;
            CountDownLatch latch = new CountDownLatch(numberOfTasks);
            
            System.out.println("Starting " + numberOfTasks + " tasks...");
            
            for (int i = 0; i < numberOfTasks; i++) {
                final int taskId = i;
                new Thread(() -> {
                    try {
                        System.out.println("Task " + taskId + " starting...");
                        Thread.sleep(1000 + (int)(Math.random() * 2000)); // Random execution time
                        System.out.println("Task " + taskId + " completed");
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    } finally {
                        latch.countDown(); // Decrement latch
                    }
                }, "Task-" + taskId).start();
            }
            
            System.out.println("Waiting for all tasks to complete...");
            latch.await(); // Wait for all tasks to finish
            System.out.println("All tasks completed!");
        }
    }
    
    // CyclicBarrier for phased execution
    public static class CyclicBarrierExample {
        public void demonstrateCyclicBarrier() throws InterruptedException {
            int numberOfThreads = 3;
            CyclicBarrier barrier = new CyclicBarrier(numberOfThreads, () -> {
                System.out.println("All threads reached the barrier! Starting next phase...");
            });
            
            for (int i = 0; i < numberOfThreads; i++) {
                final int threadId = i;
                new Thread(() -> {
                    try {
                        // Phase 1: Do some work
                        System.out.println("Thread " + threadId + " in phase 1");
                        Thread.sleep(1000 + (int)(Math.random() * 1000));
                        
                        // Wait for all threads to reach this point
                        barrier.await();
                        
                        // Phase 2: Do more work
                        System.out.println("Thread " + threadId + " in phase 2");
                        Thread.sleep(500);
                        
                        // Wait for all threads again
                        barrier.await();
                        
                        System.out.println("Thread " + threadId + " completed");
                    } catch (InterruptedException | BrokenBarrierException e) {
                        Thread.currentThread().interrupt();
                        System.err.println("Thread " + threadId + " interrupted: " + e.getMessage());
                    }
                }, "Worker-" + threadId).start();
            }
            
            Thread.sleep(10000); // Wait for all threads to complete
        }
    }
    
    // Phaser for dynamic thread registration
    public static class PhaserExample {
        public void demonstratePhaser() {
            Phaser phaser = new Phaser(1); // Register self (main thread)
            
            for (int i = 0; i < 3; i++) {
                final int taskId = i;
                phaser.register(); // Register each new thread
                new Thread(() -> {
                    try {
                        System.out.println("Task " + taskId + " starting phase 1");
                        Thread.sleep(500);
                        phaser.arriveAndAwaitAdvance(); // Wait for all to reach phase 1
                        
                        System.out.println("Task " + taskId + " starting phase 2");
                        Thread.sleep(500);
                        phaser.arriveAndAwaitAdvance(); // Wait for all to reach phase 2
                        
                        System.out.println("Task " + taskId + " starting phase 3");
                        Thread.sleep(500);
                        phaser.arriveAndDeregister(); // Complete and deregister
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }, "PhaserTask-" + taskId).start();
            }
            
            // Main thread participates in phases
            phaser.arriveAndAwaitAdvance(); // Phase 1
            phaser.arriveAndAwaitAdvance(); // Phase 2
            phaser.arriveAndAwaitAdvance(); // Phase 3 (all threads deregistered)
            
            System.out.println("All phaser tasks completed");
            phaser.arriveAndDeregister(); // Deregister main thread
        }
    }
    
    // Exchanger for thread-to-thread data exchange
    public static class ExchangerExample {
        public void demonstrateExchanger() {
            Exchanger<String> exchanger = new Exchanger<>();
            
            Thread thread1 = new Thread(() -> {
                try {
                    String data1 = "Data from Thread 1";
                    System.out.println("Thread 1 has: " + data1);
                    
                    String exchanged1 = exchanger.exchange(data1);
                    System.out.println("Thread 1 received: " + exchanged1);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }, "Thread-1");
            
            Thread thread2 = new Thread(() -> {
                try {
                    String data2 = "Data from Thread 2";
                    System.out.println("Thread 2 has: " + data2);
                    
                    String exchanged2 = exchanger.exchange(data2);
                    System.out.println("Thread 2 received: " + exchanged2);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }, "Thread-2");
            
            thread1.start();
            thread2.start();
            
            try {
                thread1.join();
                thread2.join();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
    
    public static void demonstrateAdvancedPatterns() throws InterruptedException {
        System.out.println("=== Advanced Synchronization Patterns ===");
        
        // Semaphore example
        System.out.println("\n--- Semaphore Example ---");
        SemaphoreExample semaphore = new SemaphoreExample(2); // Only 2 resources available
        
        for (int i = 0; i < 5; i++) {
            final int resourceId = i;
            new Thread(() -> semaphore.useResource("Resource-" + resourceId),
                      "User-" + resourceId).start();
        }
        
        Thread.sleep(3000); // Let some resources be used
        System.out.println("Available permits: " + semaphore.getAvailablePermits() + 
                         " out of " + semaphore.getPoolSize());
        
        // CountDownLatch example
        System.out.println("\n--- CountDownLatch Example ---");
        CountDownLatchExample latchExample = new CountDownLatchExample();
        latchExample.demonstrateCountDownLatch();
        
        // CyclicBarrier example
        System.out.println("\n--- CyclicBarrier Example ---");
        CyclicBarrierExample barrierExample = new CyclicBarrierExample();
        barrierExample.demonstrateCyclicBarrier();
        
        // Phaser example
        System.out.println("\n--- Phaser Example ---");
        PhaserExample phaserExample = new PhaserExample();
        phaserExample.demonstratePhaser();
        
        // Exchanger example
        System.out.println("\n--- Exchanger Example ---");
        ExchangerExample exchangerExample = new ExchangerExample();
        exchangerExample.demonstrateExchanger();
    }
    
    public static void main(String[] args) throws InterruptedException {
        demonstrateAdvancedPatterns();
    }
}
```

## Java Memory Model (JMM)

The Java Memory Model defines how threads interact through memory and is crucial for understanding concurrent behavior.

### JMM Fundamentals

```java
// Java Memory Model fundamentals
public class JavaMemoryModel {
    
    // Volatile variable for visibility
    private static volatile boolean flag = false;
    private static int regularInt = 0;
    private static volatile int volatileInt = 0;
    
    // Demonstrating visibility problem without volatile
    public static void demonstrateVisibilityProblem() throws InterruptedException {
        System.out.println("=== Visibility Problem Demonstration ===");
        
        Thread writer = new Thread(() -> {
            regularInt = 42;
            volatileInt = 42;
            flag = true; // Volatile write - creates happens-before relationship
            System.out.println("Writer: Set values and flag");
        }, "Writer");
        
        Thread reader = new Thread(() -> {
            while (!flag) { // Volatile read - sees updates to other variables
                // Busy wait
            }
            System.out.println("Reader: Flag is true, regularInt = " + regularInt);
            System.out.println("Reader: Flag is true, volatileInt = " + volatileInt);
        }, "Reader");
        
        reader.start();
        writer.start();
        
        writer.join();
        reader.join();
    }
    
    // Happens-before relationships
    public static class HappensBeforeExample {
        private int x = 0;
        private int y = 0;
        private volatile boolean ready = false;
        
        public void writer() {
            x = 1; // 1
            y = 2; // 2
            ready = true; // 3 - volatile write, creates happens-before
        }
        
        public void reader() {
            if (ready) { // 4 - volatile read
                System.out.println("x = " + x + ", y = " + y); // 5 - guaranteed to see 1,2
            }
        }
        
        // Demonstrate synchronized happens-before
        private final Object lock = new Object();
        private int syncX = 0;
        private int syncY = 0;
        
        public void syncWriter() {
            synchronized (lock) {
                syncX = 1;
                syncY = 2;
            } // Unlock happens-before subsequent lock
        }
        
        public void syncReader() {
            synchronized (lock) { // Subsequent lock happens-after previous unlock
                System.out.println("syncX = " + syncX + ", syncY = " + syncY);
            }
        }
    }
    
    // Atomic operations and memory effects
    public static class AtomicMemoryEffects {
        private final AtomicInteger atomicCounter = new AtomicInteger(0);
        private final AtomicReference<String> atomicRef = new AtomicReference<>();
        
        public void demonstrateAtomicOperations() {
            // All atomic operations have memory effects like volatile
            atomicCounter.incrementAndGet(); // Has memory barrier effects
            atomicRef.set("New Value"); // Volatile write semantics
            
            // Compare-and-swap operations
            atomicRef.compareAndSet("Old Value", "New Value");
        }
        
        public int getCounter() {
            return atomicCounter.get(); // Volatile read semantics
        }
    }
    
    // Memory barriers and ordering
    public static class MemoryBarriersExample {
        private final AtomicLong sequence = new AtomicLong(0);
        private final AtomicReference<Data> dataRef = new AtomicReference<>();
        
        static class Data {
            final int value;
            final String text;
            
            Data(int value, String text) {
                this.value = value;
                this.text = text;
            }
        }
        
        public void publishData(int value, String text) {
            // Create data object
            Data newData = new Data(value, text);
            
            // Publish reference atomically (ensures visibility)
            dataRef.set(newData);
            
            // Increment sequence number to signal completion
            sequence.incrementAndGet();
        }
        
        public Data consumeData() {
            long currentSeq;
            Data currentData;
            
            do {
                currentSeq = sequence.get();
                Thread.yield(); // Yield to allow other threads to complete
                currentData = dataRef.get();
            } while (sequence.get() == currentSeq); // Wait for sequence to advance
            
            return currentData;
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        demonstrateVisibilityProblem();
        
        HappensBeforeExample example = new HappensBeforeExample();
        Thread writer = new Thread(example::writer, "HappensBeforeWriter");
        Thread reader = new Thread(example::reader, "HappensBeforeReader");
        
        reader.start();
        writer.start();
        
        writer.join();
        reader.join();
        
        System.out.println("Happens-before relationships demonstrated");
    }
}
```

### JMM Advanced Concepts

```java
// Advanced JMM concepts and implementation
public class AdvancedJMM {
    
    // Double-checked locking pattern (fixed with volatile)
    public static class SafeDoubleCheckedLocking {
        private volatile static SafeDoubleCheckedLocking instance;
        
        public static SafeDoubleCheckedLocking getInstance() {
            if (instance == null) { // First check (no locking)
                synchronized (SafeDoubleCheckedLocking.class) {
                    if (instance == null) { // Second check (with locking)
                        instance = new SafeDoubleCheckedLocking(); // Volatile ensures proper publication
                    }
                }
            }
            return instance;
        }
    }
    
    // Thread-local memory effects
    public static class ThreadLocalMemoryEffects {
        private static final ThreadLocal<Integer> threadLocal = new ThreadLocal<Integer>() {
            @Override
            protected Integer initialValue() {
                return 0;
            }
        };
        
        private static final ThreadLocal<SimpleDateFormat> formatter = 
            ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
        
        public void demonstrateThreadLocal() {
            // Each thread has its own copy
            threadLocal.set(threadLocal.get() + 1);
            System.out.println("ThreadLocal value: " + threadLocal.get());
        }
        
        public String formatDate(Date date) {
            // Each thread has its own formatter instance
            return formatter.get().format(date);
        }
    }
    
    // Final field semantics
    public static class FinalFieldSemantics {
        private final int value;
        private final String text;
        private final List<String> items;
        
        public FinalFieldSemantics(int value, String text, List<String> items) {
            this.value = value;
            this.text = text;
            this.items = new ArrayList<>(items); // Defensive copy for safe publication
        }
        
        public int getValue() { return value; }
        public String getText() { return text; }
        public List<String> getItems() { return new ArrayList<>(items); } // Defensive copy for safe publication
        
        // Demonstrates that final fields are properly published
        public static void demonstrateFinalFields() {
            List<String> originalList = Arrays.asList("A", "B", "C");
            FinalFieldSemantics obj = new FinalFieldSemantics(42, "Hello", originalList);
            
            // All threads will see consistent state due to final field semantics
            System.out.println("Value: " + obj.getValue());
            System.out.println("Text: " + obj.getText());
            System.out.println("Items: " + obj.getItems());
        }
    }
    
    // Publication safety
    public static class PublicationSafety {
        private static PublicationSafety unsafeInstance;
        private static volatile PublicationSafety safeInstance;
        
        private final int value;
        private final String data;
        
        private PublicationSafety(int value, String data) {
            this.value = value;
            this.data = data;
        }
        
        // Unsafe publication - other threads might see partially constructed object
        public static void unsafePublish(int value, String data) {
            unsafeInstance = new PublicationSafety(value, data); // No happens-before guarantee
        }
        
        // Safe publication using volatile
        public static void safePublish(int value, String data) {
            safeInstance = new PublicationSafety(value, data); // Happens-before guarantee
        }
        
        public static PublicationSafety getUnsafeInstance() {
            return unsafeInstance;
        }
        
        public static PublicationSafety getSafeInstance() {
            return safeInstance;
        }
    }
    
    // Memory fence operations (Java 9+)
    public static class MemoryFencesExample {
        private static int a = 0;
        private static int b = 0;
        
        public static void demonstrateMemoryFences() {
            // Using VarHandle for explicit memory ordering (Java 9+)
            try {
                VarHandle intHandle = MethodHandles.privateLookupIn(MemoryFencesExample.class, 
                                                                 MethodHandles.lookup())
                                                  .findVarHandle(MemoryFencesExample.class, "a", int.class);
                
                // Store with release semantic
                intHandle.setRelease(null, 42);
                
                // Load with acquire semantic
                int value = (int) intHandle.getAcquire(null);
                
                System.out.println("Memory fence example: " + value);
            } catch (Exception e) {
                System.err.println("Error using VarHandle: " + e.getMessage());
            }
        }
    }
    
    public static void demonstrateAdvancedJMM() {
        System.out.println("=== Advanced JMM Concepts ===");
        
        // Final field semantics
        FinalFieldSemantics.demonstrateFinalFields();
        
        // Safe publication
        PublicationSafety.safePublish(100, "Safe data");
        PublicationSafety unsafePublish(200, "Unsafe data");
        
        PublicationSafety safe = PublicationSafety.getSafeInstance();
        if (safe != null) {
            System.out.println("Safe instance - value: " + safe.value + ", data: " + safe.data);
        }
        
        // Memory fences
        MemoryFencesExample.demonstrateMemoryFences();
    }
    
    public static void main(String[] args) {
        demonstrateAdvancedJMM();
    }
}
```

## Advanced Synchronization Utilities

The `java.util.concurrent.locks` package provides advanced synchronization utilities beyond basic `synchronized`.

### Lock Interface Extensions

```java
// Advanced lock implementations and utilities
public class AdvancedLockUtilities {
    
    // Custom lock implementation with additional features
    public static class FeatureRichLock implements Lock {
        private final ReentrantLock lock = new ReentrantLock();
        private final Condition condition = lock.newCondition();
        private final AtomicLong lockCount = new AtomicLong(0);
        private final AtomicLong unlockCount = new AtomicLong(0);
        
        @Override
        public void lock() {
            lock.lock();
            lockCount.incrementAndGet();
        }
        
        @Override
        public void lockInterruptibly() throws InterruptedException {
            lock.lockInterruptibly();
            lockCount.incrementAndGet();
        }
        
        @Override
        public boolean tryLock() {
            boolean acquired = lock.tryLock();
            if (acquired) {
                lockCount.incrementAndGet();
            }
            return acquired;
        }
        
        @Override
        public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
            boolean acquired = lock.tryLock(time, unit);
            if (acquired) {
                lockCount.incrementAndGet();
            }
            return acquired;
        }
        
        @Override
        public void unlock() {
            lock.unlock();
            unlockCount.incrementAndGet();
        }
        
        @Override
        public Condition newCondition() {
            return condition;
        }
        
        // Additional utility methods
        public long getLockCount() { return lockCount.get(); }
        public long getUnlockCount() { return unlockCount.get(); }
        public int getQueueLength() { return lock.getQueueLength(); }
        public boolean hasQueuedThreads() { return lock.hasQueuedThreads(); }
        public boolean isHeldByCurrentThread() { return lock.isHeldByCurrentThread(); }
        public int getHoldCount() { return lock.getHoldCount(); }
    }
    
    // StampedLock for optimistic locking
    public static class StampedLockExample {
        private final StampedLock lock = new StampedLock();
        private double x, y;
        
        public StampedLockExample(double x, double y) {
            this.x = x;
            this.y = y;
        }
        
        // Optimistic read - no blocking if no writes occurring
        public double distanceFromOriginOptimistic() {
            long stamp = lock.tryOptimisticRead();
            double currentX = x;
            double currentY = y;
            
            // Validate that no write occurred during read
            if (!lock.validate(stamp)) {
                // Upgrade to read lock if validation failed
                stamp = lock.readLock();
                try {
                    currentX = x;
                    currentY = y;
                } finally {
                    lock.unlockRead(stamp);
                }
            }
            
            return Math.sqrt(currentX * currentX + currentY * currentY);
        }
        
        // Read lock method
        public double distanceFromOriginRead() {
            long stamp = lock.readLock();
            try {
                return Math.sqrt(x * x + y * y);
            } finally {
                lock.unlockRead(stamp);
            }
        }
        
        // Write lock method
        public void move(double deltaX, double deltaY) {
            long stamp = lock.writeLock();
            try {
                x += deltaX;
                y += deltaY;
            } finally {
                lock.unlockWrite(stamp);
            }
        }
        
        // Try-convert read to write
        public boolean updateIfClose(double newX, double newY, double threshold) {
            long stamp = lock.readLock();
            try {
                while (distanceFromOriginRead() > threshold) {
                    long ws = lock.tryConvertToWriteLock(stamp);
                    if (ws != 0L) {
                        stamp = ws;
                        x = newX;
                        y = newY;
                        return true;
                    } else {
                        lock.unlockRead(stamp);
                        stamp = lock.writeLock();
                    }
                }
                return false;
            } finally {
                lock.unlock(stamp);
            }
        }
    }
    
    // Custom synchronization state with AbstractQueuedSynchronizer
    public static class SemaphoreAQS extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1L;
        
        SemaphoreAQS(int count) {
            setState(count);
        }
        
        @Override
        protected int tryAcquireShared(int ignored) {
            for (;;) {
                int current = getState();
                int newCount = current - 1;
                
                if (newCount < 0 || compareAndSetState(current, newCount)) {
                    return newCount;
                }
            }
        }
        
        @Override
        protected boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int newCount = current + releases;
                
                if (compareAndSetState(current, newCount)) {
                    return true;
                }
            }
        }
        
        public void acquire() throws InterruptedException {
            acquireSharedInterruptibly(1);
        }
        
        public void release() {
            releaseShared(1);
        }
        
        public boolean tryAcquire() {
            return tryAcquireShared(1) >= 0;
        }
    }
    
    public static void demonstrateAdvancedLocks() throws InterruptedException {
        System.out.println("=== Advanced Lock Utilities ===");
        
        // Feature-rich lock
        FeatureRichLock featureLock = new FeatureRichLock();
        
        Thread t1 = new Thread(() -> {
            featureLock.lock();
            try {
                System.out.println("Thread 1 acquired lock");
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                featureLock.unlock();
            }
        });
        
        Thread t2 = new Thread(() -> {
            System.out.println("Thread 2 trying to acquire lock, queue length: " + 
                             featureLock.getQueueLength());
            featureLock.lock();
            try {
                System.out.println("Thread 2 acquired lock");
            } finally {
                featureLock.unlock();
            }
        });
        
        t1.start();
        Thread.sleep(100); // Let t1 start first
        t2.start();
        
        t1.join();
        t2.join();
        
        System.out.println("Lock counts - Lock: " + featureLock.getLockCount() + 
                         ", Unlock: " + featureLock.getUnlockCount());
        
        // StampedLock example
        System.out.println("\n--- StampedLock Example ---");
        StampedLockExample stamped = new StampedLockExample(3.0, 4.0);
        
        System.out.println("Initial distance: " + stamped.distanceFromOriginOptimistic());
        
        ExecutorService executor = Executors.newFixedThreadPool(3);
        
        // Multiple readers
        for (int i = 0; i < 3; i++) {
            executor.submit(() -> {
                for (int j = 0; j < 100; j++) {
                    double distance = stamped.distanceFromOriginOptimistic();
                    System.out.println(Thread.currentThread().getName() + 
                                     " distance: " + String.format("%.2f", distance));
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
            });
        }
        
        // Single writer
        executor.submit(() -> {
            for (int i = 0; i < 10; i++) {
                stamped.move(1.0, 1.0);
                System.out.println("Moved to new position, distance: " + 
                                 String.format("%.2f", stamped.distanceFromOriginRead()));
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        });
        
        Thread.sleep(2000); // Let operations complete
        executor.shutdown();
        
        // Custom AQS-based semaphore
        System.out.println("\n--- Custom AQS Semaphore ---");
        SemaphoreAQS customSemaphore = new SemaphoreAQS(2); // 2 permits
        
        for (int i = 0; i < 5; i++) {
            final int threadId = i;
            new Thread(() -> {
                try {
                    System.out.println("Thread " + threadId + " trying to acquire permit");
                    customSemaphore.acquire();
                    System.out.println("Thread " + threadId + " acquired permit");
                    
                    Thread.sleep(1000); // Hold permit
                    
                    customSemaphore.release();
                    System.out.println("Thread " + threadId + " released permit");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }, "CustomSemaphore-" + threadId).start();
        }
        
        Thread.sleep(6000); // Wait for all to complete
    }
    
    public static void main(String[] args) throws InterruptedException {
        demonstrateAdvancedLocks();
    }
}
```

### Lock-Free Algorithms

```java
// Lock-free data structures and algorithms
public class LockFreeAlgorithms {
    
    // Lock-free stack implementation
    public static class LockFreeStack<T> {
        private final AtomicReference<Node<T>> head = new AtomicReference<>();
        
        private static class Node<T> {
            final T data;
            Node<T> next;
            
            Node(T data) {
                this.data = data;
            }
        }
        
        public void push(T item) {
            Node<T> newNode = new Node<>(item);
            Node<T> currentHead;
            
            do {
                currentHead = head.get();
                newNode.next = currentHead;
            } while (!head.compareAndSet(currentHead, newNode));
        }
        
        public T pop() {
            Node<T> currentHead;
            Node<T> newHead;
            
            do {
                currentHead = head.get();
                if (currentHead == null) {
                    return null;
                }
                newHead = currentHead.next;
            } while (!head.compareAndSet(currentHead, newHead));
            
            return currentHead.data;
        }
        
        public boolean isEmpty() {
            return head.get() == null;
        }
        
        public int size() {
            int count = 0;
            Node<T> current = head.get();
            while (current != null) {
                count++;
                current = current.next;
            }
            return count;
        }
    }
    
    // Lock-free counter
    public static class LockFreeCounter {
        private final AtomicLong value = new AtomicLong(0);
        
        public long increment() {
            return value.incrementAndGet();
        }
        
        public long decrement() {
            return value.decrementAndGet();
        }
        
        public long getValue() {
            return value.get();
        }
        
        public boolean compareAndSet(long expect, long update) {
            return value.compareAndSet(expect, update);
        }
        
        // Advanced atomic operations
        public long addAndGet(long delta) {
            return value.addAndGet(delta);
        }
        
        public long getAndAdd(long delta) {
            return value.getAndAdd(delta);
        }
        
        public long getAndSet(long newValue) {
            return value.getAndSet(newValue);
        }
    }
    
    // Lock-free queue (simplified Michael-Scott algorithm)
    public static class LockFreeQueue<T> {
        private final AtomicReference<Node<T>> head;
        private final AtomicReference<Node<T>> tail;
        
        private static class Node<T> {
            final T data;
            final AtomicReference<Node<T>> next = new AtomicReference<>();
            
            Node(T data) {
                this.data = data;
            }
        }
        
        public LockFreeQueue() {
            Node<T> dummy = new Node<>(null);
            head = new AtomicReference<>(dummy);
            tail = new AtomicReference<>(dummy);
        }
        
        public void enqueue(T item) {
            Node<T> newNode = new Node<>(item);
            
            while (true) {
                Node<T> curTail = tail.get();
                Node<T> tailNext = curTail.next.get();
                
                if (curTail == tail.get()) {
                    if (tailNext != null) {
                        // Queue was changed, try again
                        tail.compareAndSet(curTail, tailNext);
                    } else {
                        // Try to link new node as next
                        if (curTail.next.compareAndSet(null, newNode)) {
                            // Try to advance tail
                            tail.compareAndSet(curTail, newNode);
                            return;
                        }
                    }
                }
            }
        }
        
        public T dequeue() {
            while (true) {
                Node<T> curHead = head.get();
                Node<T> curTail = tail.get();
                Node<T> headNext = curHead.next.get();
                
                if (curHead == head.get()) {
                    if (curHead == curTail) {
                        if (headNext == null) {
                            return null; // Queue is empty
                        } else {
                            // Tail is falling behind, advance it
                            tail.compareAndSet(curTail, headNext);
                        }
                    } else {
                        if (headNext == null) {
                            return null; // Queue is empty
                        }
                        
                        T data = headNext.data;
                        if (head.compareAndSet(curHead, headNext)) {
                            return data;
                        }
                    }
                }
            }
        }
        
        public boolean isEmpty() {
            return head.get() == tail.get();
        }
    }
    
    // Atomic field updaters for performance
    public static class AtomicFieldUpdaterExample {
        private volatile int volatileInt = 0;
        private volatile String volatileString = null;
        
        private static final AtomicIntegerFieldUpdater<AtomicFieldUpdaterExample> 
            intUpdater = AtomicIntegerFieldUpdater.newUpdater(AtomicFieldUpdaterExample.class, "volatileInt");
        
        private static final AtomicReferenceFieldUpdater<AtomicFieldUpdaterExample, String>
            stringUpdater = AtomicReferenceFieldUpdater.newUpdater(AtomicFieldUpdaterExample.class, String.class, "volatileString");
        
        public int incrementAndGet() {
            return intUpdater.incrementAndGet(this);
        }
        
        public boolean compareAndSetString(String expect, String update) {
            return stringUpdater.compareAndSet(this, expect, update);
        }
        
        public String getString() {
            return stringUpdater.get(this);
        }
    }
    
    public static void demonstrateLockFree() throws InterruptedException {
        System.out.println("=== Lock-Free Algorithms ===");
        
        // Lock-free stack
        LockFreeStack<String> stack = new LockFreeStack<>();
        
        // Producer threads
        Thread[] producers = new Thread[3];
        for (int i = 0; i < 3; i++) {
            final int producerId = i;
            producers[i] = new Thread(() -> {
                for (int j = 0; j < 10; j++) {
                    String item = "Producer-" + producerId + "-Item-" + j;
                    stack.push(item);
                    System.out.println("Pushed: " + item);
                    try {
                        Thread.sleep(50);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        return;
                    }
                }
            }, "Producer-" + producerId);
        }
        
        // Consumer threads
        Thread[] consumers = new Thread[2];
        for (int i = 0; i < 2; i++) {
            final int consumerId = i;
            consumers[i] = new Thread(() -> {
                for (int j = 0; j < 15; j++) {
                    String item = stack.pop();
                    if (item != null) {
                        System.out.println("Consumer-" + consumerId + " popped: " + item);
                    } else {
                        System.out.println("Consumer-" + consumerId + " found empty stack");
                    }
                    try {
                        Thread.sleep(75);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        return;
                    }
                }
            }, "Consumer-" + consumerId);
        }
        
        // Start all threads
        for (Thread producer : producers) producer.start();
        for (Thread consumer : consumers) consumer.start();
        
        // Wait for completion
        for (Thread producer : producers) producer.join();
        for (Thread consumer : consumers) consumer.join();
        
        System.out.println("Final stack size: " + stack.size());
        
        // Lock-free counter
        System.out.println("\n--- Lock-Free Counter ---");
        LockFreeCounter counter = new LockFreeCounter();
        
        Thread[] counterThreads = new Thread[5];
        for (int i = 0; i < 5; i++) {
            counterThreads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    counter.increment();
                }
            });
            counterThreads[i].start();
        }
        
        for (Thread thread : counterThreads) thread.join();
        
        System.out.println("Final counter value: " + counter.getValue());
        
        // Lock-free queue
        System.out.println("\n--- Lock-Free Queue ---");
        LockFreeQueue<Integer> queue = new LockFreeQueue<>();
        
        Thread producer = new Thread(() -> {
            for (int i = 0; i < 20; i++) {
                queue.enqueue(i);
                System.out.println("Enqueued: " + i);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        });
        
        Thread consumer = new Thread(() -> {
            for (int i = 0; i < 20; i++) {
                Integer item = queue.dequeue();
                if (item != null) {
                    System.out.println("Dequeued: " + item);
                } else {
                    System.out.println("Queue empty");
                }
                try {
                    Thread.sleep(150);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        });
        
        producer.start();
        consumer.start();
        
        producer.join();
        consumer.join();
        
        System.out.println("Queue empty: " + queue.isEmpty());
        
        // Atomic field updater
        System.out.println("\n--- Atomic Field Updater ---");
        AtomicFieldUpdaterExample updaterExample = new AtomicFieldUpdaterExample();
        
        for (int i = 0; i < 5; i++) {
            final int threadId = i;
            new Thread(() -> {
                for (int j = 0; j < 100; j++) {
                    updaterExample.incrementAndGet();
                }
                System.out.println("Thread " + threadId + " finished incrementing");
            }).start();
        }
        
        Thread.sleep(1000); // Let operations complete
        System.out.println("Final int value: " + updaterExample.incrementAndGet());
    }
    
    public static void main(String[] args) throws InterruptedException {
        demonstrateLockFree();
    }
}
```

## Thread Pools and Executor Framework

The Executor framework provides a high-level abstraction for managing threads and executing tasks.

### Executor Framework Fundamentals

```java
// Executor framework and thread pool management
public class ExecutorFramework {
    
    // Custom executor service with monitoring
    public static class MonitoredExecutorService implements ExecutorService {
        private final ExecutorService executor;
        private final AtomicLong taskCount = new AtomicLong(0);
        private final AtomicLong completedTaskCount = new AtomicLong(0);
        private final AtomicLong failedTaskCount = new AtomicLong(0);
        
        public MonitoredExecutorService(ExecutorService executor) {
            this.executor = executor;
        }
        
        @Override
        public void execute(Runnable command) {
            taskCount.incrementAndGet();
            executor.execute(() -> {
                try {
                    command.run();
                    completedTaskCount.incrementAndGet();
                } catch (Exception e) {
                    failedTaskCount.incrementAndGet();
                    throw e;
                }
            });
        }
        
        // Delegate other methods to underlying executor
        @Override
        public Future<?> submit(Runnable task) {
            taskCount.incrementAndGet();
            Future<?> future = executor.submit(() -> {
                try {
                    task.run();
                    completedTaskCount.incrementAndGet();
                } catch (Exception e) {
                    failedTaskCount.incrementAndGet();
                    throw e;
                }
            });
            return future;
        }
        
        @Override
        public <T> Future<T> submit(Runnable task, T result) {
            taskCount.incrementAndGet();
            return executor.submit(() -> {
                try {
                    task.run();
                    completedTaskCount.incrementAndGet();
                    return result;
                } catch (Exception e) {
                    failedTaskCount.incrementAndGet();
                    throw e;
                }
            }, result);
        }
        
        @Override
        public <T> Future<T> submit(Callable<T> task) {
            taskCount.incrementAndGet();
            return executor.submit(() -> {
                try {
                    T result = task.call();
                    completedTaskCount.incrementAndGet();
                    return result;
                } catch (Exception e) {
                    failedTaskCount.incrementAndGet();
                    throw e;
                }
            });
        }
        
        // Monitoring methods
        public long getTaskCount() { return taskCount.get(); }
        public long getCompletedTaskCount() { return completedTaskCount.get(); }
        public long getFailedTaskCount() { return failedTaskCount.get(); }
        public double getSuccessRate() { 
            long total = taskCount.get();
            return total > 0 ? (double) completedTaskCount.get() / total * 100 : 0;
        }
        
        // Delegate remaining methods
        @Override public void shutdown() { executor.shutdown(); }
        @Override public List<Runnable> shutdownNow() { return executor.shutdownNow(); }
        @Override public boolean isShutdown() { return executor.isShutdown(); }
        @Override public boolean isTerminated() { return executor.isTerminated(); }
        @Override public boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException { 
            return executor.awaitTermination(timeout, unit); 
        }
        @Override public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException { 
            return executor.invokeAll(tasks); 
        }
        @Override public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException { 
            return executor.invokeAll(tasks, timeout, unit); 
        }
        @Override public <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException { 
            return executor.invokeAny(tasks); 
        }
        @Override public <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException { 
            return executor.invokeAny(tasks, timeout, unit); 
        }
    }
    
    // Custom thread factory for thread pools
    public static class CustomThreadFactory implements ThreadFactory {
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String namePrefix;
        private final ThreadGroup group;
        private final int priority;
        private final boolean daemon;
        
        public CustomThreadFactory(String namePrefix, int priority, boolean daemon) {
            SecurityManager s = System.getSecurityManager();
            this.group = (s != null) ? s.getThreadGroup() : Thread.currentThread().getThreadGroup();
            this.namePrefix = namePrefix + "-thread-";
            this.priority = priority;
            this.daemon = daemon;
        }
        
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(group, r, namePrefix + threadNumber.getAndIncrement(), 0);
            if (t.isDaemon() != daemon) t.setDaemon(daemon);
            if (t.getPriority() != priority) t.setPriority(priority);
            
            // Add uncaught exception handler
            t.setUncaughtExceptionHandler((thread, exception) -> {
                System.err.println("Uncaught exception in thread " + thread.getName());
                exception.printStackTrace();
            });
            
            return t;
        }
    }
    
    // Custom rejection policy
    public static class CustomRejectionPolicy implements RejectedExecutionHandler {
        private final AtomicInteger rejectionCount = new AtomicInteger(0);
        
        @Override
        public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
            rejectionCount.incrementAndGet();
            System.err.println("Task rejected by executor: " + r + ", executor info: " + 
                             executor.getPoolSize() + "/" + executor.getMaximumPoolSize());
            
            // Alternative: execute in caller's thread (blocking)
            if (!executor.isShutdown()) {
                try {
                    r.run();
                } catch (RuntimeException e) {
                    System.err.println("Task execution failed: " + e.getMessage());
                    throw e;
                }
            }
        }
        
        public int getRejectionCount() {
            return rejectionCount.get();
        }
    }
    
    public static void demonstrateExecutorFramework() throws InterruptedException {
        System.out.println("=== Executor Framework ===");
        
        // 1. Fixed thread pool
        System.out.println("\n--- Fixed Thread Pool ---");
        ExecutorService fixedPool = Executors.newFixedThreadPool(3);
        
        List<Future<String>> fixedFutures = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            final int taskId = i;
            Future<String> future = fixedPool.submit(() -> {
                Thread.sleep(1000); // Simulate work
                return "Fixed pool task " + taskId + " completed by " + 
                       Thread.currentThread().getName();
            });
            fixedFutures.add(future);
        }
        
        for (Future<String> future : fixedFutures) {
            try {
                System.out.println(future.get());
            } catch (ExecutionException e) {
                System.err.println("Task failed: " + e.getCause().getMessage());
            }
        }
        
        fixedPool.shutdown();
        fixedPool.awaitTermination(5, TimeUnit.SECONDS);
        
        // 2. Cached thread pool
        System.out.println("\n--- Cached Thread Pool ---");
        ExecutorService cachedPool = Executors.newCachedThreadPool();
        
        List<Future<Integer>> cachedFutures = new ArrayList<>();
        for (int i = 0; i < 5; i++) {
            final int taskId = i;
            Future<Integer> future = cachedPool.submit(() -> {
                Thread.sleep(500 + (int)(Math.random() * 1000)); // Random execution time
                return taskId * taskId;
            });
            cachedFutures.add(future);
        }
        
        for (Future<Integer> future : cachedFutures) {
            try {
                System.out.println("Cached pool result: " + future.get());
            } catch (ExecutionException e) {
                System.err.println("Task failed: " + e.getCause().getMessage());
            }
        }
        
        cachedPool.shutdown();
        cachedPool.awaitTermination(5, TimeUnit.SECONDS);
        
        // 3. Scheduled thread pool
        System.out.println("\n--- Scheduled Thread Pool ---");
        ScheduledExecutorService scheduledPool = Executors.newScheduledThreadPool(2);
        
        // Schedule a task to run after a delay
        ScheduledFuture<String> delayedTask = scheduledPool.schedule(() -> {
            return "Delayed task executed at " + new Date();
        }, 2, TimeUnit.SECONDS);
        
        // Schedule a task to run periodically
        ScheduledFuture<?> periodicTask = scheduledPool.scheduleAtFixedRate(() -> {
            System.out.println("Periodic task executed at " + new Date() + 
                             " by " + Thread.currentThread().getName());
        }, 0, 1, TimeUnit.SECONDS);
        
        System.out.println("Scheduled tasks, waiting...");
        System.out.println(delayedTask.get()); // Wait for delayed task
        
        Thread.sleep(5000); // Let periodic task run
        periodicTask.cancel(false); // Cancel periodic task
        
        scheduledPool.shutdown();
        scheduledPool.awaitTermination(5, TimeUnit.SECONDS);
        
        // 4. Custom thread pool with monitoring
        System.out.println("\n--- Custom Monitored Thread Pool ---");
        ThreadPoolExecutor customExecutor = new ThreadPoolExecutor(
            2,           // core pool size
            4,           // maximum pool size
            60L,         // keep alive time
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(10), // work queue
            new CustomThreadFactory("CustomPool", Thread.NORM_PRIORITY, false),
            new CustomRejectionPolicy()
        );
        
        MonitoredExecutorService monitored = new MonitoredExecutorService(customExecutor);
        
        List<Future<Void>> monitoredTasks = new ArrayList<>();
        for (int i = 0; i < 15; i++) {
            final int taskId = i;
            Future<Void> future = monitored.submit(() -> {
                Thread.sleep(200 + (int)(Math.random() * 300));
                System.out.println("Monitored task " + taskId + " completed");
                if (taskId == 7) { // Simulate a failing task
                    throw new RuntimeException("Simulated failure in task " + taskId);
                }
                return null;
            });
            monitoredTasks.add(future);
        }
        
        // Wait for all tasks to complete
        for (Future<Void> future : monitoredTasks) {
            try {
                future.get();
            } catch (ExecutionException e) {
                System.err.println("Monitored task failed: " + e.getCause().getMessage());
            }
        }
        
        // Print monitoring statistics
        System.out.println("\nMonitoring Statistics:");
        System.out.println("Total tasks: " + monitored.getTaskCount());
        System.out.println("Completed tasks: " + monitored.getCompletedTaskCount());
        System.out.println("Failed tasks: " + monitored.getFailedTaskCount());
        System.out.println("Success rate: " + String.format("%.2f%%", monitored.getSuccessRate()));
        
        CustomRejectionPolicy rejectionPolicy = (CustomRejectionPolicy) customExecutor.getRejectedExecutionHandler();
        System.out.println("Rejected tasks: " + rejectionPolicy.getRejectionCount());
        
        monitored.shutdown();
        monitored.awaitTermination(5, TimeUnit.SECONDS);
    }
    
    public static void main(String[] args) throws InterruptedException {
        demonstrateExecutorFramework();
    }
}
```

### Advanced Executor Patterns

```java
// Advanced executor patterns and techniques
public class AdvancedExecutorPatterns {
    
    // ForkJoinPool for divide-and-conquer tasks
    public static class ForkJoinExample {
        
        // Recursive task for computing sum
        public static class SumTask extends RecursiveTask<Long> {
            private final long[] array;
            private final int start;
            private final int end;
            private static final int THRESHOLD = 1000;
            
            public SumTask(long[] array, int start, int end) {
                this.array = array;
                this.start = start;
                this.end = end;
            }
            
            @Override
            protected Long compute() {
                if (end - start <= THRESHOLD) {
                    // Sequential computation for small arrays
                    long sum = 0;
                    for (int i = start; i < end; i++) {
                        sum += array[i];
                    }
                    return sum;
                } else {
                    // Divide the task
                    int mid = (start + end) / 2;
                    SumTask leftTask = new SumTask(array, start, mid);
                    SumTask rightTask = new SumTask(array, mid, end);
                    
                    // Fork the right task and compute left task directly
                    rightTask.fork();
                    Long leftResult = leftTask.compute();
                    Long rightResult = rightTask.join();
                    
                    return leftResult + rightResult;
                }
            }
        }
        
        public static long parallelSum(long[] array) {
            ForkJoinPool forkJoinPool = new ForkJoinPool();
            try {
                SumTask task = new SumTask(array, 0, array.length);
                return forkJoinPool.invoke(task);
            } finally {
                forkJoinPool.shutdown();
            }
        }
    }
    
    // CompletionService for handling results as they complete
    public static class CompletionServiceExample {
        
        public void demonstrateCompletionService() throws InterruptedException, ExecutionException {
            ExecutorService executor = Executors.newFixedThreadPool(4);
            CompletionService<String> completionService = 
                new ExecutorCompletionService<>(executor);
            
            // Submit tasks
            for (int i = 0; i < 5; i++) {
                final int taskId = i;
                completionService.submit(() -> {
                    // Simulate variable execution times
                    Thread.sleep((5 - taskId) * 1000); // Tasks finish in reverse order
                    return "Task " + taskId + " result";
                });
            }
            
            System.out.println("Processing results as they complete:");
            for (int i = 0; i < 5; i++) {
                Future<String> future = completionService.take(); // Blocks until next result available
                System.out.println("Received: " + future.get());
            }
            
            executor.shutdown();
        }
    }
    
    // Work stealing with custom ForkJoinPool
    public static class WorkStealingExample {
        
        public void demonstrateWorkStealing() throws InterruptedException {
            ForkJoinPool customPool = new ForkJoinPool(4, // parallelism level
                ForkJoinPool.defaultForkJoinWorkerThreadFactory,
                (thread, throwable) -> System.err.println("Exception in work-stealing thread: " + throwable),
                true); // asyncMode = true for event-style processing
            
            List<Future<Integer>> futures = new ArrayList<>();
            
            for (int i = 0; i < 8; i++) {
                final int taskId = i;
                Future<Integer> future = customPool.submit(() -> {
                    // Simulate some work
                    Thread.sleep(1000 + (int)(Math.random() * 2000));
                    int result = taskId * taskId;
                    System.out.println("Task " + taskId + " completed by " + 
                                     Thread.currentThread().getName() + 
                                     ", result: " + result);
                    return result;
                });
                futures.add(future);
            }
            
            // Collect results
            for (Future<Integer> future : futures) {
                try {
                    System.out.println("Final result: " + future.get());
                } catch (ExecutionException e) {
                    System.err.println("Task failed: " + e.getCause().getMessage());
                }
            }
            
            customPool.shutdown();
            customPool.awaitTermination(10, TimeUnit.SECONDS);
        }
    }
    
    // Thread pool monitoring and management
    public static class ThreadPoolMonitoring {
        private final ThreadPoolExecutor executor;
        
        public ThreadPoolMonitoring(ThreadPoolExecutor executor) {
            this.executor = executor;
        }
        
        public void printStatistics() {
            System.out.println("\n--- Thread Pool Statistics ---");
            System.out.println("Pool size: " + executor.getPoolSize());
            System.out.println("Active threads: " + executor.getActiveCount());
            System.out.println("Core pool size: " + executor.getCorePoolSize());
            System.out.println("Maximum pool size: " + executor.getMaximumPoolSize());
            System.out.println("Largest pool size: " + executor.getLargestPoolSize());
            System.out.println("Completed tasks: " + executor.getCompletedTaskCount());
            System.out.println("Scheduled tasks: " + executor.getTaskCount());
            System.out.println("Queue size: " + executor.getQueue().size());
            System.out.println("Remaining capacity: " + executor.getQueue().remainingCapacity());
        }
        
        public void adjustPoolSize(int newCoreSize, int newMaxSize) {
            if (newCoreSize <= newMaxSize) {
                executor.setCorePoolSize(newCoreSize);
                executor.setMaximumPoolSize(newMaxSize);
                System.out.println("Pool size adjusted to: " + newCoreSize + "/" + newMaxSize);
            }
        }
    }
    
    // Dynamic workload handling
    public static class DynamicWorkloadHandler {
        private final ExecutorService executor;
        private final BlockingQueue<Runnable> workQueue;
        private final ThreadPoolMonitoring monitor;
        
        public DynamicWorkloadHandler() {
            this.workQueue = new LinkedBlockingQueue<>();
            ThreadPoolExecutor threadPool = new ThreadPoolExecutor(
                2, 8, 60L, TimeUnit.SECONDS, workQueue
            );
            this.executor = threadPool;
            this.monitor = new ThreadPoolMonitoring(threadPool);
        }
        
        public void handleDynamicWorkload() throws InterruptedException {
            // Simulate dynamic workload
            ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
            
            // Periodically adjust pool size based on queue size
            scheduler.scheduleAtFixedRate(() -> {
                int queueSize = workQueue.size();
                ThreadPoolExecutor pool = (ThreadPoolExecutor) executor;
                
                if (queueSize > 10 && pool.getPoolSize() < pool.getMaximumPoolSize()) {
                    // Increase pool size if queue is backing up
                    int newCore = Math.min(pool.getCorePoolSize() + 1, pool.getMaximumPoolSize());
                    pool.setCorePoolSize(newCore);
                    System.out.println("Increased pool size to: " + newCore + " due to queue size: " + queueSize);
                } else if (queueSize < 2 && pool.getPoolSize() > pool.getCorePoolSize()) {
                    // Decrease pool size if queue is small
                    int newCore = Math.max(pool.getCorePoolSize() - 1, 2);
                    pool.setCorePoolSize(newCore);
                    System.out.println("Decreased pool size to: " + newCore + " due to queue size: " + queueSize);
                }
                
                monitor.printStatistics();
            }, 0, 2, TimeUnit.SECONDS);
            
            // Submit work dynamically
            for (int i = 0; i < 20; i++) {
                final int workId = i;
                executor.submit(() -> {
                    try {
                        Thread.sleep(500 + (int)(Math.random() * 1000));
                        System.out.println("Work item " + workId + " completed");
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                });
                
                Thread.sleep(200); // Stagger submissions
            }
            
            Thread.sleep(10000); // Let it run
            scheduler.shutdown();
            executor.shutdown();
            executor.awaitTermination(5, TimeUnit.SECONDS);
        }
    }
    
    public static void demonstrateAdvancedPatterns() throws InterruptedException, ExecutionException {
        System.out.println("=== Advanced Executor Patterns ===");
        
        // ForkJoin example
        System.out.println("\n--- ForkJoin Example ---");
        long[] largeArray = new long[1000000];
        for (int i = 0; i < largeArray.length; i++) {
            largeArray[i] = i;
        }
        
        long startTime = System.currentTimeMillis();
        long sum = ForkJoinExample.parallelSum(largeArray);
        long endTime = System.currentTimeMillis();
        
        System.out.println("Parallel sum: " + sum);
        System.out.println("Time taken: " + (endTime - startTime) + " ms");
        
        // Completion service example
        System.out.println("\n--- Completion Service Example ---");
        CompletionServiceExample completionExample = new CompletionServiceExample();
        completionExample.demonstrateCompletionService();
        
        // Work stealing example
        System.out.println("\n--- Work Stealing Example ---");
        WorkStealingExample workStealingExample = new WorkStealingExample();
        workStealingExample.demonstrateWorkStealing();
        
        // Dynamic workload handling
        System.out.println("\n--- Dynamic Workload Handling ---");
        DynamicWorkloadHandler workloadHandler = new DynamicWorkloadHandler();
        workloadHandler.handleDynamicWorkload();
    }
    
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        demonstrateAdvancedPatterns();
    }
}
```

## Fork/Join Framework

The Fork/Join framework is designed for work that can be broken into smaller pieces recursively.

### Fork/Join Fundamentals

```java
// Fork/Join framework implementation and usage
public class ForkJoinFramework {
    
    // RecursiveTask for tasks that return a result
    public static class RecursiveMaxFinder extends RecursiveTask<Integer> {
        private final int[] array;
        private final int start;
        private final int end;
        private static final int THRESHOLD = 100;
        
        public RecursiveMaxFinder(int[] array, int start, int end) {
            this.array = array;
            this.start = start;
            this.end = end;
        }
        
        @Override
        protected Integer compute() {
            if (end - start <= THRESHOLD) {
                // Sequential computation for small arrays
                int max = Integer.MIN_VALUE;
                for (int i = start; i < end; i++) {
                    if (array[i] > max) {
                        max = array[i];
                    }
                }
                return max;
            } else {
                // Divide the task
                int mid = (start + end) / 2;
                RecursiveMaxFinder leftTask = new RecursiveMaxFinder(array, start, mid);
                RecursiveMaxFinder rightTask = new RecursiveMaxFinder(array, mid, end);
                
                // Fork the left task and compute right task directly
                leftTask.fork();
                Integer rightResult = rightTask.compute();
                Integer leftResult = leftTask.join();
                
                return Math.max(leftResult, rightResult);
            }
        }
    }
    
    // RecursiveAction for tasks that don't return a result
    public static class RecursiveArraySorter extends RecursiveAction {
        private final int[] array;
        private final int start;
        private final int end;
        private static final int THRESHOLD = 50;
        
        public RecursiveArraySorter(int[] array, int start, int end) {
            this.array = array;
            this.start = start;
            this.end = end;
        }
        
        @Override
        protected void compute() {
            if (end - start <= THRESHOLD) {
                // Use insertion sort for small arrays
                insertionSort(array, start, end);
            } else {
                // Divide the array and sort recursively
                int mid = (start + end) / 2;
                RecursiveArraySorter leftSorter = new RecursiveArraySorter(array, start, mid);
                RecursiveArraySorter rightSorter = new RecursiveArraySorter(array, mid, end);
                
                // Fork both tasks and wait for completion
                invokeAll(leftSorter, rightSorter);
                
                // Merge the sorted halves
                merge(array, start, mid, end);
            }
        }
        
        private void insertionSort(int[] arr, int start, int end) {
            for (int i = start + 1; i < end; i++) {
                int key = arr[i];
                int j = i - 1;
                while (j >= start && arr[j] > key) {
                    arr[j + 1] = arr[j];
                    j--;
                }
                arr[j + 1] = key;
            }
        }
        
        private void merge(int[] arr, int start, int mid, int end) {
            int[] temp = new int[end - start];
            int i = start, j = mid, k = 0;
            
            // Merge the two halves
            while (i < mid && j < end) {
                if (arr[i] <= arr[j]) {
                    temp[k++] = arr[i++];
                } else {
                    temp[k++] = arr[j++];
                }
            }
            
            // Copy remaining elements
            while (i < mid) temp[k++] = arr[i++];
            while (j < end) temp[k++] = arr[j++];
            
            // Copy back to original array
            System.arraycopy(temp, 0, arr, start, temp.length);
        }
    }
    
    // ForkJoinTask with custom computation
    public static class MatrixMultiplier extends RecursiveTask<double[][]> {
        private final double[][] a;
        private final double[][] b;
        private final int rowStart, rowEnd;
        private final int colStart, colEnd;
        private static final int THRESHOLD = 64; // Matrix size threshold
        
        public MatrixMultiplier(double[][] a, double[][] b, int rowStart, int rowEnd, 
                              int colStart, int colEnd) {
            this.a = a;
            this.b = b;
            this.rowStart = rowStart;
            this.rowEnd = rowEnd;
            this.colStart = colStart;
            this.colEnd = colEnd;
        }
        
        @Override
        protected double[][] compute() {
            int rows = rowEnd - rowStart;
            int cols = colEnd - colStart;
            
            if (rows * cols <= THRESHOLD) {
                // Compute matrix multiplication for small matrices
                double[][] result = new double[rows][cols];
                for (int i = 0; i < rows; i++) {
                    for (int j = 0; j < cols; j++) {
                        double sum = 0;
                        for (int k = 0; k < a[0].length; k++) {
                            sum += a[rowStart + i][k] * b[k][colStart + j];
                        }
                        result[i][j] = sum;
                    }
                }
                return result;
            } else {
                // Divide and conquer
                int midRow = rowStart + rows / 2;
                int midCol = colStart + cols / 2;
                
                // Create subtasks
                MatrixMultiplier topLeft = new MatrixMultiplier(a, b, rowStart, midRow, colStart, midCol);
                MatrixMultiplier topRight = new MatrixMultiplier(a, b, rowStart, midRow, midCol, colEnd);
                MatrixMultiplier bottomLeft = new MatrixMultiplier(a, b, midRow, rowEnd, colStart, midCol);
                MatrixMultiplier bottomRight = new MatrixMultiplier(a, b, midRow, rowEnd, midCol, colEnd);
                
                // Fork three tasks, compute one directly
                topLeft.fork();
                topRight.fork();
                bottomLeft.fork();
                
                double[][] brResult = bottomRight.compute();
                double[][] blResult = bottomLeft.join();
                double[][] trResult = topRight.join();
                double[][] tlResult = topLeft.join();
                
                // Combine results
                double[][] finalResult = new double[rowEnd - rowStart][colEnd - colStart];
                
                // Copy topLeft result
                for (int i = 0; i < tlResult.length; i++) {
                    System.arraycopy(tlResult[i], 0, finalResult[i], 0, tlResult[i].length);
                }
                
                // Copy topRight result
                for (int i = 0; i < trResult.length; i++) {
                    System.arraycopy(trResult[i], 0, finalResult[i], midCol - colStart, trResult[i].length);
                }
                
                // Copy bottomLeft result
                for (int i = 0; i < blResult.length; i++) {
                    System.arraycopy(blResult[i], 0, finalResult[midRow - rowStart + i], 0, blResult[i].length);
                }
                
                // Copy bottomRight result
                for (int i = 0; i < brResult.length; i++) {
                    System.arraycopy(brResult[i], 0, finalResult[midRow - rowStart + i], midCol - colStart, brResult[i].length);
                }
                
                return finalResult;
            }
        }
    }
    
    public static void demonstrateForkJoin() throws InterruptedException {
        System.out.println("=== Fork/Join Framework ===");
        
        // 1. Recursive max finder
        System.out.println("\n--- Recursive Max Finder ---");
        int[] randomArray = new int[100000];
        Random random = new Random();
        for (int i = 0; i < randomArray.length; i++) {
            randomArray[i] = random.nextInt(10000);
        }
        
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        
        long startTime = System.currentTimeMillis();
        RecursiveMaxFinder maxFinder = new RecursiveMaxFinder(randomArray, 0, randomArray.length);
        int max = forkJoinPool.invoke(maxFinder);
        long endTime = System.currentTimeMillis();
        
        System.out.println("Max value found: " + max);
        System.out.println("Time taken: " + (endTime - startTime) + " ms");
        
        // 2. Recursive array sorter
        System.out.println("\n--- Recursive Array Sorter ---");
        int[] unsortedArray = Arrays.copyOf(randomArray, randomArray.length);
        System.out.println("Original array sum: " + Arrays.stream(unsortedArray).mapToLong(i -> i).sum());
        
        startTime = System.currentTimeMillis();
        RecursiveArraySorter sorter = new RecursiveArraySorter(unsortedArray, 0, unsortedArray.length);
        forkJoinPool.invoke(sorter);
        endTime = System.currentTimeMillis();
        
        System.out.println("Sorted array sum: " + Arrays.stream(unsortedArray).mapToLong(i -> i).sum());
        System.out.println("Time taken: " + (endTime - startTime) + " ms");
        System.out.println("Array correctly sorted: " + isSorted(unsortedArray));
        
        // 3. Matrix multiplication (simplified example)
        System.out.println("\n--- Matrix Multiplication ---");
        double[][] matrixA = {{1, 2}, {3, 4}};
        double[][] matrixB = {{5, 6}, {7, 8}};
        
        MatrixMultiplier multiplier = new MatrixMultiplier(
            matrixA, matrixB, 0, matrixA.length, 0, matrixB[0].length);
        double[][] result = forkJoinPool.invoke(multiplier);
        
        System.out.println("Matrix multiplication result:");
        for (double[] row : result) {
            System.out.println(Arrays.toString(row));
        }
        
        forkJoinPool.shutdown();
        forkJoinPool.awaitTermination(5, TimeUnit.SECONDS);
    }
    
    private static boolean isSorted(int[] array) {
        for (int i = 1; i < array.length; i++) {
            if (array[i] < array[i-1]) {
                return false;
            }
        }
        return true;
    }
    
    public static void main(String[] args) throws InterruptedException {
        demonstrateForkJoin();
    }
}
```

### Fork/Join Optimization Techniques

```java
// Advanced Fork/Join optimization techniques
public class ForkJoinOptimization {
    
    // Custom ForkJoinWorkerThread for specialized behavior
    public static class CustomForkJoinWorkerThread extends ForkJoinWorkerThread {
        
        protected CustomForkJoinWorkerThread(ForkJoinPool pool) {
            super(pool);
        }
        
        @Override
        protected void onTermination(Throwable exception) {
            if (exception != null) {
                System.err.println("Custom worker thread terminated with exception: " + exception);
                exception.printStackTrace();
            }
        }
    }
    
    // Custom ForkJoinPool with custom worker threads
    public static class CustomForkJoinPool extends ForkJoinPool {
        
        public CustomForkJoinPool() {
            super(4, // parallelism level
                new ForkJoinWorkerThreadFactory() {
                    @Override
                    public ForkJoinWorkerThread newThread(ForkJoinPool pool) {
                        return new CustomForkJoinWorkerThread(pool);
                    }
                },
                (t, e) -> System.err.println("Exception in custom fork/join thread: " + e),
                true); // asyncMode
        }
    }
    
    // Optimized recursive task with work-stealing awareness
    public static class OptimizedRecursiveTask extends RecursiveTask<Long> {
        private final long[] data;
        private final int start;
        private final int end;
        private final int originalStart;
        private final int originalEnd;
        private static final int THRESHOLD = 500;
        
        public OptimizedRecursiveTask(long[] data, int start, int end) {
            this(data, start, end, start, end);
        }
        
        private OptimizedRecursiveTask(long[] data, int start, int end, int originalStart, int originalEnd) {
            this.data = data;
            this.start = start;
            this.end = end;
            this.originalStart = originalStart;
            this.originalEnd = originalEnd;
        }
        
        @Override
        protected Long compute() {
            int size = end - start;
            
            if (size <= THRESHOLD) {
                // Sequential computation
                long sum = 0;
                for (int i = start; i < end; i++) {
                    sum += data[i];
                }
                return sum;
            } else {
                // Adaptive division based on available parallelism
                int threads = getSurplusQueuedTaskCount() + 1;
                if (threads > 1 && size > THRESHOLD * 2) {
                    // Split into more pieces if work-stealing is possible
                    int mid = start + size / 2;
                    OptimizedRecursiveTask left = new OptimizedRecursiveTask(data, start, mid, originalStart, originalEnd);
                    OptimizedRecursiveTask right = new OptimizedRecursiveTask(data, mid, end, originalStart, originalEnd);
                    
                    left.fork();
                    long rightResult = right.compute();
                    long leftResult = left.join();
                    
                    return leftResult + rightResult;
                } else {
                    // Standard binary split
                    int mid = start + size / 2;
                    OptimizedRecursiveTask left = new OptimizedRecursiveTask(data, start, mid, originalStart, originalEnd);
                    OptimizedRecursiveTask right = new OptimizedRecursiveTask(data, mid, end, originalStart, originalEnd);
                    
                    invokeAll(left, right);
                    
                    return left.join() + right.join();
                }
            }
        }
        
        // Method to check if work-stealing is beneficial
        private boolean isWorkStealingBeneficial() {
            int queueSize = getQueuedTaskCount();
            int parallelism = getParallelism();
            return queueSize > parallelism * 2;
        }
    }
    
    // ForkJoinTask for parallel prefix sum (scan operation)
    public static class ParallelPrefixSum extends RecursiveTask<long[]> {
        private final long[] input;
        private final int start;
        private final int end;
        private static final int THRESHOLD = 1000;
        
        public ParallelPrefixSum(long[] input, int start, int end) {
            this.input = input;
            this.start = start;
            this.end = end;
        }
        
        @Override
        protected long[] compute() {
            if (end - start <= THRESHOLD) {
                // Sequential prefix sum
                long[] result = new long[end - start];
                result[0] = input[start];
                for (int i = 1; i < result.length; i++) {
                    result[i] = result[i-1] + input[start + i];
                }
                return result;
            } else {
                int mid = (start + end) / 2;
                
                ParallelPrefixSum leftTask = new ParallelPrefixSum(input, start, mid);
                ParallelPrefixSum rightTask = new ParallelPrefixSum(input, mid, end);
                
                leftTask.fork();
                long[] rightResult = rightTask.compute();
                long[] leftResult = leftTask.join();
                
                // Combine results: add the last element of left to all elements of right
                long carry = leftResult.length > 0 ? leftResult[leftResult.length - 1] : 0;
                for (int i = 0; i < rightResult.length; i++) {
                    rightResult[i] += carry;
                }
                
                // Combine left and right results
                long[] combined = new long[leftResult.length + rightResult.length];
                System.arraycopy(leftResult, 0, combined, 0, leftResult.length);
                System.arraycopy(rightResult, 0, combined, leftResult.length, rightResult.length);
                
                return combined;
            }
        }
    }
    
    // ForkJoinTask with cancellation support
    public static class CancellableRecursiveTask extends RecursiveTask<Integer> {
        private final int[] data;
        private final int start;
        private final int end;
        private final AtomicInteger cancellationSignal;
        private static final int THRESHOLD = 100;
        
        public CancellableRecursiveTask(int[] data, int start, int end, AtomicInteger cancellationSignal) {
            this.data = data;
            this.start = start;
            this.end = end;
            this.cancellationSignal = cancellationSignal;
        }
        
        @Override
        protected Integer compute() {
            // Check for cancellation at the beginning of each computation
            if (cancellationSignal.get() == 1) {
                return -1; // Cancelled result
            }
            
            if (end - start <= THRESHOLD) {
                // Sequential computation with cancellation check
                int max = Integer.MIN_VALUE;
                for (int i = start; i < end; i++) {
                    if (cancellationSignal.get() == 1) {
                        return -1; // Cancelled
                    }
                    if (data[i] > max) {
                        max = data[i];
                    }
                }
                return max;
            } else {
                int mid = (start + end) / 2;
                CancellableRecursiveTask leftTask = new CancellableRecursiveTask(data, start, mid, cancellationSignal);
                CancellableRecursiveTask rightTask = new CancellableRecursiveTask(data, mid, end, cancellationSignal);
                
                // Fork right task, compute left task
                rightTask.fork();
                Integer leftResult = leftTask.compute();
                
                if (cancellationSignal.get() == 1) {
                    // If cancelled, cancel the right task too
                    rightTask.cancel(false);
                    return -1;
                }
                
                Integer rightResult = rightTask.join();
                
                if (cancellationSignal.get() == 1) {
                    return -1;
                }
                
                return Math.max(leftResult, rightResult);
            }
        }
    }
    
    public static void demonstrateOptimizations() throws InterruptedException {
        System.out.println("=== Fork/Join Optimization Techniques ===");
        
        // 1. Custom ForkJoinPool
        System.out.println("\n--- Custom ForkJoinPool ---");
        CustomForkJoinPool customPool = new CustomForkJoinPool();
        
        long[] largeArray = new long[100000];
        for (int i = 0; i < largeArray.length; i++) {
            largeArray[i] = i;
        }
        
        OptimizedRecursiveTask optimizedTask = new OptimizedRecursiveTask(largeArray, 0, largeArray.length);
        long optimizedResult = customPool.invoke(optimizedTask);
        
        System.out.println("Optimized task result: " + optimizedResult);
        System.out.println("Expected sum: " + (long)(largeArray.length - 1) * largeArray.length / 2);
        
        // 2. Parallel prefix sum
        System.out.println("\n--- Parallel Prefix Sum ---");
        long[] prefixInput = {1, 2, 3, 4, 5, 6, 7, 8};
        System.out.println("Input: " + Arrays.toString(prefixInput));
        
        ParallelPrefixSum prefixTask = new ParallelPrefixSum(prefixInput, 0, prefixInput.length);
        long[] prefixResult = customPool.invoke(prefixTask);
        
        System.out.println("Prefix sum result: " + Arrays.toString(prefixResult));
        
        // 3. Cancellable task
        System.out.println("\n--- Cancellable Task ---");
        int[] cancellationData = new int[10000];
        for (int i = 0; i < cancellationData.length; i++) {
            cancellationData[i] = (int)(Math.random() * 1000);
        }
        
        AtomicInteger cancelSignal = new AtomicInteger(0);
        CancellableRecursiveTask cancellableTask = new CancellableRecursiveTask(
            cancellationData, 0, cancellationData.length, cancelSignal);
        
        // Start the task
        ForkJoinTask<Integer> future = customPool.submit(cancellableTask);
        
        // Cancel after some time
        Thread cancelThread = new Thread(() -> {
            try {
                Thread.sleep(100); // Wait 100ms
                cancelSignal.set(1);
                System.out.println("Cancellation signal sent");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        cancelThread.start();
        
        try {
            Integer result = future.get(2, TimeUnit.SECONDS);
            System.out.println("Cancellable task result: " + result);
        } catch (TimeoutException e) {
            System.out.println("Task timed out");
            future.cancel(true);
        } catch (ExecutionException e) {
            System.out.println("Task execution failed: " + e.getCause().getMessage());
        }
        
        customPool.shutdown();
        customPool.awaitTermination(5, TimeUnit.SECONDS);
    }
    
    public static void main(String[] args) throws InterruptedException {
        demonstrateOptimizations();
    }
}
```

## CompletableFuture and Async Programming

CompletableFuture provides powerful tools for asynchronous programming and composing asynchronous operations.

### CompletableFuture Fundamentals

```java
// CompletableFuture fundamentals and basic operations
public class CompletableFutureFundamentals {
    
    // Basic CompletableFuture operations
    public static void demonstrateBasicOperations() throws ExecutionException, InterruptedException {
        System.out.println("=== CompletableFuture Basic Operations ===");
        
        // 1. Completed future
        CompletableFuture<String> completedFuture = CompletableFuture.completedFuture("Hello World");
        System.out.println("Completed future: " + completedFuture.get());
        
        // 2. Run async task (no return value)
        CompletableFuture<Void> runAsyncFuture = CompletableFuture.runAsync(() -> {
            System.out.println("Running async task in thread: " + Thread.currentThread().getName());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            System.out.println("Async task completed");
        });
        
        runAsyncFuture.get(); // Wait for completion
        
        // 3. Supply async task (with return value)
        CompletableFuture<String> supplyAsyncFuture = CompletableFuture.supplyAsync(() -> {
            System.out.println("Supplying async value in thread: " + Thread.currentThread().getName());
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "Supplied value";
        });
        
        System.out.println("Supplied value: " + supplyAsyncFuture.get());
        
        // 4. Create future with custom executor
        ExecutorService customExecutor = Executors.newFixedThreadPool(2);
        CompletableFuture<Integer> customFuture = CompletableFuture.supplyAsync(() -> {
            System.out.println("Running in custom executor: " + Thread.currentThread().getName());
            return 42;
        }, customExecutor);
        
        System.out.println("Custom executor result: " + customFuture.get());
        customExecutor.shutdown();
    }
    
    // Chaining operations
    public static void demonstrateChaining() throws ExecutionException, InterruptedException {
        System.out.println("\n=== CompletableFuture Chaining ===");
        
        // thenApply: transform result
        CompletableFuture<String> chainedFuture = CompletableFuture.supplyAsync(() -> {
            System.out.println("Initial task in thread: " + Thread.currentThread().getName());
            return "Hello";
        })
        .thenApply(s -> {
            System.out.println("First transformation in thread: " + Thread.currentThread().getName());
            return s + " World";
        })
        .thenApply(s -> {
            System.out.println("Second transformation in thread: " + Thread.currentThread().getName());
            return s.toUpperCase();
        });
        
        System.out.println("Chained result: " + chainedFuture.get());
        
        // thenCompose: flatMap for futures (returns another CompletableFuture)
        CompletableFuture<String> composeFuture = CompletableFuture.supplyAsync(() -> {
            System.out.println("Compose initial task in thread: " + Thread.currentThread().getName());
            return "Data";
        })
        .thenCompose(data -> {
            System.out.println("Compose transformation in thread: " + Thread.currentThread().getName());
            // Return another CompletableFuture
            return CompletableFuture.supplyAsync(() -> data + " Processed");
        });
        
        System.out.println("Compose result: " + composeFuture.get());
        
        // thenAccept: consume result (no return)
        CompletableFuture<Void> acceptFuture = CompletableFuture.supplyAsync(() -> {
            return "Intermediate value";
        })
        .thenAccept(value -> {
            System.out.println("Accepting value: " + value + " in thread: " + Thread.currentThread().getName());
        });
        
        acceptFuture.get();
    }
    
    // Async transformations with different executors
    public static void demonstrateAsyncTransformations() throws ExecutionException, InterruptedException {
        System.out.println("\n=== Async Transformations ===");
        
        ExecutorService ioExecutor = Executors.newFixedThreadPool(4);
        ExecutorService cpuExecutor = Executors.newFixedThreadPool(2);
        
        CompletableFuture<String> asyncTransformFuture = CompletableFuture
            .supplyAsync(() -> {
                System.out.println("Initial supply in thread: " + Thread.currentThread().getName());
                return "Raw Data";
            }, cpuExecutor)
            .thenApplyAsync(value -> {
                System.out.println("First transform in thread: " + Thread.currentThread().getName());
                return value + " | Transformed";
            }, ioExecutor)
            .thenApplyAsync(value -> {
                System.out.println("Second transform in thread: " + Thread.currentThread().getName());
                return value + " | Final";
            }, cpuExecutor);
        
        System.out.println("Async transform result: " + asyncTransformFuture.get());
        
        ioExecutor.shutdown();
        cpuExecutor.shutdown();
    }
    
    // Error handling
    public static void demonstrateErrorHandling() {
        System.out.println("\n=== CompletableFuture Error Handling ===");
        
        // Handle exceptions with exceptionally
        CompletableFuture<String> errorFuture = CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Simulated error");
        })
        .exceptionally(throwable -> {
            System.out.println("Exception caught: " + throwable.getMessage());
            return "Default value after error";
        });
        
        try {
            System.out.println("Error handling result: " + errorFuture.get());
        } catch (Exception e) {
            e.printStackTrace();
        }
        
        // Handle exceptions with handle
        CompletableFuture<String> handleFuture = CompletableFuture.supplyAsync(() -> {
            if (Math.random() > 0.5) {
                throw new RuntimeException("Random error");
            }
            return "Success value";
        })
        .handle((result, throwable) -> {
            if (throwable != null) {
                System.out.println("Handle caught exception: " + throwable.getMessage());
                return "Handled error result";
            } else {
                System.out.println("Handle received result: " + result);
                return result + " | Handled";
            }
        });
        
        try {
            System.out.println("Handle result: " + handleFuture.get());
        } catch (Exception e) {
            e.printStackTrace();
        }
        
        // Handle with async execution
        CompletableFuture<String> asyncHandleFuture = CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Async error");
        })
        .handleAsync((result, throwable) -> {
            if (throwable != null) {
                System.out.println("Async handle caught: " + throwable.getMessage());
                return "Async error handled";
            }
            return result;
        }, Executors.newSingleThreadExecutor());
        
        try {
            System.out.println("Async handle result: " + asyncHandleFuture.get());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        demonstrateBasicOperations();
        demonstrateChaining();
        demonstrateAsyncTransformations();
        demonstrateErrorHandling();
    }
}
```

### Advanced CompletableFuture Operations

```java
// Advanced CompletableFuture operations and composition
public class AdvancedCompletableFuture {
    
    // Combining futures
    public static void demonstrateCombining() throws ExecutionException, InterruptedException {
        System.out.println("=== Advanced CompletableFuture Operations ===");
        
        // thenCombine: combine two futures
        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "Hello";
        });
        
        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "World";
        });
        
        CompletableFuture<String> combinedFuture = future1.thenCombine(future2, (s1, s2) -> s1 + " " + s2);
        System.out.println("Combined result: " + combinedFuture.get());
        
        // thenAcceptBoth: accept both results (no return)
        CompletableFuture<Void> acceptBothFuture = future1.thenAcceptBoth(future2, (s1, s2) -> {
            System.out.println("Accept both: " + s1 + " and " + s2);
        });
        acceptBothFuture.get();
        
        // runAfterBoth: run after both complete (no values passed)
        CompletableFuture<Void> runAfterBothFuture = future1.runAfterBoth(future2, () -> {
            System.out.println("Both futures completed, running after both");
        });
        runAfterBothFuture.get();
        
        // applyToEither: apply function to the first completing future
        CompletableFuture<String> future3 = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(2000); // Slower future
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "Slow Result";
        });
        
        CompletableFuture<String> future4 = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(100); // Faster future
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "Fast Result";
        });
        
        CompletableFuture<String> eitherFuture = future3.applyToEither(future4, result -> {
            return "First to complete: " + result;
        });
        System.out.println("Either result: " + eitherFuture.get());
        
        // acceptEither: accept from the first completing future
        CompletableFuture<Void> acceptEitherFuture = future3.acceptEither(future4, result -> {
            System.out.println("First to complete with accept: " + result);
        });
        acceptEitherFuture.get();
        
        // runAfterEither: run after either completes
        CompletableFuture<Void> runAfterEitherFuture = future3.runAfterEither(future4, () -> {
            System.out.println("Either future completed, running after either");
        });
        runAfterEitherFuture.get();
    }
    
    // Multiple futures coordination
    public static void demonstrateMultipleFutures() throws InterruptedException {
        System.out.println("\n=== Multiple Futures Coordination ===");
        
        // allOf: wait for all futures to complete
        List<CompletableFuture<String>> futuresList = new ArrayList<>();
        
        for (int i = 0; i < 5; i++) {
            final int taskId = i;
            CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(500 + (int)(Math.random() * 1000));
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                return "Task-" + taskId + "-Result";
            });
            futuresList.add(future);
        }
        
        CompletableFuture<Void> allOfFuture = CompletableFuture.allOf(
            futuresList.toArray(new CompletableFuture[0]));
        
        allOfFuture.thenRun(() -> {
            System.out.println("All tasks completed");
        });
        
        // Get results after all complete
        CompletableFuture<List<String>> allResults = allOfFuture.thenApply(v -> {
            return futuresList.stream()
                .map(f -> {
                    try {
                        return f.get();
                    } catch (Exception e) {
                        throw new RuntimeException(e);
                    }
                })
                .collect(Collectors.toList());
        });
        
        List<String> results = allResults.get();
        System.out.println("All results: " + results);
        
        // anyOf: wait for any future to complete
        System.out.println("\n--- AnyOf Example ---");
        List<CompletableFuture<String>> anyOfFutures = new ArrayList<>();
        
        for (int i = 0; i < 3; i++) {
            final int taskId = i;
            CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep((3 - taskId) * 1000); // First task finishes fastest
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                return "AnyOf-Task-" + taskId;
            });
            anyOfFutures.add(future);
        }
        
        CompletableFuture<Object> anyOfFuture = CompletableFuture.anyOf(
            anyOfFutures.toArray(new CompletableFuture[0]));
        
        Object anyResult = anyOfFuture.get();
        System.out.println("AnyOf result: " + anyResult);
    }
    
    // Timeout and cancellation
    public static void demonstrateTimeout() {
        System.out.println("\n=== Timeout and Cancellation ===");
        
        // Timeout with orTimeout
        CompletableFuture<String> timeoutFuture = CompletableFuture
            .supplyAsync(() -> {
                try {
                    Thread.sleep(3000); // This will timeout
                    return "This won't be reached";
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return "Interrupted";
                }
            })
            .orTimeout(1, TimeUnit.SECONDS)
            .exceptionally(throwable -> {
                if (throwable.getCause() instanceof TimeoutException) {
                    System.out.println("Operation timed out");
                    return "Timeout result";
                }
                return "Error: " + throwable.getMessage();
            });
        
        try {
            String result = timeoutFuture.get();
            System.out.println("Timeout result: " + result);
        } catch (Exception e) {
            System.out.println("Exception in timeout: " + e.getMessage());
        }
        
        // CompletableFuture with manual cancellation
        CompletableFuture<String> cancellableFuture = CompletableFuture.supplyAsync(() -> {
            try {
                for (int i = 0; i < 100; i++) {
                    Thread.sleep(100);
                    if (Thread.currentThread().isInterrupted()) {
                        System.out.println("Task was interrupted");
                        return "Cancelled";
                    }
                }
                return "Completed normally";
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return "Interrupted";
            }
        });
        
        // Cancel after 1 second
        Timer timer = new Timer();
        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                System.out.println("Cancelling future...");
                cancellableFuture.cancel(true);
            }
        }, 1000);
        
        try {
            String result = cancellableFuture.get(2, TimeUnit.SECONDS);
            System.out.println("Cancellable result: " + result);
        } catch (TimeoutException e) {
            System.out.println("Future didn't complete in time");
            cancellableFuture.cancel(true);
        } catch (Exception e) {
            System.out.println("Exception in cancellable future: " + e.getMessage());
        }
        
        timer.cancel();
    }
    
    // Advanced composition patterns
    public static void demonstrateAdvancedPatterns() throws ExecutionException, InterruptedException {
        System.out.println("\n=== Advanced Composition Patterns ===");
        
        // Retry pattern with CompletableFuture
        CompletableFuture<String> retryFuture = withRetry(() -> 
            CompletableFuture.supplyAsync(() -> {
                if (Math.random() > 0.7) { // 30% success rate
                    return "Success!";
                } else {
                    throw new RuntimeException("Simulated failure");
                }
            }), 3);
        
        try {
            String result = retryFuture.get();
            System.out.println("Retry pattern result: " + result);
        } catch (Exception e) {
            System.out.println("Retry pattern failed after all attempts: " + e.getCause().getMessage());
        }
        
        // Circuit breaker pattern (simplified)
        CompletableFuture<String> circuitBreakerFuture = withCircuitBreaker(() -> 
            CompletableFuture.supplyAsync(() -> {
                if (Math.random() > 0.9) { // 10% failure rate
                    throw new RuntimeException("Service error");
                }
                return "Service response";
            }));
        
        try {
            String result = circuitBreakerFuture.get();
            System.out.println("Circuit breaker result: " + result);
        } catch (Exception e) {
            System.out.println("Circuit breaker opened: " + e.getCause().getMessage());
        }
    }
    
    // Retry pattern implementation
    private static <T> CompletableFuture<T> withRetry(Supplier<CompletableFuture<T>> operation, int maxRetries) {
        CompletableFuture<T> future = operation.get();
        
        for (int i = 0; i < maxRetries; i++) {
            final int attempt = i + 1;
            future = future.exceptionally(throwable -> {
                if (attempt >= maxRetries) {
                    throw new RuntimeException("Retry failed after " + maxRetries + " attempts", throwable);
                }
                System.out.println("Attempt " + attempt + " failed, retrying...");
                return null;
            }).thenCompose(result -> {
                if (result == null) {
                    // Need to retry
                    return CompletableFuture.supplyAsync(() -> {
                        try {
                            Thread.sleep(100 * attempt); // Exponential backoff
                        } catch (InterruptedException e) {
                            Thread.currentThread().interrupt();
                        }
                        return null;
                    }).thenCompose(v -> operation.get());
                } else {
                    // Success, return the result
                    CompletableFuture<T> successful = new CompletableFuture<>();
                    successful.complete(result);
                    return successful;
                }
            });
        }
        
        return future;
    }
    
    // Circuit breaker pattern (simplified)
    private static final AtomicBoolean circuitOpen = new AtomicBoolean(false);
    private static final AtomicInteger failureCount = new AtomicInteger(0);
    private static final int FAILURE_THRESHOLD = 3;
    
    private static <T> CompletableFuture<T> withCircuitBreaker(Supplier<CompletableFuture<T>> operation) {
        if (circuitOpen.get()) {
            CompletableFuture<T> failed = new CompletableFuture<>();
            failed.completeExceptionally(new RuntimeException("Circuit breaker is OPEN"));
            return failed;
        }
        
        return operation.get()
            .whenComplete((result, throwable) -> {
                if (throwable != null) {
                    int failures = failureCount.incrementAndGet();
                    if (failures >= FAILURE_THRESHOLD) {
                        circuitOpen.set(true);
                        System.out.println("Circuit breaker OPENED due to failures");
                        
                        // Close circuit after delay
                        CompletableFuture.delayedExecutor(5, TimeUnit.SECONDS).execute(() -> {
                            circuitOpen.set(false);
                            failureCount.set(0);
                            System.out.println("Circuit breaker CLOSED after timeout");
                        });
                    }
                } else {
                    failureCount.set(0); // Reset on success
                }
            });
    }
    
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        demonstrateCombining();
        demonstrateMultipleFutures();
        demonstrateTimeout();
        demonstrateAdvancedPatterns();
    }
}
```

## Concurrent Collections

Concurrent collections provide thread-safe alternatives to traditional collections with better performance characteristics.

### Concurrent Collection Fundamentals

```java
// Concurrent collections and their usage patterns
public class ConcurrentCollections {
    
    // ConcurrentHashMap demonstration
    public static void demonstrateConcurrentHashMap() throws InterruptedException {
        System.out.println("=== ConcurrentHashMap ===");
        
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
        
        // Concurrent operations
        ExecutorService executor = Executors.newFixedThreadPool(10);
        List<Future<?>> futures = new ArrayList<>();
        
        // Multiple threads putting values
        for (int i = 0; i < 100; i++) {
            final int value = i;
            futures.add(executor.submit(() -> {
                String key = "key-" + (value % 20); // Multiple keys
                map.compute(key, (k, v) -> v == null ? 1 : v + 1);
            }));
        }
        
        // Wait for all puts to complete
        for (Future<?> future : futures) {
            future.get();
        }
        
        System.out.println("Map size: " + map.size());
        map.forEach((k, v) -> System.out.println(k + " = " + v));
        
        // Atomic operations
        map.putIfAbsent("new-key", 100);
        map.replace("key-5", 1000);
        
        // Compute operations
        map.compute("key-1", (k, v) -> v == null ? 1 : v + 10);
        map.merge("key-2", 5, Integer::sum);
        
        // Bulk operations
        int sum = map.reduceValues(4, Integer::sum); // Parallel sum of values
        System.out.println("Sum of all values: " + sum);
        
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);
    }
    
    // ConcurrentLinkedQueue demonstration
    public static void demonstrateConcurrentLinkedQueue() throws InterruptedException {
        System.out.println("\n=== ConcurrentLinkedQueue ===");
        
        ConcurrentLinkedQueue<String> queue = new ConcurrentLinkedQueue<>();
        
        // Producer threads
        Thread[] producers = new Thread[3];
        for (int i = 0; i < 3; i++) {
            final int producerId = i;
            producers[i] = new Thread(() -> {
                for (int j = 0; j < 10; j++) {
                    String item = "Producer-" + producerId + "-Item-" + j;
                    queue.offer(item);
                    System.out.println("Offered: " + item);
                    try {
                        Thread.sleep(50);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        return;
                    }
                }
            });
        }
        
        // Consumer threads
        Thread[] consumers = new Thread[2];
        for (int i = 0; i < 2; i++) {
            final int consumerId = i;
            consumers[i] = new Thread(() -> {
                for (int j = 0; j < 15; j++) {
                    String item = queue.poll(); // Non-blocking
                    if (item != null) {
                        System.out.println("Consumer-" + consumerId + " polled: " + item);
                    } else {
                        System.out.println("Consumer-" + consumerId + " found empty queue");
                    }
                    try {
                        Thread.sleep(75);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        return;
                    }
                }
            });
        }
        
        // Start all threads
        for (Thread producer : producers) producer.start();
        for (Thread consumer : consumers) consumer.start();
        
        // Wait for completion
        for (Thread producer : producers) producer.join();
        for (Thread consumer : consumers) consumer.join();
        
        System.out.println("Final queue size: " + queue.size());
    }
    
    // ConcurrentSkipListMap demonstration
    public static void demonstrateConcurrentSkipListMap() {
        System.out.println("\n=== ConcurrentSkipListMap ===");
        
        ConcurrentSkipListMap<Integer, String> skipListMap = new ConcurrentSkipListMap<>();
        
        // Populate with random data
        Random random = new Random();
        for (int i = 0; i < 20; i++) {
            int key = random.nextInt(100);
            skipListMap.put(key, "Value-" + key);
        }
        
        System.out.println("Skip list map contents:");
        skipListMap.forEach((k, v) -> System.out.println(k + " -> " + v));
        
        // Range operations
        System.out.println("\nSubMap (20-50):");
        ConcurrentNavigableMap<Integer, String> subMap = skipListMap.subMap(20, 50);
        subMap.forEach((k, v) -> System.out.println(k + " -> " + v));
        
        // Tail map
        System.out.println("\nTailMap (starting from 50):");
        ConcurrentNavigableMap<Integer, String> tailMap = skipListMap.tailMap(50);
        tailMap.forEach((k, v) -> System.out.println(k + " -> " + v));
        
        // Navigation operations
        System.out.println("\nNavigation operations:");
        System.out.println("First entry: " + skipListMap.firstEntry());
        System.out.println("Last entry: " + skipListMap.lastEntry());
        System.out.println("Higher key than 50: " + skipListMap.higherKey(50));
        System.out.println("Lower key than 50: " + skipListMap.lowerKey(50));
        System.out.println("Ceiling key for 55: " + skipListMap.ceilingKey(55));
        System.out.println("Floor key for 55: " + skipListMap.floorKey(55));
    }
    
    // CopyOnWriteArrayList demonstration
    public static void demonstrateCopyOnWriteArrayList() {
        System.out.println("\n=== CopyOnWriteArrayList ===");
        
        CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>();
        
        // Add initial elements
        cowList.addAll(Arrays.asList("A", "B", "C", "D", "E"));
        
        System.out.println("Initial list: " + cowList);
        
        // Create iterator while modification happens
        Iterator<String> iterator = cowList.iterator();
        
        // Modify the list (this creates a new internal array)
        cowList.add("F");
        cowList.add("G");
        cowList.remove("C");
        
        System.out.println("Modified list: " + cowList);
        
        // Iterator still sees original snapshot
        System.out.println("Iterator sees: ");
        iterator.forEachRemaining(item -> System.out.print(item + " "));
        System.out.println();
        
        // Demonstrate thread safety
        ExecutorService executor = Executors.newFixedThreadPool(5);
        
        // Multiple threads reading
        for (int i = 0; i < 3; i++) {
            executor.submit(() -> {
                List<String> snapshot = new ArrayList<>(cowList);
                System.out.println("Reader thread sees: " + snapshot);
            });
        }
        
        // One thread modifying
        executor.submit(() -> {
            for (int i = 0; i < 5; i++) {
                cowList.add("New-" + i);
                System.out.println("Added New-" + i);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        });
        
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);
        
        System.out.println("Final list: " + cowList);
    }
    
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        demonstrateConcurrentHashMap();
        demonstrateConcurrentLinkedQueue();
        demonstrateConcurrentSkipListMap();
        demonstrateCopyOnWriteArrayList();
    }
}
```

### Advanced Concurrent Collection Patterns

```java
// Advanced concurrent collection usage patterns
public class AdvancedConcurrentCollections {
    
    // BlockingQueue implementations
    public static void demonstrateBlockingQueues() throws InterruptedException {
        System.out.println("=== BlockingQueue Implementations ===");
        
        // ArrayBlockingQueue - bounded, blocking
        System.out.println("\n--- ArrayBlockingQueue ---");
        BlockingQueue<String> arrayQueue = new ArrayBlockingQueue<>(3);
        
        Thread producer = new Thread(() -> {
            try {
                for (int i = 0; i < 5; i++) {
                    String item = "ArrayItem-" + i;
                    System.out.println("Producing: " + item);
                    arrayQueue.put(item); // Blocks if queue is full
                    System.out.println("Put successful: " + item);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        Thread consumer = new Thread(() -> {
            try {
                for (int i = 0; i < 5; i++) {
                    String item = arrayQueue.take(); // Blocks if queue is empty
                    System.out.println("Consumed: " + item);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        consumer.start();
        Thread.sleep(100);
        producer.start();
        
        producer.join();
        consumer.join();
        
        // LinkedBlockingQueue - optionally bounded
        System.out.println("\n--- LinkedBlockingQueue ---");
        BlockingQueue<String> linkedQueue = new LinkedBlockingQueue<>(2);
        
        ExecutorService executor = Executors.newFixedThreadPool(4);
        
        // Multiple producers
        for (int i = 0; i < 2; i++) {
            final int producerId = i;
            executor.submit(() -> {
                try {
                    for (int j = 0; j < 3; j++) {
                        String item = "LinkedProducer-" + producerId + "-Item-" + j;
                        boolean added = linkedQueue.offer(item, 1, TimeUnit.SECONDS);
                        if (added) {
                            System.out.println("Linked queue added: " + item);
                        } else {
                            System.out.println("Linked queue add timed out: " + item);
                        }
                        Thread.sleep(200);
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        
        // Multiple consumers
        for (int i = 0; i < 2; i++) {
            final int consumerId = i;
            executor.submit(() -> {
                try {
                    while (!Thread.currentThread().isInterrupted()) {
                        String item = linkedQueue.poll(500, TimeUnit.MILLISECONDS);
                        if (item != null) {
                            System.out.println("Linked consumer-" + consumerId + " got: " + item);
                        } else {
                            System.out.println("Linked consumer-" + consumerId + " timeout, exiting");
                            break;
                        }
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        
        Thread.sleep(3000);
        executor.shutdownNow();
        
        // PriorityBlockingQueue - unbounded priority queue
        System.out.println("\n--- PriorityBlockingQueue ---");
        BlockingQueue<Integer> priorityQueue = new PriorityBlockingQueue<>();
        
        // Add elements
        List<Integer> elements = Arrays.asList(3, 1, 4, 1, 5, 9, 2, 6);
        priorityQueue.addAll(elements);
        
        System.out.println("Priority queue contents: " + priorityQueue);
        
        // Consume in priority order
        System.out.println("Depleting priority queue:");
        while (!priorityQueue.isEmpty()) {
            Integer item = priorityQueue.poll();
            System.out.print(item + " ");
        }
        System.out.println();
        
        // DelayQueue - elements become available after specified delay
        System.out.println("\n--- DelayQueue ---");
        DelayQueue<DelayedElement> delayQueue = new DelayQueue<>();
        
        // Add delayed elements
        for (int i = 0; i < 3; i++) {
            delayQueue.put(new DelayedElement("DelayedItem-" + i, i * 1000)); // 0, 1, 2 seconds
        }
        
        // Consumer that waits for elements to become available
        Thread delayConsumer = new Thread(() -> {
            try {
                for (int i = 0; i < 3; i++) {
                    DelayedElement element = delayQueue.take();
                    System.out.println("DelayQueue consumed: " + element);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        long startTime = System.currentTimeMillis();
        delayConsumer.start();
        delayConsumer.join();
        long endTime = System.currentTimeMillis();
        
        System.out.println("DelayQueue processing took: " + (endTime - startTime) + " ms");
    }
    
    // Delayed element for DelayQueue
    static class DelayedElement implements Delayed {
        private final String data;
        private final long startTime;
        private final long delayInMilliseconds;
        
        public DelayedElement(String data, long delayInMilliseconds) {
            this.data = data;
            this.delayInMilliseconds = delayInMilliseconds;
            this.startTime = System.currentTimeMillis();
        }
        
        @Override
        public long getDelay(TimeUnit unit) {
            long diff = (startTime + delayInMilliseconds) - System.currentTimeMillis();
            return unit.convert(diff, TimeUnit.MILLISECONDS);
        }
        
        @Override
        public int compareTo(Delayed other) {
            if (other == this) return 0;
            long d = getDelay(TimeUnit.MILLISECONDS) - other.getDelay(TimeUnit.MILLISECONDS);
            return d == 0 ? 0 : (d < 0 ? -1 : 1);
        }
        
        @Override
        public String toString() {
            return data + " (delayed " + delayInMilliseconds + "ms)";
        }
    }
    
    // SynchronousQueue demonstration
    public static void demonstrateSynchronousQueue() throws InterruptedException {
        System.out.println("\n=== SynchronousQueue ===");
        
        SynchronousQueue<String> syncQueue = new SynchronousQueue<>();
        
        // Producer thread
        Thread producer = new Thread(() -> {
            try {
                for (int i = 0; i < 3; i++) {
                    String item = "SyncItem-" + i;
                    System.out.println("Producer waiting to put: " + item);
                    syncQueue.put(item); // Blocks until consumer takes
                    System.out.println("Producer successfully put: " + item);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        // Consumer thread
        Thread consumer = new Thread(() -> {
            try {
                for (int i = 0; i < 3; i++) {
                    System.out.println("Consumer waiting to take...");
                    String item = syncQueue.take(); // Blocks until producer puts
                    System.out.println("Consumer took: " + item);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        consumer.start();
        Thread.sleep(100);
        producer.start();
        
        producer.join();
        consumer.join();
    }
    
    // TransferQueue demonstration
    public static void demonstrateTransferQueue() throws InterruptedException {
        System.out.println("\n=== TransferQueue (LinkedTransferQueue) ===");
        
        TransferQueue<String> transferQueue = new LinkedTransferQueue<>();
        
        // Producer that transfers directly or buffers
        Thread transferProducer = new Thread(() -> {
            try {
                for (int i = 0; i < 3; i++) {
                    String item = "TransferItem-" + i;
                    
                    System.out.println("Producer trying to transfer: " + item);
                    boolean transferred = transferQueue.tryTransfer(item);
                    if (transferred) {
                        System.out.println("Item transferred directly: " + item);
                    } else {
                        System.out.println("Item added to queue: " + item);
                        transferQueue.put(item); // Add to queue if no immediate consumer
                    }
                    
                    Thread.sleep(1000);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        // Consumer with transfer
        Thread transferConsumer = new Thread(() -> {
            try {
                Thread.sleep(500); // Start consumer after producer begins
                for (int i = 0; i < 3; i++) {
                    System.out.println("Consumer about to take...");
                    String item = transferQueue.take();
                    System.out.println("Consumer got: " + item);
                    
                    // Send response back
                    String response = "Response-to-" + item;
                    transferQueue.transfer(response);
                    System.out.println("Consumer transferred response: " + response);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        // Response consumer
        Thread responseConsumer = new Thread(() -> {
            try {
                for (int i = 0; i < 3; i++) {
                    String response = transferQueue.take();
                    System.out.println("Response consumer got: " + response);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        responseConsumer.start();
        transferConsumer.start();
        Thread.sleep(100);
        transferProducer.start();
        
        transferProducer.join();
        transferConsumer.join();
        responseConsumer.join();
    }
    
    // Concurrent collection utilities and best practices
    public static void demonstrateCollectionUtilities() {
        System.out.println("\n=== Concurrent Collection Utilities ===");
        
        // Using ConcurrentHashMap as a concurrent cache
        ConcurrentHashMap<String, Object> cache = new ConcurrentHashMap<>();
        
        // Safe computeIfAbsent for lazy initialization
        Object result1 = cache.computeIfAbsent("key1", k -> {
            System.out.println("Computing value for: " + k);
            return "Computed-" + k;
        });
        
        // This won't recompute since key1 already exists
        Object result2 = cache.computeIfAbsent("key1", k -> "Should not compute");
        
        System.out.println("First access: " + result1);
        System.out.println("Second access: " + result2);
        
        // Safe conditional operations
        cache.putIfAbsent("key2", "Value2");
        boolean replaced = cache.replace("key2", "Value2", "UpdatedValue2");
        System.out.println("Replace successful: " + replaced);
        
        // Bulk operations with parallelism
        ConcurrentHashMap<String, Integer> numbers = new ConcurrentHashMap<>();
        for (int i = 1; i <= 20; i++) {
            numbers.put("num" + i, i);
        }
        
        // Parallel reduction
        int sum = numbers.reduceValues(4, Integer::sum);
        System.out.println("Sum of values: " + sum);
        
        // Parallel transformation
        ConcurrentHashMap<String, String> transformed = numbers.reduceEntries(4, (entry) -> 
            entry.getKey() + "=" + entry.getValue(), (s1, s2) -> s1 + "," + s2);
        System.out.println("Some transformed entries: " + transformed);
        
        // Key-based parallel operations
        Set<String> evenKeys = numbers.reduceKeys(4, (key) -> {
            if (key.startsWith("num") && Integer.parseInt(key.substring(3)) % 2 == 0) {
                return key;
            }
            return null;
        }, (s1, s2) -> s1 != null ? s1 : s2);
        System.out.println("Even number keys: " + evenKeys);
    }
    
    public static void main(String[] args) throws InterruptedException {
        demonstrateBlockingQueues();
        demonstrateSynchronousQueue();
        demonstrateTransferQueue();
        demonstrateCollectionUtilities();
    }
}
```

## Lock-Free Data Structures

Implementing high-performance, lock-free data structures that provide thread-safe access without explicit locking.

### Lock-Free List Implementation

```java
// Lock-free data structures implementation
public class LockFreeDataStructures {
    
    // Lock-free linked list node
    static class Node<T> {
        final T data;
        volatile Node<T> next;
        volatile boolean marked; // For deletion
        
        Node(T data) {
            this.data = data;
        }
    }
    
    // Lock-free linked list implementation
    public static class LockFreeLinkedList<T> {
        private final Node<T> head;
        private final Node<T> tail;
        
        public LockFreeLinkedList() {
            head = new Node<>(null); // Sentinel head
            tail = new Node<>(null); // Sentinel tail
            head.next = tail;
        }
        
        public boolean add(T item) {
            Node<T> newNode = new Node<>(item);
            
            while (true) {
                Node<T> prev = head;
                Node<T> curr = head.next;
                
                // Find insertion point and remove marked nodes
                while (curr != tail) {
                    boolean currMarked = curr.marked;
                    if (currMarked) {
                        Node<T> succ = curr.next;
                        if (prev.next.compareAndSet(curr, succ)) {
                            curr = succ;
                            continue;
                        } else {
                            break; // Retry from start
                        }
                    }
                    
                    if (curr.data != null && ((Comparable<T>) curr.data).compareTo(item) >= 0) {
                        break; // Found insertion point
                    }
                    
                    prev = curr;
                    curr = curr.next;
                }
                
                newNode.next = curr;
                if (prev.next.compareAndSet(curr, newNode)) {
                    return true; // Successfully inserted
                }
                // Otherwise, retry
            }
        }
        
        public boolean remove(T item) {
            while (true) {
                Node<T> prev = head;
                Node<T> curr = head.next;
                
                // Find node to remove
                while (curr != tail && curr.data != null && 
                       ((Comparable<T>) curr.data).compareTo(item) < 0) {
                    prev = curr;
                    curr = curr.next;
                }
                
                if (curr == tail || curr.data == null || 
                    ((Comparable<T>) curr.data).compareTo(item) != 0) {
                    return false; // Item not found
                }
                
                // Mark node for deletion
                Node<T> succ = curr.next;
                if (!curr.marked) {
                    if (curr.next.compareAndSet(succ, succ)) { // Mark by replacing next with itself
                        curr.marked = true;
                    }
                }
                
                // Remove from list
                if (prev.next.compareAndSet(curr, succ)) {
                    return true; // Successfully removed
                }
            }
        }
        
        public boolean contains(T item) {
            Node<T> curr = head.next;
            
            while (curr != tail && curr.data != null) {
                if (!curr.marked && ((Comparable<T>) curr.data).compareTo(item) == 0) {
                    return true;
                }
                curr = curr.next;
            }
            
            return false;
        }
        
        public List<T> toList() {
            List<T> result = new ArrayList<>();
            Node<T> curr = head.next;
            
            while (curr != tail && curr.data != null) {
                if (!curr.marked) {
                    result.add(curr.data);
                }
                curr = curr.next;
            }
            
            return result;
        }
    }
    
    // Lock-free queue implementation (Michael-Scott algorithm)
    public static class LockFreeQueue<T> {
        private final AtomicReference<Node<T>> head;
        private final AtomicReference<Node<T>> tail;
        
        public LockFreeQueue() {
            Node<T> dummy = new Node<>(null);
            head = new AtomicReference<>(dummy);
            tail = new AtomicReference<>(dummy);
        }
        
        public void enqueue(T item) {
            Node<T> newNode = new Node<>(item);
            
            while (true) {
                Node<T> curTail = tail.get();
                Node<T> tailNext = curTail.next.get();
                
                if (curTail == tail.get()) {
                    if (tailNext != null) {
                        // Another thread has enqueued, advance tail
                        tail.compareAndSet(curTail, tailNext);
                    } else {
                        // Try to link new node
                        if (curTail.next.compareAndSet(null, newNode)) {
                            // Try to advance tail
                            tail.compareAndSet(curTail, newNode);
                            return;
                        }
                    }
                }
            }
        }
        
        public T dequeue() {
            while (true) {
                Node<T> curHead = head.get();
                Node<T> curTail = tail.get();
                Node<T> headNext = curHead.next.get();
                
                if (curHead == head.get()) {
                    if (curHead == curTail) {
                        if (headNext == null) {
                            return null; // Queue is empty
                        }
                        // Tail is falling behind, advance it
                        tail.compareAndSet(curTail, headNext);
                    } else {
                        if (headNext == null) {
                            return null; // Queue is empty
                        }
                        
                        T data = headNext.data;
                        if (head.compareAndSet(curHead, headNext)) {
                            return data;
                        }
                    }
                }
            }
        }
        
        public boolean isEmpty() {
            Node<T> curHead = head.get();
            Node<T> curTail = tail.get();
            Node<T> headNext = curHead.next.get();
            
            if (curHead == curTail) {
                return headNext == null;
            } else {
                return false;
            }
        }
    }
    
    // Lock-free stack implementation
    public static class LockFreeStack<T> {
        private final AtomicReference<Node<T>> top = new AtomicReference<>();
        
        public void push(T item) {
            Node<T> newNode = new Node<>(item);
            Node<T> currentTop;
            
            do {
                currentTop = top.get();
                newNode.next = currentTop;
            } while (!top.compareAndSet(currentTop, newNode));
        }
        
        public T pop() {
            Node<T> currentTop;
            Node<T> newTop;
            
            do {
                currentTop = top.get();
                if (currentTop == null) {
                    return null;
                }
                newTop = currentTop.next;
            } while (!top.compareAndSet(currentTop, newTop));
            
            return currentTop.data;
        }
        
        public boolean isEmpty() {
            return top.get() == null;
        }
        
        public int size() {
            int count = 0;
            Node<T> current = top.get();
            while (current != null) {
                count++;
                current = current.next;
            }
            return count;
        }
    }
    
    // Hazard pointer implementation for safe memory reclamation
    public static class HazardPointer<T> {
        private static final int MAX_HAZARD_POINTERS = 100;
        private final AtomicReferenceArray<Node<T>> hazardPointers;
        private final AtomicInteger nextSlot = new AtomicInteger(0);
        
        public HazardPointer() {
            this.hazardPointers = new AtomicReferenceArray<>(MAX_HAZARD_POINTERS);
        }
        
        public Node<T> acquire(Node<T> ptr) {
            int slot = nextSlot.getAndIncrement() % MAX_HAZARD_POINTERS;
            hazardPointers.set(slot, ptr);
            return ptr;
        }
        
        public void release(Node<T> ptr) {
            for (int i = 0; i < MAX_HAZARD_POINTERS; i++) {
                if (hazardPointers.get(i) == ptr) {
                    hazardPointers.set(i, null);
                    break;
                }
            }
        }
        
        public boolean isHazardous(Node<T> ptr) {
            for (int i = 0; i < MAX_HAZARD_POINTERS; i++) {
                if (hazardPointers.get(i) == ptr) {
                    return true;
                }
            }
            return false;
        }
    }
    
    public static void demonstrateLockFreeStructures() throws InterruptedException {
        System.out.println("=== Lock-Free Data Structures ===");
        
        // Lock-free linked list
        System.out.println("\n--- Lock-Free Linked List ---");
        LockFreeLinkedList<Integer> sortedList = new LockFreeLinkedList<>();
        
        // Multiple threads adding elements
        Thread[] addThreads = new Thread[5];
        for (int i = 0; i < 5; i++) {
            final int threadId = i;
            addThreads[i] = new Thread(() -> {
                for (int j = 0; j < 10; j++) {
                    int value = threadId * 10 + j;
                    sortedList.add(value);
                    System.out.println("Thread " + threadId + " added: " + value);
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        return;
                    }
                }
            });
        }
        
        Thread[] removeThreads = new Thread[2];
        for (int i = 0; i < 2; i++) {
            final int threadId = i;
            removeThreads[i] = new Thread(() -> {
                for (int j = 0; j < 5; j++) {
                    int value = threadId * 10 + j;
                    boolean removed = sortedList.remove(value);
                    System.out.println("Thread " + threadId + " removed " + value + ": " + removed);
                    try {
                        Thread.sleep(20);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        return;
                    }
                }
            });
        }
        
        // Start all threads
        for (Thread t : addThreads) t.start();
        for (Thread t : removeThreads) t.start();
        
        // Wait for completion
        for (Thread t : addThreads) t.join();
        for (Thread t : removeThreads) t.join();
        
        System.out.println("Final sorted list: " + sortedList.toList());
        
        // Lock-free queue
        System.out.println("\n--- Lock-Free Queue ---");
        LockFreeQueue<String> queue = new LockFreeQueue<>();
        
        Thread producer = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                String item = "QueueItem-" + i;
                queue.enqueue(item);
                System.out.println("Enqueued: " + item);
                try {
                    Thread.sleep(50);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        });
        
        Thread consumer = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                String item = queue.dequeue();
                if (item != null) {
                    System.out.println("Dequeued: " + item);
                } else {
                    System.out.println("Queue empty");
                }
                try {
                    Thread.sleep(75);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        });
        
        consumer.start();
        Thread.sleep(25); // Start consumer first
        producer.start();
        
        producer.join();
        consumer.join();
        
        System.out.println("Queue empty: " + queue.isEmpty());
        
        // Lock-free stack
        System.out.println("\n--- Lock-Free Stack ---");
        LockFreeStack<Integer> stack = new LockFreeStack<>();
        
        Thread stackProducer = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                stack.push(i);
                System.out.println("Pushed: " + i);
                try {
                    Thread.sleep(30);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        });
        
        Thread stackConsumer = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                Integer item = stack.pop();
                if (item != null) {
                    System.out.println("Popped: " + item);
                } else {
                    System.out.println("Stack empty");
                }
                try {
                    Thread.sleep(50);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        });
        
        stackProducer.start();
        Thread.sleep(10);
        stackConsumer.start();
        
        stackProducer.join();
        stackConsumer.join();
        
        System.out.println("Final stack size: " + stack.size());
        System.out.println("Stack empty: " + stack.isEmpty());
    }
    
    public static void main(String[] args) throws InterruptedException {
        demonstrateLockFreeStructures();
    }
}
```

## Performance Optimization and Best Practices

Essential techniques for optimizing concurrent applications and avoiding common pitfalls.

### Performance Optimization Techniques

```java
// Performance optimization and best practices for concurrent programming
public class ConcurrencyPerformanceOptimization {
    
    // False sharing prevention
    public static class FalseSharingExample {
        // Without padding - prone to false sharing
        public static class CounterWithoutPadding {
            volatile long counter1;
            volatile long counter2;
            volatile long counter3;
            volatile long counter4;
        }
        
        // With padding to prevent false sharing
        public static class PaddedCounter {
            volatile long counter1;
            private long p1, p2, p3, p4, p5, p6, p7; // Padding
            volatile long counter2;
            private long p8, p9, p10, p11, p12, p13, p14; // Padding
            volatile long counter3;
            private long p15, p16, p17, p18, p19, p20, p21; // Padding
            volatile long counter4;
        }
        
        public static void demonstrateFalseSharing() throws InterruptedException {
            System.out.println("=== False Sharing Prevention ===");
            
            int iterations = 10_000_000;
            
            // Test without padding
            CounterWithoutPadding unpadded = new CounterWithoutPadding();
            long startTime = System.nanoTime();
            
            Thread[] threads = new Thread[4];
            for (int i = 0; i < 4; i++) {
                final int threadId = i;
                threads[i] = new Thread(() -> {
                    for (int j = 0; j < iterations / 4; j++) {
                        switch (threadId) {
                            case 0: unpadded.counter1++; break;
                            case 1: unpadded.counter2++; break;
                            case 2: unpadded.counter3++; break;
                            case 3: unpadded.counter4++; break;
                        }
                    }
                });
            }
            
            for (Thread t : threads) t.start();
            for (Thread t : threads) t.join();
            
            long unpaddedTime = System.nanoTime() - startTime;
            System.out.println("Time without padding: " + unpaddedTime / 1_000_000 + " ms");
            
            // Test with padding
            PaddedCounter padded = new PaddedCounter();
            startTime = System.nanoTime();
            
            for (int i = 0; i < 4; i++) {
                final int threadId = i;
                threads[i] = new Thread(() -> {
                    for (int j = 0; j < iterations / 4; j++) {
                        switch (threadId) {
                            case 0: padded.counter1++; break;
                            case 1: padded.counter2++; break;
                            case 2: padded.counter3++; break;
                            case 3: padded.counter4++; break;
                        }
                    }
                });
            }
            
            for (Thread t : threads) t.start();
            for (Thread t : threads) t.join();
            
            long paddedTime = System.nanoTime() - startTime;
            System.out.println("Time with padding: " + paddedTime / 1_000_000 + " ms");
            System.out.println("Improvement: " + (double) unpaddedTime / paddedTime + "x");
        }
    }
    
    // Contention reduction techniques
    public static class ContentionReduction {
        
        // Using striped locks for better concurrency
        public static class StripedCounter {
            private final AtomicIntegerArray counters;
            private final int stripeCount;
            
            public StripedCounter(int stripeCount) {
                this.stripeCount = stripeCount;
                this.counters = new AtomicIntegerArray(stripeCount);
            }
            
            public void increment() {
                int stripe = (int) (Thread.currentThread().getId() % stripeCount);
                counters.incrementAndGet(stripe);
            }
            
            public long sum() {
                long sum = 0;
                for (int i = 0; i < stripeCount; i++) {
                    sum += counters.get(i);
                }
                return sum;
            }
        }
        
        // Using LongAdder for high-contention scenarios
        public static class LongAdderExample {
            private final LongAdder adder = new LongAdder();
            private final AtomicLong atomicLong = new AtomicLong(0);
            private final volatile long volatileLong = 0L;
            
            public void addToAdder(long value) {
                adder.add(value);
            }
            
            public void addToAtomic(long value) {
                atomicLong.addAndGet(value);
            }
            
            public void addToVolatile(long value) {
                // Not thread-safe, just for comparison
            }
            
            public long getAdderValue() {
                return adder.sum();
            }
            
            public long getAtomicValue() {
                return atomicLong.get();
            }
        }
        
        public static void demonstrateContentionReduction() throws InterruptedException {
            System.out.println("\n=== Contention Reduction ===");
            
            int numThreads = 8;
            int iterations = 1_000_000;
            
            // Test striped counter
            StripedCounter stripedCounter = new StripedCounter(numThreads);
            Thread[] threads = new Thread[numThreads];
            
            long startTime = System.nanoTime();
            
            for (int i = 0; i < numThreads; i++) {
                threads[i] = new Thread(() -> {
                    for (int j = 0; j < iterations / numThreads; j++) {
                        stripedCounter.increment();
                    }
                });
            }
            
            for (Thread t : threads) t.start();
            for (Thread t : threads) t.join();
            
            long stripedTime = System.nanoTime() - startTime;
            System.out.println("Striped counter time: " + stripedTime / 1_000_000 + " ms");
            System.out.println("Striped counter sum: " + stripedCounter.sum());
            
            // Test LongAdder vs AtomicLong
            LongAdderExample example = new LongAdderExample();
            
            // LongAdder test
            startTime = System.nanoTime();
            for (int i = 0; i < numThreads; i++) {
                threads[i] = new Thread(() -> {
                    for (int j = 0; j < iterations / numThreads; j++) {
                        example.addToAdder(1);
                    }
                });
            }
            
            for (Thread t : threads) t.start();
            for (Thread t : threads) t.join();
            
            long adderTime = System.nanoTime() - startTime;
            System.out.println("LongAdder time: " + adderTime / 1_000_000 + " ms");
            System.out.println("LongAdder sum: " + example.getAdderValue());
            
            // AtomicLong test
            example = new LongAdderExample(); // Fresh instance
            startTime = System.nanoTime();
            for (int i = 0; i < numThreads; i++) {
                threads[i] = new Thread(() -> {
                    for (int j = 0; j < iterations / numThreads; j++) {
                        example.addToAtomic(1);
                    }
                });
            }
            
            for (Thread t : threads) t.start();
            for (Thread t : threads) t.join();
            
            long atomicTime = System.nanoTime() - startTime;
            System.out.println("AtomicLong time: " + atomicTime / 1_000_000 + " ms");
            System.out.println("AtomicLong sum: " + example.getAtomicValue());
            
            System.out.println("LongAdder vs AtomicLong improvement: " + 
                             (double) atomicTime / adderTime + "x");
        }
    }
    
    // Memory access optimization
    public static class MemoryAccessOptimization {
        
        // Cache-friendly data structures
        public static class CacheFriendlyList<T> {
            private static final int BATCH_SIZE = 1024;
            private final List<List<T>> batches = new ArrayList<>();
            private List<T> currentBatch = new ArrayList<>();
            
            public void add(T item) {
                if (currentBatch.size() >= BATCH_SIZE) {
                    batches.add(currentBatch);
                    currentBatch = new ArrayList<>();
                }
                currentBatch.add(item);
            }
            
            public T get(int index) {
                int batchIndex = index / BATCH_SIZE;
                int itemIndex = index % BATCH_SIZE;
                
                if (batchIndex >= batches.size()) {
                    return currentBatch.get(itemIndex);
                } else {
                    return batches.get(batchIndex).get(itemIndex);
                }
            }
            
            public int size() {
                return batches.size() * BATCH_SIZE + currentBatch.size();
            }
        }
        
        // Object pooling for performance
        public static class ObjectPool<T> {
            private final Queue<T> pool = new ConcurrentLinkedQueue<>();
            private final Supplier<T> factory;
            private final Consumer<T> resetter;
            private final int maxSize;
            
            public ObjectPool(Supplier<T> factory, Consumer<T> resetter, int maxSize) {
                this.factory = factory;
                this.resetter = resetter;
                this.maxSize = maxSize;
            }
            
            public T acquire() {
                T object = pool.poll();
                return object != null ? object : factory.get();
            }
            
            public void release(T object) {
                if (object != null && pool.size() < maxSize) {
                    resetter.accept(object);
                    pool.offer(object);
                }
            }
        }
        
        public static void demonstrateMemoryOptimization() {
            System.out.println("\n=== Memory Access Optimization ===");
            
            // Cache-friendly list example
            CacheFriendlyList<String> cacheList = new CacheFriendlyList<>();
            for (int i = 0; i < 10_000; i++) {
                cacheList.add("Item-" + i);
            }
            
            System.out.println("Cache-friendly list size: " + cacheList.size());
            
            // Object pooling example
            ObjectPool<StringBuilder> stringBuilderPool = 
                new ObjectPool<>(
                    StringBuilder::new,
                    sb -> sb.setLength(0), // Reset by clearing content
                    100
                );
            
            long startTime = System.nanoTime();
            for (int i = 0; i < 100_000; i++) {
                StringBuilder sb = stringBuilderPool.acquire();
                sb.append("Test string ").append(i);
                
                // Use the StringBuilder
                String result = sb.toString();
                
                stringBuilderPool.release(sb);
            }
            long pooledTime = System.nanoTime() - startTime;
            
            // Compare with regular allocation
            startTime = System.nanoTime();
            for (int i = 0; i < 100_000; i++) {
                StringBuilder sb = new StringBuilder();
                sb.append("Test string ").append(i);
                String result = sb.toString();
                // StringBuilder becomes eligible for GC immediately
            }
            long regularTime = System.nanoTime() - startTime;
            
            System.out.println("Object pooling improvement: " + 
                             (double) regularTime / pooledTime + "x");
        }
    }
    
    // Lock optimization techniques
    public static class LockOptimization {
        
        // Read-write lock optimization
        public static class OptimizedReadWriteMap<K, V> {
            private final Map<K, V> map = new HashMap<>();
            private final ReadWriteLock lock = new ReentrantReadWriteLock();
            private final Lock readLock = lock.readLock();
            private final Lock writeLock = lock.writeLock();
            
            public V get(K key) {
                readLock.lock();
                try {
                    return map.get(key);
                } finally {
                    readLock.unlock();
                }
            }
            
            public V put(K key, V value) {
                writeLock.lock();
                try {
                    return map.put(key, value);
                } finally {
                    writeLock.unlock();
                }
            }
            
            // Optimized bulk operation
            public void putAll(Map<? extends K, ? extends V> m) {
                writeLock.lock();
                try {
                    map.putAll(m);
                } finally {
                    writeLock.unlock();
                }
            }
            
            // Optimized read with reduced lock scope
            public boolean containsKey(K key) {
                readLock.lock();
                try {
                    return map.containsKey(key);
                } finally {
                    readLock.unlock();
                }
            }
        }
        
        // Fine-grained locking
        public static class FineGrainedLockingExample {
            private final List<ReentrantLock> locks = new ArrayList<>();
            private final List<Integer> data = new ArrayList<>();
            
            public FineGrainedLockingExample(int size) {
                for (int i = 0; i < size; i++) {
                    data.add(0);
                    locks.add(new ReentrantLock());
                }
            }
            
            public int get(int index) {
                ReentrantLock lock = locks.get(index % locks.size());
                lock.lock();
                try {
                    return data.get(index);
                } finally {
                    lock.unlock();
                }
            }
            
            public void increment(int index) {
                ReentrantLock lock = locks.get(index % locks.size());
                lock.lock();
                try {
                    int currentValue = data.get(index);
                    data.set(index, currentValue + 1);
                } finally {
                    lock.unlock();
                }
            }
        }
        
        public static void demonstrateLockOptimization() throws InterruptedException {
            System.out.println("\n=== Lock Optimization ===");
            
            // Read-write lock example
            OptimizedReadWriteMap<String, Integer> rwMap = new OptimizedReadWriteMap<>();
            
            // Populate map
            for (int i = 0; i < 100; i++) {
                rwMap.put("key" + i, i);
            }
            
            ExecutorService executor = Executors.newFixedThreadPool(10);
            CountDownLatch latch = new CountDownLatch(10);
            
            long startTime = System.currentTimeMillis();
            
            // Multiple readers
            for (int i = 0; i < 8; i++) {
                executor.submit(() -> {
                    for (int j = 0; j < 1000; j++) {
                        Integer value = rwMap.get("key" + (j % 100));
                        // Simulate read usage
                    }
                    latch.countDown();
                });
            }
            
            // Few writers
            for (int i = 0; i < 2; i++) {
                final int writerId = i;
                executor.submit(() -> {
                    for (int j = 0; j < 100; j++) {
                        rwMap.put("key" + j, j + 100 + writerId);
                    }
                    latch.countDown();
                });
            }
            
            latch.await();
            long rwTime = System.currentTimeMillis() - startTime;
            
            executor.shutdown();
            System.out.println("Read-write lock test completed in: " + rwTime + " ms");
            
            // Fine-grained locking example
            FineGrainedLockingExample fineExample = new FineGrainedLockingExample(100);
            
            executor = Executors.newFixedThreadPool(20);
            CountDownLatch fineLatch = new CountDownLatch(20);
            startTime = System.currentTimeMillis();
            
            for (int i = 0; i < 20; i++) {
                final int threadId = i;
                executor.submit(() -> {
                    for (int j = 0; j < 1000; j++) {
                        fineExample.increment((threadId * 1000 + j) % 100);
                    }
                    fineLatch.countDown();
                });
            }
            
            fineLatch.await();
            long fineTime = System.currentTimeMillis() - startTime;
            
            executor.shutdown();
            System.out.println("Fine-grained locking test completed in: " + fineTime + " ms");
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        FalseSharingExample.demonstrateFalseSharing();
        ContentionReduction.demonstrateContentionReduction();
        MemoryAccessOptimization.demonstrateMemoryOptimization();
        LockOptimization.demonstrateLockOptimization();
    }
}
```

## Advanced Concurrency Patterns

Implementing sophisticated concurrency patterns for complex applications.

### Asynchronous Programming Patterns

```java
// Advanced concurrency patterns and implementations
public class AdvancedConcurrencyPatterns {
    
    // Actor pattern implementation
    public static class Actor {
        private final Mailbox mailbox;
        private final Thread worker;
        private volatile boolean running = true;
        
        public Actor() {
            this.mailbox = new Mailbox();
            this.worker = new Thread(this::processMessages);
            this.worker.start();
        }
        
        private void processMessages() {
            while (running) {
                Message msg = mailbox.take();
                if (msg != null) {
                    try {
                        msg.execute();
                    } catch (Exception e) {
                        System.err.println("Error processing message: " + e.getMessage());
                    }
                }
            }
        }
        
        public void send(Message message) {
            mailbox.offer(message);
        }
        
        public void stop() {
            running = false;
            worker.interrupt();
        }
        
        // Message interface
        public interface Message {
            void execute();
        }
        
        // Mailbox implementation
        private static class Mailbox {
            private final LinkedBlockingQueue<Message> queue = new LinkedBlockingQueue<>();
            
            public void offer(Message message) {
                queue.offer(message);
            }
            
            public Message take() {
                try {
                    return queue.take();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return null;
                }
            }
        }
    }
    
    // Balking pattern - execute action only if object is in appropriate state
    public static class BalkingPattern {
        private volatile boolean isBusy = false;
        private final Object lock = new Object();
        
        public boolean doAction() {
            synchronized (lock) {
                if (isBusy) {
                    System.out.println("Already busy, balking...");
                    return false; // Don't execute if already busy
                }
                isBusy = true;
            }
            
            try {
                System.out.println("Performing action...");
                Thread.sleep(1000); // Simulate work
                System.out.println("Action completed");
                return true;
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return false;
            } finally {
                synchronized (lock) {
                    isBusy = false;
                }
            }
        }
    }
    
    // Guarded suspension pattern
    public static class GuardedSuspension {
        private volatile boolean condition = false;
        private final Object lock = new Object();
        
        public void waitForCondition() throws InterruptedException {
            synchronized (lock) {
                while (!condition) {
                    System.out.println("Waiting for condition...");
                    lock.wait(); // Wait until condition is true
                }
                System.out.println("Condition met, proceeding...");
            }
        }
        
        public void signalCondition() {
            synchronized (lock) {
                condition = true;
                lock.notifyAll(); // Notify all waiting threads
            }
        }
    }
    
    // Thread-specific storage pattern
    public static class ThreadSpecificStorage {
        private static final ThreadLocal<String> userContext = new ThreadLocal<>();
        private static final ThreadLocal<Long> requestStartTime = new ThreadLocal<>();
        
        public static void setCurrentUser(String user) {
            userContext.set(user);
        }
        
        public static String getCurrentUser() {
            return userContext.get();
        }
        
        public static void startRequestTimer() {
            requestStartTime.set(System.currentTimeMillis());
        }
        
        public static long getRequestDuration() {
            Long startTime = requestStartTime.get();
            return startTime != null ? System.currentTimeMillis() - startTime : 0;
        }
        
        public static void clear() {
            userContext.remove();
            requestStartTime.remove();
        }
    }
    
    // Producer-consumer with multiple producers and consumers
    public static class AdvancedProducerConsumer {
        private final BlockingQueue<String> queue;
        private final AtomicInteger producerCount = new AtomicInteger(0);
        private final AtomicInteger consumerCount = new AtomicInteger(0);
        private final AtomicInteger completedTasks = new AtomicInteger(0);
        private volatile boolean shutdown = false;
        
        public AdvancedProducerConsumer(int queueCapacity) {
            this.queue = new LinkedBlockingQueue<>(queueCapacity);
        }
        
        public void startProducers(int count) {
            for (int i = 0; i < count; i++) {
                final int producerId = i;
                new Thread(() -> {
                    producerCount.incrementAndGet();
                    while (!shutdown) {
                        try {
                            String item = "Producer-" + producerId + "-Item-" + System.nanoTime();
                            if (queue.offer(item, 100, TimeUnit.MILLISECONDS)) {
                                System.out.println("Producer " + producerId + " produced: " + item);
                            } else {
                                System.out.println("Producer " + producerId + " queue full, skipping...");
                            }
                            Thread.sleep(100);
                        } catch (InterruptedException e) {
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                    producerCount.decrementAndGet();
                }, "Producer-" + i).start();
            }
        }
        
        public void startConsumers(int count) {
            for (int i = 0; i < count; i++) {
                final int consumerId = i;
                new Thread(() -> {
                    consumerCount.incrementAndGet();
                    while (!shutdown || !queue.isEmpty()) {
                        try {
                            String item = queue.poll(100, TimeUnit.MILLISECONDS);
                            if (item != null) {
                                System.out.println("Consumer " + consumerId + " consumed: " + item);
                                completedTasks.incrementAndGet();
                            }
                            Thread.sleep(150);
                        } catch (InterruptedException e) {
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                    consumerCount.decrementAndGet();
                }, "Consumer-" + i).start();
            }
        }
        
        public void shutdown() {
            shutdown = true;
        }
        
        public int getCompletedTasks() {
            return completedTasks.get();
        }
        
        public int getActiveProducers() {
            return producerCount.get();
        }
        
        public int getActiveConsumers() {
            return consumerCount.get();
        }
    }
    
    // Future-based pattern with timeout and cancellation
    public static class FutureBasedPattern {
        
        public static <T> CompletableFuture<T> withTimeout(Supplier<CompletableFuture<T>> taskSupplier, 
                                                          long timeout, TimeUnit unit) {
            CompletableFuture<T> task = taskSupplier.get();
            CompletableFuture<T> timeoutFuture = new CompletableFuture<>();
            
            Timer timer = new Timer(true); // Daemon timer
            timer.schedule(new TimerTask() {
                @Override
                public void run() {
                    if (!task.isDone()) {
                        timeoutFuture.completeExceptionally(
                            new TimeoutException("Task timed out after " + timeout + " " + unit));
                        task.cancel(true);
                    }
                }
            }, unit.toMillis(timeout));
            
            task.whenComplete((result, throwable) -> {
                timer.cancel();
                if (throwable != null) {
                    timeoutFuture.completeExceptionally(throwable);
                } else {
                    timeoutFuture.complete(result);
                }
            });
            
            return timeoutFuture;
        }
        
        public static <T> CompletableFuture<T> withRetry(Supplier<CompletableFuture<T>> operation, 
                                                        int maxRetries, long delay, TimeUnit unit) {
            CompletableFuture<T> future = operation.get();
            
            for (int i = 0; i < maxRetries; i++) {
                final int attempt = i;
                future = future.handle((result, throwable) -> {
                    if (throwable == null) {
                        return CompletableFuture.completedFuture(result);
                    } else if (attempt >= maxRetries - 1) {
                        CompletableFuture<T> failed = new CompletableFuture<>();
                        failed.completeExceptionally(throwable);
                        return failed;
                    } else {
                        System.out.println("Attempt " + (attempt + 1) + " failed, retrying...");
                        return CompletableFuture.delayedExecutor(delay, unit).execute(() -> {
                            // This is a simplified retry - in practice you'd need more sophisticated handling
                        });
                    }
                }).thenCompose(cf -> cf);
            }
            
            return future;
        }
    }
    
    public static void demonstrateAdvancedPatterns() throws InterruptedException {
        System.out.println("=== Advanced Concurrency Patterns ===");
        
        // Actor pattern
        System.out.println("\n--- Actor Pattern ---");
        Actor actor = new Actor();
        
        // Send messages to actor
        for (int i = 0; i < 5; i++) {
            final int messageId = i;
            actor.send(() -> {
                System.out.println("Processing message " + messageId + 
                                 " in thread: " + Thread.currentThread().getName());
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        
        Thread.sleep(1000); // Let messages process
        actor.stop();
        
        // Balking pattern
        System.out.println("\n--- Balking Pattern ---");
        BalkingPattern balking = new BalkingPattern();
        
        Thread[] balkingThreads = new Thread[3];
        for (int i = 0; i < 3; i++) {
            balkingThreads[i] = new Thread(() -> {
                boolean success = balking.doAction();
                System.out.println(Thread.currentThread().getName() + " action success: " + success);
            });
        }
        
        for (Thread t : balkingThreads) t.start();
        for (Thread t : balkingThreads) t.join();
        
        // Guarded suspension
        System.out.println("\n--- Guarded Suspension ---");
        GuardedSuspension guarded = new GuardedSuspension();
        
        Thread waiter = new Thread(() -> {
            try {
                guarded.waitForCondition();
                System.out.println("Waiter proceeding after condition");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        Thread notifier = new Thread(() -> {
            try {
                Thread.sleep(1000); // Wait before signaling
                System.out.println("Signaling condition...");
                guarded.signalCondition();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        waiter.start();
        notifier.start();
        
        waiter.join();
        notifier.join();
        
        // Thread-specific storage
        System.out.println("\n--- Thread-Specific Storage ---");
        Thread userThread = new Thread(() -> {
            ThreadSpecificStorage.setCurrentUser("Alice");
            ThreadSpecificStorage.startRequestTimer();
            
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            
            System.out.println("User: " + ThreadSpecificStorage.getCurrentUser());
            System.out.println("Request duration: " + ThreadSpecificStorage.getRequestDuration() + " ms");
            
            ThreadSpecificStorage.clear();
        });
        
        userThread.start();
        userThread.join();
        
        // Advanced producer-consumer
        System.out.println("\n--- Advanced Producer-Consumer ---");
        AdvancedProducerConsumer pc = new AdvancedProducerConsumer(10);
        
        pc.startProducers(3);
        pc.startConsumers(2);
        
        Thread.sleep(3000); // Run for 3 seconds
        pc.shutdown();
        
        Thread.sleep(1000); // Let consumers finish
        System.out.println("Completed tasks: " + pc.getCompletedTasks());
        System.out.println("Active producers: " + pc.getActiveProducers());
        System.out.println("Active consumers: " + pc.getActiveConsumers());
    }
    
    public static void main(String[] args) throws InterruptedException {
        demonstrateAdvancedPatterns();
    }
}
```

## Conclusion

Advanced Java concurrency and multithreading involve understanding complex synchronization mechanisms, memory models, and performance optimization techniques. Key takeaways include:

1. **Thread Lifecycle**: Understanding thread states and proper thread management
2. **Synchronization**: Using appropriate synchronization mechanisms (synchronized, Locks, atomic variables)
3. **Java Memory Model**: Understanding visibility, atomicity, and ordering guarantees
4. **Executor Framework**: Properly managing thread pools and asynchronous tasks
5. **Fork/Join**: Effectively using divide-and-conquer algorithms
6. **CompletableFuture**: Building complex asynchronous workflows
7. **Concurrent Collections**: Using thread-safe collections appropriately
8. **Lock-Free Structures**: Implementing high-performance concurrent data structures
9. **Performance Optimization**: Applying techniques to reduce contention and improve throughput
10. **Advanced Patterns**: Implementing sophisticated concurrency patterns for complex scenarios

Mastering these concepts enables developers to build scalable, high-performance concurrent applications that can effectively utilize modern multi-core processors while maintaining correctness and reliability.