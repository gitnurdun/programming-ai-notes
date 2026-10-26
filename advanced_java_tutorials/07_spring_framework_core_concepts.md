# Advanced Spring Framework Core Concepts

## Overview
Spring Framework is the foundation of modern Java enterprise development. Understanding its core concepts, including Inversion of Control (IoC), Dependency Injection (DI), and the Application Context, is essential for building robust, maintainable applications.

## Inversion of Control and Dependency Injection

### Core Principles

Inversion of Control (IoC) is a design principle where the control of object creation and lifecycle is inverted from the application code to a framework. Dependency Injection (DI) is a specific form of IoC where dependencies are provided to objects rather than objects creating them.

```java
// Traditional approach (tight coupling)
public class OrderService {
    private final PaymentService paymentService = new PayPalPaymentService(); // Tight coupling
    private final InventoryService inventoryService = new DatabaseInventoryService(); // Tight coupling
    
    public void processOrder(Order order) {
        // Business logic
    }
}

// Spring approach (loose coupling)
@Component
public class OrderService {
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    
    // Constructor injection (preferred)
    public OrderService(PaymentService paymentService, 
                       InventoryService inventoryService) {
        this.paymentService = paymentService;
        this.inventoryService = inventoryService;
    }
    
    public void processOrder(Order order) {
        // Business logic
    }
}
```

### Dependency Injection Types

#### 1. Constructor Injection

```java
@Component
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;
    private final CacheManager cacheManager;
    
    // Required dependencies in constructor
    public UserService(UserRepository userRepository, 
                      EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
        this.cacheManager = null; // Optional dependency handled separately
    }
    
    // Multiple constructors for optional dependencies
    public UserService(UserRepository userRepository, 
                      EmailService emailService,
                      CacheManager cacheManager) {
        this.userRepository = userRepository;
        this.emailService = emailService;
        this.cacheManager = cacheManager;
    }
    
    @PostConstruct
    public void initialize() {
        // Initialization logic after all dependencies are injected
        if (cacheManager != null) {
            cacheManager.initialize();
        }
    }
}
```

#### 2. Setter Injection

```java
@Component
public class ReportService {
    private DatabaseService databaseService;
    private ReportGenerator reportGenerator;
    private FileStorageService fileStorageService;
    
    // Required dependencies
    @Autowired
    public void setDatabaseService(DatabaseService databaseService) {
        this.databaseService = databaseService;
    }
    
    @Autowired
    public void setReportGenerator(ReportGenerator reportGenerator) {
        this.reportGenerator = reportGenerator;
    }
    
    // Optional dependency
    @Autowired(required = false)
    public void setFileStorageService(FileStorageService fileStorageService) {
        this.fileStorageService = fileStorageService;
    }
    
    // Method injection (less common)
    @Autowired
    public void configure(ConfigService configService) {
        // Configure service based on configuration
    }
}
```

#### 3. Field Injection

```java
@Component
public class NotificationService {
    @Autowired
    private EmailService emailService;
    
    @Autowired
    private SmsService smsService;
    
    @Autowired(required = false)
    private PushNotificationService pushService;
    
    // Qualifier for specific implementations
    @Autowired
    @Qualifier("urgentEmailService")
    private EmailService urgentEmailService;
}
```

## Application Context and Bean Lifecycle

### Bean Lifecycle Callbacks

```java
@Component
public class DataProcessor implements InitializingBean, DisposableBean {
    private DataSource dataSource;
    private boolean initialized = false;
    
    // Method 1: InitializingBean interface
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("DataProcessor: afterPropertiesSet called");
        validateConfiguration();
        initializeResources();
        initialized = true;
    }
    
    // Method 2: Custom init method
    @PostConstruct
    public void postConstruct() {
        System.out.println("DataProcessor: @PostConstruct called");
        // Additional initialization logic
    }
    
    // Method 3: XML configuration init-method
    public void customInit() {
        System.out.println("DataProcessor: customInit called");
        // Custom initialization
    }
    
    // Method 1: DisposableBean interface
    @Override
    public void destroy() throws Exception {
        System.out.println("DataProcessor: destroy called");
        cleanupResources();
    }
    
    // Method 2: Custom destroy method
    @PreDestroy
    public void preDestroy() {
        System.out.println("DataProcessor: @PreDestroy called");
        // Cleanup logic
    }
    
    // Method 3: XML configuration destroy-method
    public void customDestroy() {
        System.out.println("DataProcessor: customDestroy called");
        // Custom cleanup
    }
    
    private void validateConfiguration() {
        // Validation logic
    }
    
    private void initializeResources() {
        // Resource initialization
    }
    
    private void cleanupResources() {
        // Resource cleanup
    }
    
    public boolean isInitialized() {
        return initialized;
    }
}
```

