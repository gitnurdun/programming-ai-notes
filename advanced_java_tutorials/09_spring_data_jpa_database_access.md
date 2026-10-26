# Advanced Spring Data JPA and Database Access

## Overview
Spring Data JPA simplifies data access layers in Java applications by reducing boilerplate code and providing powerful abstractions. Understanding its advanced features is crucial for building efficient, scalable database applications.

## JPA Repository Patterns

### Repository Interface Hierarchy

```java
// Base repository with custom functionality
@NoRepositoryBean
public interface BaseRepository<T, ID extends Serializable> 
        extends JpaRepository<T, ID>, JpaSpecificationExecutor<T> {
    
    // Custom shared methods
    List<T> findByCreatedDateAfter(LocalDateTime date);
    
    @Query("SELECT COUNT(t) FROM #{#entityName} t WHERE t.active = true")
    long countActive();
    
    @Modifying
    @Query("UPDATE #{#entityName} t SET t.lastModified = :date WHERE t.id = :id")
    void updateLastModified(@Param("id") ID id, @Param("date") LocalDateTime date);
}

// Domain-specific repository
@Repository
public interface UserRepository extends BaseRepository<User, Long> {
    
    // Query methods with derived queries
    List<User> findByEmailContainingAndActiveTrue(String email);
    
    Optional<User> findByUsername(String username);
    
    @Query("SELECT u FROM User u WHERE u.email = :email AND u.active = true")
    Optional<User> findActiveByEmail(@Param("email") String email);
    
    // Custom projections
    @Query("SELECT u.username as username, u.email as email FROM User u WHERE u.active = true")
    List<UserProjection> findActiveUsersProjection();
    
    // Dynamic projections
    <T> List<T> findByEmailContaining(String email, Class<T> type);
    
    // Batch operations
    @Modifying
    @Query("UPDATE User u SET u.lastLogin = :loginTime WHERE u.id IN :ids")
    int updateLastLoginForUsers(@Param("ids") List<Long> ids, 
                               @Param("loginTime") LocalDateTime loginTime);
    
    // Native queries with pagination
    @Query(value = "SELECT * FROM users WHERE email LIKE %:pattern%", 
           countQuery = "SELECT count(*) FROM users WHERE email LIKE %:pattern%",
           nativeQuery = true)
    Page<User> findByEmailPattern(@Param("pattern") String pattern, Pageable pageable);
}

// Projection interfaces
public interface UserProjection {
    String getUsername();
    String getEmail();
}

// Dynamic projection with SpEL
public interface UserSummary {
    @Value("#{target.firstName + ' ' + target.lastName}")
    String getFullName();
    
    String getEmail();
}
```

### Custom Repository Implementations

