# Advanced Microservices Architecture and Design Patterns

## Overview
Microservices architecture has become the standard for building scalable, maintainable, and resilient distributed systems. This tutorial covers advanced microservices design patterns, architecture principles, communication strategies, and implementation best practices for building enterprise-grade microservices.

## Microservices Architecture Fundamentals

### Service Decomposition and Domain-Driven Design

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.function.Function;
import java.util.stream.Collectors;

public class MicroservicesArchitecture {
    
    // Domain-Driven Design implementation for microservices
    public static class DomainDrivenDesign {
        
        // Bounded Context representation
        public static class BoundedContext {
            private final String name;
            private final String description;
            private final Set<Aggregate> aggregates;
            private final Set<ValueObject> valueObjects;
            private final Set<Entity> entities;
            
            public BoundedContext(String name, String description) {
                this.name = name;
                this.description = description;
                this.aggregates = ConcurrentHashMap.newKeySet();
                this.valueObjects = ConcurrentHashMap.newKeySet();
                this.entities = ConcurrentHashMap.newKeySet();
            }
            
            public void addAggregate(Aggregate aggregate) {
                aggregates.add(aggregate);
            }
            
            public void addValueObject(ValueObject valueObject) {
                valueObjects.add(valueObject);
            }
            
            public void addEntity(Entity entity) {
                entities.add(entity);
            }
            
            public String getName() { return name; }
            public String getDescription() { return description; }
            public Set<Aggregate> getAggregates() { return new HashSet<>(aggregates); }
            public Set<ValueObject> getValueObjects() { return new HashSet<>(valueObjects); }
            public Set<Entity> getEntities() { return new HashSet<>(entities); }
        }
        
        // Aggregate root pattern
        public static abstract class Aggregate {
            protected String id;
            protected long version;
            protected List<DomainEvent> domainEvents = new ArrayList<>();
            
            public Aggregate(String id) {
                this.id = id;
                this.version = 0;
            }
            
            protected void applyEvent(DomainEvent event) {
                domainEvents.add(event);
                version++;
            }
            
            public String getId() { return id; }
            public long getVersion() { return version; }
            public List<DomainEvent> getDomainEvents() { return new ArrayList<>(domainEvents); }
            public void clearDomainEvents() { domainEvents.clear(); }
        }
        
        // Entity with identity
        public static abstract class Entity {
            protected String id;
            
            public Entity(String id) {
                this.id = id;
            }
            
            public String getId() { return id; }
            
            @Override
            public boolean equals(Object obj) {
                if (this == obj) return true;
                if (obj == null || getClass() != obj.getClass()) return false;
                Entity entity = (Entity) obj;
                return Objects.equals(id, entity.id);
            }
            
            @Override
            public int hashCode() {
                return Objects.hash(id);
            }
        }
        
        // Value object (immutable, no identity)
        public static abstract class ValueObject {
            @Override
            public boolean equals(Object obj) {
                if (this == obj) return true;
                if (obj == null || getClass() != obj.getClass()) return false;
                ValueObject that = (ValueObject) obj;
                return Objects.equals(getAttributes(), that.getAttributes());
            }
            
            @Override
            public int hashCode() {
                return Objects.hash(getAttributes());
            }
            
            protected abstract List<Object> getAttributes();
        }
        
        // Domain event
        public static abstract class DomainEvent {
            private final String eventType;
            private final long timestamp;
            private final String aggregateId;
            
            public DomainEvent(String eventType, String aggregateId) {
                this.eventType = eventType;
                this.aggregateId = aggregateId;
                this.timestamp = System.currentTimeMillis();
            }
            
            public String getEventType() { return eventType; }
            public long getTimestamp() { return timestamp; }
            public String getAggregateId() { return aggregateId; }
        }
        
        // Service layer with domain logic
        public static class DomainService {
            private final Map<String, BoundedContext> boundedContexts = new ConcurrentHashMap<>();
            
            public void registerBoundedContext(BoundedContext context) {
                boundedContexts.put(context.getName(), context);
            }
            
            public BoundedContext getBoundedContext(String name) {
                return boundedContexts.get(name);
            }
            
            public List<String> validateServiceBoundaries(String serviceName, 
                                                        List<String> dependencies) {
                List<String> violations = new ArrayList<>();
                
                BoundedContext context = boundedContexts.get(serviceName);
                if (context == null) {
                    violations.add("Service " + serviceName + " not found");
                    return violations;
                }
                
                // Validate that dependencies don't cross bounded context boundaries unnecessarily
                for (String dependency : dependencies) {
                    BoundedContext depContext = boundedContexts.get(dependency);
                    if (depContext != null && !depContext.equals(context)) {
                        // Check if dependency is allowed
                        if (!isAllowedCrossContextDependency(serviceName, dependency)) {
                            violations.add("Unauthorized cross-context dependency: " + 
                                         serviceName + " -> " + dependency);
                        }
                    }
                }
                
                return violations;
            }
            
            private boolean isAllowedCrossContextDependency(String source, String target) {
                // Implement cross-context dependency rules
                // This would typically be configured in a dependency matrix
                return true; // Simplified
            }
        }
    }
    
    // Service decomposition strategies
    public static class ServiceDecomposition {
        
        // Business capability-based decomposition
        public static class BusinessCapabilityDecomposer {
            private final Map<String, BusinessCapability> capabilities = new HashMap<>();
            
            public void addCapability(String name, String description, 
                                    List<String> domainEntities) {
                capabilities.put(name, new BusinessCapability(name, description, domainEntities));
            }
            
            public List<ServiceDefinition> decomposeToServices() {
                List<ServiceDefinition> services = new ArrayList<>();
                
                for (BusinessCapability capability : capabilities.values()) {
                    ServiceDefinition service = new ServiceDefinition(
                        capability.getName(),
                        capability.getDescription(),
                        new ArrayList<>(capability.getDomainEntities())
                    );
                    services.add(service);
                }
                
                return services;
            }
            
            public ServiceDecompositionResult analyzeDependencies(
                    Map<String, List<String>> serviceDependencies) {
                
                ServiceDecompositionResult result = new ServiceDecompositionResult();
                
                // Analyze coupling between services
                for (Map.Entry<String, List<String>> entry : serviceDependencies.entrySet()) {
                    String service = entry.getKey();
                    List<String> dependencies = entry.getValue();
                    
                    if (dependencies.size() > 5) { // High coupling threshold
                        result.addHighCouplingService(service, dependencies.size());
                    }
                    
                    // Check for circular dependencies
                    for (String dependency : dependencies) {
                        List<String> reverseDeps = serviceDependencies.get(dependency);
                        if (reverseDeps != null && reverseDeps.contains(service)) {
                            result.addCircularDependency(service, dependency);
                        }
                    }
                }
                
                return result;
            }
            
            private static class BusinessCapability {
                private final String name;
                private final String description;
                private final List<String> domainEntities;
                
                public BusinessCapability(String name, String description, List<String> domainEntities) {
                    this.name = name;
                    this.description = description;
                    this.domainEntities = domainEntities;
                }
                
                public String getName() { return name; }
                public String getDescription() { return description; }
                public List<String> getDomainEntities() { return domainEntities; }
            }
        }
        
        // Data-driven decomposition
        public static class DataDrivenDecomposer {
            private final Map<String, DatabaseTable> tables = new HashMap<>();
            
