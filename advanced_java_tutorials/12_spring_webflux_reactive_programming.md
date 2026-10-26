# Advanced Spring WebFlux and Reactive Programming

## Overview
Spring WebFlux is Spring's reactive web framework built on Project Reactor, providing non-blocking, event-driven processing for building scalable, high-performance applications. Understanding reactive programming principles and WebFlux capabilities is essential for modern Java development.

## Reactive Programming Fundamentals

### Publisher, Subscriber, and Backpressure

```java
import reactor.core.publisher.*;
import reactor.core.scheduler.Schedulers;
import java.util.concurrent.TimeUnit;

public class ReactiveFundamentals {
    
    // Mono (0 or 1 item) examples
    public void monoExamples() {
        // Creating Monos
        Mono<String> justMono = Mono.just("Hello");
        Mono<String> emptyMono = Mono.empty();
        Mono<String> errorMono = Mono.error(new RuntimeException("Error"));
        Mono<String> deferMono = Mono.defer(() -> Mono.just("Deferred"));
        
        // Transforming Monos
        Mono<String> transformed = justMono
            .map(String::toUpperCase)
            .filter(s -> s.length() > 3)
            .switchIfEmpty(Mono.just("DEFAULT"));
        
        // Error handling
        Mono<String> withErrorHandling = errorMono
            .onErrorReturn("Error handled")
            .onErrorResume(throwable -> Mono.just("Resume"));
        
        // Subscribing
        transformed.subscribe(
            value -> System.out.println("Value: " + value),
            error -> System.out.println("Error: " + error),
            () -> System.out.println("Completed")
        );
    }
    
    // Flux (0 to N items) examples
    public void fluxExamples() {
        // Creating Fluxes
        Flux<String> justFlux = Flux.just("A", "B", "C");
        Flux<Integer> rangeFlux = Flux.range(1, 10);
        Flux<String> fromIterable = Flux.fromIterable(Arrays.asList("X", "Y", "Z"));
        Flux<Long> intervalFlux = Flux.interval(Duration.ofSeconds(1));
        
        // Transforming Fluxes
        Flux<String> transformed = justFlux
            .map(String::toLowerCase)
            .filter(s -> s.length() > 0)
            .take(3);
        
        // Combining Fluxes
        Flux<String> combined = Flux.merge(
            Flux.just("A", "B"),
            Flux.just("C", "D")
        );
        
        // Backpressure handling
        Flux<Integer> backpressured = Flux.range(1, 1000)
            .onBackpressureBuffer(100) // Buffer up to 100 items
            .observeOn(Schedulers.boundedElastic()); // Run on bounded elastic scheduler
        
        // Subscribing with different backpressure strategies
        backpressured
            .subscribe(
                System.out::println, // onNext
                System.err::println, // onError
                () -> System.out.println("Completed"), // onComplete
                subscription -> subscription.request(10) // request 10 items
            );
    }
    
    // Backpressure demonstration
    public void demonstrateBackpressure() {
        Flux.range(1, 1000)
            .doOnNext(i -> System.out.println("Emitting: " + i))
            .onBackpressureBuffer(50, (overflow, dropped) -> {
                System.out.println("Dropped: " + dropped);
            })
            .observeOn(Schedulers.boundedElastic())
            .subscribe(
                value -> {
                    try {
                        Thread.sleep(100); // Slow consumer
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                    System.out.println("Processing: " + value);
                },
                error -> System.err.println("Error: " + error)
            );
    }
}
```

### Reactor Operators and Patterns

