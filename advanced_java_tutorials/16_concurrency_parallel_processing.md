# Advanced Concurrency and Parallel Processing

## Overview
Java provides powerful concurrency utilities in the `java.util.concurrent` package and beyond. Understanding advanced concurrency patterns, parallel processing techniques, and thread safety mechanisms is crucial for building high-performance, scalable applications.

## Advanced Concurrency Utilities

### Fork/Join Framework

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.stream.*;

public class ForkJoinFramework {
    
    // Custom RecursiveTask for CPU-intensive computation
    public static class FibonacciTask extends RecursiveTask<Long> {
        private final long n;
        private static final int THRESHOLD = 10;
        
        public FibonacciTask(long n) {
            this.n = n;
        }
        
        @Override
        protected Long compute() {
            if (n <= THRESHOLD) {
                // Direct computation for small values
                return fibonacci(n);
            } else {
                // Divide and conquer
                FibonacciTask leftTask = new FibonacciTask(n - 1);
                FibonacciTask rightTask = new FibonacciTask(n - 2);
                
                // Fork subtasks
                leftTask.fork();
                rightTask.fork();
                
                // Join results
                long rightResult = rightTask.join();
                long leftResult = leftTask.join();
                
                return leftResult + rightResult;
            }
        }
        
        private long fibonacci(long n) {
            if (n <= 1) return n;
            long a = 0, b = 1;
            for (long i = 2; i <= n; i++) {
                long temp = a + b;
                a = b;
                b = temp;
            }
            return b;
        }
    }
    
    // Fork/Join for parallel array processing
    public static class ParallelArrayProcessor extends RecursiveAction {
        private final int[] array;
        private final int start;
        private final int end;
        private final int threshold = 1000;
        
        public ParallelArrayProcessor(int[] array, int start, int end) {
            this.array = array;
            this.start = start;
            this.end = end;
        }
        
        @Override
        protected void compute() {
            if (end - start <= threshold) {
                // Direct processing for small arrays
                processArray(array, start, end);
            } else {
                // Divide and conquer
                int mid = (start + end) / 2;
                ParallelArrayProcessor leftTask = new ParallelArrayProcessor(array, start, mid);
                ParallelArrayProcessor rightTask = new ParallelArrayProcessor(array, mid, end);
                
                invokeAll(leftTask, rightTask);
            }
        }
        
        private void processArray(int[] array, int start, int end) {
            for (int i = start; i < end; i++) {
                array[i] = array[i] * array[i]; // Square each element
            }
        }
    }
    
    // Fork/Join for complex data processing
    public static class DataAggregationTask extends RecursiveTask<AggregationResult> {
        private final List<DataPoint> dataPoints;
        private final int start;
        private final int end;
        private final int threshold = 1000;
        
        public DataAggregationTask(List<DataPoint> dataPoints, int start, int end) {
            this.dataPoints = dataPoints;
            this.start = start;
            this.end = end;
        }
        
        @Override
        protected AggregationResult compute() {
            if (end - start <= threshold) {
                return processChunk(dataPoints, start, end);
            } else {
                int mid = (start + end) / 2;
                DataAggregationTask leftTask = new DataAggregationTask(dataPoints, start, mid);
                DataAggregationTask rightTask = new DataAggregationTask(dataPoints, mid, end);
                
                leftTask.fork();
                AggregationResult rightResult = rightTask.compute();
                AggregationResult leftResult = leftTask.join();
                
                return mergeResults(leftResult, rightResult);
            }
        }
        
        private AggregationResult processChunk(List<DataPoint> dataPoints, int start, int end) {
            AggregationResult result = new AggregationResult();
            
            for (int i = start; i < end; i++) {
                DataPoint point = dataPoints.get(i);
                result.count++;
                result.sum += point.getValue();
                result.max = Math.max(result.max, point.getValue());
                result.min = Math.min(result.min, point.getValue());
            }
            
            return result;
        }
        
        private AggregationResult mergeResults(AggregationResult left, AggregationResult right) {
            AggregationResult merged = new AggregationResult();
            merged.count = left.count + right.count;
            merged.sum = left.sum + right.sum;
            merged.max = Math.max(left.max, right.max);
            merged.min = Math.min(left.min, right.min);
            return merged;
        }
    }
    
    // Demonstration and performance comparison
    public static class ForkJoinDemo {
        private static final ForkJoinPool forkJoinPool = new ForkJoinPool();
        
