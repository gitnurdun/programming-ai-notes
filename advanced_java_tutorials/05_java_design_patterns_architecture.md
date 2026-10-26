# Advanced Java Design Patterns and Architecture

## Overview
Design patterns provide proven solutions to common software design problems. Understanding advanced design patterns and architectural principles is crucial for creating maintainable, scalable, and robust enterprise applications. This tutorial explores both classical GoF patterns and modern architectural patterns used in enterprise Java applications.

## Creational Patterns

### Singleton Pattern (Advanced Implementation)

```java
import java.io.*;
import java.util.concurrent.atomic.AtomicReference;
import java.util.function.Supplier;

// Thread-safe Singleton implementations
public class AdvancedSingleton {
    
    // 1. Eager Initialization (Thread-safe)
    private static final AdvancedSingleton EAGER_INSTANCE = new AdvancedSingleton();
    
    // 2. Double-Checked Locking (Thread-safe)
    private static volatile AdvancedSingleton dclInstance;
    
    // 3. Bill Pugh Solution (Initialization-on-demand holder idiom)
    private static class SingletonHelper {
        private static final AdvancedSingleton INSTANCE = new AdvancedSingleton();
    }
    
    // 4. Enum Singleton (Most robust)
    public enum EnumSingleton {
        INSTANCE;
        
        public void doSomething() {
            System.out.println("Enum singleton operation");
        }
    }
    
    // 5. ThreadLocal Singleton
    private static final ThreadLocal<AdvancedSingleton> threadLocalInstance = 
        ThreadLocal.withInitial(AdvancedSingleton::new);
    
    private AdvancedSingleton() {
        // Prevent reflection attacks
        if (EAGER_INSTANCE != null) {
            throw new RuntimeException("Use getInstance() method to create instance.");
        }
    }
    
    public static AdvancedSingleton getEagerInstance() {
        return EAGER_INSTANCE;
    }
    
    public static AdvancedSingleton getDCLInstance() {
        if (dclInstance == null) {
            synchronized (AdvancedSingleton.class) {
                if (dclInstance == null) {
                    dclInstance = new AdvancedSingleton();
                }
            }
        }
        return dclInstance;
    }
    
    public static AdvancedSingleton getBillPughInstance() {
        return SingletonHelper.INSTANCE;
    }
    
    public static AdvancedSingleton getThreadLocalInstance() {
        return threadLocalInstance.get();
    }
    
    // Prevent deserialization attacks
    protected Object readResolve() {
        return getBillPughInstance();
    }
    
    public void doSomething() {
        System.out.println("Singleton operation performed");
    }
}

// Abstract Factory Pattern
interface DatabaseConnection {
    void connect();
    void disconnect();
}

class MySQLConnection implements DatabaseConnection {
    @Override
    public void connect() {
        System.out.println("Connected to MySQL");
    }
    
    @Override
    public void disconnect() {
        System.out.println("Disconnected from MySQL");
    }
}

class PostgreSQLConnection implements DatabaseConnection {
    @Override
    public void connect() {
        System.out.println("Connected to PostgreSQL");
    }
    
    @Override
    public void disconnect() {
        System.out.println("Disconnected from PostgreSQL");
    }
}

interface DatabaseFactory {
    DatabaseConnection createConnection();
}

class MySQLFactory implements DatabaseFactory {
    @Override
    public DatabaseConnection createConnection() {
        return new MySQLConnection();
    }
}

class PostgreSQLFactory implements DatabaseFactory {
    @Override
    public DatabaseConnection createConnection() {
        return new PostgreSQLConnection();
    }
}

class ConnectionFactory {
    public static DatabaseFactory getFactory(String databaseType) {
        switch (databaseType.toLowerCase()) {
            case "mysql":
                return new MySQLFactory();
            case "postgresql":
                return new PostgreSQLFactory();
            default:
                throw new IllegalArgumentException("Unknown database type: " + databaseType);
        }
    }
}
```

### Builder Pattern (Advanced Implementation)

