# Advanced Spring Cloud and Microservices

## Overview
Spring Cloud provides tools for building distributed systems and microservices architectures. Understanding service discovery, configuration management, circuit breakers, and distributed tracing is essential for building scalable, resilient microservices.

## Service Discovery with Eureka

### Eureka Server Configuration

```java
// Eureka Server Application
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}

// Eureka Server Configuration
@Configuration
@EnableWebSecurity
public class EurekaServerConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/eureka/**").hasRole("EUREKA")
                .anyRequest().authenticated()
            )
            .httpBasic();
        
        return http.build();
    }
    
    @Bean
    public InMemoryUserDetailsManager userDetailsService() {
        UserDetails user = User.withUsername("eureka")
            .password("{noop}password")
            .roles("EUREKA")
            .build();
        return new InMemoryUserDetailsManager(user);
    }
}

// Eureka Server Properties (application.yml)
/*
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  server:
    enable-self-preservation: false
    eviction-interval-timer-in-ms: 30000

spring:
  application:
    name: eureka-server
*/
```

### Eureka Client Configuration

```java
// Service Application with Eureka Client
@SpringBootApplication
@EnableEurekaClient
@EnableDiscoveryClient
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}

// Service Configuration
@Configuration
public class ServiceConfig {
    
    @Bean
    @LoadBalanced
    public WebClient.Builder loadBalancedWebClientBuilder() {
        return WebClient.builder();
    }
    
    @Bean
    @LoadBalanced
    public RestTemplate loadBalancedRestTemplate() {
        return new RestTemplate();
    }
}

// Service Properties (application.yml)
/*
server:
  port: 8081

spring:
  application:
    name: user-service
  cloud:
    inetutils:
      preferred-networks:
        - 192.168
        - 10.0

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
    registry-fetch-interval-seconds: 30
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 90
    metadata-map:
      instanceId: ${spring.application.name}:${server.port}
      version: 1.0.0
      description: User Management Service

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
*/
```

### Service Registration and Discovery

```java
@RestController
@RequestMapping("/api/users")
@Slf4j
public class UserController {
    
    private final UserService userService;
    private final DiscoveryClient discoveryClient;
    private final LoadBalancerClient loadBalancerClient;
    
    public UserController(UserService userService, 
                         DiscoveryClient discoveryClient,
                         LoadBalancerClient loadBalancerClient) {
        this.userService = userService;
        this.discoveryClient = discoveryClient;
        this.loadBalancerClient = loadBalancerClient;
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
        
        // Add service metadata to response
        addServiceMetadata(user);
        
        return ResponseEntity.ok(user);
    }
    
    @GetMapping("/discovery-info")
    public ResponseEntity<ServiceDiscoveryInfo> getDiscoveryInfo() {
        ServiceDiscoveryInfo info = new ServiceDiscoveryInfo();
        
        // Get local service instance
        ServiceInstance localInstance = discoveryClient.getLocalServiceInstance();
        info.setLocalInstance(localInstance);
        
        // Get all registered services
        List<String> services = discoveryClient.getServices();
        info.setAvailableServices(services);
        
        // Get instances for each service
        Map<String, List<ServiceInstance>> serviceInstances = new HashMap<>();
        services.forEach(service -> {
            List<ServiceInstance> instances = discoveryClient.getInstances(service);
            serviceInstances.put(service, instances);
        });
        info.setServiceInstances(serviceInstances);
        
        return ResponseEntity.ok(info);
    }
    
    // Load balancing with RestTemplate
    @GetMapping("/orders/{userId}")
    public ResponseEntity<List<Order>> getUserOrders(@PathVariable Long userId) {
        ServiceInstance orderServiceInstance = loadBalancerClient.choose("order-service");
        
        String orderServiceUrl = orderServiceInstance.getUri().toString();
        String ordersUrl = orderServiceUrl + "/api/orders/user/" + userId;
        
        RestTemplate restTemplate = new RestTemplate();
        List<Order> orders = restTemplate.getForObject(ordersUrl, List.class);
        
        return ResponseEntity.ok(orders);
    }
    
    // Load balancing with WebClient
    @GetMapping("/notifications/{userId}")
    public Mono<ResponseEntity<List<Notification>>> getUserNotifications(@PathVariable Long userId) {
        return loadBalancedWebClientBuilder()
            .build()
            .get()
            .uri("lb://notification-service/api/notifications/user/{userId}", userId)
            .retrieve()
            .bodyToFlux(Notification.class)
            .collectList()
            .map(ResponseEntity::ok)
            .onErrorReturn(ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).build());
    }
    
    private void addServiceMetadata(User user) {
        ServiceInstance instance = discoveryClient.getLocalServiceInstance();
        if (instance != null) {
            user.setServiceMetadata(Map.of(
                "serviceId", instance.getServiceId(),
                "host", instance.getHost(),
                "port", instance.getPort(),
                "metadata", instance.getMetadata()
            ));
        }
    }
    
    private WebClient.Builder loadBalancedWebClientBuilder() {
        LoadBalancerExchangeFilterFunction lbFunction = 
            new LoadBalancerExchangeFilterFunction(loadBalancerClient);
        return WebClient.builder().filter(lbFunction);
    }
}
```