        public void demonstrateForkJoin() {
            // Fibonacci computation
            long startTime = System.nanoTime();
            FibonacciTask fibTask = new FibonacciTask(40);
            long fibResult = forkJoinPool.invoke(fibTask);
            long forkJoinTime = System.nanoTime() - startTime;
            
            System.out.println("Fibonacci(40) = " + fibResult);
            System.out.println("Fork/Join time: " + forkJoinTime / 1_000_000 + "ms");
            
            // Array processing
            int[] array = new int[1000000];
            for (int i = 0; i < array.length; i++) {
                array[i] = i;
            }
            
            startTime = System.nanoTime();
            ParallelArrayProcessor processor = new ParallelArrayProcessor(array, 0, array.length);
            forkJoinPool.invoke(processor);
            long arrayProcessingTime = System.nanoTime() - startTime;
            
            System.out.println("Array processing time: " + arrayProcessingTime / 1_000_000 + "ms");
            
            // Data aggregation
            List<DataPoint> dataPoints = generateDataPoints(1000000);
            
            startTime = System.nanoTime();
            DataAggregationTask aggregationTask = new DataAggregationTask(dataPoints, 0, dataPoints.size());
            AggregationResult result = forkJoinPool.invoke(aggregationTask);
            long aggregationTime = System.nanoTime() - startTime;
            
            System.out.println("Aggregation result: " + result);
            System.out.println("Aggregation time: " + aggregationTime / 1_000_000 + "ms");
        }
        
        private List<DataPoint> generateDataPoints(int count) {
            List<DataPoint> points = new ArrayList<>(count);
            Random random = new Random();
            for (int i = 0; i < count; i++) {
                points.add(new DataPoint(random.nextDouble() * 1000));
            }
            return points;
        }
    }
    
    // Supporting classes
    static class DataPoint {
        private final double value;
        
        public DataPoint(double value) {
            this.value = value;
        }
        
        public double getValue() { return value; }
    }
    
    static class AggregationResult {
        long count = 0;
        double sum = 0;
        double max = Double.MIN_VALUE;
        double min = Double.MAX_VALUE;
        
        @Override
        public String toString() {
            return String.format("Count: %d, Sum: %.2f, Avg: %.2f, Max: %.2f, Min: %.2f", 
                               count, sum, count > 0 ? sum / count : 0, max, min);
        }
    }
}
```

### CompletableFuture for Asynchronous Programming

```java
import java.util.concurrent.*;
import java.util.function.*;
import java.util.stream.*;
import java.time.LocalDateTime;

public class CompletableFutureAdvanced {
    
    private static final ExecutorService executor = 
        Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
    
    // Chaining and combining CompletableFutures
    public static class CompletableFutureChaining {
        
        public CompletableFuture<String> processOrderAsync(Order order) {
            return CompletableFuture
                // Step 1: Validate order
                .supplyAsync(() -> validateOrder(order), executor)
                .thenCompose(validatedOrder -> 
                    // Step 2: Process payment
                    processPaymentAsync(validatedOrder))
                .thenCompose(paymentResult -> 
                    // Step 3: Update inventory
                    updateInventoryAsync(paymentResult))
                .thenCompose(inventoryResult -> 
                    // Step 4: Send notification
                    sendNotificationAsync(inventoryResult))
                .thenApply(notificationResult -> 
                    // Final transformation
                    "Order " + order.getId() + " processed successfully at " + LocalDateTime.now())
                .exceptionally(throwable -> {
                    // Handle any errors in the chain
                    logError("Order processing failed", throwable);
                    return "Order " + order.getId() + " processing failed: " + throwable.getMessage();
                });
        }
        
        private Order validateOrder(Order order) {
            // Simulate validation
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            
            if (order.getItems().isEmpty()) {
                throw new IllegalArgumentException("Order must have items");
            }
            
            return order;
        }
        