```java
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import reactor.util.retry.Retry;
import java.time.Duration;
import java.util.concurrent.atomic.AtomicInteger;

public class ReactorOperators {
    
    public void transformationOperators() {
        Flux.range(1, 10)
            .map(i -> i * 2) // Transform each element
            .filter(i -> i % 4 == 0) // Filter elements
            .take(3) // Take first 3 elements
            .log() // Add logging
            .subscribe(System.out::println);
    }
    
    public void combinationOperators() {
        Flux<String> flux1 = Flux.just("A", "B", "C");
        Flux<String> flux2 = Flux.just("1", "2", "3");
        
        // Combine with zip
        Flux.zip(flux1, flux2)
            .map(tuple -> tuple.getT1() + tuple.getT2())
            .subscribe(System.out::println);
        
        // Combine with merge
        Flux.merge(flux1, flux2)
            .subscribe(System.out::println);
        
        // Combine with concat
        Flux.concat(flux1, flux2)
            .subscribe(System.out::println);
    }
    
    public void errorHandling() {
        Flux.range(1, 10)
            .map(i -> {
                if (i == 5) {
                    throw new RuntimeException("Error at " + i);
                }
                return i * 2;
            })
            .onErrorResume(throwable -> {
                System.out.println("Resuming after error: " + throwable.getMessage());
                return Flux.just(-1); // Return fallback value
            })
            .subscribe(
                System.out::println,
                error -> System.out.println("Final error: " + error)
            );
    }
    
    public void retryPattern() {
        AtomicInteger attemptCount = new AtomicInteger(0);
        
        Mono<String> retryable = Mono.fromCallable(() -> {
            int attempt = attemptCount.incrementAndGet();
            if (attempt < 3) {
                throw new RuntimeException("Attempt " + attempt + " failed");
            }
            return "Success on attempt " + attempt;
        })
        .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
                   .filter(throwable -> throwable instanceof RuntimeException));
        
        retryable.subscribe(
            result -> System.out.println("Result: " + result),
            error -> System.out.println("Final error: " + error)
        );
    }
    
    public void conditionalProcessing() {
        Mono<String> conditional = Mono.just("test")
            .filter(s -> s.length() > 5)
            .switchIfEmpty(Mono.just("default"))
            .flatMap(s -> Mono.just(s.toUpperCase()));
        
        conditional.subscribe(System.out::println);
    }
}
```

## Spring WebFlux Controllers

### Reactive Controller Implementation