## Configuration Management with Spring Cloud Config

### Config Server Setup

```java
// Config Server Application
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}

// Config Server Properties (application.yml)
/*
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo
          username: ${GIT_USERNAME}
          password: ${GIT_PASSWORD}
          clone-on-start: true
          force-pull: true
          search-paths:
            - '{application}'
            - 'common'
        encrypt:
          enabled: true
      encrypt:
        key: ${ENCRYPT_KEY:your-encryption-key}
  security:
    user:
      name: config-user
      password: ${CONFIG_PASSWORD:config-password}

management:
  endpoints:
    web:
      exposure:
        include: health,info
*/

// Config Server Security
@Configuration
@EnableWebSecurity
public class ConfigServerSecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/actuator/**").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic();
        
        return http.build();
    }
}
```

### Config Client Configuration

```java
// Service with Config Client
@SpringBootApplication
@EnableEurekaClient
@RefreshScope
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}

// Service Properties (bootstrap.yml)
/*
spring:
  application:
    name: order-service
  cloud:
    config:
      uri: http://localhost:8888
      username: config-user
      password: config-password
      fail-fast: true
      retry:
        initial-interval: 1000
        multiplier: 1.1
        max-interval: 2000
        max-attempts: 6
  profiles:
    active: dev

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
*/

// Configuration Properties with Refresh
@Component
@RefreshScope
@ConfigurationProperties(prefix = "order.service")
@Data
public class OrderServiceProperties {
    
    private String defaultCurrency = "USD";
    private int maxOrderItems = 100;
    private BigDecimal maxOrderAmount = new BigDecimal("10000.00");
    private List<String> allowedPaymentMethods = Arrays.asList("CREDIT_CARD", "PAYPAL");
    private Map<String, String> serviceEndpoints = new HashMap<>();
    
    // These values will be refreshed when /actuator/refresh is called
}

// Controller using Refreshable Properties
@RestController
@RequestMapping("/api/orders")
@Slf4j
public class OrderController {
    
    private final OrderServiceProperties properties;
    
    public OrderController(OrderServiceProperties properties) {
        this.properties = properties;
    }
    
    @GetMapping("/config")
    public ResponseEntity<OrderServiceProperties> getConfigProperties() {
        return ResponseEntity.ok(properties);
    }
    
    @PostMapping
    public ResponseEntity<Order> createOrder(@Valid @RequestBody CreateOrderRequest request) {
        // Use configuration properties in business logic
        if (request.getItems().size() > properties.getMaxOrderItems()) {
            throw new BusinessValidationException(
                "Order items exceed maximum of " + properties.getMaxOrderItems());
        }
        
        // Process order...
        Order order = new Order();
        return ResponseEntity.ok(order);
    }
}
```

## Circuit Breaker with Resilience4j

### Circuit Breaker Configuration