            public void addTable(DatabaseTable table) {
                tables.put(table.getName(), table);
            }
            
            public List<ServiceDefinition> decomposeByDataRelationships() {
                Map<String, Set<String>> clusters = new HashMap<>();
                UnionFind unionFind = new UnionFind(tables.size());
                Map<String, Integer> tableIndexMap = new HashMap<>();
                
                // Create index mapping
                int index = 0;
                for (String tableName : tables.keySet()) {
                    tableIndexMap.put(tableName, index++);
                }
                
                // Cluster related tables
                for (DatabaseTable table : tables.values()) {
                    for (ForeignKey foreignKey : table.getForeignKeys()) {
                        String referencedTable = foreignKey.getReferencedTable();
                        if (tables.containsKey(referencedTable)) {
                            int tableIndex = tableIndexMap.get(table.getName());
                            int refIndex = tableIndexMap.get(referencedTable);
                            unionFind.union(tableIndex, refIndex);
                        }
                    }
                }
                
                // Group tables by cluster
                Map<Integer, Set<String>> clusterGroups = new HashMap<>();
                for (String tableName : tables.keySet()) {
                    int root = unionFind.find(tableIndexMap.get(tableName));
                    clusterGroups.computeIfAbsent(root, k -> new HashSet<>()).add(tableName);
                }
                
                // Create services from clusters
                List<ServiceDefinition> services = new ArrayList<>();
                for (Set<String> cluster : clusterGroups.values()) {
                    if (!cluster.isEmpty()) {
                        String serviceName = "service_" + cluster.iterator().next();
                        services.add(new ServiceDefinition(
                            serviceName,
                            "Data-clustered service",
                            new ArrayList<>(cluster)
                        ));
                    }
                }
                
                return services;
            }
            
            private static class UnionFind {
                private final int[] parent;
                private final int[] rank;
                
                public UnionFind(int size) {
                    parent = new int[size];
                    rank = new int[size];
                    for (int i = 0; i < size; i++) {
                        parent[i] = i;
                    }
                }
                
                public int find(int x) {
                    if (parent[x] != x) {
                        parent[x] = find(parent[x]); // Path compression
                    }
                    return parent[x];
                }
                
                public void union(int x, int y) {
                    int rootX = find(x);
                    int rootY = find(y);
                    
                    if (rootX != rootY) {
                        // Union by rank
                        if (rank[rootX] < rank[rootY]) {
                            parent[rootX] = rootY;
                        } else if (rank[rootX] > rank[rootY]) {
                            parent[rootY] = rootX;
                        } else {
                            parent[rootY] = rootX;
                            rank[rootX]++;
                        }
                    }
                }
            }
        }
        
        // Supporting classes
        public static class ServiceDefinition {
            private final String name;
            private final String description;
            private final List<String> domainEntities;
            private final Map<String, String> configuration;
            
            public ServiceDefinition(String name, String description, List<String> domainEntities) {
                this.name = name;
                this.description = description;
                this.domainEntities = domainEntities;
                this.configuration = new HashMap<>();
            }
            
            public void addConfiguration(String key, String value) {
                configuration.put(key, value);
            }
            
            public String getName() { return name; }
            public String getDescription() { return description; }
            public List<String> getDomainEntities() { return domainEntities; }
            public Map<String, String> getConfiguration() { return configuration; }
        }
        
        public static class DatabaseTable {
            private final String name;
            private final List<ForeignKey> foreignKeys;
            
            public DatabaseTable(String name) {
                this.name = name;
                this.foreignKeys = new ArrayList<>();
            }
            
            public void addForeignKey(ForeignKey foreignKey) {
                foreignKeys.add(foreignKey);
            }
            
            public String getName() { return name; }
            public List<ForeignKey> getForeignKeys() { return foreignKeys; }
        }
        
        public static class ForeignKey {
            private final String columnName;
            private final String referencedTable;
            private final String referencedColumn;
            
            public ForeignKey(String columnName, String referencedTable, String referencedColumn) {
                this.columnName = columnName;
                this.referencedTable = referencedTable;
                this.referencedColumn = referencedColumn;
            }
            
            public String getColumnName() { return columnName; }
            public String getReferencedTable() { return referencedTable; }
            public String getReferencedColumn() { return referencedColumn; }
        }
        
        public static class ServiceDecompositionResult {
            private final List<String> highCouplingServices = new ArrayList<>();
            private final List<String> circularDependencies = new ArrayList<>();
            private final Map<String, Integer> couplingCounts = new HashMap<>();
            
            public void addHighCouplingService(String service, int couplingCount) {
                highCouplingServices.add(service);
                couplingCounts.put(service, couplingCount);
            }
            
            public void addCircularDependency(String service1, String service2) {
                circularDependencies.add(service1 + " <-> " + service2);
            }
            
            public List<String> getHighCouplingServices() { return highCouplingServices; }
            public List<String> getCircularDependencies() { return circularDependencies; }
            public Map<String, Integer> getCouplingCounts() { return couplingCounts; }
        }
    }
}
```

## Microservices Communication Patterns

### Advanced Communication Strategies

```java
import reactor.core.publisher.Mono;
import reactor.core.publisher.Flux;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;
import java.util.function.Function;
import java.time.Duration;
import java.util.concurrent.atomic.AtomicInteger;

public class MicroservicesCommunication {
    
    // Synchronous communication with resilience patterns
    public static class SynchronousCommunication {
        
        // HTTP client with circuit breaker and retry
        public static class ResilientHttpClient {
            private final CircuitBreaker circuitBreaker;
            private final RetryPolicy retryPolicy;
            private final LoadBalancer loadBalancer;
            
            public ResilientHttpClient(CircuitBreaker circuitBreaker, 
                                     RetryPolicy retryPolicy,
                                     LoadBalancer loadBalancer) {
                this.circuitBreaker = circuitBreaker;
                this.retryPolicy = retryPolicy;
                this.loadBalancer = loadBalancer;
            }
            
            public <T> CompletableFuture<HttpResponse<T>> sendRequest(
                    HttpRequest request, 
                    Class<T> responseType) {
                
                return circuitBreaker.execute(
                    () -> retryPolicy.executeWithRetry(
                        () -> sendRequestInternal(request, responseType)
                    )
                );
            }
            
            private <T> CompletableFuture<HttpResponse<T>> sendRequestInternal(
                    HttpRequest request, 
                    Class<T> responseType) {
                
                String serviceUrl = loadBalancer.selectService(request.getServiceName());
                String fullUrl = serviceUrl + request.getPath();
                
                return CompletableFuture.supplyAsync(() -> {
                    try {
                        // Simulate HTTP call
                        Thread.sleep(100); // Simulate network delay
                        
                        if (Math.random() < 0.1) { // 10% failure rate
                            throw new RuntimeException("Service temporarily unavailable");
                        }
                        
                        T response = parseResponse(responseType);
                        return new HttpResponse<>(200, response, System.currentTimeMillis());
                    } catch (Exception e) {
                        throw new RuntimeException("HTTP request failed", e);
                    }
                });
            }
            
            private <T> T parseResponse(Class<T> responseType) {
                // Simulate response parsing
                return null; // In real implementation, parse actual response
            }
        }
        