```java
import java.time.LocalDateTime;
import java.util.*;
import java.util.function.Consumer;

// Advanced Builder with validation and immutability
public class User {
    private final String username;
    private final String email;
    private final int age;
    private final String address;
    private final LocalDateTime createdDate;
    private final List<String> roles;
    private final Map<String, Object> metadata;
    
    private User(Builder builder) {
        this.username = builder.username;
        this.email = builder.email;
        this.age = builder.age;
        this.address = builder.address;
        this.createdDate = builder.createdDate != null ? 
                          builder.createdDate : LocalDateTime.now();
        this.roles = Collections.unmodifiableList(new ArrayList<>(builder.roles));
        this.metadata = Collections.unmodifiableMap(new HashMap<>(builder.metadata));
    }
    
    // Getters
    public String getUsername() { return username; }
    public String getEmail() { return email; }
    public int getAge() { return age; }
    public String getAddress() { return address; }
    public LocalDateTime getCreatedDate() { return createdDate; }
    public List<String> getRoles() { return roles; }
    public Map<String, Object> getMetadata() { return metadata; }
    
    public static class Builder {
        private String username;
        private String email;
        private int age;
        private String address;
        private LocalDateTime createdDate;
        private final List<String> roles = new ArrayList<>();
        private final Map<String, Object> metadata = new HashMap<>();
        
        public Builder setUsername(String username) {
            this.username = username;
            return this;
        }
        
        public Builder setEmail(String email) {
            this.email = email;
            return this;
        }
        
        public Builder setAge(int age) {
            if (age < 0 || age > 150) {
                throw new IllegalArgumentException("Age must be between 0 and 150");
            }
            this.age = age;
            return this;
        }
        
        public Builder setAddress(String address) {
            this.address = address;
            return this;
        }
        
        public Builder setCreatedDate(LocalDateTime createdDate) {
            this.createdDate = createdDate;
            return this;
        }
        
        public Builder addRole(String role) {
            this.roles.add(role);
            return this;
        }
        
        public Builder addRoles(String... roles) {
            this.roles.addAll(Arrays.asList(roles));
            return this;
        }
        
        public Builder addRoles(List<String> roles) {
            this.roles.addAll(roles);
            return this;
        }
        
        public Builder putMetadata(String key, Object value) {
            this.metadata.put(key, value);
            return this;
        }
        
        public Builder putAllMetadata(Map<String, Object> metadata) {
            this.metadata.putAll(metadata);
            return this;
        }
        
        // Fluent API for nested configuration
        public Builder configureMetadata(Consumer<Map<String, Object>> config) {
            config.accept(this.metadata);
            return this;
        }
        
        public User build() {
            validate();
            return new User(this);
        }
        
        private void validate() {
            if (username == null || username.trim().isEmpty()) {
                throw new IllegalStateException("Username is required");
            }
            if (email == null || !email.contains("@")) {
                throw new IllegalStateException("Valid email is required");
            }
            if (age <= 0) {
                throw new IllegalStateException("Age must be positive");
            }
        }
    }
    
    // Static factory method for common user types
    public static User createBasicUser(String username, String email) {
        return new Builder()
            .setUsername(username)
            .setEmail(email)
            .setAge(18)
            .addRole("USER")
            .build();
    }
    
    public static User createAdminUser(String username, String email) {
        return new Builder()
            .setUsername(username)
            .setEmail(email)
            .setAge(18)
            .addRoles("USER", "ADMIN")
            .putMetadata("adminLevel", 10)
            .build();
    }
}
```

## Structural Patterns

### Composite Pattern (Advanced Implementation)