```java
// Circuit Breaker Configuration
@Configuration
@EnableCircuitBreaker
public class CircuitBreakerConfig {
    
    @Bean
    public CircuitBreakerRegistry circuitBreakerRegistry() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(10))
            .permittedNumberOfCallsInHalfOpenState(3)
            .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.TIME_BASED)
            .slidingWindowSize(10)
            .recordExceptions(HttpServerErrorException.class, IOException.class)
            .recordException(throwable -> 
                throwable instanceof WebClientResponseException && 
                ((WebClientResponseException) throwable).getStatusCode().is5xxServerError())
            .build();
        
        return CircuitBreakerRegistry.of(config);
    }
    
    @Bean
    public TimeLimiterRegistry timeLimiterRegistry() {
        TimeLimiterConfig config = TimeLimiterConfig.custom()
            .timeoutDuration(Duration.ofSeconds(5))
            .cancelRunningFuture(true)
            .build();
        
        return TimeLimiterRegistry.of(config);
    }
    
    @Bean
    public RetryRegistry retryRegistry() {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofMillis(500))
            .retryExceptions(TimeoutException.class, IOException.class)
            .build();
        
        return RetryRegistry.of(config);
    }
}

// Service with Circuit Breaker
@Service
@Slf4j
public class ExternalApiService {
    
    private final WebClient webClient;
    private final CircuitBreaker circuitBreaker;
    private final TimeLimiter timeLimiter;
    private final Retry retry;
    
    public ExternalApiService(WebClient webClient,
                             CircuitBreakerRegistry circuitBreakerRegistry,
                             TimeLimiterRegistry timeLimiterRegistry,
                             RetryRegistry retryRegistry) {
        this.webClient = webClient;
        this.circuitBreaker = circuitBreakerRegistry.circuitBreaker("externalApi");
        this.timeLimiter = timeLimiterRegistry.timeLimiter("externalApi");
        this.retry = retryRegistry.retry("externalApi");
        
        // Register event consumers for monitoring
        circuitBreaker.getEventPublisher()
            .onStateTransition(event -> log.info("Circuit breaker {} transitioned from {} to {}", 
                event.getCircuitBreakerName(), 
                event.getStateTransition().getFromState(),
                event.getStateTransition().getToState()));
    }
    
    @CircuitBreaker(name = "externalApi", fallbackMethod = "getExternalDataFallback")
    @TimeLimiter(name = "externalApi")
    @Retry(name = "externalApi")
    public Mono<ApiResponse> getExternalData(String id) {
        return webClient.get()
            .uri("/api/data/{id}", id)
            .retrieve()
            .bodyToMono(ApiResponse.class)
            .timeout(Duration.ofSeconds(5));
    }
    
    // Fallback method for circuit breaker
    public Mono<ApiResponse> getExternalDataFallback(String id, Throwable throwable) {
        log.warn("External API call failed for id: {}, using fallback", id, throwable);
        
        ApiResponse fallbackResponse = new ApiResponse();
        fallbackResponse.setId(id);
        fallbackResponse.setStatus("FALLBACK");
        fallbackResponse.setMessage("Service temporarily unavailable");
        
        return Mono.just(fallbackResponse);
    }
    
    // Bulkhead pattern for resource isolation
    @Bulkhead(name = "externalApi", type = Bulkhead.Type.THREADPOOL)
    public Mono<ApiResponse> getExternalDataWithBulkhead(String id) {
        return getExternalData(id);
    }
    
    // Rate limiting
    @RateLimiter(name = "externalApi")
    public Mono<ApiResponse> getExternalDataWithRateLimit(String id) {
        return getExternalData(id);
    }
}

// Controller with Circuit Breaker Metrics
@RestController
@RequestMapping("/api/monitoring")
@Slf4j
public class MonitoringController {
    
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final MeterRegistry meterRegistry;
    
    public MonitoringController(CircuitBreakerRegistry circuitBreakerRegistry,
                               MeterRegistry meterRegistry) {
        this.circuitBreakerRegistry = circuitBreakerRegistry;
        this.meterRegistry = meterRegistry;
    }
    
    @GetMapping("/circuit-breakers")
    public ResponseEntity<Map<String, CircuitBreakerState>> getCircuitBreakerStates() {
        Map<String, CircuitBreakerState> states = new HashMap<>();
        
        circuitBreakerRegistry.getAllCircuitBreakers()
            .forEach(circuitBreaker -> {
                CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
                states.put(circuitBreaker.getName(), CircuitBreakerState.builder()
                    .name(circuitBreaker.getName())
                    .state(circuitBreaker.getState())
                    .failureRate(metrics.getFailureRate())
                    .slowCallRate(metrics.getSlowCallRate())
                    .numberOfSuccessfulCalls(metrics.getNumberOfSuccessfulCalls())
                    .numberOfFailedCalls(metrics.getNumberOfFailedCalls())
                    .numberOfSlowCalls(metrics.getNumberOfSlowCalls())
                    .build());
            });
        
        return ResponseEntity.ok(states);
    }
    
    @PostMapping("/circuit-breakers/{name}/reset")
    public ResponseEntity<Void> resetCircuitBreaker(@PathVariable String name) {
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker(name);
        if (circuitBreaker != null) {
            circuitBreaker.reset();
            return ResponseEntity.ok().build();
        }
        return ResponseEntity.notFound().build();
    }
    
    @GetMapping("/metrics")
    public ResponseEntity<Map<String, Object>> getMetrics() {
        Map<String, Object> metrics = new HashMap<>();
        
        // Add custom metrics
        Collection<Meter> meters = meterRegistry.getMeters();
        meters.forEach(meter -> {
            String name = meter.getId().getName();
            List<Tag> tags = meter.getId().getTags();
            
            if (meter instanceof Timer) {
                Timer timer = (Timer) meter;
                metrics.put(name + ".count", timer.count());
                metrics.put(name + ".totalTime", timer.totalTime(TimeUnit.MILLISECONDS));
                metrics.put(name + ".meanTime", timer.mean(TimeUnit.MILLISECONDS));
            } else if (meter instanceof Counter) {
                Counter counter = (Counter) meter;
                metrics.put(name, counter.count());
            }
        });
        
        return ResponseEntity.ok(metrics);
    }
}
```