```java
// Custom repository interface
public interface UserRepositoryCustom {
    List<User> findUsersWithComplexCriteria(UserSearchCriteria criteria);
    Page<UserStatistics> getUserStatistics(Pageable pageable);
    void bulkUpdateUserStatus(List<Long> userIds, UserStatus status);
}

// Custom repository implementation
@Repository
public class UserRepositoryImpl implements UserRepositoryCustom {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public List<User> findUsersWithComplexCriteria(UserSearchCriteria criteria) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> user = query.from(User.class);
        
        List<Predicate> predicates = new ArrayList<>();
        
        if (criteria.getEmail() != null && !criteria.getEmail().isEmpty()) {
            predicates.add(cb.like(cb.lower(user.get("email")), 
                                 "%" + criteria.getEmail().toLowerCase() + "%"));
        }
        
        if (criteria.getCreatedAfter() != null) {
            predicates.add(cb.greaterThanOrEqualTo(user.get("createdDate"), 
                                                 criteria.getCreatedAfter()));
        }
        
        if (criteria.getStatus() != null) {
            predicates.add(cb.equal(user.get("status"), criteria.getStatus()));
        }
        
        query.where(predicates.toArray(new Predicate[0]));
        query.orderBy(cb.desc(user.get("createdDate")));
        
        return entityManager.createQuery(query).getResultList();
    }
    
    @Override
    public Page<UserStatistics> getUserStatistics(Pageable pageable) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        
        // Main query for data
        CriteriaQuery<UserStatistics> query = cb.createQuery(UserStatistics.class);
        Root<User> user = query.from(User.class);
        
        query.multiselect(
            user.get("status").alias("status"),
            cb.count(user).alias("count"),
            cb.max(user.get("lastLogin")).alias("lastLogin")
        );
        
        query.groupBy(user.get("status"));
        
        // Count query
        CriteriaQuery<Long> countQuery = cb.createQuery(Long.class);
        Root<User> countRoot = countQuery.from(User.class);
        countQuery.select(cb.countDistinct(countRoot.get("status")));
        
        // Execute queries with pagination
        List<UserStatistics> results = entityManager.createQuery(query)
            .setFirstResult((int) pageable.getOffset())
            .setMaxResults(pageable.getPageSize())
            .getResultList();
        
        Long total = entityManager.createQuery(countQuery).getSingleResult();
        
        return new PageImpl<>(results, pageable, total);
    }
    
    @Override
    @Transactional
    public void bulkUpdateUserStatus(List<Long> userIds, UserStatus status) {
        if (userIds.isEmpty()) return;
        
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaUpdate<User> update = cb.createCriteriaUpdate(User.class);
        Root<User> user = update.from(User.class);
        
        update.set("status", status)
              .set("lastModified", LocalDateTime.now())
              .where(user.get("id").in(userIds));
        
        entityManager.createQuery(update).executeUpdate();
    }
}

// Extend JpaRepository with custom implementation
@Repository
public interface UserRepository extends JpaRepository<User, Long>, 
                                         JpaSpecificationExecutor<User>,
                                         UserRepositoryCustom {
    // Standard methods plus custom ones
}
```

## Advanced JPA Features

### Entity Relationships and Performance

```java
@Entity
@Table(name = "users")
@NamedEntityGraph(
    name = "User.withRolesAndDepartment",
    attributeNodes = {
        @NamedAttributeNode("roles"),
        @NamedAttributeNode(value = "department", subgraph = "department-graph")
    },
    subgraphs = {
        @NamedSubgraph(
            name = "department-graph",
            attributeNodes = @NamedAttributeNode("organization")
        )
    }
)
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true)
    private String username;
    
    @Column(nullable = false)
    private String email;
    
    // Lazy loading with fetch strategy
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    @BatchSize(size = 10)
    private Set<Role> roles = new HashSet<>();
    
    // Eager loading with entity graph
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;
    
    // Optimistic locking
    @Version
    private Long version;
    
    // Element collection with ordering
    @ElementCollection
    @CollectionTable(name = "user_preferences", joinColumns = @JoinColumn(name = "user_id"))
    @MapKeyColumn(name = "preference_key")
    @Column(name = "preference_value")
    private Map<String, String> preferences = new HashMap<>();
    
    // Constructors, getters, setters
}

@Entity
@Table(name = "orders")
@org.springframework.data.elasticsearch.annotations.Document(indexName = "orders")
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("position ASC")
    private List<OrderItem> items = new ArrayList<>();
    
    // Read-only computed field
    @Formula("(SELECT SUM(oi.quantity * oi.price) FROM order_items oi WHERE oi.order_id = id)")
    private BigDecimal totalAmount;
    
    // Auditing fields
    @CreatedDate
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdDate;
    
    @LastModifiedDate
    @Column(nullable = false)
    private LocalDateTime lastModifiedDate;
    
    // Constructors, getters, setters
}

@Entity
@Table(name = "order_items")
@Immutable
public class OrderItem {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;
    
    @Column(nullable = false)
    private Integer quantity;
    
    @Column(nullable = false, precision = 19, scale = 2)
    private BigDecimal price;
    
    // Constructors, getters, setters
}
```

### QueryDSL Integration