```java
import java.util.*;

// Advanced Composite pattern with traversal and visitor support
interface Component {
    String getName();
    void execute();
    int getSize();
    Component getParent();
    void setParent(Component parent);
    boolean isLeaf();
}

abstract class AbstractComponent implements Component {
    protected String name;
    protected Component parent;
    
    public AbstractComponent(String name) {
        this.name = name;
    }
    
    @Override
    public String getName() {
        return name;
    }
    
    @Override
    public Component getParent() {
        return parent;
    }
    
    @Override
    public void setParent(Component parent) {
        this.parent = parent;
    }
    
    @Override
    public int getSize() {
        return 1;
    }
    
    @Override
    public String toString() {
        return getName();
    }
}

class Leaf extends AbstractComponent {
    private final String value;
    
    public Leaf(String name, String value) {
        super(name);
        this.value = value;
    }
    
    @Override
    public void execute() {
        System.out.println("Executing leaf: " + name + " with value: " + value);
    }
    
    @Override
    public boolean isLeaf() {
        return true;
    }
    
    public String getValue() {
        return value;
    }
}

class Composite extends AbstractComponent {
    private final List<Component> children = new ArrayList<>();
    
    public Composite(String name) {
        super(name);
    }
    
    public void add(Component component) {
        children.add(component);
        component.setParent(this);
    }
    
    public void remove(Component component) {
        children.remove(component);
        component.setParent(null);
    }
    
    public List<Component> getChildren() {
        return new ArrayList<>(children);
    }
    
    @Override
    public void execute() {
        System.out.println("Executing composite: " + name);
        for (Component child : children) {
            child.execute();
        }
    }
    
    @Override
    public int getSize() {
        return children.stream().mapToInt(Component::getSize).sum() + 1;
    }
    
    @Override
    public boolean isLeaf() {
        return false;
    }
    
    // Traversal methods
    public void traverseDFS(Consumer<Component> visitor) {
        visitor.accept(this);
        for (Component child : children) {
            if (child instanceof Composite) {
                ((Composite) child).traverseDFS(visitor);
            } else {
                visitor.accept(child);
            }
        }
    }
    
    public void traverseBFS(Consumer<Component> visitor) {
        Queue<Component> queue = new LinkedList<>();
        queue.offer(this);
        
        while (!queue.isEmpty()) {
            Component current = queue.poll();
            visitor.accept(current);
            
            if (current instanceof Composite) {
                Composite composite = (Composite) current;
                queue.addAll(composite.getChildren());
            }
        }
    }
    
    public Optional<Component> findByName(String name) {
        if (this.name.equals(name)) {
            return Optional.of(this);
        }
        
        for (Component child : children) {
            if (child.getName().equals(name)) {
                return Optional.of(child);
            }
            
            if (child instanceof Composite) {
                Optional<Component> found = ((Composite) child).findByName(name);
                if (found.isPresent()) {
                    return found;
                }
            }
        }
        
        return Optional.empty();
    }
}
```

### Decorator Pattern (Advanced Implementation)

```java
import java.math.BigDecimal;
import java.util.function.Function;

// Advanced decorator pattern with functional programming
interface PricingService {
    BigDecimal calculatePrice(String productId, int quantity);
}

class BasicPricingService implements PricingService {
    @Override
    public BigDecimal calculatePrice(String productId, int quantity) {
        // Simulate base price calculation
        return new BigDecimal("10.00").multiply(BigDecimal.valueOf(quantity));
    }
}

// Functional decorator approach
class PricingDecorator implements PricingService {
    private final PricingService pricingService;
    private final Function<BigDecimal, BigDecimal> priceModifier;
    
    public PricingDecorator(PricingService pricingService, 
                           Function<BigDecimal, BigDecimal> priceModifier) {
        this.pricingService = pricingService;
        this.priceModifier = priceModifier;
    }
    
    @Override
    public BigDecimal calculatePrice(String productId, int quantity) {
        BigDecimal basePrice = pricingService.calculatePrice(productId, quantity);
        return priceModifier.apply(basePrice);
    }
}

// Specific decorators
class TaxDecorator extends PricingDecorator {
    private final BigDecimal taxRate;
    
    public TaxDecorator(PricingService pricingService, BigDecimal taxRate) {
        super(pricingService, price -> price.multiply(BigDecimal.ONE.add(taxRate)));
        this.taxRate = taxRate;
    }
}

class DiscountDecorator extends PricingDecorator {
    private final BigDecimal discountRate;
    
    public DiscountDecorator(PricingService pricingService, BigDecimal discountRate) {
        super(pricingService, price -> price.multiply(BigDecimal.ONE.subtract(discountRate)));
        this.discountRate = discountRate;
    }
}

class LoyaltyDecorator extends PricingDecorator {
    private final BigDecimal loyaltyDiscount;
    
    public LoyaltyDecorator(PricingService pricingService, BigDecimal loyaltyDiscount) {
        super(pricingService, price -> price.subtract(loyaltyDiscount));
        this.loyaltyDiscount = loyaltyDiscount;
    }
}

// Builder for creating complex decorator chains
class PricingServiceBuilder {
    private PricingService baseService = new BasicPricingService();
    
    public PricingServiceBuilder withTax(BigDecimal taxRate) {
        baseService = new TaxDecorator(baseService, taxRate);
        return this;
    }
    
    public PricingServiceBuilder withDiscount(BigDecimal discountRate) {
        baseService = new DiscountDecorator(baseService, discountRate);
        return this;
    }
    
    public PricingServiceBuilder withLoyaltyDiscount(BigDecimal loyaltyDiscount) {
        baseService = new LoyaltyDecorator(baseService, loyaltyDiscount);
        return this;
    }
    
    public PricingService build() {
        return baseService;
    }
}
```