## API Gateway with Spring Cloud Gateway

### Gateway Configuration

```java
// API Gateway Application
@SpringBootApplication
@EnableEurekaClient
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}

// Gateway Routes Configuration
@Configuration
public class GatewayConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // User Service Route
            .route("user-service", r -> r
                .path("/api/users/**")
                .filters(f -> f
                    .rewritePath("/api/users/(?<segment>.*)", "/${segment}")
                    .addRequestHeader("X-Gateway", "Spring Cloud Gateway")
                    .hystrix(config -> config
                        .setName("user-service")
                        .setFallbackUri("forward:/fallback/user"))
                    .requestRateLimiter(config -> config
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver()))
                )
                .uri("lb://user-service"))
            
            // Order Service Route
            .route("order-service", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .rewritePath("/api/orders/(?<segment>.*)", "/${segment}")
                    .addRequestHeader("X-Gateway", "Spring Cloud Gateway")
                    .hystrix(config -> config
                        .setName("order-service")
                        .setFallbackUri("forward:/fallback/order"))
                )
                .uri("lb://order-service"))
            
            // Static Content Route
            .route("static-content", r -> r
                .path("/static/**")
                .uri("file:///var/www/static/"))
            
            // Websocket Route
            .route("websocket-service", r -> r
                .path("/websocket/**")
                .uri("lb:ws://websocket-service"))
            
            .build();
    }
    
    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(10, 20); // 10 requests per second, burst of 20
    }
    
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            String userId = exchange.getRequest().getHeaders().getFirst("X-User-ID");
            if (userId != null) {
                return Mono.just(userId);
            }
            return Mono.just(exchange.getRequest().getRemoteAddress().getAddress().getHostAddress());
        };
    }
}

// Gateway Properties (application.yml)
/*
server:
  port: 8080

spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=2
            - name: Hystrix
              args:
                name: user-service
                fallbackUri: forward:/fallback/user
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=2
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
    inetutils:
      preferred-networks:
        - 192.168
        - 10.0

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/

management:
  endpoints:
    web:
      exposure:
        include: health,gateway
*/
```

### Gateway Filters and Security