```java
// Generated QueryDSL classes (typically auto-generated)
public class UserQueryService {
    
    private final JPAQueryFactory queryFactory;
    
    public UserQueryService(EntityManager entityManager) {
        this.queryFactory = new JPAQueryFactory(entityManager);
    }
    
    public List<User> findUsersWithComplexCriteria(UserSearchCriteria criteria) {
        QUser user = QUser.user;
        QRole role = QRole.role;
        
        JPAQuery<User> query = queryFactory.selectFrom(user);
        
        if (criteria.getEmail() != null && !criteria.getEmail().isEmpty()) {
            query.where(user.email.containsIgnoreCase(criteria.getEmail()));
        }
        
        if (criteria.getCreatedAfter() != null) {
            query.where(user.createdDate.goe(criteria.getCreatedAfter()));
        }
        
        if (criteria.getStatus() != null) {
            query.where(user.status.eq(criteria.getStatus()));
        }
        
        if (criteria.getRoleNames() != null && !criteria.getRoleNames().isEmpty()) {
            query.join(user.roles, role)
                 .where(role.name.in(criteria.getRoleNames()));
        }
        
        return query.orderBy(user.createdDate.desc()).fetch();
    }
    
    public Page<User> findUsersWithPagination(UserSearchCriteria criteria, Pageable pageable) {
        QUser user = QUser.user;
        
        JPAQuery<User> query = queryFactory.selectFrom(user);
        
        // Apply filters
        applyFilters(query, user, criteria);
        
        // Apply pagination
        long total = query.fetchCount();
        List<User> users = query
            .offset(pageable.getOffset())
            .limit(pageable.getPageSize())
            .orderBy(getOrderSpecifiers(user, pageable.getSort()))
            .fetch();
        
        return new PageImpl<>(users, pageable, total);
    }
    
    private void applyFilters(JPAQuery<User> query, QUser user, UserSearchCriteria criteria) {
        if (criteria.getEmail() != null) {
            query.where(user.email.containsIgnoreCase(criteria.getEmail()));
        }
        if (criteria.getStatus() != null) {
            query.where(user.status.eq(criteria.getStatus()));
        }
    }
    
    private OrderSpecifier<?>[] getOrderSpecifiers(QUser user, Sort sort) {
        return sort.stream()
            .map(order -> {
                switch (order.getProperty()) {
                    case "email":
                        return order.isAscending() ? user.email.asc() : user.email.desc();
                    case "createdDate":
                        return order.isAscending() ? user.createdDate.asc() : user.createdDate.desc();
                    default:
                        return user.id.asc();
                }
            })
            .toArray(OrderSpecifier[]::new);
    }
}
```

## Performance Optimization

### Caching Strategies

```java
@Entity
@Table(name = "products")
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(precision = 19, scale = 2)
    private BigDecimal price;
    
    // Second-level cache with custom region
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    @org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE, 
                                   region = "product.category")
    private Category category;
    
    // Query cache
    @org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
    public BigDecimal getDiscountedPrice() {
        return price.multiply(BigDecimal.valueOf(0.9));
    }
}

@Repository
public class ProductRepositoryImpl {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    // Query with cache
    @QueryHints(@QueryHint(name = org.hibernate.annotations.QueryHints.CACHEABLE, value = "true"))
    public List<Product> findActiveProducts() {
        return entityManager.createQuery(
            "SELECT p FROM Product p WHERE p.active = true", Product.class)
            .setHint("org.hibernate.cacheable", true)
            .getResultList();
    }
    
    // Native query with cache
    @Query(value = "SELECT * FROM products WHERE active = true", 
           nativeQuery = true,
           hints = @QueryHint(name = org.hibernate.annotations.QueryHints.CACHEABLE, value = "true"))
    List<Product> findActiveProductsNative();
}

// Cache configuration
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .recordStats());
        return cacheManager;
    }
    
    @Bean
    public CacheResolver customCacheResolver(CacheManager cacheManager) {
        SimpleCacheResolver resolver = new SimpleCacheResolver();
        resolver.setCacheManager(cacheManager);
        return resolver;
    }
}
```

