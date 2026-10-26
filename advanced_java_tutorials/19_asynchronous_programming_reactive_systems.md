# Advanced Asynchronous Programming and Reactive Systems

## Overview
Asynchronous programming and reactive systems are fundamental for building responsive, resilient, and scalable applications. This tutorial covers advanced patterns including CompletableFuture, reactive streams, event-driven architectures, and reactive programming principles with Project Reactor.

## CompletableFuture Advanced Patterns

### Complex Chaining and Composition

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.*;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class CompletableFutureAdvancedPatterns {
    
    // Complex chaining with error handling
    public static class CompletableFutureChaining {
        
        public CompletableFuture<String> processOrderAsync(Order order) {
            return CompletableFuture
                // Step 1: Validate order
                .supplyAsync(() -> validateOrder(order), ForkJoinPool.commonPool())
                .thenCompose(validatedOrder -> 
                    // Step 2: Process payment
                    processPaymentAsync(validatedOrder.getPaymentInfo())
                        .handle((paymentResult, throwable) -> {
                            if (throwable != null) {
                                logError("Payment failed", throwable);
                                return PaymentResult.failed("Payment failed: " + throwable.getMessage());
                            }
                            return paymentResult;
                        })
                )
                .thenCompose(paymentResult -> {
                    if (!paymentResult.isSuccess()) {
                        throw new PaymentException("Payment failed: " + paymentResult.getErrorMessage());
                    }
                    return updateInventoryAsync(order.getItems());
                })
                .thenCompose(inventoryResult -> 
                    sendConfirmationEmailAsync(order.getCustomerEmail())
                )
                .thenApply(confirmation -> 
                    "Order " + order.getId() + " processed successfully at " + java.time.LocalDateTime.now()
                )
                .exceptionally(throwable -> {
                    logError("Order processing failed", throwable);
                    return "Order " + order.getId() + " failed: " + throwable.getMessage();
                });
        }
        
        private Order validateOrder(Order order) {
            if (order.getItems().isEmpty()) {
                throw new IllegalArgumentException("Order must have items");
            }
            if (order.getCustomerEmail() == null || order.getCustomerEmail().isEmpty()) {
                throw new IllegalArgumentException("Customer email is required");
            }
            return order;
        }
        
        private CompletableFuture<PaymentResult> processPaymentAsync(PaymentInfo paymentInfo) {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(2000); // Simulate payment processing
                    if (Math.random() < 0.1) { // 10% failure rate
                        throw new RuntimeException("Payment gateway timeout");
                    }
                    return PaymentResult.success("TXN_" + System.currentTimeMillis(), "Payment successful");
                } catch (Exception e) {
                    return PaymentResult.failed(e.getMessage());
                }
            });
        }
        
        private CompletableFuture<InventoryResult> updateInventoryAsync(java.util.List<OrderItem> items) {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(1000); // Simulate inventory update
                    return new InventoryResult(true, "Inventory updated successfully");
                } catch (Exception e) {
                    return new InventoryResult(false, e.getMessage());
                }
            });
        }
        
        private CompletableFuture<EmailResult> sendConfirmationEmailAsync(String email) {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(500); // Simulate email sending
                    return new EmailResult(true, "Confirmation sent to " + email);
                } catch (Exception e) {
                    return new EmailResult(false, e.getMessage());
                }
            });
        }
        
        private void logError(String message, Throwable throwable) {
            System.err.println(message + ": " + throwable.getMessage());
        }
    }
    
    // Combining multiple futures with different strategies
    public static class FutureCombining {
        
        // All of pattern with timeout
        public CompletableFuture<AggregatedData> aggregateDataAsync() {
            CompletableFuture<UserData> userFuture = fetchUserDataAsync();
            CompletableFuture<OrderData> orderFuture = fetchOrderDataAsync();
            CompletableFuture<InventoryData> inventoryFuture = fetchInventoryDataAsync();
            
            // Apply timeout to all futures
            CompletableFuture<UserData> userWithTimeout = userFuture
                .orTimeout(3, TimeUnit.SECONDS)
                .exceptionally(throwable -> {
                    logWarning("User data fetch timed out", throwable);
                    return new UserData("Unknown", "unknown@example.com");
                });
            
            CompletableFuture<OrderData> orderWithTimeout = orderFuture
                .orTimeout(3, TimeUnit.SECONDS)
                .exceptionally(throwable -> {
                    logWarning("Order data fetch timed out", throwable);
                    return new OrderData();
                });
            
            CompletableFuture<InventoryData> inventoryWithTimeout = inventoryFuture
                .orTimeout(3, TimeUnit.SECONDS)
                .exceptionally(throwable -> {
                    logWarning("Inventory data fetch timed out", throwable);
                    return new InventoryData();
                });
            
            return CompletableFuture.allOf(userWithTimeout, orderWithTimeout, inventoryWithTimeout)
                .thenApply(v -> {
                    try {
                        UserData user = userWithTimeout.getNow(new UserData("Unknown", "unknown@example.com"));
                        OrderData order = orderWithTimeout.getNow(new OrderData());
                        InventoryData inventory = inventoryWithTimeout.getNow(new InventoryData());
                        
                        return new AggregatedData(user, order, inventory);
                    } catch (Exception e) {
                        throw new RuntimeException("Failed to aggregate data", e);
                    }
                });
        }
        
        // Any of pattern for fallback
        public CompletableFuture<String> getBestPriceAsync(String productId) {
            CompletableFuture<String> primaryPrice = fetchPriceFromPrimaryService(productId);
            CompletableFuture<String> backupPrice = fetchPriceFromBackupService(productId);
            CompletableFuture<String> cachePrice = fetchPriceFromCache(productId);
            
            return CompletableFuture.anyOf(primaryPrice, backupPrice, cachePrice)
                .thenApply(result -> (String) result)
                .orTimeout(5, TimeUnit.SECONDS)
                .exceptionally(throwable -> {
                    logError("All price services failed", throwable);
                    return "N/A";
                });
        }
        
        // Race condition handling with proper cleanup
        public CompletableFuture<ProcessingResult> raceConditionExample() {
            CompletableFuture<ProcessingResult> fastService = callFastService();
            CompletableFuture<ProcessingResult> slowService = callSlowService();
            
            return fastService
                .applyToEither(slowService, Function.identity())
                .thenCompose(result -> {
                    // Cancel the slower future if it's still running
                    CompletableFuture<ProcessingResult> slowerFuture = 
                        fastService == result ? slowService : fastService;
                    
                    // Attempt to cancel the slower operation
                    slowerFuture.cancel(true);
                    
                    return CompletableFuture.completedFuture(result);
                });
        }
        
        private CompletableFuture<UserData> fetchUserDataAsync() {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(1500);
                    return new UserData("John Doe", "john@example.com");
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            });
        }
        
        private CompletableFuture<OrderData> fetchOrderDataAsync() {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(2000);
                    return new OrderData(12345L, 299.99);
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            });
        }
        
        private CompletableFuture<InventoryData> fetchInventoryDataAsync() {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(1000);
                    return new InventoryData(100, 50);
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            });
        }
        
        private CompletableFuture<String> fetchPriceFromPrimaryService(String productId) {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(1000); // Simulate service call
                    return "$" + (Math.random() * 100);
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            });
        }
        
        private CompletableFuture<String> fetchPriceFromBackupService(String productId) {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(2000); // Slower backup service
                    return "$" + (Math.random() * 100 + 10);
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            });
        }
        
        private CompletableFuture<String> fetchPriceFromCache(String productId) {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(100); // Fast cache lookup
                    return "$" + (Math.random() * 90);
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            });
        }
        
        private CompletableFuture<ProcessingResult> callFastService() {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(500);
                    return new ProcessingResult("Fast service result", true);
                } catch (Exception e) {
                    return new ProcessingResult("Fast service failed", false);
                }
            });
        }
        
        private CompletableFuture<ProcessingResult> callSlowService() {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    Thread.sleep(2000);
                    return new ProcessingResult("Slow service result", true);
                } catch (Exception e) {
                    return new ProcessingResult("Slow service failed", false);
                }
            });
        }
        
        private void logWarning(String message, Throwable throwable) {
            System.out.println("WARNING: " + message + " - " + throwable.getMessage());
        }
        
        private void logError(String message, Throwable throwable) {
            System.err.println("ERROR: " + message + " - " + throwable.getMessage());
        }
    }
    
    // Retry mechanisms with exponential backoff
    public static class RetryMechanisms {
        
        public <T> CompletableFuture<T> retryWithBackoff(
                Supplier<CompletableFuture<T>> operation,
                int maxRetries,
                long initialDelayMs,
                double backoffMultiplier) {
            
            CompletableFuture<T> result = new CompletableFuture<>();
            AtomicInteger attempt = new AtomicInteger(0);
            
            executeWithRetry(operation, maxRetries, attempt, initialDelayMs, backoffMultiplier, result);
            
            return result;
        }
        
        private <T> void executeWithRetry(
                Supplier<CompletableFuture<T>> operation,
                int maxRetries,
                AtomicInteger attempt,
                long delay,
                double backoffMultiplier,
                CompletableFuture<T> result) {
            
            operation.get()
                .whenComplete((value, throwable) -> {
                    int currentAttempt = attempt.get();
                    
                    if (throwable == null) {
                        // Success
                        result.complete(value);
                    } else if (currentAttempt < maxRetries) {
                        // Retry
                        attempt.incrementAndGet();
                        
                        CompletableFuture.delayedExecutor(delay, TimeUnit.MILLISECONDS)
                            .execute(() -> executeWithRetry(
                                operation,
                                maxRetries,
                                attempt,
                                (long) (delay * backoffMultiplier),
                                backoffMultiplier,
                                result
                            ));
                    } else {
                        // Max retries reached
                        result.completeExceptionally(throwable);
                    }
                });
        }
        
        // Circuit breaker pattern
        public static class CircuitBreaker {
            private final int failureThreshold;
            private final long timeoutMillis;
            private final AtomicInteger failureCount = new AtomicInteger(0);
            private volatile long lastFailureTime = 0;
            private final AtomicInteger state = new AtomicInteger(0); // 0=Closed, 1=Open, 2=Half-Open
            
            public CircuitBreaker(int failureThreshold, long timeoutMillis) {
                this.failureThreshold = failureThreshold;
                this.timeoutMillis = timeoutMillis;
            }
            
            public <T> CompletableFuture<T> execute(Supplier<CompletableFuture<T>> operation) {
                if (isCircuitOpen()) {
                    return CompletableFuture.failedFuture(new CircuitBreakerOpenException("Circuit is OPEN"));
                }
                
                CompletableFuture<T> future = operation.get();
                
                future.whenComplete((result, throwable) -> {
                    if (throwable != null) {
                        recordFailure();
                    } else {
                        recordSuccess();
                    }
                });
                
                return future;
            }
            
            private boolean isCircuitOpen() {
                if (state.get() == 0) {
                    return false; // Closed
                }
                
                if (state.get() == 1) {
                    // Check if timeout has passed
                    if (System.currentTimeMillis() - lastFailureTime > timeoutMillis) {
                        state.set(2); // Half-open
                        return false;
                    }
                    return true; // Still open
                }
                
                return false; // Half-open
            }
            
            private void recordFailure() {
                failureCount.incrementAndGet();
                lastFailureTime = System.currentTimeMillis();
                
                if (failureCount.get() >= failureThreshold) {
                    state.set(1); // Open the circuit
                }
            }
            
            private void recordSuccess() {
                failureCount.set(0);
                state.set(0); // Close the circuit
            }
        }
        
        public static class CircuitBreakerOpenException extends Exception {
            public CircuitBreakerOpenException(String message) {
                super(message);
            }
        }
    }
    
    // Supporting classes
    static class Order {
        private Long id;
        private java.util.List<OrderItem> items;
        private PaymentInfo paymentInfo;
        private String customerEmail;
        
        public Order(Long id, java.util.List<OrderItem> items, PaymentInfo paymentInfo, String customerEmail) {
            this.id = id;
            this.items = items;
            this.paymentInfo = paymentInfo;
            this.customerEmail = customerEmail;
        }
        
        // Getters
        public Long getId() { return id; }
        public java.util.List<OrderItem> getItems() { return items; }
        public PaymentInfo getPaymentInfo() { return paymentInfo; }
        public String getCustomerEmail() { return customerEmail; }
    }
    
    static class OrderItem {
        private String productId;
        private int quantity;
        
        public OrderItem(String productId, int quantity) {
            this.productId = productId;
            this.quantity = quantity;
        }
        
        // Getters
        public String getProductId() { return productId; }
        public int getQuantity() { return quantity; }
    }
    
    static class PaymentInfo {
        private String cardNumber;
        private String expiryDate;
        private String cvv;
        
        public PaymentInfo(String cardNumber, String expiryDate, String cvv) {
            this.cardNumber = cardNumber;
            this.expiryDate = expiryDate;
            this.cvv = cvv;
        }
        
        // Getters
        public String getCardNumber() { return cardNumber; }
        public String getExpiryDate() { return expiryDate; }
        public String getCvv() { return cvv; }
    }
    
    static class PaymentResult {
        private String transactionId;
        private String status;
        private String errorMessage;
        
        public PaymentResult(String transactionId, String status, String errorMessage) {
            this.transactionId = transactionId;
            this.status = status;
            this.errorMessage = errorMessage;
        }
        
        public boolean isSuccess() { return "SUCCESS".equals(status); }
        
        public static PaymentResult success(String transactionId, String message) {
            return new PaymentResult(transactionId, "SUCCESS", null);
        }
        
        public static PaymentResult failed(String message) {
            return new PaymentResult(null, "FAILED", message);
        }
        
        // Getters
        public String getTransactionId() { return transactionId; }
        public String getStatus() { return status; }
        public String getErrorMessage() { return errorMessage; }
    }
    
    static class InventoryResult {
        private boolean success;
        private String message;
        
        public InventoryResult(boolean success, String message) {
            this.success = success;
            this.message = message;
        }
        
        // Getters
        public boolean isSuccess() { return success; }
        public String getMessage() { return message; }
    }
    
    static class EmailResult {
        private boolean success;
        private String message;
        
        public EmailResult(boolean success, String message) {
            this.success = success;
            this.message = message;
        }
        
        // Getters
        public boolean isSuccess() { return success; }
        public String getMessage() { return message; }
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
        
        public OrderData() {}
        
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
        
        public InventoryData() {}
        
        public InventoryData(int totalItems, int availableItems) {
            this.totalItems = totalItems;
            this.availableItems = availableItems;
        }
        
        // Getters
        public int getTotalItems() { return totalItems; }
        public int getAvailableItems() { return availableItems; }
    }
    
    static class AggregatedData {
        private UserData userData;
        private OrderData orderData;
        private InventoryData inventoryData;
        
        public AggregatedData(UserData userData, OrderData orderData, InventoryData inventoryData) {
            this.userData = userData;
            this.orderData = orderData;
            this.inventoryData = inventoryData;
        }
        
        // Getters
        public UserData getUserData() { return userData; }
        public OrderData getOrderData() { return orderData; }
        public InventoryData getInventoryData() { return inventoryData; }
    }
    
    static class ProcessingResult {
        private String result;
        private boolean success;
        
        public ProcessingResult(String result, boolean success) {
            this.result = result;
            this.success = success;
        }
        
        // Getters
        public String getResult() { return result; }
        public boolean isSuccess() { return success; }
    }
    
    static class PaymentException extends RuntimeException {
        public PaymentException(String message) {
            super(message);
        }
    }
}
```

## Reactive Programming with Project Reactor

### Advanced Reactor Patterns

```java
import reactor.core.publisher.*;
import reactor.core.scheduler.Schedulers;
import reactor.util.retry.Retry;
import reactor.core.Disposable;
import java.time.Duration;
import java.util.concurrent.atomic.AtomicInteger;