### Bean Scopes

```java
// Singleton scope (default)
@Component
@Scope("singleton")
public class SingletonService {
    private int counter = 0;
    
    public void increment() {
        counter++;
    }
    
    public int getCounter() {
        return counter;
    }
}

// Prototype scope
@Component
@Scope("prototype")
public class PrototypeService {
    private String instanceId = UUID.randomUUID().toString();
    
    public String getInstanceId() {
        return instanceId;
    }
}

// Request scope (web-aware)
@Component
@Scope("request")
public class RequestScopedService {
    private LocalDateTime requestTime = LocalDateTime.now();
    
    public LocalDateTime getRequestTime() {
        return requestTime;
    }
}

// Session scope (web-aware)
@Component
@Scope("session")
public class SessionScopedService {
    private Map<String, Object> sessionData = new HashMap<>();
    
    public void setData(String key, Object value) {
        sessionData.put(key, value);
    }
    
    public Object getData(String key) {
        return sessionData.get(key);
    }
}

// Custom scope
@Scope("thread")
public class ThreadScopedService {
    private String threadData;
    
    public void setThreadData(String data) {
        this.threadData = data;
    }
    
    public String getThreadData() {
        return threadData;
    }
}
```

### Custom Bean Post Processors

```java
@Component
public class CustomBeanPostProcessor implements BeanPostProcessor {
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) 
            throws BeansException {
        System.out.println("Before initialization of bean: " + beanName);
        
        // Modify bean before initialization
        if (bean instanceof ConfigurableService) {
            ((ConfigurableService) bean).setConfigurationSource("custom");
        }
        
        return bean;
    }
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) 
            throws BeansException {
        System.out.println("After initialization of bean: " + beanName);
        
        // Create proxy or modify bean after initialization
        if (bean instanceof ServiceInterface) {
            return Proxy.newProxyInstance(
                bean.getClass().getClassLoader(),
                new Class[]{ServiceInterface.class},
                new LoggingInvocationHandler(bean)
            );
        }
        
        return bean;
    }
}

@Component
public class CustomBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) 
            throws BeansException {
        System.out.println("Processing bean factory");
        
        // Modify bean definitions before instantiation
        for (String beanName : beanFactory.getBeanDefinitionNames()) {
            BeanDefinition beanDefinition = beanFactory.getBeanDefinition(beanName);
            // Modify bean definition properties
        }
    }
}
```

## Configuration Approaches

### Java Configuration

```java
@Configuration
@ComponentScan(basePackages = "com.example")
@EnableConfigurationProperties
public class ApplicationConfig {
    
    // Bean definition with dependencies
    @Bean
    @Primary
    public DatabaseService databaseService(@Qualifier("primaryDataSource") DataSource dataSource) {
        return new DatabaseServiceImpl(dataSource);
    }
    
    // Conditional bean creation
    @Bean
    @ConditionalOnProperty(name = "app.cache.enabled", havingValue = "true")
    public CacheManager cacheManager() {
        return new RedisCacheManager();
    }
    
    // Profile-specific beans
    @Bean
    @Profile("development")
    public EmailService developmentEmailService() {
        return new MockEmailService();
    }
    
    @Bean
    @Profile("production")
    public EmailService productionEmailService() {
        return new SmtpEmailService();
    }
    
    // Lazy initialization
    @Bean
    @Lazy
    public ExpensiveResource expensiveResource() {
        return new ExpensiveResource();
    }
    
    // Bean with custom initialization
    @Bean(initMethod = "initialize", destroyMethod = "cleanup")
    public ResourceManager resourceManager() {
        return new ResourceManager();
    }
    
    // Factory method pattern
    @Bean
    public ServiceFactory serviceFactory() {
        return new ServiceFactory();
    }
    
    @Bean
    public PaymentService paymentService(ServiceFactory factory) {
        return factory.createPaymentService();
    }
}
```