```java
@RestController
@RequestMapping("/api/reactive")
@Slf4j
public class ReactiveController {
    
    private final ReactiveUserService userService;
    private final ReactiveEmailService emailService;
    
    public ReactiveController(ReactiveUserService userService, 
                             ReactiveEmailService emailService) {
        this.userService = userService;
        this.emailService = emailService;
    }
    
    // GET - Return Mono for single resource
    @GetMapping("/users/{id}")
    public Mono<ResponseEntity<User>> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .map(user -> ResponseEntity.ok().eTag("\"" + user.getVersion() + "\"").body(user))
            .switchIfEmpty(Mono.just(ResponseEntity.notFound().build()))
            .doOnNext(response -> {
                if (response.getStatusCode().is2xxSuccessful()) {
                    log.info("User {} retrieved successfully", id);
                }
            });
    }
    
    // GET - Return Flux for multiple resources
    @GetMapping("/users")
    public Flux<User> getUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        return userService.findAll(PageRequest.of(page, size))
            .doOnNext(user -> log.debug("Processing user: {}", user.getId()))
            .doOnComplete(() -> log.info("User stream completed"));
    }
    
    // POST - Process Mono and return Mono
    @PostMapping("/users")
    public Mono<ResponseEntity<User>> createUser(@Valid @RequestBody Mono<CreateUserRequest> request) {
        return request
            .flatMap(req -> userService.create(toUser(req)))
            .map(user -> ResponseEntity.status(HttpStatus.CREATED)
                      .location(URI.create("/api/reactive/users/" + user.getId()))
                      .body(user))
            .doOnNext(response -> log.info("User created with ID: {}", 
                         response.getBody().getId()))
            .onErrorReturn(ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build());
    }
    
    // PUT - Process Mono and return Mono with conditional logic
    @PutMapping("/users/{id}")
    public Mono<ResponseEntity<User>> updateUser(
            @PathVariable Long id,
            @Valid @RequestBody Mono<UpdateUserRequest> request,
            @RequestHeader(value = "If-Match", required = false) String ifMatch) {
        
        return userService.findById(id)
            .switchIfEmpty(Mono.error(new UserNotFoundException(id)))
            .flatMap(existingUser -> {
                // Check ETag for optimistic locking
                if (ifMatch != null && !ifMatch.equals("\"" + existingUser.getVersion() + "\"")) {
                    return Mono.error(new OptimisticLockingException("Resource modified"));
                }
                
                return request.flatMap(req -> {
                    User updated = updateUserFromRequest(existingUser, req);
                    return userService.update(updated);
                });
            })
            .map(user -> ResponseEntity.ok()
                      .eTag("\"" + user.getVersion() + "\"")
                      .body(user))
            .onErrorResume(UserNotFoundException.class, 
                          ex -> Mono.just(ResponseEntity.notFound().build()))
            .onErrorResume(OptimisticLockingException.class,
                          ex -> Mono.just(ResponseEntity.status(HttpStatus.CONFLICT).build()));
    }
    
    // DELETE - Return Mono<Void>
    @DeleteMapping("/users/{id}")
    public Mono<ResponseEntity<Void>> deleteUser(@PathVariable Long id) {
        return userService.findById(id)
            .switchIfEmpty(Mono.error(new UserNotFoundException(id)))
            .flatMap(user -> userService.delete(id))
            .then(Mono.just(ResponseEntity.noContent().<Void>build()))
            .onErrorResume(UserNotFoundException.class,
                          ex -> Mono.just(ResponseEntity.notFound().build()));
    }
    
    // Streaming endpoint
    @GetMapping(value = "/users/stream", produces = MediaType.APPLICATION_NDJSON_VALUE)
    public Flux<User> streamUsers() {
        return userService.findAll()
            .delayElements(Duration.ofMillis(500)) // Simulate streaming
            .doOnNext(user -> log.info("Streaming user: {}", user.getId()));
    }
    
    // Server-Sent Events
    @GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> streamEvents() {
        return Flux.interval(Duration.ofSeconds(1))
            .map(sequence -> ServerSentEvent.builder("Event " + sequence)
                       .id(String.valueOf(sequence))
                       .event("user-event")
                       .build());
    }
    
    // Parallel processing
    @PostMapping("/users/batch")
    public Mono<ResponseEntity<List<User>>> createBatch(
            @RequestBody Flux<CreateUserRequest> requests) {
        
        return requests
            .flatMap(req -> userService.create(toUser(req))
                            .onErrorResume(throwable -> {
                                log.error("Error creating user: {}", throwable.getMessage());
                                return Mono.empty(); // Skip failed users
                            }))
            .collectList()
            .map(users -> ResponseEntity.status(HttpStatus.CREATED).body(users));
    }
    
    // Concurrency handling
    @GetMapping("/users/{id}/with-associations")
    public Mono<UserWithAssociations> getUserWithAssociations(@PathVariable Long id) {
        Mono<User> userMono = userService.findById(id);
        Mono<List<Order>> ordersMono = userService.getUserOrders(id);
        Mono<List<Preference>> preferencesMono = userService.getUserPreferences(id);
        
        return Mono.zip(userMono, ordersMono, preferencesMono)
            .map(tuple -> new UserWithAssociations(
                tuple.getT1(), tuple.getT2(), tuple.getT3()));
    }
    
    private User toUser(CreateUserRequest request) {
        User user = new User();
        user.setUsername(request.getUsername());
        user.setEmail(request.getEmail());
        user.setPassword(request.getPassword());
        return user;
    }
    
    private User updateUserFromRequest(User existing, UpdateUserRequest request) {
        existing.setUsername(request.getUsername());
        existing.setEmail(request.getEmail());
        return existing;
    }
}
```

### Reactive Service Implementation