public class ReactorAdvancedPatterns {
    
    // Advanced flux operations and transformations
    public static class AdvancedFluxOperations {
        
        // Backpressure handling
        public Flux<String> handleBackpressure() {
            return Flux.range(1, 10000)
                .map(i -> "Item " + i)
                .onBackpressureBuffer(1000, (overflow) -> {
                    System.out.println("Dropped: " + overflow);
                })
                .observeOn(Schedulers.boundedElastic())
                .delayElements(Duration.ofMillis(10));
        }
        
        // Complex windowing and grouping
        public Flux<java.util.List<String>> windowProcessing() {
            return Flux.range(1, 100)
                .map(i -> "Data-" + i)
                .window(Duration.ofSeconds(2))
                .concatMap(window -> window.collectList())
                .filter(list -> !list.isEmpty());
        }
        
        // Combining multiple streams with different strategies
        public Flux<String> combineMultipleStreams() {
            Flux<String> stream1 = Flux.just("A", "B", "C").delayElements(Duration.ofMillis(100));
            Flux<String> stream2 = Flux.just("1", "2", "3").delayElements(Duration.ofMillis(150));
            Flux<String> stream3 = Flux.just("X", "Y", "Z").delayElements(Duration.ofMillis(200));
            
            return Flux.merge(stream1, stream2, stream3);
        }
        