## Behavioral Patterns

### Observer Pattern (Advanced Implementation)

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

// Advanced observer pattern with async support and filtering
interface EventListener<T> {
    void onEvent(T event);
    default boolean canHandle(Object event) {
        return true;
    }
}

class EventPublisher<T> {
    private final Map<Class<?>, List<EventListener<?>>> listeners = 
        new ConcurrentHashMap<>();
    private final ExecutorService executorService = Executors.newCachedThreadPool();
    private final boolean async;
    
    public EventPublisher(boolean async) {
        this.async = async;
    }
    
    @SuppressWarnings("unchecked")
    public <E> void subscribe(Class<E> eventType, EventListener<E> listener) {
        listeners.computeIfAbsent(eventType, k -> new CopyOnWriteArrayList<>())
                 .add((EventListener<?>) listener);
    }
    
    public <E> void unsubscribe(Class<E> eventType, EventListener<E> listener) {
        List<EventListener<?>> eventListeners = listeners.get(eventType);
        if (eventListeners != null) {
            eventListeners.remove(listener);
        }
    }
    
    @SuppressWarnings("unchecked")
    public void publish(Object event) {
        List<EventListener<?>> eventListeners = listeners.get(event.getClass());
        if (eventListeners == null || eventListeners.isEmpty()) {
            return;
        }
        
        for (EventListener<?> listener : eventListeners) {
            if (listener.canHandle(event)) {
                if (async) {
                    executorService.submit(() -> {
                        try {
                            ((EventListener<Object>) listener).onEvent(event);
                        } catch (Exception e) {
                            System.err.println("Error in event listener: " + e.getMessage());
                        }
                    });
                } else {
                    try {
                        ((EventListener<Object>) listener).onEvent(event);
                    } catch (Exception e) {
                        System.err.println("Error in event listener: " + e.getMessage());
                    }
                }
            }
        }
    }
    
    public void shutdown() {
        executorService.shutdown();
    }
}

// Example usage with custom events
class OrderEvent {
    private final String orderId;
    private final String status;
    
    public OrderEvent(String orderId, String status) {
        this.orderId = orderId;
        this.status = status;
    }
    
    public String getOrderId() { return orderId; }
    public String getStatus() { return status; }
}

class InventoryListener implements EventListener<OrderEvent> {
    @Override
    public void onEvent(OrderEvent event) {
        if ("CREATED".equals(event.getStatus())) {
            System.out.println("Updating inventory for order: " + event.getOrderId());
        }
    }
}

class NotificationListener implements EventListener<OrderEvent> {
    @Override
    public void onEvent(OrderEvent event) {
        System.out.println("Sending notification for order: " + event.getOrderId() + 
                          ", status: " + event.getStatus());
    }
}
```

### Strategy Pattern (Advanced Implementation)

```java
import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;

// Advanced Strategy pattern with context and strategy selection
interface PaymentStrategy {
    PaymentResult processPayment(PaymentRequest request);
    boolean isApplicable(PaymentRequest request);
    String getStrategyName();
}

class PaymentRequest {
    private final String orderId;
    private final BigDecimal amount;
    private final String paymentMethod;
    private final Customer customer;
    