```java
@Service
@Slf4j
public class ReactiveUserService {
    
    private final UserRepository userRepository;
    private final ReactiveEmailService emailService;
    private final WebClient webClient;
    
    public ReactiveUserService(UserRepository userRepository,
                              ReactiveEmailService emailService,
                              WebClient webClient) {
        this.userRepository = userRepository;
        this.emailService = emailService;
        this.webClient = webClient;
    }
    
    public Mono<User> findById(Long id) {
        return userRepository.findById(id)
            .doOnNext(user -> log.debug("Found user: {}", user.getId()))
            .doOnError(error -> log.error("Error finding user: {}", error.getMessage()));
    }
    
    public Flux<User> findAll() {
        return userRepository.findAll()
            .doOnNext(user -> log.debug("Processing user: {}", user.getId()))
            .doOnError(error -> log.error("Error finding all users: {}", error.getMessage()));
    }
    
    public Flux<User> findAll(Pageable pageable) {
        return userRepository.findAllBy(PageRequest.of(
            pageable.getPageNumber(), 
            pageable.getPageSize(),
            pageable.getSort()))
            .doOnNext(user -> log.debug("Processing user: {}", user.getId()))
            .doOnError(error -> log.error("Error finding users: {}", error.getMessage()));
    }
    
    public Mono<User> create(User user) {
        user.setCreatedDate(LocalDateTime.now());
        user.setPassword(encodePassword(user.getPassword()));
        
        return userRepository.save(user)
            .flatMap(savedUser -> {
                // Send welcome email asynchronously
                return emailService.sendWelcomeEmail(savedUser)
                    .then(Mono.just(savedUser))
                    .onErrorResume(throwable -> {
                        log.error("Email sending failed: {}", throwable.getMessage());
                        return Mono.just(savedUser); // Don't fail user creation
                    });
            })
            .doOnNext(user -> log.info("User created: {}", user.getId()))
            .doOnError(error -> log.error("Error creating user: {}", error.getMessage()));
    }
    
    public Mono<User> update(User user) {
        user.setLastModifiedDate(LocalDateTime.now());
        
        return userRepository.save(user)
            .doOnNext(updated -> log.info("User updated: {}", updated.getId()))
            .doOnError(error -> log.error("Error updating user: {}", error.getMessage()));
    }
    
    public Mono<Void> delete(Long id) {
        return userRepository.deleteById(id)
            .doOnSuccess(v -> log.info("User deleted: {}", id))
            .doOnError(error -> log.error("Error deleting user: {}", error.getMessage()));
    }
    
    public Flux<User> findByEmailContaining(String email) {
        return userRepository.findByEmailContaining(email)
            .doOnNext(user -> log.debug("Found user by email: {}", user.getEmail()))
            .doOnError(error -> log.error("Error finding users by email: {}", error.getMessage()));
    }
    
    public Mono<List<Order>> getUserOrders(Long userId) {
        return webClient.get()
            .uri("/api/orders/user/{userId}", userId)
            .retrieve()
            .bodyToFlux(Order.class)
            .collectList()
            .doOnNext(orders -> log.debug("Retrieved {} orders for user {}", 
                         orders.size(), userId))
            .onErrorReturn(List.of());
    }
    
    public Mono<List<Preference>> getUserPreferences(Long userId) {
        return webClient.get()
            .uri("/api/preferences/user/{userId}", userId)
            .retrieve()
            .bodyToFlux(Preference.class)
            .collectList()
            .doOnNext(preferences -> log.debug("Retrieved {} preferences for user {}", 
                         preferences.size(), userId))
            .onErrorReturn(List.of());
    }
    
    public Mono<Boolean> existsByUsername(String username) {
        return userRepository.existsByUsername(username)
            .doOnNext(exists -> log.debug("Username {} exists: {}", username, exists))
            .onErrorReturn(false);
    }
    
    public Mono<User> findByUsername(String username) {
        return userRepository.findByUsername(username)
            .doOnNext(user -> log.debug("Found user by username: {}", username))
            .doOnError(error -> log.error("Error finding user by username: {}", 
                         error.getMessage()));
    }
    
    private String encodePassword(String password) {
        // Password encoding logic
        return password; // Simplified for example
    }
}
```

## WebClient for Reactive HTTP Calls

### Advanced WebClient Configuration