        // Advanced error handling with retry and fallback
        public Flux<String> errorHandlingWithRetry() {
            AtomicInteger attemptCount = new AtomicInteger(0);
            
            return Flux.range(1, 10)
                .flatMap(i -> {
                    if (i == 5 && attemptCount.incrementAndGet() < 3) {
                        return Mono.error(new RuntimeException("Simulated error at " + i));
                    }
                    return Mono.just("Success-" + i);
                })
                .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
                          .filter(throwable -> throwable instanceof RuntimeException)
                          .onRetryExhaustedThrow((retryBackoffSpec, retrySignal) -> 
                              new RuntimeException("Retry exhausted")))
                .onErrorResume(throwable -> {
                    System.out.println("Fallback due to: " + throwable.getMessage());
                    return Mono.just("Fallback value");
                });
        }
        
        // Conditional processing with switchIfEmpty
        public Flux<String> conditionalProcessing() {
            return Flux.just("A", "B", "C")
                .filter(s -> s.length() > 1)
                .switchIfEmpty(Flux.just("Default"))
                .map(String::toUpperCase);
        }
        
        // Stateful processing with scan
        public Flux<Integer> statefulProcessing() {
            return Flux.range(1, 10)
                .scan(0, Integer::sum)
                .map(sum -> sum * 2);
        }
        