    public PaymentRequest(String orderId, BigDecimal amount, String paymentMethod, Customer customer) {
        this.orderId = orderId;
        this.amount = amount;
        this.paymentMethod = paymentMethod;
        this.customer = customer;
    }
    
    // Getters
    public String getOrderId() { return orderId; }
    public BigDecimal getAmount() { return amount; }
    public String getPaymentMethod() { return paymentMethod; }
    public Customer getCustomer() { return customer; }
}

class PaymentResult {
    private final boolean success;
    private final String transactionId;
    private final String message;
    
    public PaymentResult(boolean success, String transactionId, String message) {
        this.success = success;
        this.transactionId = transactionId;
        this.message = message;
    }
    
    // Getters
    public boolean isSuccess() { return success; }
    public String getTransactionId() { return transactionId; }
    public String getMessage() { return message; }
}

class Customer {
    private final String customerId;
    private final String creditScore;
    
    public Customer(String customerId, String creditScore) {
        this.customerId = customerId;
        this.creditScore = creditScore;
    }
    
    public String getCustomerId() { return customerId; }
    public String getCreditScore() { return creditScore; }
}

// Concrete strategies
class CreditCardStrategy implements PaymentStrategy {
    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        // Simulate credit card processing
        return new PaymentResult(true, "CC_" + UUID.randomUUID().toString(), 
                                "Credit card payment processed");
    }
    
    @Override
    public boolean isApplicable(PaymentRequest request) {
        return "CREDIT_CARD".equalsIgnoreCase(request.getPaymentMethod());
    }
    
    @Override
    public String getStrategyName() {
        return "CREDIT_CARD";
    }
}

class DebitCardStrategy implements PaymentStrategy {
    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        return new PaymentResult(true, "DC_" + UUID.randomUUID().toString(), 
                                "Debit card payment processed");
    }
    
    @Override
    public boolean isApplicable(PaymentRequest request) {
        return "DEBIT_CARD".equalsIgnoreCase(request.getPaymentMethod()) &&
               !request.getCustomer().getCreditScore().equals("LOW");
    }
    
    @Override
    public String getStrategyName() {
        return "DEBIT_CARD";
    }
}

class PayPalStrategy implements PaymentStrategy {
    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        return new PaymentResult(true, "PP_" + UUID.randomUUID().toString(), 
                                "PayPal payment processed");
    }
    
    @Override
    public boolean isApplicable(PaymentRequest request) {
        return "PAYPAL".equalsIgnoreCase(request.getPaymentMethod());
    }
    
    @Override
    public String getStrategyName() {
        return "PAYPAL";
    }
}

// Strategy context with dynamic selection
class PaymentProcessor {
    private final Map<String, PaymentStrategy> strategies;
    private final List<PaymentStrategy> strategyList;
    
    public PaymentProcessor() {
        List<PaymentStrategy> tempStrategies = Arrays.asList(
            new CreditCardStrategy(),
            new DebitCardStrategy(),
            new PayPalStrategy()
        );
        
        this.strategyList = Collections.unmodifiableList(tempStrategies);
        this.strategies = tempStrategies.stream()
            .collect(Collectors.toMap(
                PaymentStrategy::getStrategyName,
                strategy -> strategy,
                (existing, replacement) -> replacement
            ));
    }
    
    public PaymentResult processPayment(PaymentRequest request) {
        // Find appropriate strategy
        Optional<PaymentStrategy> strategy = strategyList.stream()
            .filter(s -> s.isApplicable(request))
            .findFirst();
        
        if (!strategy.isPresent()) {
            throw new IllegalArgumentException("No applicable payment strategy found");
        }
        
        return strategy.get().processPayment(request);
    }
    
    public List<String> getApplicableStrategies(PaymentRequest request) {
        return strategyList.stream()
            .filter(s -> s.isApplicable(request))
            .map(PaymentStrategy::getStrategyName)
            .collect(Collectors.toList());
    }
    
    public PaymentStrategy getStrategy(String name) {
        return strategies.get(name);
    }
}
```

## Architectural Patterns

### Repository Pattern

```java
import java.util.*;
import java.util.function.Predicate;
import java.util.stream.Collectors;