```java
// Custom Gateway Filter
@Component
@Slf4j
public class AuthenticationGatewayFilter implements GlobalFilter, Ordered {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        
        // Skip authentication for public endpoints
        String path = request.getPath().value();
        if (isPublicEndpoint(path)) {
            return chain.filter(exchange);
        }
        
        // Extract and validate JWT token
        String authHeader = request.getHeaders().getFirst("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return onError(exchange, "Missing or invalid Authorization header", HttpStatus.UNAUTHORIZED);
        }
        
        String token = authHeader.substring(7);
        try {
            Claims claims = validateToken(token);
            String userId = claims.getSubject();
            
            // Add user info to request headers
            ServerHttpRequest modifiedRequest = request.mutate()
                .header("X-User-ID", userId)
                .header("X-User-Roles", String.join(",", getRoles(claims)))
                .build();
            
            ServerWebExchange modifiedExchange = exchange.mutate()
                .request(modifiedRequest)
                .build();
            
            return chain.filter(modifiedExchange);
        } catch (JwtException e) {
            return onError(exchange, "Invalid token", HttpStatus.UNAUTHORIZED);
        }
    }
    
    @Override
    public int getOrder() {
        return -1; // High priority
    }
    
    private Mono<Void> onError(ServerWebExchange exchange, String err, HttpStatus httpStatus) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(httpStatus);
        
        log.warn("Authentication failed: {}", err);
        
        return response.setComplete();
    }
    
    private boolean isPublicEndpoint(String path) {
        return path.startsWith("/api/auth/") || 
               path.startsWith("/health") || 
               path.startsWith("/actuator/");
    }
    
    private Claims validateToken(String token) {
        return Jwts.parser()
            .setSigningKey("your-secret-key")
            .parseClaimsJws(token)
            .getBody();
    }
    
    private List<String> getRoles(Claims claims) {
        Object roles = claims.get("roles");
        if (roles instanceof List) {
            return (List<String>) roles;
        }
        return Collections.emptyList();
    }
}

// Gateway Fallback Controller
@RestController
public class FallbackController {
    
    @RequestMapping("/fallback/user")
    public ResponseEntity<Map<String, String>> userFallback() {
        Map<String, String> response = Map.of(
            "message", "User service is temporarily unavailable",
            "status", "SERVICE_UNAVAILABLE",
            "timestamp", LocalDateTime.now().toString()
        );
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).body(response);
    }
    
    @RequestMapping("/fallback/order")
    public ResponseEntity<Map<String, String>> orderFallback() {
        Map<String, String> response = Map.of(
            "message", "Order service is temporarily unavailable",
            "status", "SERVICE_UNAVAILABLE",
            "timestamp", LocalDateTime.now().toString()
        );
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).body(response);
    }
}
```

## Distributed Tracing with Sleuth and Zipkin

### Tracing Configuration