        // Buffering with time and count
        public Flux<java.util.List<String>> bufferingExample() {
            return Flux.interval(Duration.ofMillis(100))
                .take(50)
                .map(i -> "Event-" + i)
                .bufferTimeout(5, Duration.ofMillis(500))
                .filter(list -> !list.isEmpty());
        }
    }
    
    // Mono advanced patterns
    public static class AdvancedMonoOperations {
        
        // Conditional mono with switchIfEmpty
        public Mono<String> conditionalMono(String input) {
            return Mono.justOrEmpty(input)
                .filter(s -> s.length() > 3)
                .switchIfEmpty(Mono.defer(() -> fallbackOperation()));
        }
        
        private Mono<String> fallbackOperation() {
            return Mono.just("Default Value");
        }
        
        // Mono with side effects
        public Mono<String> monoWithSideEffects() {
            return Mono.just("Hello")
                .doOnNext(value -> System.out.println("Processing: " + value))
                .map(String::toUpperCase)
                .doOnSuccess(result -> System.out.println("Success: " + result))
                .doOnError(error -> System.out.println("Error: " + error.getMessage()));
        }
        
        // Mono composition with flatMap and zip
        public Mono<CombinedResult> composeMonos() {
            Mono<User> userMono = fetchUser("123");
            Mono<Preferences> preferencesMono = fetchPreferences("123");
            Mono<UserProfile> profileMono = fetchUserProfile("123");
            
            return Mono.zip(userMono, preferencesMono, profileMono)
                .map(tuple -> new CombinedResult(tuple.getT1(), tuple.getT2(), tuple.getT3()));
        }
        
        private Mono<User> fetchUser(String id) {
            return Mono.fromCallable(() -> new User(id, "John Doe"))
                      .subscribeOn(Schedulers.boundedElastic());
        }
        
        private Mono<Preferences> fetchPreferences(String id) {
            return Mono.fromCallable(() -> new Preferences("en", true, false))
                      .subscribeOn(Schedulers.boundedElastic());
        }
        
        private Mono<UserProfile> fetchUserProfile(String id) {
            return Mono.fromCallable(() -> new UserProfile("john", "john@example.com"))
                      .subscribeOn(Schedulers.boundedElastic());
        }
        
        // Timeout handling
        public Mono<String> timeoutHandling() {
            return Mono.fromCallable(() -> {
                    Thread.sleep(2000); // Simulate slow operation
                    return "Slow result";
                })
                .subscribeOn(Schedulers.boundedElastic())
                .timeout(Duration.ofMillis(1000))
                .onErrorResume(TimeoutException.class, 
                    ex -> Mono.just("Timeout occurred, using fallback"));
        }
        
        // Retry with exponential backoff
        public Mono<String> retryWithExponentialBackoff() {
            AtomicInteger attemptCount = new AtomicInteger(0);
            
            return Mono.fromCallable(() -> {
                    int attempt = attemptCount.incrementAndGet();
                    if (attempt < 3) {
                        throw new RuntimeException("Attempt " + attempt + " failed");
                    }
                    return "Success on attempt " + attempt;
                })
                .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
                          .maxBackoff(Duration.ofSeconds(2)))
                .doOnError(error -> System.out.println("Final error: " + error.getMessage()));
        }
    }
    
    // Hot and Cold Publishers
    public static class HotColdPublishers {
        
        // Cold publisher (each subscription creates new sequence)
        public Flux<String> coldPublisher() {
            return Flux.range(1, 5)
                .map(i -> "Cold-" + i + "-" + System.currentTimeMillis());
        }
        
        // Hot publisher with share (multiple subscribers share same sequence)
        public Flux<String> hotPublisher() {
            return Flux.interval(Duration.ofMillis(500))
                .take(10)
                .map(i -> "Hot-" + i)
                .share(); // Shares the subscription
        }
        
        // Connectable flux (manual control over when to start)
        public ConnectableFlux<String> connectablePublisher() {
            return Flux.interval(Duration.ofMillis(100))
                .take(20)
                .map(i -> "Connectable-" + i)
                .publish(); // Creates a connectable publisher
        }
        
        // Replay processor (replays values to new subscribers)
        public ReplayProcessor<String> replayProcessor() {
            ReplayProcessor<String> processor = ReplayProcessor.create(3); // Keep last 3 values
            
            Flux.interval(Duration.ofMillis(200))
                .take(10)
                .map(i -> "Replay-" + i)
                .subscribe(processor);
            
            return processor;
        }
        
        // Behavior processor (keeps and replays latest value)
        public BehaviorProcessor<String> behaviorProcessor() {
            BehaviorProcessor<String> processor = BehaviorProcessor.create("Initial");
            
            Flux.interval(Duration.ofMillis(300))
                .take(10)
                .map(i -> "Behavior-" + i)
                .subscribe(processor);
            
            return processor;
        }
    }
    
    // Reactive stream processing patterns
    public static class ReactiveStreamProcessing {
        
        // Batch processing with flatMap
        public Flux<ProcessedItem> processBatch(java.util.List<String> items) {
            return Flux.fromIterable(items)
                .flatMap(item -> processItem(item), 5) // Process up to 5 in parallel
                .onErrorContinue((error, item) -> {
                    System.out.println("Error processing item: " + item + ", error: " + error.getMessage());
                });
        }
        
        private Mono<ProcessedItem> processItem(String item) {
            return Mono.fromCallable(() -> {
                    // Simulate processing
                    Thread.sleep(100);
                    return new ProcessedItem(item, item.toUpperCase());
                })
                .subscribeOn(Schedulers.boundedElastic())
                .timeout(Duration.ofSeconds(2))
                .onErrorResume(TimeoutException.class, 
                    ex -> Mono.just(new ProcessedItem(item, "TIMEOUT")));
        }
        
        // Aggregation and reduction
        public Mono<AggregatedData> aggregateData() {
            return Flux.range(1, 1000)
                .map(i -> Math.random() * 100)
                .collect(Collectors::createAggregatedData, AggregatedData::accumulate);
        }
        
        // Window-based processing
        public Flux<WindowedResult> windowedProcessing() {
            return Flux.interval(Duration.ofMillis(100))
                .take(100)
                .window(Duration.ofSeconds(2))
                .concatMap(window -> 
                    window.count()
                        .map(count -> new WindowedResult(count, System.currentTimeMillis()))
                );
        }
        
        // Throttling and rate limiting
        public Flux<String> throttledStream() {
            return Flux.interval(Duration.ofMillis(10))
                .take(1000)
                .map(i -> "Item-" + i)
                .sample(Duration.ofMillis(100)); // Take one sample every 100ms
        }
        
        // Concurrency control with parallel flux
        public Flux<String> parallelProcessing() {
            return Flux.range(1, 100)
                .parallel(4) // Use 4 parallel threads
                .runOn(Schedulers.parallel())
                .map(i -> "Processed-" + i + "-Thread-" + Thread.currentThread().getName())
                .sequential(); // Convert back to sequential flux
        }
    }
    
    // Supporting classes
    static class User {
        private String id;
        private String name;
        
        public User(String id, String name) {
            this.id = id;
            this.name = name;
        }
        
        // Getters
        public String getId() { return id; }
        public String getName() { return name; }
    }
    
    static class Preferences {
        private String language;
        private boolean notifications;
        private boolean darkMode;
        
        public Preferences(String language, boolean notifications, boolean darkMode) {
            this.language = language;
            this.notifications = notifications;
            this.darkMode = darkMode;
        }
        
        // Getters
        public String getLanguage() { return language; }
        public boolean isNotifications() { return notifications; }
        public boolean isDarkMode() { return darkMode; }
    }
    
    static class UserProfile {
        private String username;
        private String email;
        
        public UserProfile(String username, String email) {
            this.username = username;
            this.email = email;
        }
        
        // Getters
        public String getUsername() { return username; }
        public String getEmail() { return email; }
    }
    
    static class CombinedResult {
        private User user;
        private Preferences preferences;
        private UserProfile profile;
        
        public CombinedResult(User user, Preferences preferences, UserProfile profile) {
            this.user = user;
            this.preferences = preferences;
            this.profile = profile;
        }
        
        // Getters
        public User getUser() { return user; }
        public Preferences getPreferences() { return preferences; }
        public UserProfile getProfile() { return profile; }
    }
    
    static class ProcessedItem {
        private String original;
        private String processed;
        
        public ProcessedItem(String original, String processed) {
            this.original = original;
            this.processed = processed;
        }
        
        // Getters
        public String getOriginal() { return original; }
        public String getProcessed() { return processed; }
    }
    
    static class AggregatedData {
        private double sum = 0;
        private int count = 0;
        private double min = Double.MAX_VALUE;
        private double max = Double.MIN_VALUE;
        
        public static AggregatedData createAggregatedData() {
            return new AggregatedData();
        }
        
        public void accumulate(Double value) {
            sum += value;
            count++;
            min = Math.min(min, value);
            max = Math.max(max, value);
        }
        
        // Getters
        public double getSum() { return sum; }
        public int getCount() { return count; }
        public double getMin() { return min; }
        public double getMax() { return max; }
        public double getAverage() { return count > 0 ? sum / count : 0; }
    }
    
    static class WindowedResult {
        private long count;
        private long timestamp;
        
        public WindowedResult(long count, long timestamp) {
            this.count = count;
            this.timestamp = timestamp;
        }
        
        // Getters
        public long getCount() { return count; }
        public long getTimestamp() { return timestamp; }
    }
    
    static class Collectors {
        public static AggregatedData createAggregatedData() {
            return new AggregatedData();
        }
    }
}
```

## Event-Driven Architecture

### Advanced Event Processing

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;
import java.util.function.Consumer;
import java.util.function.Function;
import java.util.stream.Collectors;

public class EventDrivenArchitecture {
    
    // Event bus implementation
    public static class EventBus {
        private final ExecutorService executor = Executors.newCachedThreadPool();
        private final java.util.Map<Class<?>, java.util.List<Consumer<?>>> handlers = 
            new ConcurrentHashMap<>();
        private final AtomicLong eventCounter = new AtomicLong(0);
        
        public <T> void subscribe(Class<T> eventType, Consumer<T> handler) {
            handlers.computeIfAbsent(eventType, k -> new CopyOnWriteArrayList<>())
                   .add(handler);
        }
        
        public <T> void publish(T event) {
            Class<?> eventType = event.getClass();
            java.util.List<Consumer<?>> eventHandlers = handlers.get(eventType);
            
            if (eventHandlers != null && !eventHandlers.isEmpty()) {
                eventCounter.incrementAndGet();
                
                for (Consumer<?> handler : eventHandlers) {
                    @SuppressWarnings("unchecked")
                    Consumer<T> typedHandler = (Consumer<T>) handler;
                    
                    executor.submit(() -> {
                        try {
                            typedHandler.accept(event);
                        } catch (Exception e) {
                            System.err.println("Error processing event: " + e.getMessage());
                        }
                    });
                }
            }
        }
        
        public void shutdown() {
            executor.shutdown();
            try {
                if (!executor.awaitTermination(10, TimeUnit.SECONDS)) {
                    executor.shutdownNow();
                }
            } catch (InterruptedException e) {
                executor.shutdownNow();
                Thread.currentThread().interrupt();
            }
        }
        
        public long getEventCount() {
            return eventCounter.get();
        }
    }
    
    // Event sourcing implementation
    public static class EventSourcing {
        private final java.util.List<Event> eventStore = new CopyOnWriteArrayList<>();
        private final java.util.Map<String, Aggregate> aggregates = new ConcurrentHashMap<>();
        
        public void saveEvent(String aggregateId, Event event) {
            eventStore.add(event);
            updateAggregate(aggregateId, event);
        }
        
        public java.util.List<Event> getEventsForAggregate(String aggregateId) {
            return eventStore.stream()
                .filter(event -> aggregateId.equals(event.getAggregateId()))
                .collect(Collectors.toList());
        }
        
        public Aggregate getAggregate(String aggregateId) {
            Aggregate aggregate = aggregates.get(aggregateId);
            if (aggregate == null) {
                aggregate = rebuildAggregate(aggregateId);
                aggregates.put(aggregateId, aggregate);
            }
            return aggregate;
        }
        
        private void updateAggregate(String aggregateId, Event event) {
            Aggregate aggregate = aggregates.computeIfAbsent(aggregateId, 
                id -> new Aggregate(id));
            aggregate.applyEvent(event);
        }
        
        private Aggregate rebuildAggregate(String aggregateId) {
            java.util.List<Event> events = getEventsForAggregate(aggregateId);
            Aggregate aggregate = new Aggregate(aggregateId);
            
            for (Event event : events) {
                aggregate.applyEvent(event);
            }
            
            return aggregate;
        }
        
        // Event and Aggregate base classes
        public static abstract class Event {
            private final String aggregateId;
            private final long sequenceNumber;
            private final java.time.Instant timestamp;
            
            public Event(String aggregateId, long sequenceNumber) {
                this.aggregateId = aggregateId;
                this.sequenceNumber = sequenceNumber;
                this.timestamp = java.time.Instant.now();
            }
            
            public String getAggregateId() { return aggregateId; }
            public long getSequenceNumber() { return sequenceNumber; }
            public java.time.Instant getTimestamp() { return timestamp; }
        }
        
        public static abstract class Aggregate {
            private final String id;
            private long version = 0;
            
            public Aggregate(String id) {
                this.id = id;
            }
            
            public abstract void applyEvent(Event event);
            
            public String getId() { return id; }
            public long getVersion() { return version; }
            
            protected void incrementVersion() {
                version++;
            }
        }
    }
    
    // CQRS implementation
    public static class CQRS {
        private final EventSourcing eventStore;
        private final QueryService queryService;
        
        public CQRS(EventSourcing eventStore, QueryService queryService) {
            this.eventStore = eventStore;
            this.queryService = queryService;
        }
        
        // Command side - writes
        public <T extends Command> void executeCommand(T command) {
            // Validate command
            if (validateCommand(command)) {
                // Create event from command
                Event event = createEventFromCommand(command);
                
                // Save to event store
                eventStore.saveEvent(command.getAggregateId(), event);
                
                // Update query model
                queryService.updateFromEvent(event);
            }
        }
        
        // Query side - reads
        public <T> T query(Query<T> query) {
            return queryService.executeQuery(query);
        }
        
        private boolean validateCommand(Command command) {
            // Command validation logic
            return true;
        }
        
        private Event createEventFromCommand(Command command) {
            // Create appropriate event based on command
            if (command instanceof CreateAccountCommand) {
                CreateAccountCommand cmd = (CreateAccountCommand) command;
                return new AccountCreatedEvent(cmd.getAggregateId(), cmd.getCustomerId(), cmd.getInitialBalance());
            }
            // Add more command mappings as needed
            return null;
        }
        
        // Base classes for CQRS
        public static abstract class Command {
            private final String aggregateId;
            
            public Command(String aggregateId) {
                this.aggregateId = aggregateId;
            }
            
            public String getAggregateId() { return aggregateId; }
        }
        
        public static abstract class Query<T> {
            private final String queryType;
            
            public Query(String queryType) {
                this.queryType = queryType;
            }
            
            public String getQueryType() { return queryType; }
        }
        
        public static class CreateAccountCommand extends Command {
            private final String customerId;
            private final double initialBalance;
            
            public CreateAccountCommand(String aggregateId, String customerId, double initialBalance) {
                super(aggregateId);
                this.customerId = customerId;
                this.initialBalance = initialBalance;
            }
            
            public String getCustomerId() { return customerId; }
            public double getInitialBalance() { return initialBalance; }
        }
        
        public static class AccountCreatedEvent extends EventSourcing.Event {
            private final String customerId;
            private final double initialBalance;
            
            public AccountCreatedEvent(String aggregateId, String customerId, double initialBalance) {
                super(aggregateId, System.nanoTime());
                this.customerId = customerId;
                this.initialBalance = initialBalance;
            }
            
            public String getCustomerId() { return customerId; }
            public double getInitialBalance() { return initialBalance; }
        }
    }
    
    // Query service implementation
    public static class QueryService {
        private final java.util.Map<String, Object> queryModels = new ConcurrentHashMap<>();
        
        public <T> T executeQuery(CQRS.Query<T> query) {
            // Execute query against appropriate query model
            Object model = queryModels.get(query.getQueryType());
            if (model != null) {
                @SuppressWarnings("unchecked")
                T result = (T) model;
                return result;
            }
            return null;
        }
        
        public void updateFromEvent(EventSourcing.Event event) {
            // Update query models based on event
            if (event instanceof CQRS.AccountCreatedEvent) {
                CQRS.AccountCreatedEvent accountEvent = (CQRS.AccountCreatedEvent) event;
                updateAccountQueryModel(accountEvent);
            }
            // Add more event handlers as needed
        }
        
        private void updateAccountQueryModel(CQRS.AccountCreatedEvent event) {
            // Update account query model
            AccountQueryModel model = (AccountQueryModel) queryModels.computeIfAbsent(
                "account_" + event.getAggregateId(), 
                k -> new AccountQueryModel()
            );
            
            model.setCustomerId(event.getCustomerId());
            model.setBalance(event.getInitialBalance());
            model.setCreated(true);
        }
        
        // Query model classes
        public static class AccountQueryModel {
            private String customerId;
            private double balance;
            private boolean created;
            
            public String getCustomerId() { return customerId; }
            public void setCustomerId(String customerId) { this.customerId = customerId; }
            
            public double getBalance() { return balance; }
            public void setBalance(double balance) { this.balance = balance; }
            
            public boolean isCreated() { return created; }
            public void setCreated(boolean created) { this.created = created; }
        }
    }
    
    // Message queue implementation
    public static class MessageQueue {
        private final BlockingQueue<Message> queue = new LinkedBlockingQueue<>();
        private final java.util.List<MessageConsumer> consumers = new CopyOnWriteArrayList<>();
        private final ExecutorService executor = Executors.newCachedThreadPool();
        
        public void sendMessage(Message message) {
            queue.offer(message);
        }
        
        public void addConsumer(MessageConsumer consumer) {
            consumers.add(consumer);
            
            // Start consumer in background
            executor.submit(() -> {
                while (!Thread.currentThread().isInterrupted()) {
                    try {
                        Message message = queue.take();
                        consumer.handleMessage(message);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        break;
                    } catch (Exception e) {
                        System.err.println("Error processing message: " + e.getMessage());
                    }
                }
            });
        }
        
        public void shutdown() {
            executor.shutdown();
            try {
                if (!executor.awaitTermination(10, TimeUnit.SECONDS)) {
                    executor.shutdownNow();
                }
            } catch (InterruptedException e) {
                executor.shutdownNow();
                Thread.currentThread().interrupt();
            }
        }
        
        public int getQueueSize() {
            return queue.size();
        }
        
        // Message and Consumer interfaces
        public interface Message {
            String getId();
            String getType();
            java.util.Map<String, Object> getPayload();
        }
        
        public interface MessageConsumer {
            void handleMessage(Message message) throws Exception;
        }
    }
    
    // Event processor with backpressure handling
    public static class EventProcessor {
        private final FluxSink<String> sink;
        private final Disposable subscription;
        
        public EventProcessor(Consumer<String> eventHandler) {
            Sinks.Many<String> sink = Sinks.many().multicast().onBackpressureBuffer(1000);
            
            this.sink = sink.asFlux()
                .onBackpressureBuffer(1000)
                .observeOn(Schedulers.boundedElastic())
                .subscribe(eventHandler::accept);
                
            this.subscription = sink.asFlux()
                .subscribe(event -> {
                    try {
                        eventHandler.accept(event);
                    } catch (Exception e) {
                        System.err.println("Error processing event: " + e.getMessage());
                    }
                });
        }
        
        public void processEvent(String event) {
            Sinks.EmitResult result = ((Sinks.Many<String>) sink).tryEmitNext(event);
            if (result == Sinks.EmitResult.FAIL_NON_SERIALIZED) {
                System.out.println("Event dropped due to backpressure");
            }
        }
        
        public void shutdown() {
            subscription.dispose();
        }
    }
    
    // Saga pattern implementation
    public static class SagaOrchestrator {
        private final java.util.Map<String, SagaInstance> activeSagas = new ConcurrentHashMap<>();
        
        public void startSaga(String sagaId, SagaDefinition definition) {
            SagaInstance instance = new SagaInstance(sagaId, definition);
            activeSagas.put(sagaId, instance);
            
            // Execute first step
            executeStep(instance, 0);
        }
        
        private void executeStep(SagaInstance instance, int stepIndex) {
            if (stepIndex >= instance.getDefinition().getSteps().size()) {
                // Saga completed successfully
                instance.setStatus(SagaStatus.COMPLETED);
                activeSagas.remove(instance.getId());
                return;
            }
            
            SagaStep step = instance.getDefinition().getSteps().get(stepIndex);
            step.execute()
                .subscribe(
                    result -> {
                        instance.getStepResults().put(stepIndex, result);
                        executeStep(instance, stepIndex + 1);
                    },
                    error -> handleStepFailure(instance, stepIndex, error)
                );
        }
        
        private void handleStepFailure(SagaInstance instance, int stepIndex, Throwable error) {
            instance.setStatus(SagaStatus.FAILED);
            
            // Execute compensating transactions in reverse order
            for (int i = stepIndex - 1; i >= 0; i--) {
                SagaStep compensation = instance.getDefinition().getSteps().get(i).getCompensation();
                if (compensation != null) {
                    compensation.execute().subscribe();
                }
            }
        }
        
        // Saga classes
        public static class SagaDefinition {
            private final java.util.List<SagaStep> steps = new java.util.ArrayList<>();
            
            public void addStep(SagaStep step) {
                steps.add(step);
            }
            
            public java.util.List<SagaStep> getSteps() { return steps; }
        }
        
        public static class SagaStep {
            private final Function<Void, Mono<Object>> executeFunction;
            private final Function<Void, Mono<Object>> compensationFunction;
            
            public SagaStep(Function<Void, Mono<Object>> executeFunction) {
                this.executeFunction = executeFunction;
                this.compensationFunction = null;
            }
            
            public SagaStep(Function<Void, Mono<Object>> executeFunction, 
                           Function<Void, Mono<Object>> compensationFunction) {
                this.executeFunction = executeFunction;
                this.compensationFunction = compensationFunction;
            }
            
            public Mono<Object> execute() {
                return executeFunction.apply(null);
            }
            
            public Mono<Object> compensate() {
                if (compensationFunction != null) {
                    return compensationFunction.apply(null);
                }
                return Mono.empty();
            }
            
            public SagaStep getCompensation() {
                return compensationFunction != null ? 
                    new SagaStep(compensationFunction) : null;
            }
        }
        
        public static class SagaInstance {
            private final String id;
            private final SagaDefinition definition;
            private SagaStatus status = SagaStatus.PENDING;
            private final java.util.Map<Integer, Object> stepResults = new java.util.HashMap<>();
            
            public SagaInstance(String id, SagaDefinition definition) {
                this.id = id;
                this.definition = definition;
            }
            
            // Getters and setters
            public String getId() { return id; }
            public SagaDefinition getDefinition() { return definition; }
            public SagaStatus getStatus() { return status; }
            public void setStatus(SagaStatus status) { this.status = status; }
            public java.util.Map<Integer, Object> getStepResults() { return stepResults; }
        }
        
        public enum SagaStatus {
            PENDING, EXECUTING, COMPLETED, FAILED, COMPENSATED
        }
    }
}
```