        // Circuit breaker implementation
        public static class CircuitBreaker {
            private final String name;
            private final int failureThreshold;
            private final long timeoutMillis;
            private final AtomicInteger failureCount = new AtomicInteger(0);
            private volatile long lastFailureTime = 0;
            private volatile CircuitState state = CircuitState.CLOSED;
            
            public CircuitBreaker(String name, int failureThreshold, long timeoutMillis) {
                this.name = name;
                this.failureThreshold = failureThreshold;
                this.timeoutMillis = timeoutMillis;
            }
            
            public <T> CompletableFuture<T> execute(Supplier<CompletableFuture<T>> operation) {
                if (state == CircuitState.OPEN) {
                    if (System.currentTimeMillis() - lastFailureTime > timeoutMillis) {
                        state = CircuitState.HALF_OPEN;
                    } else {
                        return CompletableFuture.failedFuture(
                            new CircuitBreakerOpenException("Circuit breaker is OPEN for " + name));
                    }
                }
                
                return operation.get()
                    .whenComplete((result, throwable) -> {
                        if (throwable != null) {
                            recordFailure();
                        } else {
                            recordSuccess();
                        }
                    });
            }
            
            private void recordFailure() {
                int count = failureCount.incrementAndGet();
                lastFailureTime = System.currentTimeMillis();
                
                if (count >= failureThreshold) {
                    state = CircuitState.OPEN;
                    System.out.println("Circuit breaker " + name + " OPENED");
                }
            }
            
            private void recordSuccess() {
                failureCount.set(0);
                if (state == CircuitState.HALF_OPEN) {
                    state = CircuitState.CLOSED;
                    System.out.println("Circuit breaker " + name + " CLOSED");
                }
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
        
        // Retry policy with exponential backoff
        public static class RetryPolicy {
            private final int maxRetries;
            private final long initialDelayMillis;
            private final double backoffMultiplier;
            
            public RetryPolicy(int maxRetries, long initialDelayMillis, double backoffMultiplier) {
                this.maxRetries = maxRetries;
                this.initialDelayMillis = initialDelayMillis;
                this.backoffMultiplier = backoffMultiplier;
            }
            
            public <T> CompletableFuture<T> executeWithRetry(Supplier<CompletableFuture<T>> operation) {
                return executeWithRetryInternal(operation, 0, initialDelayMillis);
            }
            
            private <T> CompletableFuture<T> executeWithRetryInternal(
                    Supplier<CompletableFuture<T>> operation,
                    int attempt,
                    long delayMillis) {
                
                return operation.get()
                    .handle((result, throwable) -> {
                        if (throwable == null) {
                            return CompletableFuture.completedFuture(result);
                        } else if (attempt < maxRetries) {
                            return CompletableFuture.delayedExecutor(delayMillis, TimeUnit.MILLISECONDS)
                                .execute(() -> executeWithRetryInternal(
                                    operation, 
                                    attempt + 1, 
                                    (long) (delayMillis * backoffMultiplier)
                                ));
                        } else {
                            return CompletableFuture.<T>failedFuture(throwable);
                        }
                    })
                    .thenCompose(Function.identity());
            }
        }
        
        // Load balancer with different strategies
        public static class LoadBalancer {
            private final String serviceName;
            private final List<String> serviceUrls;
            private final LoadBalancingStrategy strategy;
            private final AtomicInteger currentIndex = new AtomicInteger(0);
            
            public LoadBalancer(String serviceName, List<String> serviceUrls, 
                              LoadBalancingStrategy strategy) {
                this.serviceName = serviceName;
                this.serviceUrls = new ArrayList<>(serviceUrls);
                this.strategy = strategy;
            }
            
            public String selectService(String serviceName) {
                if (serviceUrls.isEmpty()) {
                    throw new IllegalStateException("No service URLs available");
                }
                
                switch (strategy) {
                    case ROUND_ROBIN:
                        return serviceUrls.get(
                            currentIndex.getAndUpdate(i -> (i + 1) % serviceUrls.size())
                        );
                    case RANDOM:
                        return serviceUrls.get(new Random().nextInt(serviceUrls.size()));
                    case WEIGHTED:
                        return selectWeightedService();
                    default:
                        return serviceUrls.get(0);
                }
            }
            
            private String selectWeightedService() {
                // Simplified weighted selection
                return serviceUrls.get(new Random().nextInt(serviceUrls.size()));
            }
            
            public enum LoadBalancingStrategy {
                ROUND_ROBIN, RANDOM, WEIGHTED
            }
        }
        
        // Supporting classes
        public static class HttpRequest {
            private final String serviceName;
            private final String path;
            private final String method;
            private final Map<String, String> headers;
            private final String body;
            
            public HttpRequest(String serviceName, String path, String method) {
                this.serviceName = serviceName;
                this.path = path;
                this.method = method;
                this.headers = new HashMap<>();
                this.body = null;
            }
            
            public void addHeader(String key, String value) {
                headers.put(key, value);
            }
            
            public String getServiceName() { return serviceName; }
            public String getPath() { return path; }
            public String getMethod() { return method; }
            public Map<String, String> getHeaders() { return headers; }
            public String getBody() { return body; }
        }
        
        public static class HttpResponse<T> {
            private final int statusCode;
            private final T body;
            private final long timestamp;
            
            public HttpResponse(int statusCode, T body, long timestamp) {
                this.statusCode = statusCode;
                this.body = body;
                this.timestamp = timestamp;
            }
            
            public int getStatusCode() { return statusCode; }
            public T getBody() { return body; }
            public long getTimestamp() { return timestamp; }
        }
        
        @FunctionalInterface
        public interface Supplier<T> {
            T get() throws Exception;
        }
    }
    
    // Asynchronous communication with message queues
    public static class AsynchronousCommunication {
        
        // Event-driven messaging system
        public static class EventDrivenMessaging {
            private final Map<String, List<MessageHandler>> handlers = new ConcurrentHashMap<>();
            private final ExecutorService executor = Executors.newCachedThreadPool();
            
            public void subscribe(String eventType, MessageHandler handler) {
                handlers.computeIfAbsent(eventType, k -> new CopyOnWriteArrayList<>())
                       .add(handler);
            }
            