        private CompletableFuture<PaymentResult> processPaymentAsync(Order order) {
            return CompletableFuture.supplyAsync(() -> {
                // Simulate payment processing
                try {
                    Thread.sleep(200);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                
                return new PaymentResult("TXN_" + order.getId(), order.getAmount(), "SUCCESS");
            }, executor);
        }
        
        private CompletableFuture<InventoryResult> updateInventoryAsync(PaymentResult paymentResult) {
            return CompletableFuture.supplyAsync(() -> {
                // Simulate inventory update
                try {
                    Thread.sleep(150);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                
                return new InventoryResult(paymentResult.getTransactionId(), true);
            }, executor);
        }
        
        private CompletableFuture<NotificationResult> sendNotificationAsync(InventoryResult inventoryResult) {
            return CompletableFuture.supplyAsync(() -> {
                // Simulate notification
                try {
                    Thread.sleep(50);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                
                return new NotificationResult(inventoryResult.getTransactionId(), true);
            }, executor);
        }
        
        private void logError(String message, Throwable throwable) {
            System.err.println(message + ": " + throwable.getMessage());
        }
    }
    
    // Combining multiple futures
    public static class FutureCombining {
        
        public CompletableFuture<AggregatedResult> aggregateDataFromMultipleSources() {
            // Start multiple async operations simultaneously
            CompletableFuture<UserData> userFuture = fetchUserDataAsync();
            CompletableFuture<OrderData> orderFuture = fetchOrderDataAsync();
            CompletableFuture<InventoryData> inventoryFuture = fetchInventoryDataAsync();
            
            // Combine results when all complete
            return CompletableFuture.allOf(userFuture, orderFuture, inventoryFuture)
                .thenApply(v -> {
                    try {
                        UserData user = userFuture.get();
                        OrderData order = orderFuture.get();
                        InventoryData inventory = inventoryFuture.get();
                        
                        return new AggregatedResult(user, order, inventory);
                    } catch (Exception e) {
                        throw new RuntimeException("Failed to aggregate data", e);
                    }
                })
                .orTimeout(5, TimeUnit.SECONDS)
                .exceptionally(throwable -> {
                    System.err.println("Aggregation failed: " + throwable.getMessage());
                    return new AggregatedResult(); // Return empty result
                });
        }
        
        // Apply different logic based on which future completes first
        public CompletableFuture<String> raceConditionExample() {
            CompletableFuture<String> fastService = callFastService();
            CompletableFuture<String> slowService = callSlowService();
            
            // Use the result from whichever completes first
            return CompletableFuture.anyOf(fastService, slowService)
                .thenApply(result -> (String) result)
                .thenApply(result -> "Fastest result: " + result);
        }
        
        // Complex combination with different success/failure handling
        public CompletableFuture<ProcessedResult> complexCombination() {
            CompletableFuture<UserProfile> userProfileFuture = fetchUserProfile();
            CompletableFuture<List<Permission>> permissionsFuture = fetchUserPermissions();
            
            return userProfileFuture.thenCombine(
                permissionsFuture,
                (profile, permissions) -> {
                    // Combine profile and permissions
                    return new ProcessedResult(profile, permissions);
                }
            ).handle((result, throwable) -> {
                if (throwable != null) {
                    // Handle partial failures
                    System.err.println("Processing failed: " + throwable.getMessage());
                    return new ProcessedResult(); // Return default/fallback result
                }
                return result;
            });
        }
        
        private CompletableFuture<UserData> fetchUserDataAsync() {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(300);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                return new UserData("John Doe", "john@example.com");
            }, executor);
        }
        
        private CompletableFuture<OrderData> fetchOrderDataAsync() {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(250);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                return new OrderData(12345L, 299.99);
            }, executor);
        }
        