### Batch Processing and Performance Tuning

```java
@Service
@Transactional
public class BatchUserService {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    @Autowired
    private UserRepository userRepository;
    
    // Batch insert with flush and clear
    public void batchInsertUsers(List<User> users) {
        int batchSize = 50;
        
        for (int i = 0; i < users.size(); i++) {
            entityManager.persist(users.get(i));
            
            if (i % batchSize == 0 && i > 0) {
                // Flush and clear to manage memory
                entityManager.flush();
                entityManager.clear();
            }
        }
        
        // Flush remaining entities
        entityManager.flush();
        entityManager.clear();
    }
    
    // Stateless session for large batch operations
    public void processLargeUserBatch(List<UserData> userDataList) {
        Session session = entityManager.unwrap(Session.class);
        StatelessSession statelessSession = session.getSessionFactory().openStatelessSession();
        
        Transaction transaction = statelessSession.beginTransaction();
        
        try {
            for (UserData userData : userDataList) {
                User user = new User();
                user.setUsername(userData.getUsername());
                user.setEmail(userData.getEmail());
                // Set other properties
                
                statelessSession.insert(user);
            }
            
            transaction.commit();
        } catch (Exception e) {
            transaction.rollback();
            throw new RuntimeException("Batch processing failed", e);
        } finally {
            statelessSession.close();
        }
    }
    
    // Bulk update with CriteriaUpdate
    public int bulkUpdateUserStatus(UserStatus oldStatus, UserStatus newStatus) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaUpdate<User> update = cb.createCriteriaUpdate(User.class);
        Root<User> root = update.from(User.class);
        
        update.set("status", newStatus)
              .set("lastModified", LocalDateTime.now())
              .where(cb.equal(root.get("status"), oldStatus));
        
        return entityManager.createQuery(update).executeUpdate();
    }
    
    // Fetch join optimization
    @EntityGraph("User.withRolesAndDepartment")
    public List<User> findUsersWithRolesAndDepartments() {
        return userRepository.findAll();
    }
}
```

## Transaction Management

### Advanced Transaction Configuration

```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    // Programmatic transaction management
    @Autowired
    private PlatformTransactionManager transactionManager;
    
    // Transaction with custom isolation level
    @Transactional(isolation = Isolation.REPEATABLE_READ)
    public void transferCredits(Long fromUserId, Long toUserId, BigDecimal amount) {
        User fromUser = userRepository.findById(fromUserId)
            .orElseThrow(() -> new UserNotFoundException(fromUserId));
            
        User toUser = userRepository.findById(toUserId)
            .orElseThrow(() -> new UserNotFoundException(toUserId));
        
        if (fromUser.getCredits().compareTo(amount) < 0) {
            throw new InsufficientCreditsException();
        }
        
        fromUser.setCredits(fromUser.getCredits().subtract(amount));
        toUser.setCredits(toUser.getCredits().add(amount));
        
        userRepository.save(fromUser);
        userRepository.save(toUser);
    }
    
    // Nested transactions with propagation
    @Transactional(propagation = Propagation.REQUIRED)
    public void processUserRegistration(UserRegistrationRequest request) {
        User user = createUser(request);
        assignDefaultRoles(user);
        sendWelcomeEmail(user);
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendWelcomeEmail(User user) {
        // Send email in separate transaction
        // If this fails, user creation still succeeds
    }
    
    // Transaction with timeout and rollback rules
    @Transactional(timeout = 30, 
                   rollbackFor = {DataAccessException.class, ValidationException.class},
                   noRollbackFor = {BusinessException.class})
    public void complexBusinessOperation() {
        // Long-running operation with custom rollback behavior
    }
    
    // Programmatic transaction usage
    public void programmaticTransaction() {
        TransactionDefinition def = new DefaultTransactionDefinition();
        TransactionStatus status = transactionManager.getTransaction(def);
        
        try {
            // Business logic
            userRepository.save(new User());
            
            transactionManager.commit(status);
        } catch (Exception e) {
            transactionManager.rollback(status);
            throw e;
        }
    }
}

// Transaction event listeners
@Component
public class TransactionEventListener {
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleUserCreated(UserCreatedEvent event) {
        // Send email after transaction commits
        // This ensures email is only sent if user creation succeeds
    }
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void handleUserCreationFailed(UserCreationFailedEvent event) {
        // Clean up resources if transaction rolls back
    }
    
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void beforeCommit(UserBeforeCommitEvent event) {
        // Validation or final processing before commit
    }
}
```