```java
@Configuration
@EnableWebFlux
public class WebClientConfig {
    
    @Bean
    public WebClient externalApiWebClient() {
        return WebClient.builder()
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .defaultHeader(HttpHeaders.USER_AGENT, "Spring WebFlux Client")
            .codecs(configurer -> configurer.defaultCodecs().maxInMemorySize(2 * 1024 * 1024)) // 2MB
            .clientConnector(new ReactorClientHttpConnector(
                HttpClient.create()
                    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
                    .responseTimeout(Duration.ofSeconds(10))
                    .doOnConnected(conn -> 
                        conn.addHandlerLast(new ReadTimeoutHandler(10))
                            .addHandlerLast(new WriteTimeoutHandler(10)))
            ))
            .build();
    }
    
    @Bean
    public WebClient loadBalancedWebClient(LoadBalancerExchangeFilterFunction lbFunction) {
        return WebClient.builder()
            .filter(lbFunction)
            .build();
    }
}

@Service
@Slf4j
public class ExternalApiService {
    
    private final WebClient externalWebClient;
    private final WebClient loadBalancedWebClient;
    
    public ExternalApiService(WebClient externalApiWebClient,
                             WebClient loadBalancedWebClient) {
        this.externalWebClient = externalApiWebClient;
        this.loadBalancedWebClient = loadBalancedWebClient;
    }
    
    public Mono<ApiResponse> getExternalData(String id) {
        return externalWebClient
            .get()
            .uri("/data/{id}", id)
            .retrieve()
            .onStatus(HttpStatus::is4xxClientError, response -> {
                log.warn("Client error: {}", response.statusCode());
                return Mono.error(new ExternalApiException("Client error"));
            })
            .onStatus(HttpStatus::is5xxServerError, response -> {
                log.warn("Server error: {}", response.statusCode());
                return Mono.error(new ExternalApiException("Server error"));
            })
            .bodyToMono(ApiResponse.class)
            .doOnSuccess(response -> log.info("External API call successful"))
            .doOnError(error -> log.error("External API call failed: {}", error.getMessage()))
            .timeout(Duration.ofSeconds(5))
            .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
                       .filter(throwable -> throwable instanceof ExternalApiException));
    }
    
    public Flux<ApiResponse> getExternalDataStream() {
        return externalWebClient
            .get()
            .uri("/data/stream")
            .retrieve()
            .bodyToFlux(ApiResponse.class)
            .doOnNext(response -> log.info("Received stream item: {}", response.getId()))
            .doOnError(error -> log.error("Stream error: {}", error.getMessage()))
            .take(Duration.ofMinutes(1)); // Timeout after 1 minute
    }
    
    public Mono<ApiResponse> postData(ApiRequest request) {
        return externalWebClient
            .post()
            .uri("/data")
            .bodyValue(request)
            .retrieve()
            .bodyToMono(ApiResponse.class)
            .doOnSuccess(response -> log.info("POST successful: {}", response.getId()))
            .timeout(Duration.ofSeconds(10));
    }
    
    // Load balanced service call
    public Mono<ServiceResponse> callLoadBalancedService(String serviceId) {
        return loadBalancedWebClient
            .get()
            .uri("lb://service-name/api/data/{id}", serviceId)
            .retrieve()
            .bodyToMono(ServiceResponse.class);
    }
    
    // Batch requests
    public Flux<ApiResponse> batchGet(List<String> ids) {
        return Flux.fromIterable(ids)
            .flatMap(id -> getExternalData(id)
                           .onErrorResume(throwable -> {
                               log.warn("Failed to get data for id: {}", id);
                               return Mono.empty();
                           }),
                       5) // Parallelism of 5
            .doOnNext(response -> log.info("Batch response: {}", response.getId()));
    }
}
```

## Reactive Data Access with R2DBC

### R2DBC Repository Implementation