        private CompletableFuture<InventoryData> fetchInventoryDataAsync() {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(200);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                return new InventoryData(100, 50);
            }, executor);
        }
        
        private CompletableFuture<String> callFastService() {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                return "Fast service result";
            }, executor);
        }
        
        private CompletableFuture<String> callSlowService() {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                return "Slow service result";
            }, executor);
        }
        
        private CompletableFuture<UserProfile> fetchUserProfile() {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(150);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                return new UserProfile("user123", "John", "Doe");
            }, executor);
        }
        
        private CompletableFuture<List<Permission>> fetchUserPermissions() {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(200);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                return Arrays.asList(new Permission("READ"), new Permission("WRITE"));
            }, executor);
        }
    }
    
    // Custom CompletableFuture utilities
    public static class CompletableFutureUtils {
        
        // Retry mechanism with exponential backoff
        public static <T> CompletableFuture<T> retryWithBackoff(
                Supplier<CompletableFuture<T>> operation,
                int maxRetries,
                long initialDelayMs,
                double backoffMultiplier) {
            
            CompletableFuture<T> future = operation.get();
            
            for (int i = 0; i < maxRetries; i++) {
                final int retryCount = i;
                future = future.handle((result, throwable) -> {
                    if (throwable == null) {
                        return CompletableFuture.completedFuture(result);
                    } else if (retryCount < maxRetries - 1) {
                        long delay = (long) (initialDelayMs * Math.pow(backoffMultiplier, retryCount));
                        return CompletableFuture.supplyAsync(() -> {
                            try {
                                Thread.sleep(delay);
                            } catch (InterruptedException e) {
                                Thread.currentThread().interrupt();
                            }
                            return null;
                        }, executor).thenCompose(v -> operation.get());
                    } else {
                        CompletableFuture<T> failed = new CompletableFuture<>();
                        failed.completeExceptionally(throwable);
                        return failed;
                    }
                }).thenCompose(Function.identity());
            }
            
            return future;
        }
        
        // Timeout with fallback
        public static <T> CompletableFuture<T> withTimeoutAndFallback(
                CompletableFuture<T> future,
                long timeout,
                TimeUnit unit,
                Supplier<T> fallback) {
            
            CompletableFuture<T> timeoutFuture = CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(unit.toMillis(timeout));
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                return null;
            }, executor).thenApply(v -> fallback.get());
            
            return future.applyToEither(timeoutFuture, Function.identity());
        }
        
        // Bulkhead pattern for resource limiting
        public static <T> CompletableFuture<T> withBulkhead(
                Supplier<CompletableFuture<T>> operation,
                Semaphore semaphore) {
            
            return CompletableFuture.supplyAsync(() -> {
                try {
                    semaphore.acquire();
                    return operation.get();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    CompletableFuture<T> failed = new CompletableFuture<>();
                    failed.completeExceptionally(e);
                    return failed;
                }
            }, executor).thenCompose(Function.identity())
              .whenComplete((result, throwable) -> semaphore.release());
        }
    }
    
    // Supporting classes
    static class Order {
        private Long id;
        private double amount;
        private List<String> items;
        
        public Order(Long id, double amount, List<String> items) {
            this.id = id;
            this.amount = amount;
            this.items = items;
        }
        
        // Getters
        public Long getId() { return id; }
        public double getAmount() { return amount; }
        public List<String> getItems() { return items; }
    }
    
    static class PaymentResult {
        private String transactionId;
        private double amount;
        private String status;
        
        public PaymentResult(String transactionId, double amount, String status) {
            this.transactionId = transactionId;
            this.amount = amount;
            this.status = status;
        }
        
        // Getters
        public String getTransactionId() { return transactionId; }
        public double getAmount() { return amount; }
        public String getStatus() { return status; }
    }
    
    static class InventoryResult {
        private String transactionId;
        private boolean updated;
        
        public InventoryResult(String transactionId, boolean updated) {
            this.transactionId = transactionId;
            this.updated = updated;
        }
        
        // Getters
        public String getTransactionId() { return transactionId; }
        public boolean isUpdated() { return updated; }
    }
    
    static class NotificationResult {
        private String transactionId;
        private boolean sent;
        
        public NotificationResult(String transactionId, boolean sent) {
            this.transactionId = transactionId;
            this.sent = sent;
        }
        
        // Getters
        public String getTransactionId() { return transactionId; }
        public boolean isSent() { return sent; }
    }
    
    static class UserData {
        private String name;
        private String email;
        
        public UserData(String name, String email) {
            this.name = name;
            this.email = email;
        }
        
        // Getters
        public String getName() { return name; }
        public String getEmail() { return email; }
    }
    
    static class OrderData {
        private Long orderId;
        private double amount;
        
        public OrderData(Long orderId, double amount) {
            this.orderId = orderId;
            this.amount = amount;
        }
        
        // Getters
        public Long getOrderId() { return orderId; }
        public double getAmount() { return amount; }
    }
    
    static class InventoryData {
        private int totalItems;
        private int availableItems;
        
        public InventoryData(int totalItems, int availableItems) {
            this.totalItems = totalItems;
            this.availableItems = availableItems;
        }
        
        // Getters
        public int getTotalItems() { return totalItems; }
        public int getAvailableItems() { return availableItems; }
    }
    
    static class AggregatedResult {
        private UserData userData;
        private OrderData orderData;
        private InventoryData inventoryData;
        
        public AggregatedResult() {}
        
        public AggregatedResult(UserData userData, OrderData orderData, InventoryData inventoryData) {
            this.userData = userData;
            this.orderData = orderData;
            this.inventoryData = inventoryData;
        }
        
        // Getters
        public UserData getUserData() { return userData; }
        public OrderData getOrderData() { return orderData; }
        public InventoryData getInventoryData() { return inventoryData; }
    }
    
    static class UserProfile {
        private String userId;
        private String firstName;
        private String lastName;
        
        public UserProfile(String userId, String firstName, String lastName) {
            this.userId = userId;
            this.firstName = firstName;
            this.lastName = lastName;
        }
        
        // Getters
        public String getUserId() { return userId; }
        public String getFirstName() { return firstName; }
        public String getLastName() { return lastName; }
    }
    
    static class Permission {
        private String name;
        
        public Permission(String name) {
            this.name = name;
        }
        
        // Getters
        public String getName() { return name; }
    }
    
    static class ProcessedResult {
        private UserProfile profile;
        private List<Permission> permissions;
        
        public ProcessedResult() {}
        
        public ProcessedResult(UserProfile profile, List<Permission> permissions) {
            this.profile = profile;
            this.permissions = permissions;
        }
        
        // Getters
        public UserProfile getProfile() { return profile; }
        public List<Permission> getPermissions() { return permissions; }
    }
}
```

## Advanced Locking Mechanisms

### StampedLock and Other Advanced Locks

```java
import java.util.concurrent.locks.*;
import java.util.concurrent.atomic.*;
import java.util.concurrent.*;
import java.util.*;