            public void publish(String eventType, Object eventData) {
                List<MessageHandler> eventHandlers = handlers.get(eventType);
                if (eventHandlers != null && !eventHandlers.isEmpty()) {
                    for (MessageHandler handler : eventHandlers) {
                        executor.submit(() -> {
                            try {
                                handler.handle(eventType, eventData);
                            } catch (Exception e) {
                                System.err.println("Error handling event " + eventType + ": " + e.getMessage());
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
            
            @FunctionalInterface
            public interface MessageHandler {
                void handle(String eventType, Object eventData) throws Exception;
            }
        }
        
        // Message queue implementation
        public static class MessageQueue {
            private final String queueName;
            private final BlockingQueue<Message> queue = new LinkedBlockingQueue<>();
            private final List<MessageConsumer> consumers = new CopyOnWriteArrayList<>();
            private final ExecutorService executor = Executors.newCachedThreadPool();
            private final AtomicInteger messageCounter = new AtomicInteger(0);
            
            public MessageQueue(String queueName) {
                this.queueName = queueName;
            }
            
            public void sendMessage(Message message) {
                messageCounter.incrementAndGet();
                queue.offer(message);
            }
            
            public void addConsumer(MessageConsumer consumer) {
                consumers.add(consumer);
                
                // Start consumer thread
                executor.submit(() -> {
                    while (!Thread.currentThread().isInterrupted()) {
                        try {
                            Message message = queue.take();
                            consumer.consume(message);
                        } catch (InterruptedException e) {
                            Thread.currentThread().interrupt();
                            break;
                        } catch (Exception e) {
                            System.err.println("Error consuming message: " + e.getMessage());
                        }
                    }
                });
            }
            
            public int getQueueSize() {
                return queue.size();
            }
            
            public int getTotalMessagesProcessed() {
                return messageCounter.get();
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
            
            public interface MessageConsumer {
                void consume(Message message) throws Exception;
            }
        }
        
        // Reactive messaging with backpressure
        public static class ReactiveMessaging {
            private final Sinks.Many<Message> messageSink;
            private final Flux<Message> messageStream;
            
            public ReactiveMessaging() {
                this.messageSink = Sinks.many().multicast().onBackpressureBuffer(1000);
                this.messageStream = messageSink.asFlux()
                    .onBackpressureBuffer(1000)
                    .share();
            }
            
            public void publishMessage(Message message) {
                Sinks.EmitResult result = messageSink.tryEmitNext(message);
                if (result == Sinks.EmitResult.FAIL_NON_SERIALIZED) {
                    System.out.println("Message dropped due to backpressure");
                }
            }
            
            public Flux<Message> getMessageStream() {
                return messageStream;
            }
            
            public Flux<Message> getMessageStreamWithFilter(Predicate<Message> filter) {
                return messageStream.filter(filter);
            }
        }
        
        // Dead letter queue implementation
        public static class DeadLetterQueue {
            private final BlockingQueue<DeadLetterMessage> dlq = new LinkedBlockingQueue<>();
            private final String serviceName;
            private final int maxRetries;
            
            public DeadLetterQueue(String serviceName, int maxRetries) {
                this.serviceName = serviceName;
                this.maxRetries = maxRetries;
            }
            
            public void addMessage(Message message, Exception error, int retryCount) {
                if (retryCount >= maxRetries) {
                    DeadLetterMessage dlMessage = new DeadLetterMessage(
                        message, error, retryCount, System.currentTimeMillis()
                    );
                    dlq.offer(dlMessage);
                    System.err.println("Message moved to DLQ: " + message.getId());
                }
            }
            
            public DeadLetterMessage pollMessage() {
                return dlq.poll();
            }
            
            public int size() {
                return dlq.size();
            }
            
            public void processDeadLetters(DeadLetterHandler handler) {
                DeadLetterMessage message;
                while ((message = dlq.poll()) != null) {
                    try {
                        handler.handle(message);
                    } catch (Exception e) {
                        System.err.println("Error processing dead letter: " + e.getMessage());
                        // Re-queue if handler fails
                        dlq.offer(message);
                    }
                }
            }
            
            public interface DeadLetterHandler {
                void handle(DeadLetterMessage message) throws Exception;
            }
        }
        
        // Supporting classes
        public static class Message {
            private final String id;
            private final String type;
            private final Object payload;
            private final long timestamp;
            private final Map<String, String> headers;
            
            public Message(String type, Object payload) {
                this.id = UUID.randomUUID().toString();
                this.type = type;
                this.payload = payload;
                this.timestamp = System.currentTimeMillis();
                this.headers = new HashMap<>();
            }
            
            public void addHeader(String key, String value) {
                headers.put(key, value);
            }
            
            public String getId() { return id; }
            public String getType() { return type; }
            public Object getPayload() { return payload; }
            public long getTimestamp() { return timestamp; }
            public Map<String, String> getHeaders() { return headers; }
        }
        
        public static class DeadLetterMessage {
            private final Message originalMessage;
            private final Exception error;
            private final int retryCount;
            private final long timestamp;
            
            public DeadLetterMessage(Message originalMessage, Exception error, 
                                   int retryCount, long timestamp) {
                this.originalMessage = originalMessage;
                this.error = error;
                this.retryCount = retryCount;
                this.timestamp = timestamp;
            }
            
            public Message getOriginalMessage() { return originalMessage; }
            public Exception getError() { return error; }
            public int getRetryCount() { return retryCount; }
            public long getTimestamp() { return timestamp; }
        }
    }
}
```

## Microservices Design Patterns

### Advanced Microservices Patterns

```java
import java.util.concurrent.atomic.AtomicLong;
import java.util.function.Predicate;
import java.util.stream.Collectors;

public class MicroservicesDesignPatterns {
    
    // API Gateway pattern implementation
    public static class ApiGateway {
        private final Map<String, ServiceRoute> routes = new ConcurrentHashMap<>();
        private final RateLimiter rateLimiter;
        private final CircuitBreaker circuitBreaker;
        private final LoadBalancer loadBalancer;
        private final SecurityFilter securityFilter;
        
        public ApiGateway(RateLimiter rateLimiter, CircuitBreaker circuitBreaker, 
                         LoadBalancer loadBalancer, SecurityFilter securityFilter) {
            this.rateLimiter = rateLimiter;
            this.circuitBreaker = circuitBreaker;
            this.loadBalancer = loadBalancer;
            this.securityFilter = securityFilter;
        }
        
        public void addRoute(String path, ServiceRoute route) {
            routes.put(path, route);
        }
        
        public GatewayResponse handleRequest(GatewayRequest request) {
            try {
                // Security check
                if (!securityFilter.isAuthorized(request)) {
                    return GatewayResponse.unauthorized("Access denied");
                }
                
                // Rate limiting
                if (!rateLimiter.allowRequest(request.getClientId())) {
                    return GatewayResponse.tooManyRequests("Rate limit exceeded");
                }
                
                // Route matching
                ServiceRoute route = findMatchingRoute(request.getPath());
                if (route == null) {
                    return GatewayResponse.notFound("Route not found");
                }
                
                // Circuit breaker check
                if (circuitBreaker.isCircuitOpen(route.getServiceName())) {
                    return GatewayResponse.serviceUnavailable("Service temporarily unavailable");
                }
                
                // Transform and forward request
                ServiceRequest serviceRequest = transformRequest(request, route);
                
                // Forward to service
                ServiceResponse serviceResponse = forwardToService(serviceRequest, route);
                
                // Transform response
                GatewayResponse gatewayResponse = transformResponse(serviceResponse);
                
                return gatewayResponse;
            } catch (Exception e) {
                return GatewayResponse.internalServerError("Internal server error: " + e.getMessage());
            }
        }
        
        private ServiceRoute findMatchingRoute(String path) {
            // Find the best matching route
            return routes.values().stream()
                .filter(route -> path.startsWith(route.getPathPrefix()))
                .findFirst()
                .orElse(null);
        }
        
        private ServiceRequest transformRequest(GatewayRequest request, ServiceRoute route) {
            // Transform gateway request to service request
            String serviceUrl = loadBalancer.selectService(route.getServiceName());
            return new ServiceRequest(
                serviceUrl + route.getPathPrefix(),
                request.getMethod(),
                request.getHeaders(),
                request.getBody()
            );
        }
        
        private ServiceResponse forwardToService(ServiceRequest request, ServiceRoute route) {
            // Forward request to service with circuit breaker
            return circuitBreaker.execute(route.getServiceName(), () -> {
                // Simulate service call
                try {
                    Thread.sleep(100);
                    if (Math.random() < 0.05) { // 5% failure rate
                        throw new RuntimeException("Service error");
                    }
                    return new ServiceResponse(200, "Success", System.currentTimeMillis());
                } catch (Exception e) {
                    throw new RuntimeException("Service call failed", e);
                }
            });
        }
        
        private GatewayResponse transformResponse(ServiceResponse serviceResponse) {
            // Transform service response to gateway response
            return new GatewayResponse(
                serviceResponse.getStatusCode(),
                serviceResponse.getBody(),
                serviceResponse.getTimestamp()
            );
        }
        
        // Supporting classes
        public static class GatewayRequest {
            private final String path;
            private final String method;
            private final Map<String, String> headers;
            private final String body;
            private final String clientId;
            
            public GatewayRequest(String path, String method, Map<String, String> headers, 
                                String body, String clientId) {
                this.path = path;
                this.method = method;
                this.headers = headers != null ? new HashMap<>(headers) : new HashMap<>();
                this.body = body;
                this.clientId = clientId;
            }
            
            public String getPath() { return path; }
            public String getMethod() { return method; }
            public Map<String, String> getHeaders() { return headers; }
            public String getBody() { return body; }
            public String getClientId() { return clientId; }
        }
        
        public static class GatewayResponse {
            private final int statusCode;
            private final String body;
            private final long timestamp;
            
            public GatewayResponse(int statusCode, String body, long timestamp) {
                this.statusCode = statusCode;
                this.body = body;
                this.timestamp = timestamp;
            }
            
            public static GatewayResponse unauthorized(String message) {
                return new GatewayResponse(401, message, System.currentTimeMillis());
            }
            
            public static GatewayResponse tooManyRequests(String message) {
                return new GatewayResponse(429, message, System.currentTimeMillis());
            }
            
            public static GatewayResponse notFound(String message) {
                return new GatewayResponse(404, message, System.currentTimeMillis());
            }
            
            public static GatewayResponse serviceUnavailable(String message) {
                return new GatewayResponse(503, message, System.currentTimeMillis());
            }
            
            public static GatewayResponse internalServerError(String message) {
                return new GatewayResponse(500, message, System.currentTimeMillis());
            }
            
            public int getStatusCode() { return statusCode; }
            public String getBody() { return body; }
            public long getTimestamp() { return timestamp; }
        }
        
        public static class ServiceRoute {
            private final String pathPrefix;
            private final String serviceName;
            private final Map<String, String> transformations;
            
            public ServiceRoute(String pathPrefix, String serviceName) {
                this.pathPrefix = pathPrefix;
                this.serviceName = serviceName;
                this.transformations = new HashMap<>();
            }
            
            public void addTransformation(String from, String to) {
                transformations.put(from, to);
            }
            
            public String getPathPrefix() { return pathPrefix; }
            public String getServiceName() { return serviceName; }
            public Map<String, String> getTransformations() { return transformations; }
        }
        
        public static class ServiceRequest {
            private final String url;
            private final String method;
            private final Map<String, String> headers;
            private final String body;
            
            public ServiceRequest(String url, String method, Map<String, String> headers, String body) {
                this.url = url;
                this.method = method;
                this.headers = headers;
                this.body = body;
            }
            
            public String getUrl() { return url; }
            public String getMethod() { return method; }
            public Map<String, String> getHeaders() { return headers; }
            public String getBody() { return body; }
        }
        
        public static class ServiceResponse {
            private final int statusCode;
            private final String body;
            private final long timestamp;
            
            public ServiceResponse(int statusCode, String body, long timestamp) {
                this.statusCode = statusCode;
                this.body = body;
                this.timestamp = timestamp;
            }
            
            public int getStatusCode() { return statusCode; }
            public String getBody() { return body; }
            public long getTimestamp() { return timestamp; }
        }
    }
    
    // Circuit breaker pattern
    public static class CircuitBreaker {
        private final Map<String, ServiceCircuit> circuits = new ConcurrentHashMap<>();
        
        public <T> T execute(String serviceName, Supplier<T> operation) {
            ServiceCircuit circuit = circuits.computeIfAbsent(serviceName, 
                k -> new ServiceCircuit(serviceName));
            return circuit.execute(operation);
        }
        
        public boolean isCircuitOpen(String serviceName) {
            ServiceCircuit circuit = circuits.get(serviceName);
            return circuit != null && circuit.isCircuitOpen();
        }
        
        public void recordSuccess(String serviceName) {
            ServiceCircuit circuit = circuits.get(serviceName);
            if (circuit != null) {
                circuit.recordSuccess();
            }
        }
        
        public void recordFailure(String serviceName) {
            ServiceCircuit circuit = circuits.get(serviceName);
            if (circuit != null) {
                circuit.recordFailure();
            }
        }
        
        private static class ServiceCircuit {
            private final String serviceName;
            private final int failureThreshold = 5;
            private final long timeoutMillis = 60000; // 1 minute
            private final AtomicInteger failureCount = new AtomicInteger(0);
            private volatile long lastFailureTime = 0;
            private volatile CircuitState state = CircuitState.CLOSED;
            
            public ServiceCircuit(String serviceName) {
                this.serviceName = serviceName;
            }
            
            public <T> T execute(Supplier<T> operation) {
                if (state == CircuitState.OPEN) {
                    if (System.currentTimeMillis() - lastFailureTime > timeoutMillis) {
                        state = CircuitState.HALF_OPEN;
                    } else {
                        throw new CircuitBreakerOpenException("Circuit breaker is OPEN for " + serviceName);
                    }
                }
                
                try {
                    T result = operation.get();
                    recordSuccess();
                    return result;
                } catch (Exception e) {
                    recordFailure();
                    throw e;
                }
            }
            
            public void recordSuccess() {
                failureCount.set(0);
                if (state == CircuitState.HALF_OPEN) {
                    state = CircuitState.CLOSED;
                    System.out.println("Circuit breaker for " + serviceName + " CLOSED");
                }
            }
            
            public void recordFailure() {
                int count = failureCount.incrementAndGet();
                lastFailureTime = System.currentTimeMillis();
                
                if (count >= failureThreshold) {
                    state = CircuitState.OPEN;
                    System.out.println("Circuit breaker for " + serviceName + " OPENED");
                }
            }
            
            public boolean isCircuitOpen() {
                return state == CircuitState.OPEN;
            }
            
            private enum CircuitState {
                CLOSED, OPEN, HALF_OPEN
            }
        }
        
        public static class CircuitBreakerOpenException extends RuntimeException {
            public CircuitBreakerOpenException(String message) {
                super(message);
            }
        }
        
        @FunctionalInterface
        public interface Supplier<T> {
            T get();
        }
    }
    
    // Rate limiting implementation
    public static class RateLimiter {
        private final Map<String, TokenBucket> clientBuckets = new ConcurrentHashMap<>();
        private final int requestsPerSecond;
        private final long windowMillis;
        
        public RateLimiter(int requestsPerSecond, long windowMillis) {
            this.requestsPerSecond = requestsPerSecond;
            this.windowMillis = windowMillis;
        }
        
        public boolean allowRequest(String clientId) {
            TokenBucket bucket = clientBuckets.computeIfAbsent(clientId, 
                k -> new TokenBucket(requestsPerSecond, windowMillis));
            return bucket.tryConsume(1);
        }
        
        public void resetClient(String clientId) {
            clientBuckets.remove(clientId);
        }
        
        public Map<String, Integer> getClientUsage() {
            return clientBuckets.entrySet().stream()
                .collect(Collectors.toMap(
                    Map.Entry::getKey,
                    entry -> entry.getValue().getTokens()
                ));
        }
        
        private static class TokenBucket {
            private final int capacity;
            private final long refillIntervalMillis;
            private final AtomicInteger tokens;
            private volatile long lastRefillTime;
            
            public TokenBucket(int capacity, long refillIntervalMillis) {
                this.capacity = capacity;
                this.refillIntervalMillis = refillIntervalMillis;
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
                
                if (timePassed >= refillIntervalMillis) {
                    int newTokens = (int) (timePassed / refillIntervalMillis) * capacity;
                    if (newTokens > 0) {
                        tokens.accumulateAndGet(Math.min(capacity, tokens.get() + newTokens), 
                            (current, increment) -> Math.min(capacity, current + increment));
                        lastRefillTime = now;
                    }
                }
            }
            
            public int getTokens() {
                refill();
                return tokens.get();
            }
        }
    }
    
    // Security filter implementation
    public static class SecurityFilter {
        private final JwtTokenValidator tokenValidator;
        private final Map<String, Set<String>> rolePermissions = new ConcurrentHashMap<>();
        
        public SecurityFilter(JwtTokenValidator tokenValidator) {
            this.tokenValidator = tokenValidator;
        }
        
        public boolean isAuthorized(ApiGateway.GatewayRequest request) {
            String authHeader = request.getHeaders().get("Authorization");
            if (authHeader == null || !authHeader.startsWith("Bearer ")) {
                return false;
            }
            
            String token = authHeader.substring(7);
            try {
                JwtClaims claims = tokenValidator.validateToken(token);
                
                // Check permissions based on role
                String role = claims.getStringClaim("role");
                String path = request.getPath();
                String method = request.getMethod();
                
                return hasPermission(role, path, method);
            } catch (Exception e) {
                return false;
            }
        }
        
        public void addRolePermission(String role, String resource, String action) {
            rolePermissions.computeIfAbsent(role, k -> ConcurrentHashMap.newKeySet())
                          .add(resource + ":" + action);
        }
        
        private boolean hasPermission(String role, String resource, String action) {
            Set<String> permissions = rolePermissions.get(role);
            if (permissions == null) return false;
            
            // Check direct permission
            if (permissions.contains(resource + ":" + action)) {
                return true;
            }
            
            // Check wildcard permissions
            for (String permission : permissions) {
                if (permission.equals("*:*") || 
                    (permission.startsWith(resource + ":") && permission.endsWith(":*"))) {
                    return true;
                }
            }
            
            return false;
        }
    }
    
    // JWT token validator
    public static class JwtTokenValidator {
        private final String secretKey;
        
        public JwtTokenValidator(String secretKey) {
            this.secretKey = secretKey;
        }
        
        public JwtClaims validateToken(String token) {
            if (token == null || token.split("\\.").length != 3) {
                throw new IllegalArgumentException("Invalid token format");
            }
            
            // In a real implementation, validate signature and parse claims
            // This is a simplified version
            return new JwtClaims();
        }
        
        public static class JwtClaims {
            private final Map<String, Object> claims = new HashMap<>();
            
            public void addClaim(String key, Object value) {
                claims.put(key, value);
            }
            
            public String getStringClaim(String key) {
                Object value = claims.get(key);
                return value instanceof String ? (String) value : null;
            }
            
            public Long getLongClaim(String key) {
                Object value = claims.get(key);
                return value instanceof Long ? (Long) value : null;
            }
            
            public Boolean getBooleanClaim(String key) {
                Object value = claims.get(key);
                return value instanceof Boolean ? (Boolean) value : null;
            }
        }
    }
    
    // Load balancer implementation
    public static class LoadBalancer {
        private final Map<String, List<String>> serviceUrls = new ConcurrentHashMap<>();
        private final Map<String, AtomicInteger> serviceIndices = new ConcurrentHashMap<>();
        
        public void registerService(String serviceName, List<String> urls) {
            serviceUrls.put(serviceName, new ArrayList<>(urls));
            serviceIndices.put(serviceName, new AtomicInteger(0));
        }
        
        public String selectService(String serviceName) {
            List<String> urls = serviceUrls.get(serviceName);
            if (urls == null || urls.isEmpty()) {
                throw new IllegalArgumentException("Service not found: " + serviceName);
            }
            
            AtomicInteger index = serviceIndices.get(serviceName);
            int currentIndex = index.getAndUpdate(i -> (i + 1) % urls.size());
            return urls.get(currentIndex);
        }
        
        public List<String> getServiceUrls(String serviceName) {
            List<String> urls = serviceUrls.get(serviceName);
            return urls != null ? new ArrayList<>(urls) : new ArrayList<>();
        }
    }
}
```

## Microservices Observability

### Advanced Monitoring and Tracing

```java
import java.util.concurrent.atomic.AtomicLong;
import java.util.function.Supplier;
import java.time.Instant;

public class MicroservicesObservability {
    
    // Distributed tracing implementation
    public static class DistributedTracing {
        private final Map<String, TraceContext> activeTraces = new ConcurrentHashMap<>();
        private final TraceExporter traceExporter;
        private final AtomicLong traceIdCounter = new AtomicLong(0);
        
        public DistributedTracing(TraceExporter traceExporter) {
            this.traceExporter = traceExporter;
        }
        
        public TraceContext startTrace(String operationName) {
            String traceId = generateTraceId();
            String spanId = generateSpanId();
            
            TraceContext context = new TraceContext(traceId, spanId, null, operationName);
            activeTraces.put(traceId, context);
            
            return context;
        }
        
        public Span startSpan(TraceContext parentContext, String operationName) {
            String spanId = generateSpanId();
            Span span = new Span(parentContext.getTraceId(), spanId, 
                               parentContext.getSpanId(), operationName);
            span.start();
            return span;
        }
        
        public void endSpan(Span span) {
            span.end();
            traceExporter.exportSpan(span);
        }
        
        public void endTrace(TraceContext context) {
            activeTraces.remove(context.getTraceId());
        }
        
        public TraceContext getTraceContext(String traceId) {
            return activeTraces.get(traceId);
        }
        
        private String generateTraceId() {
            return "trace-" + traceIdCounter.incrementAndGet();
        }
        
        private String generateSpanId() {
            return "span-" + UUID.randomUUID().toString().substring(0, 8);
        }
        
        // Supporting classes
        public static class TraceContext {
            private final String traceId;
            private final String spanId;
            private final String parentSpanId;
            private final String operationName;
            private final long startTime;
            
            public TraceContext(String traceId, String spanId, String parentSpanId, 
                              String operationName) {
                this.traceId = traceId;
                this.spanId = spanId;
                this.parentSpanId = parentSpanId;
                this.operationName = operationName;
                this.startTime = System.currentTimeMillis();
            }
            
            public String getTraceId() { return traceId; }
            public String getSpanId() { return spanId; }
            public String getParentSpanId() { return parentSpanId; }
            public String getOperationName() { return operationName; }
            public long getStartTime() { return startTime; }
        }
        
        public static class Span {
            private final String traceId;
            private final String spanId;
            private final String parentSpanId;
            private final String operationName;
            private final long startTime;
            private long endTime;
            private final Map<String, String> tags = new HashMap<>();
            private final List<LogEntry> logs = new ArrayList<>();
            private boolean ended = false;
            
            public Span(String traceId, String spanId, String parentSpanId, String operationName) {
                this.traceId = traceId;
                this.spanId = spanId;
                this.parentSpanId = parentSpanId;
                this.operationName = operationName;
                this.startTime = System.currentTimeMillis();
            }
            
            public void start() {
                logEvent("start", "Span started");
            }
            
            public void end() {
                if (!ended) {
                    this.endTime = System.currentTimeMillis();
                    this.ended = true;
                    logEvent("end", "Span ended");
                }
            }
            
            public void tag(String key, String value) {
                tags.put(key, value);
            }
            
            public void logEvent(String event, String message) {
                logs.add(new LogEntry(System.currentTimeMillis(), event, message));
            }
            
            public void logError(String error) {
                tag("error", "true");
                logEvent("error", error);
            }
            
            public String getTraceId() { return traceId; }
            public String getSpanId() { return spanId; }
            public String getParentSpanId() { return parentSpanId; }
            public String getOperationName() { return operationName; }
            public long getStartTime() { return startTime; }
            public long getEndTime() { return endTime; }
            public long getDuration() { return endTime - startTime; }
            public Map<String, String> getTags() { return new HashMap<>(tags); }
            public List<LogEntry> getLogs() { return new ArrayList<>(logs); }
            public boolean isEnded() { return ended; }
        }
        
        public static class LogEntry {
            private final long timestamp;
            private final String event;
            private final String message;
            
            public LogEntry(long timestamp, String event, String message) {
                this.timestamp = timestamp;
                this.event = event;
                this.message = message;
            }
            
            public long getTimestamp() { return timestamp; }
            public String getEvent() { return event; }
            public String getMessage() { return message; }
        }
        
        public interface TraceExporter {
            void exportSpan(Span span);
        }
    }
    
    // Metrics collection and monitoring
    public static class MetricsCollection {
        private final Map<String, Metric> metrics = new ConcurrentHashMap<>();
        private final MetricExporter metricExporter;
        private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
        
        public MetricsCollection(MetricExporter metricExporter) {
            this.metricExporter = metricExporter;
            startExporting();
        }
        
        public void incrementCounter(String name, String... tags) {
            Counter counter = (Counter) metrics.computeIfAbsent(name, 
                k -> new Counter(name, tags));
            counter.increment();
        }
        
        public void recordTimer(String name, long duration, String... tags) {
            Timer timer = (Timer) metrics.computeIfAbsent(name, 
                k -> new Timer(name, tags));
            timer.record(duration);
        }
        
        public void updateGauge(String name, double value, String... tags) {
            Gauge gauge = (Gauge) metrics.computeIfAbsent(name, 
                k -> new Gauge(name, tags));
            gauge.set(value);
        }
        
        public void recordHistogram(String name, double value, String... tags) {
            Histogram histogram = (Histogram) metrics.computeIfAbsent(name, 
                k -> new Histogram(name, tags));
            histogram.record(value);
        }
        
        public Map<String, Metric> getMetrics() {
            return new HashMap<>(metrics);
        }
        
        private void startExporting() {
            scheduler.scheduleAtFixedRate(this::exportMetrics, 0, 30, TimeUnit.SECONDS);
        }
        
        private void exportMetrics() {
            metrics.values().forEach(metricExporter::exportMetric);
        }
        
        public void shutdown() {
            scheduler.shutdown();
            try {
                if (!scheduler.awaitTermination(10, TimeUnit.SECONDS)) {
                    scheduler.shutdownNow();
                }
            } catch (InterruptedException e) {
                scheduler.shutdownNow();
                Thread.currentThread().interrupt();
            }
        }
        
        // Abstract metric class
        public abstract static class Metric {
            protected final String name;
            protected final String[] tags;
            protected final long createdTime;
            
            public Metric(String name, String[] tags) {
                this.name = name;
                this.tags = tags != null ? tags : new String[0];
                this.createdTime = System.currentTimeMillis();
            }
            
            public String getName() { return name; }
            public String[] getTags() { return tags; }
            public long getCreatedTime() { return createdTime; }
        }
        
        public static class Counter extends Metric {
            private final AtomicLong value = new AtomicLong(0);
            
            public Counter(String name, String[] tags) {
                super(name, tags);
            }
            
            public void increment() {
                value.incrementAndGet();
            }
            
            public void increment(long delta) {
                value.addAndGet(delta);
            }
            
            public long getValue() {
                return value.get();
            }
        }
        
        public static class Timer extends Metric {
            private final AtomicLong count = new AtomicLong(0);
            private final AtomicLong totalDuration = new AtomicLong(0);
            private final AtomicLong maxDuration = new AtomicLong(0);
            
            public Timer(String name, String[] tags) {
                super(name, tags);
            }
            
            public void record(long duration) {
                count.incrementAndGet();
                totalDuration.addAndGet(duration);
                maxDuration.accumulateAndGet(duration, Math::max);
            }
            
            public long getCount() { return count.get(); }
            public long getTotalDuration() { return totalDuration.get(); }
            public long getMaxDuration() { return maxDuration.get(); }
            public double getAverageDuration() {
                long c = count.get();
                return c > 0 ? (double) totalDuration.get() / c : 0;
            }
        }
        
        public static class Gauge extends Metric {
            private volatile double value = 0;
            
            public Gauge(String name, String[] tags) {
                super(name, tags);
            }
            
            public void set(double value) {
                this.value = value;
            }
            
            public double getValue() {
                return value;
            }
        }
        
        public static class Histogram extends Metric {
            private final AtomicLong count = new AtomicLong(0);
            private final AtomicLong sum = new AtomicLong(0);
            private final AtomicLong min = new AtomicLong(Long.MAX_VALUE);
            private final AtomicLong max = new AtomicLong(Long.MIN_VALUE);
            
            public Histogram(String name, String[] tags) {
                super(name, tags);
            }
            
            public void record(double value) {
                count.incrementAndGet();
                sum.addAndGet((long) value);
                min.accumulateAndGet((long) value, Math::min);
                max.accumulateAndGet((long) value, Math::max);
            }
            
            public long getCount() { return count.get(); }
            public long getSum() { return sum.get(); }
            public long getMin() { return min.get(); }
            public long getMax() { return max.get(); }
            public double getAverage() {
                long c = count.get();
                return c > 0 ? (double) sum.get() / c : 0;
            }
        }
        
        public interface MetricExporter {
            void exportMetric(Metric metric);
        }
    }
    
    // Health check and circuit breaker monitoring
    public static class HealthMonitoring {
        private final Map<String, ServiceHealth> serviceHealth = new ConcurrentHashMap<>();
        private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
        private final HealthNotifier healthNotifier;
        
        public HealthMonitoring(HealthNotifier healthNotifier) {
            this.healthNotifier = healthNotifier;
            startMonitoring();
        }
        
        public void registerService(String serviceName, HealthCheck healthCheck) {
            serviceHealth.put(serviceName, new ServiceHealth(serviceName, healthCheck));
        }
        
        public ServiceStatus getServiceStatus(String serviceName) {
            ServiceHealth health = serviceHealth.get(serviceName);
            return health != null ? health.getStatus() : ServiceStatus.UNKNOWN;
        }
        
        public Map<String, ServiceStatus> getAllServiceStatuses() {
            return serviceHealth.entrySet().stream()
                .collect(Collectors.toMap(
                    Map.Entry::getKey,
                    entry -> entry.getValue().getStatus()
                ));
        }
        
        private void startMonitoring() {
            scheduler.scheduleAtFixedRate(this::checkHealth, 0, 30, TimeUnit.SECONDS);
        }
        
        private void checkHealth() {
            serviceHealth.values().parallelStream().forEach(serviceHealth -> {
                ServiceStatus oldStatus = serviceHealth.getStatus();
                serviceHealth.checkHealth();
                ServiceStatus newStatus = serviceHealth.getStatus();
                
                if (oldStatus != newStatus) {
                    healthNotifier.notifyStatusChange(
                        serviceHealth.getServiceName(), 
                        oldStatus, 
                        newStatus
                    );
                }
            });
        }
        
        public void shutdown() {
            scheduler.shutdown();
            try {
                if (!scheduler.awaitTermination(10, TimeUnit.SECONDS)) {
                    scheduler.shutdownNow();
                }
            } catch (InterruptedException e) {
                scheduler.shutdownNow();
                Thread.currentThread().interrupt();
            }
        }
        
        // Supporting classes
        public static class ServiceHealth {
            private final String serviceName;
            private final HealthCheck healthCheck;
            private volatile ServiceStatus status = ServiceStatus.UNKNOWN;
            private volatile String lastCheckMessage = "";
            private volatile long lastCheckTime = 0;
            
            public ServiceHealth(String serviceName, HealthCheck healthCheck) {
                this.serviceName = serviceName;
                this.healthCheck = healthCheck;
            }
            
            public void checkHealth() {
                try {
                    HealthCheck.Result result = healthCheck.check();
                    this.status = result.isHealthy() ? ServiceStatus.HEALTHY : ServiceStatus.UNHEALTHY;
                    this.lastCheckMessage = result.getMessage();
                    this.lastCheckTime = System.currentTimeMillis();
                } catch (Exception e) {
                    this.status = ServiceStatus.UNHEALTHY;
                    this.lastCheckMessage = "Check failed: " + e.getMessage();
                    this.lastCheckTime = System.currentTimeMillis();
                }
            }
            
            public String getServiceName() { return serviceName; }
            public ServiceStatus getStatus() { return status; }
            public String getLastCheckMessage() { return lastCheckMessage; }
            public long getLastCheckTime() { return lastCheckTime; }
        }
        
        public enum ServiceStatus {
            HEALTHY, UNHEALTHY, UNKNOWN
        }
        
        public interface HealthCheck {
            Result check();
            
            class Result {
                private final boolean healthy;
                private final String message;
                
                public Result(boolean healthy, String message) {
                    this.healthy = healthy;
                    this.message = message;
                }
                
                public static Result healthy(String message) {
                    return new Result(true, message);
                }
                
                public static Result unhealthy(String message) {
                    return new Result(false, message);
                }
                
                public boolean isHealthy() { return healthy; }
                public String getMessage() { return message; }
            }
        }
        
        public interface HealthNotifier {
            void notifyStatusChange(String serviceName, ServiceStatus oldStatus, ServiceStatus newStatus);
        }
    }
    
    // Log aggregation and analysis
    public static class LogAggregation {
        private final LogExporter logExporter;
        private final Map<String, List<LogEntry>> serviceLogs = new ConcurrentHashMap<>();
        private final int maxLogsPerService = 1000;
        
        public LogAggregation(LogExporter logExporter) {
            this.logExporter = logExporter;
        }
        
        public void log(String serviceName, LogLevel level, String message, 
                       Map<String, String> context) {
            LogEntry entry = new LogEntry(
                System.currentTimeMillis(),
                serviceName,
                level,
                message,
                context,
                Thread.currentThread().getName()
            );
            
            // Store log entry
            serviceLogs.computeIfAbsent(serviceName, k -> new ArrayList<>())
                      .add(entry);
            
            // Limit log storage
            List<LogEntry> logs = serviceLogs.get(serviceName);
            if (logs.size() > maxLogsPerService) {
                synchronized (logs) {
                    if (logs.size() > maxLogsPerService) {
                        logs.subList(0, logs.size() - maxLogsPerService).clear();
                    }
                }
            }
            
            // Export log
            logExporter.export(entry);
        }
        
        public List<LogEntry> getLogs(String serviceName, int limit) {
            List<LogEntry> logs = serviceLogs.get(serviceName);
            if (logs == null) return new ArrayList<>();
            
            int startIndex = Math.max(0, logs.size() - limit);
            return new ArrayList<>(logs.subList(startIndex, logs.size()));
        }
        
        public List<LogEntry> searchLogs(String serviceName, String searchTerm, 
                                       LogLevel level, long startTime, long endTime) {
            List<LogEntry> logs = serviceLogs.get(serviceName);
            if (logs == null) return new ArrayList<>();
            
            return logs.stream()
                .filter(entry -> entry.getTimestamp() >= startTime && entry.getTimestamp() <= endTime)
                .filter(entry -> level == null || entry.getLevel() == level)
                .filter(entry -> searchTerm == null || 
                    entry.getMessage().contains(searchTerm) ||
                    entry.getContext().values().stream().anyMatch(v -> v.contains(searchTerm)))
                .collect(Collectors.toList());
        }
        
        // Supporting classes
        public static class LogEntry {
            private final long timestamp;
            private final String serviceName;
            private final LogLevel level;
            private final String message;
            private final Map<String, String> context;
            private final String threadName;
            
            public LogEntry(long timestamp, String serviceName, LogLevel level, 
                          String message, Map<String, String> context, String threadName) {
                this.timestamp = timestamp;
                this.serviceName = serviceName;
                this.level = level;
                this.message = message;
                this.context = context != null ? new HashMap<>(context) : new HashMap<>();
                this.threadName = threadName;
            }
            
            public long getTimestamp() { return timestamp; }
            public String getServiceName() { return serviceName; }
            public LogLevel getLevel() { return level; }
            public String getMessage() { return message; }
            public Map<String, String> getContext() { return new HashMap<>(context); }
            public String getThreadName() { return threadName; }
        }
        
        public enum LogLevel {
            TRACE, DEBUG, INFO, WARN, ERROR, FATAL
        }
        
        public interface LogExporter {
            void export(LogEntry entry);
        }
    }
}
```

## Conclusion

Advanced Microservices Architecture and Design Patterns provide:
- Comprehensive understanding of microservices decomposition strategies
- Implementation of advanced communication patterns with resilience
- Design patterns for building scalable, maintainable microservices
- Observability frameworks for monitoring and tracing
- Security and reliability patterns for production systems

Mastering these concepts is essential for:
- Building enterprise-grade microservices architectures
- Implementing scalable and resilient distributed systems
- Creating maintainable and observable microservices
- Ensuring security and compliance in microservices environments
- Optimizing performance and resource utilization