```java
@Repository
public class ReactiveUserRepository implements ReactiveCrudRepository<User, Long> {
    
    private final DatabaseClient databaseClient;
    private final R2dbcEntityTemplate template;
    
    public ReactiveUserRepository(DatabaseClient databaseClient,
                                 R2dbcEntityTemplate template) {
        this.databaseClient = databaseClient;
        this.template = template;
    }
    
    @Override
    public <S extends User> Mono<S> save(S entity) {
        if (entity.getId() == null) {
            return insert(entity);
        } else {
            return update(entity);
        }
    }
    
    private <S extends User> Mono<S> insert(S entity) {
        return databaseClient.sql("INSERT INTO users(username, email, password, created_date) VALUES(:username, :email, :password, :createdDate)")
            .bind("username", entity.getUsername())
            .bind("email", entity.getEmail())
            .bind("password", entity.getPassword())
            .bind("createdDate", Timestamp.valueOf(entity.getCreatedDate()))
            .fetch()
            .first()
            .map(result -> {
                Long id = result.get("id", Long.class);
                entity.setId(id);
                return entity;
            });
    }
    
    private <S extends User> Mono<S> update(S entity) {
        return databaseClient.sql("UPDATE users SET username = :username, email = :email, password = :password, last_modified = :lastModified WHERE id = :id")
            .bind("username", entity.getUsername())
            .bind("email", entity.getEmail())
            .bind("password", entity.getPassword())
            .bind("lastModified", Timestamp.valueOf(entity.getLastModifiedDate()))
            .bind("id", entity.getId())
            .then()
            .then(Mono.just(entity));
    }
    
    @Override
    public Mono<User> findById(Long id) {
        return databaseClient.sql("SELECT * FROM users WHERE id = :id")
            .bind("id", id)
            .map((row, metadata) -> mapRowToUser(row))
            .first();
    }
    
    @Override
    public Flux<User> findAll() {
        return databaseClient.sql("SELECT * FROM users ORDER BY id")
            .map((row, metadata) -> mapRowToUser(row))
            .all();
    }
    
    public Flux<User> findByEmailContaining(String email) {
        return databaseClient.sql("SELECT * FROM users WHERE email LIKE :email")
            .bind("email", "%" + email + "%")
            .map((row, metadata) -> mapRowToUser(row))
            .all();
    }
    
    public Mono<Boolean> existsByUsername(String username) {
        return databaseClient.sql("SELECT COUNT(*) FROM users WHERE username = :username")
            .bind("username", username)
            .map(row -> row.get(0, Long.class) > 0)
            .first();
    }
    
    public Flux<User> findAllBy(Pageable pageable) {
        String sql = "SELECT * FROM users ORDER BY " + 
                    pageable.getSort().stream()
                           .map(order -> order.getProperty() + " " + order.getDirection())
                           .collect(Collectors.joining(", ")) +
                    " LIMIT :limit OFFSET :offset";
        
        return databaseClient.sql(sql)
            .bind("limit", pageable.getPageSize())
            .bind("offset", pageable.getOffset())
            .map((row, metadata) -> mapRowToUser(row))
            .all();
    }
    
    private User mapRowToUser(Row row) {
        User user = new User();
        user.setId(row.get("id", Long.class));
        user.setUsername(row.get("username", String.class));
        user.setEmail(row.get("email", String.class));
        user.setPassword(row.get("password", String.class));
        user.setCreatedDate(row.get("created_date", Timestamp.class).toLocalDateTime());
        
        Timestamp lastModified = row.get("last_modified", Timestamp.class);
        if (lastModified != null) {
            user.setLastModifiedDate(lastModified.toLocalDateTime());
        }
        
        return user;
    }
    
    // Custom query with parameter binding
    public Flux<User> findUsersByCriteria(UserSearchCriteria criteria) {
        StringBuilder sql = new StringBuilder("SELECT * FROM users WHERE 1=1");
        MapSqlParameterSource parameters = new MapSqlParameterSource();
        
        if (criteria.getUsername() != null) {
            sql.append(" AND username LIKE :username");
            parameters.addValue("username", "%" + criteria.getUsername() + "%");
        }
        
        if (criteria.getEmail() != null) {
            sql.append(" AND email LIKE :email");
            parameters.addValue("email", "%" + criteria.getEmail() + "%");
        }
        
        if (criteria.getCreatedAfter() != null) {
            sql.append(" AND created_date >= :createdAfter");
            parameters.addValue("createdAfter", Timestamp.valueOf(criteria.getCreatedAfter()));
        }
        
        return databaseClient.sql(sql.toString())
            .bindAll(parameters)
            .map((row, metadata) -> mapRowToUser(row))
            .all();
    }
}
```

## Testing Reactive Applications

### WebFlux Test Configuration