## Database Migration and Schema Management

### Flyway Integration

```java
@Configuration
public class DatabaseMigrationConfig {
    
    @Bean
    public Flyway flyway(DataSource dataSource) {
        Flyway flyway = Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration", "classpath:db/callback")
            .baselineOnMigrate(true)
            .validateOnMigrate(true)
            .outOfOrder(true)
            .placeholders(Map.of(
                "schema.name", "public",
                "table.prefix", "app_"
            ))
            .load();
            
        // Custom migration callback
        flyway.execute(new FlywayCallback() {
            @Override
            public void afterEachMigrate(Flyway flyway, MigrationInfo info) {
                // Post-migration tasks
            }
        });
        
        return flyway;
    }
}

// Example migration scripts
// V1__Create_user_table.sql
/*
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    created_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_modified TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created_date ON users(created_date);
*/

// V2__Add_user_roles_table.sql
/*
CREATE TABLE roles (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    description TEXT
);

CREATE TABLE user_roles (
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id BIGINT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);
*/
```

## Testing Data Access Layers

### Repository Testing

```java
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
    void testFindByEmailContainingAndActiveTrue() {
        // Given
        User activeUser = new User("john", "john@example.com", true);
        User inactiveUser = new User("jane", "jane@example.com", false);
        entityManager.persistAndFlush(activeUser);
        entityManager.persistAndFlush(inactiveUser);
        
        // When
        List<User> results = userRepository.findByEmailContainingAndActiveTrue("john");
        
        // Then
        assertThat(results).hasSize(1);
        assertThat(results.get(0).getUsername()).isEqualTo("john");
    }
    
    @Test
    void testCustomRepositoryMethod() {
        // Given
        UserSearchCriteria criteria = new UserSearchCriteria();
        criteria.setEmail("example");
        criteria.setStatus(UserStatus.ACTIVE);
        
        User user = new User("test", "test@example.com", UserStatus.ACTIVE);
        entityManager.persistAndFlush(user);
        
        // When
        List<User> results = ((UserRepositoryImpl) userRepository).findUsersWithComplexCriteria(criteria);
        
        // Then
        assertThat(results).hasSize(1);
    }
}

// Integration testing with Testcontainers
@SpringBootTest
@Testcontainers
class UserRepositoryIntegrationTest {
    
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
    private UserRepository userRepository;
    
    @Test
    @Transactional
    void testBulkUpdate() {
        // Given
        List<User> users = createTestUsers(100);
        userRepository.saveAll(users);
        
        List<Long> userIds = users.stream()
            .map(User::getId)
            .collect(Collectors.toList());
        
        // When
        int updated = ((UserRepositoryImpl) userRepository).bulkUpdateUserStatus(userIds, UserStatus.INACTIVE);
        
        // Then
        assertThat(updated).isEqualTo(100);
        List<User> updatedUsers = userRepository.findAllById(userIds);
        assertThat(updatedUsers).allMatch(user -> user.getStatus() == UserStatus.INACTIVE);
    }
}
```

## Conclusion

Advanced Spring Data JPA provides:
- Powerful repository abstractions with custom implementations
- Performance optimization through caching and batch processing
- Sophisticated query capabilities with QueryDSL and Specifications
- Advanced transaction management features
- Comprehensive testing support
- Database migration and schema management

Mastering these features is essential for:
- Senior Java developer roles
- High-performance database applications
- Enterprise application architecture
- Scalable system design
- Performance optimization and tuning