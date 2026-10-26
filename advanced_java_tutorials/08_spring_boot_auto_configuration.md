# Advanced Spring Boot and Auto-Configuration

## Overview
Spring Boot revolutionizes Spring application development by providing convention-over-configuration principles, auto-configuration, and production-ready features. Understanding its internals and advanced features is crucial for building modern, cloud-native applications.

## Auto-Configuration Deep Dive

### How Auto-Configuration Works

Spring Boot's auto-configuration is based on the principle of "convention over configuration." It automatically configures your Spring application based on the jar dependencies you have added to the classpath.

```java
// Example of how @EnableAutoConfiguration works
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(Application.class);
        // Customization before run
        app.setBannerMode(Banner.Mode.OFF);
        app.run(args);
    }
}

// Behind the scenes, @SpringBootApplication is equivalent to:
// @Configuration
// @EnableAutoConfiguration
// @ComponentScan

// Auto-configuration implementation example
@Configuration
@ConditionalOnClass(DataSource.class)
@ConditionalOnMissingBean(DataSource.class)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
}
```

### Creating Custom Auto-Configuration

```java
// Custom auto-configuration class
@Configuration
@ConditionalOnClass(NotificationService.class)
@EnableConfigurationProperties(NotificationProperties.class)
@AutoConfigureAfter(JacksonAutoConfiguration.class)
public class NotificationAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(name = "notification.enabled", havingValue = "true", 
                          matchIfMissing = true)
    public NotificationService notificationService(
            NotificationProperties properties,
            ObjectProvider<NotificationTemplateEngine> templateEngineProvider) {
        
        NotificationTemplateEngine templateEngine = 
            templateEngineProvider.getIfAvailable(() -> new DefaultTemplateEngine());
            
        return new NotificationServiceImpl(properties, templateEngine);
    }
    
    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(name = "notification.email.enabled", havingValue = "true")
    public EmailNotificationService emailNotificationService(
            NotificationProperties properties) {
        return new EmailNotificationService(properties);
    }
    
    @Configuration
    @ConditionalOnClass(Metrics.class)
    @ConditionalOnProperty(name = "notification.metrics.enabled", havingValue = "true")
    public static class NotificationMetricsConfiguration {
        
        @Bean
        public NotificationMetrics notificationMetrics(MeterRegistry meterRegistry) {
            return new NotificationMetrics(meterRegistry);
        }
    }
}

// Configuration properties
@ConfigurationProperties(prefix = "notification")
public class NotificationProperties {
    
    private boolean enabled = true;
    private Email email = new Email();
    private Sms sms = new Sms();
    private Push push = new Push();
    
    public static class Email {
        private boolean enabled = true;
        private String host;
        private int port = 587;
        private String username;
        private String password;
        // getters and setters
    }
    
    public static class Sms {
        private boolean enabled = false;
        private String provider;
        private String apiKey;
        // getters and setters
    }
    
    public static class Push {
        private boolean enabled = false;
        private String serviceUrl;
        // getters and setters
    }
    
    // getters and setters for all properties
}

// Conditional annotations for fine-grained control
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Conditional(OnNotificationServiceCondition.class)
public @interface ConditionalOnNotificationService {
    String value() default "";
}

class OnNotificationServiceCondition implements Condition {
    
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Environment env = context.getEnvironment();
        String serviceType = env.getProperty("notification.service.type", "default");
        
        Map<String, Object> attributes = metadata.getAnnotationAttributes(
            ConditionalOnNotificationService.class.getName());
        String requiredType = (String) attributes.get("value");
        
        return requiredType.isEmpty() || requiredType.equals(serviceType);
    }
}
```

### Auto-Configuration Metadata

```java
// META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.autoconfiguration.NotificationAutoConfiguration
com.example.autoconfiguration.SecurityAutoConfiguration
com.example.autoconfiguration.MonitoringAutoConfiguration

// META-INF/spring-configuration-metadata.json
{
  "groups": [
    {
      "name": "notification",
      "type": "com.example.properties.NotificationProperties",
      "description": "Notification service configuration."
    }
  ],
  "properties": [
    {
      "name": "notification.enabled",
      "type": "java.lang.Boolean",
      "description": "Whether to enable notification service.",
      "defaultValue": true
    },
    {
      "name": "notification.email.host",
      "type": "java.lang.String",
      "description": "SMTP server host."
    }
  ]
}
```

## Spring Boot Starters