```java
@ExtendWith(SpringExtension.class)
@WebFluxTest(UserController.class)
@Import({UserMapperImpl.class, GlobalExceptionHandler.class})
class ReactiveUserControllerTest {
    
    @Autowired
    private WebTestClient webTestClient;
    
    @MockBean
    private ReactiveUserService userService;
    
    @Test
    void testGetUser_Success() {
        // Given
        User user = new User(1L, "john_doe", "john@example.com");
        when(userService.findById(1L)).thenReturn(Mono.just(user));
        
        // When & Then
        webTestClient.get()
            .uri("/api/reactive/users/1")
            .accept(MediaType.APPLICATION_JSON)
            .exchange()
            .expectStatus().isOk()
            .expectHeader().contentType(MediaType.APPLICATION_JSON)
            .expectBody(User.class)
            .value(userResponse -> {
                assertThat(userResponse.getId()).isEqualTo(1L);
                assertThat(userResponse.getUsername()).isEqualTo("john_doe");
                assertThat(userResponse.getEmail()).isEqualTo("john@example.com");
            });
    }
    
    @Test
    void testGetUser_NotFound() {
        // Given
        when(userService.findById(999L)).thenReturn(Mono.empty());
        
        // When & Then
        webTestClient.get()
            .uri("/api/reactive/users/999")
            .accept(MediaType.APPLICATION_JSON)
            .exchange()
            .expectStatus().isNotFound();
    }
    
    @Test
    void testCreateUser_Success() {
        // Given
        CreateUserRequest request = new CreateUserRequest();
        request.setUsername("jane_doe");
        request.setEmail("jane@example.com");
        request.setPassword("Password123");
        
        User user = new User(2L, "jane_doe", "jane@example.com");
        when(userService.create(any(User.class))).thenReturn(Mono.just(user));
        
        // When & Then
        webTestClient.post()
            .uri("/api/reactive/users")
            .contentType(MediaType.APPLICATION_JSON)
            .body(Mono.just(request), CreateUserRequest.class)
            .exchange()
            .expectStatus().isCreated()
            .expectHeader().value("Location", containsString("/api/reactive/users/2"))
            .expectBody(User.class)
            .value(userResponse -> {
                assertThat(userResponse.getId()).isEqualTo(2L);
                assertThat(userResponse.getUsername()).isEqualTo("jane_doe");
            });
    }
    
    @Test
    void testGetUsers_Stream() {
        // Given
        List<User> users = Arrays.asList(
            new User(1L, "user1", "user1@example.com"),
            new User(2L, "user2", "user2@example.com")
        );
        
        when(userService.findAll()).thenReturn(Flux.fromIterable(users));
        
        // When & Then
        webTestClient.get()
            .uri("/api/reactive/users")
            .accept(MediaType.APPLICATION_NDJSON)
            .exchange()
            .expectStatus().isOk()
            .expectBodyList(User.class)
            .hasSize(2);
    }
    
    @Test
    void testServerSentEvents() {
        // Given
        when(userService.findAll()).thenReturn(
            Flux.just(
                new User(1L, "user1", "user1@example.com"),
                new User(2L, "user2", "user2@example.com")
            ).delayElements(Duration.ofMillis(100)) // Add delay for streaming
        );
        
        // When & Then
        StepVerifier.create(
            webTestClient.get()
                .uri("/api/reactive/events")
                .accept(MediaType.TEXT_EVENT_STREAM)
                .exchange()
                .expectStatus().isOk()
                .returnResult(String.class)
                .getResponseBody()
        )
        .expectNextCount(2)
        .verifyComplete();
    }
    
    @Test
    void testErrorHandling() {
        // Given
        when(userService.findById(999L)).thenReturn(
            Mono.error(new UserNotFoundException(999L))
        );
        
        // When & Then
        webTestClient.get()
            .uri("/api/reactive/users/999")
            .accept(MediaType.APPLICATION_JSON)
            .exchange()
            .expectStatus().isNotFound()
            .expectBody()
            .jsonPath("$.error").isEqualTo("User Not Found");
    }
}

// Integration testing with Testcontainers
@SpringBootTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class ReactiveUserIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:13")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.r2dbc.url", () -> 
            "r2dbc:postgresql://" + postgres.getHost() + ":" + postgres.getMappedPort(5432) + "/testdb");
        registry.add("spring.r2dbc.username", postgres::getUsername);
        registry.add("spring.r2dbc.password", postgres::getPassword);
    }
    
    @Autowired
    private TestWebClient webClient;
    
    @Test
    void testUserLifecycle() {
        // 1. Create user
        CreateUserRequest createRequest = new CreateUserRequest();
        createRequest.setUsername("testuser");
        createRequest.setEmail("test@example.com");
        createRequest.setPassword("Password123");
        
        User createdUser = webClient.post()
            .uri("/api/reactive/users")
            .body(Mono.just(createRequest), CreateUserRequest.class)
            .exchange()
            .expectStatus().isCreated()
            .expectBody(User.class)
            .returnResult()
            .getResponseBody();
        
        assertThat(createdUser).isNotNull();
        assertThat(createdUser.getUsername()).isEqualTo("testuser");
        
        Long userId = createdUser.getId();
        
        // 2. Get user
        User retrievedUser = webClient.get()
            .uri("/api/reactive/users/{id}", userId)
            .exchange()
            .expectStatus().isOk()
            .expectBody(User.class)
            .returnResult()
            .getResponseBody();
        
        assertThat(retrievedUser.getId()).isEqualTo(userId);
        
        // 3. Delete user
        webClient.delete()
            .uri("/api/reactive/users/{id}", userId)
            .exchange()
            .expectStatus().isNoContent();
        
        // 4. Verify deletion
        webClient.get()
            .uri("/api/reactive/users/{id}", userId)
            .exchange()
            .expectStatus().isNotFound();
    }
}
```