```java
// Tracing Configuration
@Configuration
public class TracingConfig {
    
    @Bean
    public Sampler sampler() {
        // Sample 10% of requests
        return Sampler.create(0.1f);
    }
    
    @Bean
    public SpanAdjuster spanAdjuster() {
        return span -> {
            // Add custom tags to spans
            span.tag("environment", "production");
            span.tag("version", "1.0.0");
            return span;
        };
    }
    
    @Bean
    public TracerCustomizer tracerCustomizer() {
        return tracer -> tracer.currentSpanCustomizer()
            .tag("custom-tag", "custom-value");
    }
}

// Service with Custom Tracing
@Service
@Slf4j
public class OrderProcessingService {
    
    private final Tracer tracer;
    private final UserServiceClient userServiceClient;
    private final PaymentServiceClient paymentServiceClient;
    
    public OrderProcessingService(Tracer tracer,
                                 UserServiceClient userServiceClient,
                                 PaymentServiceClient paymentServiceClient) {
        this.tracer = tracer;
        this.userServiceClient = userServiceClient;
        this.paymentServiceClient = paymentServiceClient;
    }
    
    public Mono<Order> processOrder(CreateOrderRequest request) {
        Span processOrderSpan = tracer.nextSpan().name("process-order").start();
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(processOrderSpan)) {
            processOrderSpan.tag("order.id", request.getOrderId());
            processOrderSpan.tag("user.id", request.getUserId());
            
            return validateUser(request.getUserId())
                .flatMap(user -> processPayment(request.getPaymentInfo()))
                .flatMap(paymentResult -> createOrder(request))
                .doOnSuccess(order -> processOrderSpan.tag("order.status", "SUCCESS"))
                .doOnError(error -> {
                    processOrderSpan.tag("order.status", "FAILED");
                    processOrderSpan.tag("error", error.getMessage());
                });
        } finally {
            processOrderSpan.end();
        }
    }
    
    private Mono<User> validateUser(String userId) {
        Span span = tracer.nextSpan().name("validate-user").start();
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            span.tag("user.id", userId);
            
            return userServiceClient.getUser(userId)
                .doOnSuccess(user -> span.tag("user.status", "VALID"))
                .doOnError(error -> span.tag("user.status", "INVALID"));
        } finally {
            span.end();
        }
    }
    
    private Mono<PaymentResult> processPayment(PaymentInfo paymentInfo) {
        Span span = tracer.nextSpan().name("process-payment").start();
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            span.tag("payment.method", paymentInfo.getMethod());
            span.tag("payment.amount", paymentInfo.getAmount().toString());
            
            return paymentServiceClient.processPayment(paymentInfo)
                .doOnSuccess(result -> span.tag("payment.status", result.getStatus()))
                .doOnError(error -> {
                    span.tag("payment.status", "FAILED");
                    span.tag("error", error.getMessage());
                });
        } finally {
            span.end();
        }
    }
    
    private Mono<Order> createOrder(CreateOrderRequest request) {
        Span span = tracer.nextSpan().name("create-order").start();
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            // Create order logic
            Order order = new Order();
            order.setId(request.getOrderId());
            order.setUserId(request.getUserId());
            order.setItems(request.getItems());
            
            span.tag("order.items", String.valueOf(request.getItems().size()));
            
            return Mono.just(order);
        } finally {
            span.end();
        }
    }
}

// Controller with Tracing
@RestController
@RequestMapping("/api/orders")
@Slf4j
public class OrderController {
    
    private final OrderProcessingService orderProcessingService;
    private final Tracer tracer;
    
    public OrderController(OrderProcessingService orderProcessingService, Tracer tracer) {
        this.orderProcessingService = orderProcessingService;
        this.tracer = tracer;
    }
    
    @PostMapping
    public Mono<ResponseEntity<Order>> createOrder(@Valid @RequestBody Mono<CreateOrderRequest> request) {
        Span controllerSpan = tracer.currentSpan();
        if (controllerSpan != null) {
            controllerSpan.tag("endpoint", "create-order");
        }
        
        return request
            .flatMap(req -> {
                Span businessSpan = tracer.nextSpan().name("business-logic").start();
                try (Tracer.SpanInScope ws = tracer.withSpanInScope(businessSpan)) {
                    businessSpan.tag("order.type", req.getOrderType());
                    return orderProcessingService.processOrder(req);
                } finally {
                    businessSpan.end();
                }
            })
            .map(order -> ResponseEntity.status(HttpStatus.CREATED).body(order))
            .onErrorResume(throwable -> {
                log.error("Order creation failed", throwable);
                return Mono.just(ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build());
            });
    }
    
    @GetMapping("/trace-info")
    public ResponseEntity<Map<String, String>> getTraceInfo() {
        Span currentSpan = tracer.currentSpan();
        Map<String, String> traceInfo = new HashMap<>();
        
        if (currentSpan != null) {
            traceInfo.put("traceId", currentSpan.context().traceId());
            traceInfo.put("spanId", currentSpan.context().spanId());
            traceInfo.put("parentId", currentSpan.context().parentId());
        }
        
        return ResponseEntity.ok(traceInfo);
    }
}
```

## Microservices Testing

### Integration Testing with Testcontainers