### Creating Custom Starters

```java
// pom.xml for custom starter
/*
<project>
    <groupId>com.example</groupId>
    <artifactId>example-spring-boot-starter</artifactId>
    <version>1.0.0</version>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>example-library</artifactId>
            <version>1.0.0</version>
        </dependency>
    </dependencies>
</project>
*/

// Auto-configuration class in starter
package com.example.starter;

@Configuration
@ConditionalOnClass(ExampleService.class)
@EnableConfigurationProperties(ExampleProperties.class)
public class ExampleAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public ExampleService exampleService(ExampleProperties properties) {
        return new ExampleServiceImpl(properties);
    }
    
    @Bean
    @ConditionalOnMissingBean
    public ExampleClient exampleClient(ExampleProperties properties) {
        return new ExampleClient(properties);
    }
}

// Properties class
@ConfigurationProperties(prefix = "example")
public class ExampleProperties {
    private String apiUrl = "https://api.example.com";
    private int timeout = 5000;
    private boolean enabled = true;
    
    // getters and setters
}

// pom.xml for auto-configuration module
/*
<project>
    <groupId>com.example</groupId>
    <artifactId>example-spring-boot-autoconfigure</artifactId>
    <version>1.0.0</version>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
        </dependency>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>example-library</artifactId>
            <version>1.0.0</version>
            <optional>true</optional>
        </dependency>
    </dependencies>
</project>
*/
```

## Spring Boot Actuator