## Reactive Integration Patterns

### Advanced Integration Scenarios

```java
import reactor.core.publisher.Sinks;
import reactor.util.retry.Retry;
import java.time.Duration;
import java.util.concurrent.atomic.AtomicInteger;

public class ReactiveIntegrationPatterns {
    
    // Reactive API Gateway
    public static class ReactiveApiGateway {
        private final java.util.Map<String, ServiceClient> serviceClients = new java.util.HashMap<>();
        
        public void registerService(String serviceName, ServiceClient client) {
            serviceClients.put(serviceName, client);
        }
        
        public Mono<ApiResponse> routeRequest(String serviceName, ApiRequest request) {
            ServiceClient client = serviceClients.get(serviceName);
            if (client == null) {
                return Mono.error(new IllegalArgumentException("Service not found: " + serviceName));
            }
            
            return client.call(request)
                .timeout(Duration.ofSeconds(10))
                .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
                          .filter(throwable -> isRetryable(throwable)))
                .onErrorResume(throwable -> 
                    handleFallback(serviceName, request, throwable));
        }
        
        private boolean isRetryable(Throwable throwable) {
            // Determine if the error is retryable
            return throwable instanceof java.net.ConnectException ||
                   throwable instanceof java.util.concurrent.TimeoutException ||
                   (throwable instanceof java.net.SocketException && 
                    throwable.getMessage().contains("Connection reset"));
        }
        
        private Mono<ApiResponse> handleFallback(String serviceName, ApiRequest request, Throwable error) {
            System.out.println("Using fallback for service: " + serviceName + ", error: " + error.getMessage());
            
            // Implement circuit breaker fallback logic
            return Mono.just(ApiResponse.fallback("Service unavailable: " + serviceName));
        }
    }
    
    // Service client interface
    public interface ServiceClient {
        Mono<ApiResponse> call(ApiRequest request);
    }
    
    // Reactive caching with Redis
    public static class ReactiveCache {
        private final java.util.Map<String, Mono<Object>> cache = new java.util.concurrent.ConcurrentHashMap<>();
        
        public <T> Mono<T> get(String key, Class<T> type) {
            return Optional.ofNullable((Mono<T>) cache.get(key))
                .orElse(Mono.empty());
        }
        
        public <T> Mono<T> getOrLoad(String key, Supplier<Mono<T>> loader) {
            return get(key, Object.class)
                .cast(Object.class)
                .switchIfEmpty(
                    loader.get()
                        .doOnNext(value -> cache.put(key, Mono.just(value)))
                        .cache(Duration.ofMinutes(5)) // Cache for 5 minutes
                );
        }
        
        public Mono<Void> invalidate(String key) {
            cache.remove(key);
            return Mono.empty();
        }
        
        public Mono<Void> clear() {
            cache.clear();
            return Mono.empty();
        }
    }
    
    // Reactive rate limiting
    public static class ReactiveRateLimiter {
        private final java.util.Map<String, TokenBucket> buckets = new java.util.concurrent.ConcurrentHashMap<>();
        private final int requestsPerSecond;
        
        public ReactiveRateLimiter(int requestsPerSecond) {
            this.requestsPerSecond = requestsPerSecond;
        }
        
        public Mono<Boolean> acquire(String clientId) {
            TokenBucket bucket = buckets.computeIfAbsent(clientId, 
                id -> new TokenBucket(requestsPerSecond));
            
            return Mono.fromCallable(() -> bucket.tryConsume(1))
                .subscribeOn(Schedulers.boundedElastic())
                .timeout(Duration.ofMillis(100));
        }
        
        private static class TokenBucket {
            private final int capacity;
            private final double refillRate; // tokens per millisecond
            private final AtomicInteger tokens;
            private volatile long lastRefillTime;
            
            public TokenBucket(int requestsPerSecond) {
                this.capacity = requestsPerSecond; // Burst capacity
                this.refillRate = requestsPerSecond / 1000.0; // tokens per ms
                this.tokens = new AtomicInteger(capacity);
                this.lastRefillTime = System.currentTimeMillis();
            }
            
            public boolean tryConsume(int numTokens) {
                refill();
                
                int currentTokens = tokens.get();
                if (currentTokens >= numTokens) {
                    return tokens.compareAndSet(currentTokens, currentTokens - numTokens);
                }
                return false;
            }
            
            private void refill() {
                long now = System.currentTimeMillis();
                long timePassed = now - lastRefillTime;
                
                int newTokens = (int) (timePassed * refillRate);
                if (newTokens > 0) {
                    tokens.accumulateAndGet(Math.min(capacity, tokens.get() + newTokens), 
                        (current, increment) -> Math.min(capacity, current + increment));
                    lastRefillTime = now;
                }
            }
        }
    }
    
    // Reactive bulkhead pattern
    public static class ReactiveBulkhead {
        private final Semaphore semaphore;
        private final int maxConcurrent;
        
        public ReactiveBulkhead(int maxConcurrent) {
            this.maxConcurrent = maxConcurrent;
            this.semaphore = new Semaphore(maxConcurrent);
        }
        
        public <T> Mono<T> execute(Supplier<Mono<T>> operation) {
            return Mono.fromCallable(semaphore::acquire)
                .then(operation.get())
                .doFinally(signalType -> semaphore.release())
                .timeout(Duration.ofSeconds(30));
        }
        
        public int getAvailablePermits() {
            return semaphore.availablePermits();
        }
        
        public int getMaxConcurrent() {
            return maxConcurrent;
        }
    }
    
    // Reactive circuit breaker
    public static class ReactiveCircuitBreaker {
        private final int failureThreshold;
        private final Duration timeout;
        private final AtomicInteger failureCount = new AtomicInteger(0);
        private volatile long lastFailureTime = 0;
        private volatile CircuitState state = CircuitState.CLOSED;
        private final AtomicInteger attemptCount = new AtomicInteger(0);
        
        public ReactiveCircuitBreaker(int failureThreshold, Duration timeout) {
            this.failureThreshold = failureThreshold;
            this.timeout = timeout;
        }
        
        public <T> Mono<T> execute(Supplier<Mono<T>> operation) {
            if (state == CircuitState.OPEN) {
                if (System.currentTimeMillis() - lastFailureTime > timeout.toMillis()) {
                    state = CircuitState.HALF_OPEN;
                    attemptCount.set(0);
                } else {
                    return Mono.error(new CircuitBreakerOpenException("Circuit is OPEN"));
                }
            }
            
            return operation.get()
                .doOnSuccess(result -> {
                    if (state == CircuitState.HALF_OPEN) {
                        // Success in half-open state closes the circuit
                        failureCount.set(0);
                        state = CircuitState.CLOSED;
                    }
                })
                .doOnError(error -> {
                    recordFailure();
                })
                .timeout(timeout);
        }
        
        private void recordFailure() {
            int count = failureCount.incrementAndGet();
            lastFailureTime = System.currentTimeMillis();
            
            if (count >= failureThreshold) {
                state = CircuitState.OPEN;
            }
        }
        
        public CircuitState getState() {
            return state;
        }
        
        public int getFailureCount() {
            return failureCount.get();
        }
        
        public enum CircuitState {
            CLOSED, OPEN, HALF_OPEN
        }
        
        public static class CircuitBreakerOpenException extends RuntimeException {
            public CircuitBreakerOpenException(String message) {
                super(message);
            }
        }
    }
    
    // Supporting classes for integration
    public static class ApiRequest {
        private final String method;
        private final String path;
        private final java.util.Map<String, String> headers;
        private final String body;
        
        public ApiRequest(String method, String path, java.util.Map<String, String> headers, String body) {
            this.method = method;
            this.path = path;
            this.headers = headers != null ? new java.util.HashMap<>(headers) : new java.util.HashMap<>();
            this.body = body;
        }
        
        // Getters
        public String getMethod() { return method; }
        public String getPath() { return path; }
        public java.util.Map<String, String> getHeaders() { return headers; }
        public String getBody() { return body; }
    }
    
    public static class ApiResponse {
        private final int statusCode;
        private final java.util.Map<String, String> headers;
        private final String body;
        private final boolean isFallback;
        
        public ApiResponse(int statusCode, java.util.Map<String, String> headers, String body) {
            this.statusCode = statusCode;
            this.headers = headers != null ? new java.util.HashMap<>(headers) : new java.util.HashMap<>();
            this.body = body;
            this.isFallback = false;
        }
        
        public static ApiResponse fallback(String message) {
            ApiResponse response = new ApiResponse(503, null, message);
            response.isFallback = true;
            return response;
        }
        
        // Getters
        public int getStatusCode() { return statusCode; }
        public java.util.Map<String, String> getHeaders() { return headers; }
        public String getBody() { return body; }
        public boolean isFallback() { return isFallback; }
    }
    
    @FunctionalInterface
    public interface Supplier<T> {
        T get() throws Exception;
    }
}
```

## Conclusion

Advanced Asynchronous Programming and Reactive Systems provide:
- Sophisticated CompletableFuture chaining and composition patterns
- Reactive programming with Project Reactor for non-blocking, event-driven applications
- Event-driven architecture for decoupled, scalable systems
- Advanced integration patterns for reactive microservices

Mastering these concepts is essential for:
- Building responsive, resilient, and scalable applications
- Implementing event-driven and reactive architectures
- Creating high-performance, non-blocking systems
- Handling complex asynchronous workflows
- Building modern microservices and cloud-native applications