### Component Scanning and Stereotype Annotations

```java
// Custom stereotype annotation
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Component
public @interface ServiceComponent {
    String value() default "";
    boolean transactional() default false;
}

// Custom scanning filter
@Configuration
@ComponentScan(
    basePackages = "com.example",
    includeFilters = @ComponentScan.Filter(
        type = FilterType.ANNOTATION, 
        classes = ServiceComponent.class
    ),
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.REGEX, 
        pattern = "com\\.example\\.deprecated\\..*"
    )
)
public class ComponentScanConfig {
}

// Usage of custom stereotype
@ServiceComponent(transactional = true)
public class UserService {
    // Implementation
}

@Repository
@Profile("production")
public class JdbcUserRepository implements UserRepository {
    // Implementation
}
```

## Advanced Dependency Injection Features

### Qualifiers and Custom Qualifiers

```java
// Custom qualifier
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface PaymentMethod {
    String value();
}

// Implementation with custom qualifier
@Component
@PaymentMethod("credit")
public class CreditCardPaymentService implements PaymentService {
    @Override
    public void processPayment(BigDecimal amount) {
        System.out.println("Processing credit card payment: " + amount);
    }
}

@Component
@PaymentMethod("paypal")
public class PayPalPaymentService implements PaymentService {
    @Override
    public void processPayment(BigDecimal amount) {
        System.out.println("Processing PayPal payment: " + amount);
    }
}

// Usage with custom qualifier
@Service
public class OrderService {
    private final PaymentService creditPaymentService;
    private final PaymentService paypalPaymentService;
    
    public OrderService(
            @PaymentMethod("credit") PaymentService creditPaymentService,
            @PaymentMethod("paypal") PaymentService paypalPaymentService) {
        this.creditPaymentService = creditPaymentService;
        this.paypalPaymentService = paypalPaymentService;
    }
}
```

### Collection Injection

```java
@Service
public class NotificationService {
    
    // Inject all beans of type
    private final List<NotificationChannel> channels;
    
    // Inject as map with bean names as keys
    private final Map<String, NotificationChannel> channelMap;
    
    // Inject with specific ordering
    private final List<NotificationProcessor> processors;
    
    public NotificationService(
            List<NotificationChannel> channels,
            Map<String, NotificationChannel> channelMap,
            @Autowired @NotificationPriorityOrdered List<NotificationProcessor> processors) {
        this.channels = channels;
        this.channelMap = channelMap;
        this.processors = processors;
    }
    
    public void sendNotification(String message) {
        channels.forEach(channel -> channel.send(message));
    }
    
    public void sendNotificationToChannel(String channelName, String message) {
        NotificationChannel channel = channelMap.get(channelName);
        if (channel != null) {
            channel.send(message);
        }
    }
}

// Ordered collection injection
@Order(1)
@Component
public class HighPriorityProcessor implements NotificationProcessor {
    @Override
    public void process(String notification) {
        System.out.println("High priority processing");
    }
}

@Order(2)
@Component
public class MediumPriorityProcessor implements NotificationProcessor {
    @Override
    public void process(String notification) {
        System.out.println("Medium priority processing");
    }
}
```

## Environment and Property Management