### Custom Health Indicators

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    private final DataSource dataSource;
    
    public DatabaseHealthIndicator(DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    @Override
    public Health health() {
        try {
            Connection connection = dataSource.getConnection();
            boolean isValid = connection.isValid(1);
            connection.close();
            
            if (isValid) {
                return Health.up()
                    .withDetail("database", "Available")
                    .withDetail("validationQuery", "SELECT 1")
                    .build();
            } else {
                return Health.down()
                    .withDetail("database", "Not responding")
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("database", "Connection failed")
                .withException(e)
                .build();
        }
    }
}

@Component
public class CacheHealthIndicator implements HealthIndicator {
    
    private final CacheManager cacheManager;
    
    public CacheHealthIndicator(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }
    
    @Override
    public Health health() {
        try {
            Collection<String> cacheNames = cacheManager.getCacheNames();
            Map<String, Object> details = new HashMap<>();
            
            for (String cacheName : cacheNames) {
                Cache cache = cacheManager.getCache(cacheName);
                if (cache != null) {
                    // Custom cache statistics if available
                    details.put(cacheName, getCacheDetails(cache));
                }
            }
            
            return Health.up()
                .withDetails(details)
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("cache", "Health check failed")
                .withException(e)
                .build();
        }
    }
    
    private Map<String, Object> getCacheDetails(Cache cache) {
        Map<String, Object> details = new HashMap<>();
        // Add cache-specific details
        details.put("name", cache.getName());
        details.put("type", cache.getClass().getSimpleName());
        return details;
    }
}
```

### Custom Endpoints

```java
@Component
@Endpoint(id = "custom-info")
public class CustomInfoEndpoint {
    
    private final ApplicationProperties properties;
    
    public CustomInfoEndpoint(ApplicationProperties properties) {
        this.properties = properties;
    }
    
    @ReadOperation
    public CustomInfo info() {
        return new CustomInfo(
            properties.getApplicationName(),
            properties.getVersion(),
            LocalDateTime.now()
        );
    }
    
    @WriteOperation
    public void updateInfo(@Selector String key, String value) {
        // Update operation
    }
    
    @DeleteOperation
    public void resetInfo(@Selector String key) {
        // Delete operation
    }
    
    public static class CustomInfo {
        private final String applicationName;
        private final String version;
        private final LocalDateTime timestamp;
        
        public CustomInfo(String applicationName, String version, LocalDateTime timestamp) {
            this.applicationName = applicationName;
            this.version = version;
            this.timestamp = timestamp;
        }
        
        // getters
        public String getApplicationName() { return applicationName; }
        public String getVersion() { return version; }
        public LocalDateTime getTimestamp() { return timestamp; }
    }
}

@Component
@Endpoint(id = "system-metrics")
public class SystemMetricsEndpoint {
    
    @ReadOperation
    public Map<String, Object> metrics() {
        Map<String, Object> metrics = new HashMap<>();
        
        // JVM metrics
        Runtime runtime = Runtime.getRuntime();
        metrics.put("memory.total", runtime.totalMemory());
        metrics.put("memory.free", runtime.freeMemory());
        metrics.put("memory.max", runtime.maxMemory());
        metrics.put("processors", runtime.availableProcessors());
        
        // System metrics
        metrics.put("timestamp", System.currentTimeMillis());
        
        return metrics;
    }
    
    @ReadOperation
    public Object metric(@Selector String name) {
        switch (name) {
            case "memory.total":
                return Runtime.getRuntime().totalMemory();
            case "memory.free":
                return Runtime.getRuntime().freeMemory();
            case "processors":
                return Runtime.getRuntime().availableProcessors();
            default:
                return null;
        }
    }
}
```

## Configuration Properties Validation

```java
@ConfigurationProperties(prefix = "app")
@Validated
public class ApplicationProperties {
    
    @NotBlank(message = "Application name must not be blank")
    @Length(min = 3, max = 50, message = "Application name must be between 3 and 50 characters")
    private String name;
    
    @Min(value = 1024, message = "Port must be greater than 1024")
    @Max(value = 65535, message = "Port must be less than 65536")
    private int port = 8080;
    
    @Valid
    private Database database = new Database();
    
    @NotEmpty(message = "At least one admin email must be configured")
    private List<@Email String> adminEmails = new ArrayList<>();
    
    @Pattern(regexp = "^[A-Za-z0-9_]+$", message = "Environment name must be alphanumeric")
    private String environment;
    
    public static class Database {
        @NotBlank
        private String url;
        
        @NotBlank
        @Length(min = 1, max = 100)
        private String username;
        
        @NotBlank
        @Length(min = 8, message = "Password must be at least 8 characters")
        private String password;
        
        @Min(1)
        @Max(100)
        private int maxConnections = 10;
        
        // getters and setters
    }
    
    // getters and setters
}

// Configuration properties validator
@Component
public class PropertiesValidator implements SmartInitializingSingleton {
    
    private final ApplicationProperties properties;
    private final Validator validator;
    
    public PropertiesValidator(ApplicationProperties properties, Validator validator) {
        this.properties = properties;
        this.validator = validator;
    }
    
    @Override
    public void afterSingletonsInstantiated() {
        Set<ConstraintViolation<ApplicationProperties>> violations = 
            validator.validate(properties);
        
        if (!violations.isEmpty()) {
            StringBuilder sb = new StringBuilder();
            sb.append("Configuration validation failed:\n");
            for (ConstraintViolation<ApplicationProperties> violation : violations) {
                sb.append("  - ")
                  .append(violation.getPropertyPath())
                  .append(": ")
                  .append(violation.getMessage())
                  .append("\n");
            }
            throw new IllegalStateException(sb.toString());
        }
    }
}
```

## Externalized Configuration

### Configuration Processing

```java
@Configuration
public class ConfigurationProcessingConfig {
    
    @Bean
    @ConfigurationPropertiesBinding
    public Converter<String, Duration> durationConverter() {
        return new StringToDurationConverter();
    }
    
    @Bean
    @ConfigurationPropertiesBinding
    public Converter<String, DataSize> dataSizeConverter() {
        return new StringToDataSizeConverter();
    }
}

class StringToDurationConverter implements Converter<String, Duration> {
    
    @Override
    public Duration convert(String source) {
        if (source == null || source.isEmpty()) {
            return null;
        }
        
        try {
            return Duration.parse(source);
        } catch (DateTimeParseException e) {
            // Try custom formats
            return parseCustomFormat(source);
        }
    }
    
    private Duration parseCustomFormat(String source) {
        // Custom parsing logic for formats like "10s", "5m", "2h"
        if (source.endsWith("s")) {
            return Duration.ofSeconds(Long.parseLong(source.substring(0, source.length() - 1)));
        } else if (source.endsWith("m")) {
            return Duration.ofMinutes(Long.parseLong(source.substring(0, source.length() - 1)));
        } else if (source.endsWith("h")) {
            return Duration.ofHours(Long.parseLong(source.substring(0, source.length() - 1)));
        }
        throw new IllegalArgumentException("Invalid duration format: " + source);
    }
}

// Property sources ordering and resolution
@Configuration
public class CustomPropertySourcesConfig {
    
    @PostConstruct
    public void customizePropertySources(ConfigurableEnvironment environment) {
        // Add custom property sources with specific precedence
        MutablePropertySources propertySources = environment.getPropertySources();
        
        // Add high-priority property source
        propertySources.addFirst(new MapPropertySource("high-priority", 
            Map.of("app.priority", "high")));
            
        // Add low-priority property source
        propertySources.addLast(new MapPropertySource("low-priority", 
            Map.of("app.priority", "low")));
    }
}
```

### Configuration Properties Migration

```java
@ConfigurationProperties(prefix = "app")
@Validated
public class ApplicationProperties {
    
    // Deprecated property with migration
    @DeprecatedConfigurationProperty(replacement = "app.new-property")
    public String getOldProperty() {
        return newProperty;
    }
    
    public void setOldProperty(String oldProperty) {
        this.newProperty = oldProperty;
        // Log deprecation warning
        log.warn("Property 'app.old-property' is deprecated, use 'app.new-property' instead");
    }
    
    private String newProperty;
    
    // New property
    public String getNewProperty() {
        return newProperty;
    }
    
    public void setNewProperty(String newProperty) {
        this.newProperty = newProperty;
    }
}
```

## Spring Boot Testing

### Slice Testing

```java
// Data JPA slice test
@DataJpaTest
@TestPropertySource(properties = {
    "spring.jpa.hibernate.ddl-auto=create-drop",
    "spring.datasource.url=jdbc:h2:mem:testdb"
})
class UserRepositoryTest {
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testFindByEmail() {
        // Given
        User user = new User("John Doe", "john@example.com");
        entityManager.persistAndFlush(user);
        
        // When
        Optional<User> found = userRepository.findByEmail("john@example.com");
        
        // Then
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("John Doe");
    }
}

// Web MVC slice test
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    void testGetUser() throws Exception {
        // Given
        User user = new User(1L, "John Doe", "john@example.com");
        when(userService.findById(1L)).thenReturn(Optional.of(user));
        
        // When & Then
        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John Doe"))
            .andExpect(jsonPath("$.email").value("john@example.com"));
    }
}