// Generic repository interface
interface Repository<T, ID> {
    Optional<T> findById(ID id);
    List<T> findAll();
    List<T> findBy(Predicate<T> predicate);
    T save(T entity);
    void delete(ID id);
    boolean exists(ID id);
    long count();
}

// JPA-like implementation
class InMemoryRepository<T, ID> implements Repository<T, ID> {
    private final Map<ID, T> storage = new HashMap<>();
    private final Function<T, ID> idExtractor;
    
    public InMemoryRepository(Function<T, ID> idExtractor) {
        this.idExtractor = idExtractor;
    }
    
    @Override
    public Optional<T> findById(ID id) {
        return Optional.ofNullable(storage.get(id));
    }
    
    @Override
    public List<T> findAll() {
        return new ArrayList<>(storage.values());
    }
    
    @Override
    public List<T> findBy(Predicate<T> predicate) {
        return storage.values().stream()
            .filter(predicate)
            .collect(Collectors.toList());
    }
    
    @Override
    public T save(T entity) {
        ID id = idExtractor.apply(entity);
        storage.put(id, entity);
        return entity;
    }
    
    @Override
    public void delete(ID id) {
        storage.remove(id);
    }
    
    @Override
    public boolean exists(ID id) {
        return storage.containsKey(id);
    }
    
    @Override
    public long count() {
        return storage.size();
    }
}

// Entity example
class UserEntity {
    private final Long id;
    private String name;
    private String email;
    private int age;
    
    public UserEntity(Long id, String name, String email, int age) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.age = age;
    }
    
    // Getters and setters
    public Long getId() { return id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
}

// Repository implementation
class UserRepository extends InMemoryRepository<UserEntity, Long> {
    public UserRepository() {
        super(UserEntity::getId);
    }
    
    public Optional<UserEntity> findByEmail(String email) {
        return findBy(user -> user.getEmail().equals(email)).stream().findFirst();
    }
    
    public List<UserEntity> findByAgeRange(int minAge, int maxAge) {
        return findBy(user -> user.getAge() >= minAge && user.getAge() <= maxAge);
    }
    
    public List<UserEntity> findByNameContaining(String name) {
        return findBy(user -> user.getName().toLowerCase()
                                .contains(name.toLowerCase()));
    }
}
```

### Service Layer Pattern

```java
import java.util.*;
import java.util.stream.Collectors;

// Service interface with business logic
interface UserService {
    UserEntity createUser(String name, String email, int age);
    Optional<UserEntity> getUserById(Long id);
    List<UserEntity> getAllUsers();
    UserEntity updateUser(Long id, String name, String email, int age);
    void deleteUser(Long id);
    List<UserEntity> findUsersByAgeRange(int minAge, int maxAge);
    Optional<UserEntity> getUserByEmail(String email);
}

// Service implementation with validation and business rules
class UserServiceImpl implements UserService {
    private final UserRepository userRepository;
    