## Performance and Monitoring

### Reactive Metrics and Monitoring

```java
@Component
public class ReactiveMetricsService {
    
    private final MeterRegistry meterRegistry;
    
    public ReactiveMetricsService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    public <T> Mono<T> withMetrics(String operationName, Mono<T> operation) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        return operation
            .doOnSuccess(result -> 
                sample.stop(Timer.builder("reactive.operation")
                    .tag("operation", operationName)
                    .tag("status", "success")
                    .register(meterRegistry)))
            .doOnError(error -> 
                sample.stop(Timer.builder("reactive.operation")
                    .tag("operation", operationName)
                    .tag("status", "error")
                    .tag("error", error.getClass().getSimpleName())
                    .register(meterRegistry)));
    }
    
    public <T> Flux<T> withMetrics(String operationName, Flux<T> operation) {
        Timer.Sample sample = Timer.start(meterRegistry);
        AtomicInteger count = new AtomicInteger(0);
        
        return operation
            .doOnNext(item -> count.incrementAndGet())
            .doOnComplete(() -> 
                sample.stop(Timer.builder("reactive.operation")
                    .tag("operation", operationName)
                    .tag("status", "completed")
                    .tag("count", String.valueOf(count.get()))
                    .register(meterRegistry)))
            .doOnError(error -> 
                sample.stop(Timer.builder("reactive.operation")
                    .tag("operation", operationName)
                    .tag("status", "error")
                    .tag("error", error.getClass().getSimpleName())
                    .register(meterRegistry)));
    }
    
    // Track backpressure
    public <T> Flux<T> withBackpressureMetrics(Flux<T> flux) {
        AtomicInteger requested = new AtomicInteger(0);
        
        return flux
            .doOnRequest(n -> requested.addAndGet(n.intValue()))
            .doOnNext(item -> meterRegistry.counter("reactive.items.processed").increment())
            .doOnSubscribe(subscription -> 
                meterRegistry.gauge("reactive.requested.items", requested));
    }
    
    // Track buffer usage
    public <T> Flux<T> withBufferMetrics(Flux<T> flux) {
        return flux
            .onBackpressureBuffer(100, 
                (overflow, dropped) -> meterRegistry.counter("reactive.dropped.items").increment())
            .doOnNext(item -> 
                meterRegistry.gauge("reactive.buffer.size", 
                    flux.log().doOnNext(i -> {}).subscribe())
            );
    }
}

// Reactive health indicators
@Component
public class DatabaseHealthIndicator implements ReactiveHealthIndicator {
    
    private final ReactiveUserRepository userRepository;
    
    public DatabaseHealthIndicator(ReactiveUserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    @Override
    public Mono<Health> health() {
        return userRepository.count()
            .map(count -> Health.up()
                .withDetail("userCount", count)
                .withDetail("status", "available")
                .build())
            .onErrorMap(throwable -> Health.down()
                .withDetail("error", throwable.getMessage())
                .build())
            .cast(Health.class);
    }
}
```

## Conclusion

Spring WebFlux and Reactive Programming provide:
- Non-blocking, event-driven processing for high performance
- Backpressure handling for flow control
- Comprehensive error handling and retry mechanisms
- Reactive data access with R2DBC
- Advanced testing capabilities for reactive applications
- Performance monitoring and metrics

Mastering these concepts is essential for:
- High-performance web applications
- Microservices with reactive communication
- Real-time data streaming applications
- Scalable system architecture
- Modern cloud-native application development