// JSON slice test
@JsonTest
class UserJsonTest {
    
    @Autowired
    private JacksonTester<User> json;
    
    @Test
    void testSerialize() throws Exception {
        User user = new User(1L, "John Doe", "john@example.com");
        assertThat(json.write(user)).isStrictlyEqualToJson("user.json");
    }
    
    @Test
    void testDeserialize() throws Exception {
        User user = new User(1L, "John Doe", "john@example.com");
        assertThat(json.parse("user.json")).isEqualTo(user);
    }
}
```

### Integration Testing

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class UserIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:13")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testCreateUser() {
        // Given
        UserDto userDto = new UserDto("John Doe", "john@example.com");
        
        // When
        ResponseEntity<UserDto> response = restTemplate.postForEntity(
            "/users", userDto, UserDto.class);
        
        // Then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getName()).isEqualTo("John Doe");
        
        // Verify in database
        List<User> users = userRepository.findAll();
        assertThat(users).hasSize(1);
        assertThat(users.get(0).getName()).isEqualTo("John Doe");
    }
}
```

## Production-Ready Features

### Customizing SpringApplication

```java
@SpringBootApplication
public class CustomApplication {
    
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(CustomApplication.class);
        
        // Custom banner
        app.setBanner((environment, sourceClass, out) -> 
            out.println("==================================\n" +
                       "  Custom Application Starting\n" +
                       "=================================="));
        
        // Custom listeners
        app.addListeners(new ApplicationStartupListener());
        
        // Custom initialization
        app.addInitializers(new CustomApplicationContextInitializer());
        
        app.run(args);
    }
}

class ApplicationStartupListener implements ApplicationListener<ApplicationReadyEvent> {
    
    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        System.out.println("Application is ready!");
        // Custom startup logic
    }
}

class CustomApplicationContextInitializer 
        implements ApplicationContextInitializer<ConfigurableApplicationContext> {
    
    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        // Custom context initialization
        ConfigurableEnvironment environment = applicationContext.getEnvironment();
        environment.setRequiredProperties("app.name", "app.version");
    }
}
```

## Conclusion

Spring Boot Advanced Features provide:
- Powerful auto-configuration mechanisms
- Production-ready actuator endpoints
- Comprehensive testing support
- Flexible externalized configuration
- Custom starter creation capabilities

Mastering these features is essential for:
- Senior Spring developer roles
- Enterprise application architecture
- Cloud-native application development
- DevOps and operations integration
- Performance optimization and monitoring