```java
@Configuration
@ConfigurationProperties(prefix = "app")
@Validated
public class ApplicationProperties {
    @NotBlank
    private String name;
    
    @Min(1)
    @Max(65535)
    private int port = 8080;
    
    private Database database = new Database();
    
    private List<String> allowedOrigins = new ArrayList<>();
    
    private Map<String, String> additionalProperties = new HashMap<>();
    
    // Nested configuration
    public static class Database {
        @NotBlank
        private String url;
        
        @NotBlank
        private String username;
        
        private String password;
        
        // Getters and setters
        public String getUrl() { return url; }
        public void setUrl(String url) { this.url = url; }
        public String getUsername() { return username; }
        public void setUsername(String username) { this.username = username; }
        public String getPassword() { return password; }
        public void setPassword(String password) { this.password = password; }
    }
    
    // Getters and setters
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public int getPort() { return port; }
    public void setPort(int port) { this.port = port; }
    public Database getDatabase() { return database; }
    public void setDatabase(Database database) { this.database = database; }
    public List<String> getAllowedOrigins() { return allowedOrigins; }
    public void setAllowedOrigins(List<String> allowedOrigins) { this.allowedOrigins = allowedOrigins; }
    public Map<String, String> getAdditionalProperties() { return additionalProperties; }
    public void setAdditionalProperties(Map<String, String> additionalProperties) { this.additionalProperties = additionalProperties; }
}

// Usage in service
@Service
public class ApplicationService {
    private final ApplicationProperties properties;
    
    public ApplicationService(ApplicationProperties properties) {
        this.properties = properties;
    }
    
    @PostConstruct
    public void initialize() {
        System.out.println("Application name: " + properties.getName());
        System.out.println("Server port: " + properties.getPort());
        System.out.println("Database URL: " + properties.getDatabase().getUrl());
    }
}
```

## AOP Integration

```java
@Aspect
@Component
public class LoggingAspect {
    
    // Before advice
    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        System.out.println("Before method: " + joinPoint.getSignature().getName());
    }
    
    // After advice
    @After("execution(* com.example.service.*.*(..))")
    public void logAfter(JoinPoint joinPoint) {
        System.out.println("After method: " + joinPoint.getSignature().getName());
    }
    
    // After returning advice
    @AfterReturning(pointcut = "execution(* com.example.service.*.*(..))", 
                   returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        System.out.println("Method returned: " + result);
    }
    
    // Around advice
    @Around("execution(* com.example.service.*.*(..))")
    public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {
        long startTime = System.currentTimeMillis();
        try {
            Object result = joinPoint.proceed();
            long endTime = System.currentTimeMillis();
            System.out.println("Method execution time: " + (endTime - startTime) + "ms");
            return result;
        } catch (Exception e) {
            System.out.println("Method threw exception: " + e.getMessage());
            throw e;
        }
    }
    
    // Pointcut definitions
    @Pointcut("within(com.example.service.*)")
    public void inServiceLayer() {}
    
    @Pointcut("execution(* *.*(..))")
    public void anyMethod() {}
    
    @Pointcut("inServiceLayer() && anyMethod()")
    public void serviceLayerExecution() {}
}
```

## Testing with Spring Context

```java
@SpringBootTest
@TestPropertySource(locations = "classpath:application-test.properties")
@ActiveProfiles("test")
class UserServiceTest {
    
    @Autowired
    private UserService userService;
    
    @MockBean
    private UserRepository userRepository;
    
    @Test
    void testCreateUser() {
        // Given
        User user = new User("John", "john@example.com");
        when(userRepository.save(any(User.class))).thenReturn(user);
        
        // When
        User result = userService.createUser("John", "john@example.com");
        
        // Then
        assertThat(result).isNotNull();
        assertThat(result.getName()).isEqualTo("John");
        verify(userRepository).save(any(User.class));
    }
    
    @TestConfiguration
    static class TestConfig {
        @Bean
        @Primary
        public EmailService mockEmailService() {
            return mock(EmailService.class);
        }
    }
}

// Integration testing with test slices
@DataJpaTest
class UserRepositoryTest {
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testFindByEmail() {
        // Given
        User user = new User("John", "john@example.com");
        entityManager.persistAndFlush(user);
        
        // When
        Optional<User> result = userRepository.findByEmail("john@example.com");
        
        // Then
        assertThat(result).isPresent();
        assertThat(result.get().getName()).isEqualTo("John");
    }
}
```

## Conclusion

Spring Framework Core Concepts provide:
- Powerful IoC container with flexible DI mechanisms
- Comprehensive lifecycle management
- Multiple configuration approaches
- Advanced features like AOP and testing support
- Strong foundation for enterprise applications

Mastering these concepts is essential for:
- Senior Java developer roles
- Enterprise application development
- Framework customization and extension
- Performance optimization
- Troubleshooting complex dependency issues