```java
@SpringBootTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserServiceIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:13")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");
    
    @Container
    static GenericContainer<?> eurekaServer = new GenericContainer<>("eureka-server:latest")
            .withExposedPorts(8761);
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("eureka.client.service-url.defaultZone", 
                    () -> "http://" + eurekaServer.getHost() + ":" + eurekaServer.getMappedPort(8761) + "/eureka/");
    }
    
    @Autowired
    private TestWebClient webClient;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testUserRegistrationAndDiscovery() {
        // 1. Create user
        CreateUserRequest createRequest = new CreateUserRequest();
        createRequest.setUsername("testuser");
        createRequest.setEmail("test@example.com");
        createRequest.setPassword("Password123");
        
        User createdUser = webClient.post()
            .uri("/api/users")
            .body(Mono.just(createRequest), CreateUserRequest.class)
            .exchange()
            .expectStatus().isCreated()
            .expectBody(User.class)
            .returnResult()
            .getResponseBody();
        
        assertThat(createdUser).isNotNull();
        assertThat(createdUser.getUsername()).isEqualTo("testuser");
        
        // 2. Verify user in database
        Optional<User> dbUser = userRepository.findByUsername("testuser");
        assertThat(dbUser).isPresent();
        
        // 3. Verify service registration
        webClient.get()
            .uri("/api/users/discovery-info")
            .exchange()
            .expectStatus().isOk()
            .expectBody(ServiceDiscoveryInfo.class)
            .value(info -> {
                assertThat(info.getAvailableServices()).contains("user-service");
            });
    }
    
    @Test
    void testCircuitBreakerFunctionality() {
        // Simulate external service failure
        wireMockServer.stubFor(WireMock.get(WireMock.urlEqualTo("/api/external/data"))
            .willReturn(WireMock.serverError()));
        
        // Make multiple calls to trigger circuit breaker
        for (int i = 0; i < 10; i++) {
            webClient.get()
                .uri("/api/users/external-data")
                .exchange()
                .expectStatus().is5xxServerError();
        }
        
        // Verify circuit breaker is open
        webClient.get()
            .uri("/api/monitoring/circuit-breakers")
            .exchange()
            .expectStatus().isOk()
            .expectBody(new ParameterizedTypeReference<Map<String, CircuitBreakerState>>() {})
            .value(states -> {
                CircuitBreakerState state = states.get("externalApi");
                assertThat(state).isNotNull();
                assertThat(state.getState()).isEqualTo(CircuitBreaker.State.OPEN);
            });
    }
}

// Contract Testing with Spring Cloud Contract
public class UserServiceBase {
    
    @Autowired
    private WebTestClient webTestClient;
    
    @BeforeEach
    public void setup() {
        // Setup test data
        User user = new User();
        user.setId(1L);
        user.setUsername("testuser");
        user.setEmail("test@example.com");
        
        // Save to repository or mock service
    }
    
    // Contract test for user retrieval
    public void getUser() {
        webTestClient.get()
            .uri("/api/users/1")
            .accept(MediaType.APPLICATION_JSON)
            .exchange()
            .expectStatus().isOk()
            .expectHeader().contentType(MediaType.APPLICATION_JSON)
            .expectBody()
            .jsonPath("$.id").isEqualTo(1)
            .jsonPath("$.username").isEqualTo("testuser")
            .jsonPath("$.email").isEqualTo("test@example.com");
    }
    
    // Contract test for user creation
    public void createUser() {
        CreateUserRequest request = new CreateUserRequest();
        request.setUsername("newuser");
        request.setEmail("new@example.com");
        request.setPassword("Password123");
        
        webTestClient.post()
            .uri("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .body(Mono.just(request), CreateUserRequest.class)
            .exchange()
            .expectStatus().isCreated()
            .expectHeader().value("Location", Matchers.containsString("/api/users/"));
    }
}
```

## Conclusion

Spring Cloud and Microservices provide:
- Service discovery for dynamic service registration
- Centralized configuration management
- Circuit breakers for fault tolerance
- API gateway for routing and security
- Distributed tracing for observability
- Comprehensive testing strategies

Mastering these concepts is essential for:
- Building scalable microservices architectures
- Implementing resilient distributed systems
- Managing complex service interactions
- Ensuring system observability and monitoring
- Achieving enterprise-grade reliability and performance