    public UserServiceImpl(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    @Override
    public UserEntity createUser(String name, String email, int age) {
        // Validation
        validateUser(name, email, age);
        
        // Business rule: check if email already exists
        if (userRepository.findByEmail(email).isPresent()) {
            throw new IllegalArgumentException("User with email already exists: " + email);
        }
        
        // Generate ID and create entity
        Long id = generateId();
        UserEntity user = new UserEntity(id, name, email, age);
        return userRepository.save(user);
    }
    
    @Override
    public Optional<UserEntity> getUserById(Long id) {
        return userRepository.findById(id);
    }
    
    @Override
    public List<UserEntity> getAllUsers() {
        return userRepository.findAll();
    }
    
    @Override
    public UserEntity updateUser(Long id, String name, String email, int age) {
        Optional<UserEntity> existingUser = userRepository.findById(id);
        if (!existingUser.isPresent()) {
            throw new IllegalArgumentException("User not found with ID: " + id);
        }
        
        validateUser(name, email, age);
        
        UserEntity user = existingUser.get();
        user.setName(name);
        user.setEmail(email);
        user.setAge(age);
        
        return userRepository.save(user);
    }
    
    @Override
    public void deleteUser(Long id) {
        if (!userRepository.exists(id)) {
            throw new IllegalArgumentException("User not found with ID: " + id);
        }
        userRepository.delete(id);
    }
    
    @Override
    public List<UserEntity> findUsersByAgeRange(int minAge, int maxAge) {
        if (minAge > maxAge) {
            throw new IllegalArgumentException("Min age cannot be greater than max age");
        }
        return userRepository.findByAgeRange(minAge, maxAge);
    }
    
    @Override
    public Optional<UserEntity> getUserByEmail(String email) {
        validateEmail(email);
        return userRepository.findByEmail(email);
    }
    
    private void validateUser(String name, String email, int age) {
        if (name == null || name.trim().isEmpty()) {
            throw new IllegalArgumentException("Name cannot be null or empty");
        }
        if (age < 0 || age > 150) {
            throw new IllegalArgumentException("Age must be between 0 and 150");
        }
        validateEmail(email);
    }
    
    private void validateEmail(String email) {
        if (email == null || !email.contains("@")) {
            throw new IllegalArgumentException("Valid email is required");
        }
    }
    
    private Long generateId() {
        // In a real implementation, this would use a proper ID generation strategy
        return System.currentTimeMillis();
    }
}
```

## Microservices Architecture Patterns

### Circuit Breaker Pattern

```java
import java.time.LocalDateTime;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicReference;

enum CircuitState {
    CLOSED, OPEN, HALF_OPEN
}

class CircuitBreaker {
    private final int failureThreshold;
    private final long timeoutInMillis;
    
    private final AtomicInteger failureCount = new AtomicInteger(0);
    private final AtomicReference<CircuitState> state = new AtomicReference<>(CircuitState.CLOSED);
    private volatile LocalDateTime lastFailureTime;
    
    public CircuitBreaker(int failureThreshold, long timeoutInMillis) {
        this.failureThreshold = failureThreshold;
        this.timeoutInMillis = timeoutInMillis;
    }
    
    public <T> T execute(SupplierWithException<T> supplier) throws Exception {
        if (state.get() == CircuitState.OPEN) {
            // Check if timeout has passed
            if (isTimeoutExpired()) {
                state.compareAndSet(CircuitState.OPEN, CircuitState.HALF_OPEN);
            } else {
                throw new CircuitBreakerOpenException("Circuit breaker is OPEN");
            }
        }
        
        try {
            T result = supplier.get();
            onSuccess();
            return result;
        } catch (Exception e) {
            onFailure();
            throw e;
        }
    }
    
    private void onSuccess() {
        failureCount.set(0);
        state.set(CircuitState.CLOSED);
    }
    
    private void onFailure() {
        int count = failureCount.incrementAndGet();
        if (count >= failureThreshold) {
            state.set(CircuitState.OPEN);
            lastFailureTime = LocalDateTime.now();
        }
    }
    
    private boolean isTimeoutExpired() {
        if (lastFailureTime == null) {
            return true;
        }
        return java.time.Duration.between(lastFailureTime, LocalDateTime.now())
                                .toMillis() > timeoutInMillis;
    }
    
    public CircuitState getState() {
        return state.get();
    }
    
    @FunctionalInterface
    public interface SupplierWithException<T> {
        T get() throws Exception;
    }
}

class CircuitBreakerOpenException extends RuntimeException {
    public CircuitBreakerOpenException(String message) {
        super(message);
    }
}

// Usage example
class ExternalServiceClient {
    private final CircuitBreaker circuitBreaker;
    
    public ExternalServiceClient() {
        this.circuitBreaker = new CircuitBreaker(3, 5000); // 3 failures, 5 second timeout
    }
    
    public String callExternalService() throws Exception {
        return circuitBreaker.execute(() -> {
            // Simulate external service call
            if (Math.random() > 0.7) { // 30% chance of failure
                throw new RuntimeException("External service failed");
            }
            return "Success: " + System.currentTimeMillis();
        });
    }
}
```

## Conclusion

Advanced design patterns and architecture provide:
- Proven solutions to common design problems
- Enhanced code maintainability and extensibility
- Better separation of concerns
- Scalable and robust application architecture
- Foundation for enterprise-level applications

Understanding these patterns is essential for:
- Senior Java developer roles
- System architecture design
- Framework development
- Performance optimization
- Code quality improvement