public class AdvancedLocking {
    
    // StampedLock example with optimistic reading
    public static class StampedLockExample {
        private final StampedLock lock = new StampedLock();
        private double x, y;
        
        // Optimistic read - fastest but may need to retry
        public double distanceFromOrigin() {
            long stamp = lock.tryOptimisticRead();
            double currentX = x;
            double currentY = y;
            
            // Validate that no write occurred during read
            if (!lock.validate(stamp)) {
                // Retry with read lock
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
        
        // Write operation
        public void move(double deltaX, double deltaY) {
            long stamp = lock.writeLock();
            try {
                x += deltaX;
                y += deltaY;
            } finally {
                lock.unlockWrite(stamp);
            }
        }
        
        // Read operation that needs guaranteed consistency
        public String getCoordinates() {
            long stamp = lock.readLock();
            try {
                return String.format("(%.2f, %.2f)", x, y);
            } finally {
                lock.unlockRead(stamp);
            }
        }
        
        // Upgrade from read to write lock
        public void moveIfAtOrigin(double deltaX, double deltaY) {
            long stamp = lock.readLock();
            try {
                while (x == 0.0 && y == 0.0) {
                    // Try to upgrade to write lock
                    long writeStamp = lock.tryConvertToWriteLock(stamp);
                    if (writeStamp != 0L) {
                        // Successfully upgraded
                        stamp = writeStamp;
                        x += deltaX;
                        y += deltaY;
                        break;
                    } else {
                        // Could not upgrade, release read lock and acquire write lock
                        lock.unlockRead(stamp);
                        stamp = lock.writeLock();
                    }
                }
            } finally {
                lock.unlock(stamp);
            }
        }
    }
    
    // ReentrantReadWriteLock with advanced features
    public static class AdvancedReadWriteLock {
        private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
        private final ReadLock readLock = rwLock.readLock();
        private final WriteLock writeLock = rwLock.writeLock();
        private final Map<String, String> data = new HashMap<>();
        
        public String getWithTimeout(String key, long timeout, TimeUnit unit) 
                throws TimeoutException, InterruptedException {
            if (readLock.tryLock(timeout, unit)) {
                try {
                    return data.get(key);
                } finally {
                    readLock.unlock();
                }
            } else {
                throw new TimeoutException("Could not acquire read lock within timeout");
            }
        }
        
        public void putWithCondition(String key, String value, Predicate<String> condition) {
            writeLock.lock();
            try {
                String currentValue = data.get(key);
                if (condition.test(currentValue)) {
                    data.put(key, value);
                }
            } finally {
                writeLock.unlock();
            }
        }
        
        // Fair locking example
        private final ReentrantReadWriteLock fairLock = 
            new ReentrantReadWriteLock(true); // Fair locking
        
        public void demonstrateFairLocking() {
            // Fair locking ensures threads acquire locks in the order they requested them
            // This can reduce starvation but may reduce throughput
        }
        
        // Lock downgrading example
        public String processDataWithDowngrade(String key) {
            writeLock.lock();
            try {
                // Perform write operation
                String value = data.get(key);
                if (value == null) {
                    value = "processed_" + key;
                    data.put(key, value);
                }
                
                // Downgrade to read lock to allow concurrent reads
                readLock.lock();
                return value;
            } finally {
                writeLock.unlock(); // Release write lock
                // Keep read lock for subsequent operations
            }
            // Read lock released in calling code or separate finally block
        }
    }
    
    // Condition variables for complex synchronization
    public static class ConditionVariables {
        private final ReentrantLock lock = new ReentrantLock();
        private final Condition notFull = lock.newCondition();
        private final Condition notEmpty = lock.newCondition();
        private final String[] buffer = new String[100];
        private int putIndex = 0;
        private int takeIndex = 0;
        private int count = 0;
        
        public void put(String item) throws InterruptedException {
            lock.lock();
            try {
                // Wait while buffer is full
                while (count == buffer.length) {
                    notFull.await();
                }
                
                buffer[putIndex] = item;
                putIndex = (putIndex + 1) % buffer.length;
                count++;
                
                // Signal that buffer is no longer empty
                notEmpty.signal();
            } finally {
                lock.unlock();
            }
        }
        
        public String take() throws InterruptedException {
            lock.lock();
            try {
                // Wait while buffer is empty
                while (count == 0) {
                    notEmpty.await();
                }
                
                String item = buffer[takeIndex];
                buffer[takeIndex] = null;
                takeIndex = (takeIndex + 1) % buffer.length;
                count--;
                
                // Signal that buffer is no longer full
                notFull.signal();
                
                return item;
            } finally {
                lock.unlock();
            }
        }
        
        // Multiple conditions example
        public static class MultiConditionExample {
            private final ReentrantLock lock = new ReentrantLock();
            private final Condition conditionA = lock.newCondition();
            private final Condition conditionB = lock.newCondition();
            private final Condition conditionC = lock.newCondition();
            
            private volatile String state = "A";
            
            public void awaitStateA() throws InterruptedException {
                lock.lock();
                try {
                    while (!"A".equals(state)) {
                        conditionA.await();
                    }
                } finally {
                    lock.unlock();
                }
            }
            
            public void awaitStateB() throws InterruptedException {
                lock.lock();
                try {
                    while (!"B".equals(state)) {
                        conditionB.await();
                    }
                } finally {
                    lock.unlock();
                }
            }
            
            public void setState(String newState) {
                lock.lock();
                try {
                    this.state = newState;
                    
                    // Signal appropriate condition
                    switch (newState) {
                        case "A":
                            conditionA.signalAll();
                            break;
                        case "B":
                            conditionB.signalAll();
                            break;
                        case "C":
                            conditionC.signalAll();
                            break;
                    }
                } finally {
                    lock.unlock();
                }
            }
        }
    }
    
    // Non-blocking algorithms with atomic references
    public static class NonBlockingAlgorithms {
        
        // Lock-free stack implementation
        public static class LockFreeStack<T> {
            private final AtomicReference<Node<T>> head = new AtomicReference<>();
            
            private static class Node<T> {
                final T data;
                final Node<T> next;
                
                Node(T data, Node<T> next) {
                    this.data = data;
                    this.next = next;
                }
            }
            
            public void push(T item) {
                Node<T> newNode = new Node<>(item, null);
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
        }
        
        // AtomicStampedReference for ABA problem prevention
        public static class VersionedReference<T> {
            private final AtomicStampedReference<T> reference;
            
            public VersionedReference(T initialRef) {
                this.reference = new AtomicStampedReference<>(initialRef, 0);
            }
            
            public boolean compareAndSet(T expectedRef, T newRef) {
                int[] stampHolder = new int[1];
                T currentRef = reference.get(stampHolder);
                int currentStamp = stampHolder[0];
                
                return reference.compareAndSet(expectedRef, newRef, currentStamp, currentStamp + 1);
            }
            
            public T get() {
                return reference.getReference();
            }
            
            public int getVersion() {
                return reference.getStamp();
            }
        }
        
        // AtomicIntegerFieldUpdater for memory-efficient atomic operations
        public static class AtomicFieldUpdaterExample {
            public volatile int counter = 0;
            public volatile long timestamp = 0L;
            
            private static final AtomicIntegerFieldUpdater<AtomicFieldUpdaterExample> COUNTER_UPDATER =
                AtomicIntegerFieldUpdater.newUpdater(AtomicFieldUpdaterExample.class, "counter");
            
            private static final AtomicLongFieldUpdater<AtomicFieldUpdaterExample> TIMESTAMP_UPDATER =
                AtomicLongFieldUpdater.newUpdater(AtomicFieldUpdaterExample.class, "timestamp");
            
            public void incrementCounter() {
                COUNTER_UPDATER.incrementAndGet(this);
            }
            
            public void updateTimestamp() {
                TIMESTAMP_UPDATER.set(this, System.currentTimeMillis());
            }
            
            public int getCounter() {
                return counter;
            }
            
            public long getTimestamp() {
                return timestamp;
            }
        }
    }
}
```

## Parallel Collections and Streams

### Advanced Parallel Processing

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.function.*;
import java.util.stream.*;

public class ParallelCollectionsStreams {
    
    // Custom parallel collector
    public static class CustomParallelCollectors {
        
        // Parallel grouping collector
        public static <T, K, V> Collector<T, ?, Map<K, List<V>>> parallelGroupingBy(
                Function<T, K> classifier,
                Function<T, V> mapper,
                int parallelism) {
            
            return Collector.of(
                () -> new ConcurrentHashMap<K, List<V>>(),
                (map, item) -> {
                    K key = classifier.apply(item);
                    V value = mapper.apply(item);
                    map.computeIfAbsent(key, k -> new ArrayList<>()).add(value);
                },
                (map1, map2) -> {
                    map2.forEach((key, values) -> 
                        map1.computeIfAbsent(key, k -> new ArrayList<>()).addAll(values));
                    return map1;
                },
                Collector.Characteristics.CONCURRENT,
                Collector.Characteristics.UNORDERED
            );
        }
        
        // Parallel reduction with custom combiner
        public static <T, R> Collector<T, ?, R> parallelReducing(
                Supplier<R> supplier,
                BiConsumer<R, T> accumulator,
                BinaryOperator<R> combiner,
                int parallelism) {
            
            return Collector.of(
                () -> {
                    // Thread-local state
                    return new ArrayList<T>();
                },
                (list, item) -> list.add(item),
                (list1, list2) -> {
                    list1.addAll(list2);
                    return list1;
                },
                list -> {
                    // Parallel processing of accumulated items
                    return list.parallelStream()
                        .collect(Collector.of(supplier, accumulator, combiner));
                },
                Collector.Characteristics.UNORDERED
            );
        }
    }
    
    // Parallel processing with custom executors
    public static class CustomParallelProcessing {
        
        private static final ForkJoinPool customPool = 
            new ForkJoinPool(Runtime.getRuntime().availableProcessors() * 2);
        
        public static <T, R> List<R> parallelMap(
                Collection<T> collection,
                Function<T, R> mapper,
                ExecutorService executor) {
            
            return collection.parallelStream()
                .map(item -> CompletableFuture.supplyAsync(() -> mapper.apply(item), executor))
                .collect(Collectors.toList())
                .parallelStream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList());
        }
        
        // Parallel processing with batching
        public static <T, R> List<R> parallelBatchProcess(
                List<T> items,
                Function<List<T>, List<R>> batchProcessor,
                int batchSize) {
            
            return IntStream.range(0, (items.size() + batchSize - 1) / batchSize)
                .parallel()
                .mapToObj(i -> {
                    int start = i * batchSize;
                    int end = Math.min(start + batchSize, items.size());
                    return items.subList(start, end);
                })
                .map(batchProcessor)
                .flatMap(List::stream)
                .collect(Collectors.toList());
        }
        
        // Parallel processing with progress tracking
        public static <T, R> List<R> parallelProcessWithProgress(
                List<T> items,
                Function<T, R> processor,
                Consumer<Integer> progressCallback) {
            
            AtomicInteger processedCount = new AtomicInteger(0);
            int totalItems = items.size();
            
            return items.parallelStream()
                .map(item -> {
                    R result = processor.apply(item);
                    int count = processedCount.incrementAndGet();
                    if (count % (totalItems / 10) == 0 || count == totalItems) {
                        progressCallback.accept(count);
                    }
                    return result;
                })
                .collect(Collectors.toList());
        }
    }
    
    // Performance monitoring for parallel operations
    public static class ParallelPerformanceMonitor {
        
        public static <T, R> R monitorParallelOperation(
                Supplier<R> operation,
                String operationName) {
            
            long startTime = System.nanoTime();
            long startCpuTime = getCpuTime();
            
            try {
                R result = operation.get();
                
                long endTime = System.nanoTime();
                long endCpuTime = getCpuTime();
                
                System.out.printf("%s - Wall time: %d ms, CPU time: %d ms%n",
                    operationName,
                    (endTime - startTime) / 1_000_000,
                    (endCpuTime - startCpuTime) / 1_000_000);
                
                return result;
            } catch (Exception e) {
                long endTime = System.nanoTime();
                System.out.printf("%s - Failed after %d ms%n",
                    operationName,
                    (endTime - startTime) / 1_000_000);
                throw e;
            }
        }
        
        private static long getCpuTime() {
            ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
            if (threadBean.isThreadCpuTimeSupported()) {
                return threadBean.getCurrentThreadCpuTime();
            }
            return 0;
        }
        
        // Thread pool monitoring
        public static void monitorThreadPool(ExecutorService executor, String poolName) {
            if (executor instanceof ThreadPoolExecutor) {
                ThreadPoolExecutor threadPool = (ThreadPoolExecutor) executor;
                
                System.out.printf("%s - Active threads: %d, Pool size: %d, Queue size: %d%n",
                    poolName,
                    threadPool.getActiveCount(),
                    threadPool.getPoolSize(),
                    threadPool.getQueue().size());
            }
        }
    }
    
    // Parallel stream optimization examples
    public static class ParallelStreamOptimization {
        
        // Efficient parallel reduction
        public static long parallelSum(List<Integer> numbers) {
            return numbers.parallelStream()
                .mapToLong(Integer::longValue)
                .sum();
        }
        
        // Parallel processing with early termination
        public static Optional<Integer> findFirstParallel(
                List<Integer> numbers,
                Predicate<Integer> condition) {
            
            return numbers.parallelStream()
                .filter(condition)
                .findFirst(); // Short-circuits the parallel stream
        }
        
        // Parallel processing with ordered results
        public static List<String> processInOrder(List<Integer> numbers) {
            return numbers.parallelStream()
                .map(i -> processNumber(i))
                .collect(Collectors.toList()); // Maintains encounter order
        }
        
        private static String processNumber(Integer number) {
            // Simulate processing time
            try {
                Thread.sleep(1);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "Processed: " + number;
        }
        
        // Parallel processing with custom characteristics
        public static <T> Stream<T> optimizedParallelStream(
                Collection<T> collection) {
            
            // Use custom spliterator for better splitting
            return StreamSupport.stream(
                collection.spliterator(),
                true // parallel
            ).onClose(() -> {
                // Cleanup resources if needed
                System.out.println("Parallel stream completed");
            });
        }
        
        // Memory-efficient parallel processing
        public static void memoryEfficientParallelProcessing(
                Stream<DataItem> dataStream) {
            
            dataStream
                .parallel()
                .filter(item -> item.getValue() > 0)
                .map(DataItem::process)
                .forEach(item -> {
                    // Process items without accumulating in memory
                    // This is memory-efficient for large datasets
                });
        }
    }
    
    // Supporting classes
    static class DataItem {
        private final int value;
        private final String data;
        
        public DataItem(int value, String data) {
            this.value = value;
            this.data = data;
        }
        
        public int getValue() { return value; }
        public String getData() { return data; }
        
        public DataItem process() {
            // Processing logic
            return new DataItem(value * 2, data.toUpperCase());
        }
    }
    
    // Demonstration and benchmarking
    public static class ParallelProcessingDemo {
        
        public void demonstrateParallelProcessing() {
            // Create test data
            List<Integer> numbers = IntStream.range(0, 1000000)
                .boxed()
                .collect(Collectors.toList());
            
            // Sequential processing
            long startTime = System.currentTimeMillis();
            long sequentialSum = numbers.stream()
                .mapToLong(i -> i * i)
                .sum();
            long sequentialTime = System.currentTimeMillis() - startTime;
            
            // Parallel processing
            startTime = System.currentTimeMillis();
            long parallelSum = numbers.parallelStream()
                .mapToLong(i -> i * i)
                .sum();
            long parallelTime = System.currentTimeMillis() - startTime;
            
            System.out.println("Sequential sum: " + sequentialSum + 
                             " in " + sequentialTime + "ms");
            System.out.println("Parallel sum: " + parallelSum + 
                             " in " + parallelTime + "ms");
            System.out.println("Speedup: " + (double) sequentialTime / parallelTime + "x");
            
            // Custom parallel processing
            List<String> results = CustomParallelProcessing.parallelBatchProcess(
                numbers,
                batch -> batch.parallelStream()
                    .map(i -> "Item: " + i)
                    .collect(Collectors.toList()),
                1000
            );
            
            System.out.println("Processed " + results.size() + " items in batches");
        }
    }
}
```

## Conclusion

Advanced Concurrency and Parallel Processing provide:
- Powerful Fork/Join framework for divide-and-conquer algorithms
- Flexible CompletableFuture for asynchronous programming
- Advanced locking mechanisms for complex synchronization
- Efficient parallel collections and streams
- Performance monitoring and optimization techniques

Mastering these concepts is essential for:
- Building high-performance, scalable applications
- Implementing efficient concurrent algorithms
- Managing complex thread interactions
- Optimizing resource utilization
- Ensuring thread safety in concurrent environments