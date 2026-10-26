# Advanced Testing Strategies and Mocking

## Overview
Testing is fundamental to software quality and reliability. Advanced testing strategies include unit testing, integration testing, contract testing, and end-to-end testing. Effective mocking techniques are essential for isolating components and creating controlled test environments.

## JUnit 5 Advanced Features

### Dynamic Tests and Parameterized Tests

```java
import org.junit.jupiter.api.*;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.*;
import org.junit.jupiter.api.DynamicTest;
import org.junit.jupiter.api.TestFactory;

import java.util.*;
import java.util.stream.Stream;

import static org.junit.jupiter.api.Assertions.*;
import static org.junit.jupiter.api.DynamicTest.*;

class AdvancedJUnitTests {
    
    private Calculator calculator;
    
    @BeforeEach
    void setUp() {
        calculator = new Calculator();
    }
    
    // Parameterized tests with multiple sources
    @ParameterizedTest(name = "Test {index}: {0} + {1} = {2}")
    @CsvSource({
        "1, 2, 3",
        "5, 7, 12",
        "0, 0, 0",
        "-1, 1, 0"
    })
    void testAddition(int a, int b, int expected) {
        int result = calculator.add(a, b);
        assertEquals(expected, result, 
            () -> String.format("%d + %d should equal %d", a, b, expected));
    }
    
    // Method source for complex parameterized tests
    @ParameterizedTest
    @MethodSource("provideUserValidationData")
    void testUserValidation(UserInput input, boolean expectedValid) {
        boolean isValid = validateUserInput(input);
        assertEquals(expectedValid, isValid, 
            "User validation should return " + expectedValid);
    }
    
    static Stream<Arguments> provideUserValidationData() {
        return Stream.of(
            Arguments.of(new UserInput("john", "john@example.com", 25), true),
            Arguments.of(new UserInput("", "john@example.com", 25), false),
            Arguments.of(new UserInput("john", "invalid-email", 25), false),
            Arguments.of(new UserInput("john", "john@example.com", 150), false),
            Arguments.of(new UserInput("validname", "valid@example.com", 30), true)
        );
    }
    
    // Test factory for dynamic tests
    @TestFactory
    Collection<DynamicTest> dynamicTestsFromCollection() {
        List<User> users = Arrays.asList(
            new User("Alice", 25),
            new User("Bob", 30),
            new User("Charlie", 35)
        );
        
        return users.stream()
            .map(user -> dynamicTest(
                "Test user age: " + user.getName(),
                () -> assertTrue(user.getAge() > 0, 
                    "Age should be positive for user: " + user.getName())
            ))
            .collect(Collectors.toList());
    }
    
    // Dynamic tests with external data
    @TestFactory
    Stream<DynamicTest> dynamicTestsFromStream() {
        Map<String, Integer> testCases = Map.of(
            "positive number", 42,
            "zero", 0,
            "negative number", -5
        );
        
        return testCases.entrySet().stream()
            .map(entry -> dynamicTest(
                "Square root test: " + entry.getKey(),
                () -> {
                    if (entry.getValue() >= 0) {
                        double result = Math.sqrt(entry.getValue());
                        assertTrue(result >= 0, "Result should be non-negative");
                    } else {
                        assertThrows(ArithmeticException.class, 
                            () -> Math.sqrt(entry.getValue()));
                    }
                }
            ));
    }
    
    // Nested tests for better organization
    @Nested
    @DisplayName("Calculator Division Tests")
    class DivisionTests {
        
        @Test
        @DisplayName("Divide positive numbers")
        void testPositiveDivision() {
            double result = calculator.divide(10, 2);
            assertEquals(5.0, result, 0.001);
        }
        
        @Test
        @DisplayName("Divide by zero throws exception")
        void testDivisionByZero() {
            ArithmeticException exception = assertThrows(
                ArithmeticException.class,
                () -> calculator.divide(10, 0)
            );
            assertEquals("Division by zero", exception.getMessage());
        }
        
        @Nested
        @DisplayName("Edge cases")
        class EdgeCases {
            
            @Test
            @DisplayName("Divide with floating point precision")
            void testFloatingPointDivision() {
                double result = calculator.divide(10.0, 3.0);
                assertEquals(3.3333333333333335, result, 0.000000000000001);
            }
        }
    }
    
    // Repeated tests
    @RepeatedTest(value = 5, name = "Attempt {currentRepetition} of {totalRepetitions}")
    void repeatedTest() {
        int randomValue = new Random().nextInt(100);
        assertTrue(randomValue >= 0 && randomValue <= 100);
    }
    
    // Conditional tests
    @Test
    @EnabledOnOs(OS.WINDOWS)
    void onlyRunOnWindows() {
        assertTrue(System.getProperty("os.name").toLowerCase().contains("windows"));
    }
    
    @Test
    @EnabledOnJre(JRE.JAVA_17)
    void onlyRunOnJava17() {
        assertEquals(17, Runtime.version().feature());
    }
    
    @Test
    @EnabledIfSystemProperty(named = "test.profile", matches = "integration")
    void conditionalTest() {
        // This test runs only if -Dtest.profile=integration
    }
    
    private boolean validateUserInput(UserInput input) {
        return input.getName() != null && !input.getName().trim().isEmpty() &&
               input.getEmail() != null && input.getEmail().contains("@") &&
               input.getAge() > 0 && input.getAge() < 150;
    }
}

// Helper classes for testing
class Calculator {
    public int add(int a, int b) { return a + b; }
    public double divide(double a, double b) {
        if (b == 0) throw new ArithmeticException("Division by zero");
        return a / b;
    }
}

class UserInput {
    private String name;
    private String email;
    private int age;
    
    public UserInput(String name, String email, int age) {
        this.name = name;
        this.email = email;
        this.age = age;
    }
    
    // getters
    public String getName() { return name; }
    public String getEmail() { return email; }
    public int getAge() { return age; }
}

class User {
    private String name;
    private int age;
    
    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    // getters
    public String getName() { return name; }
    public int getAge() { return age; }
}
```

### Advanced Assertions and Assumptions

```java
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.condition.EnabledIfEnvironmentVariable;
import org.junit.jupiter.api.extension.ExtendWith;

import java.time.LocalDateTime;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

import static org.junit.jupiter.api.Assertions.*;

class AdvancedAssertionsTests {
    
    // Assertions with suppliers for lazy evaluation
    @Test
    void testWithLazyEvaluation() {
        List<String> expensiveList = getExpensiveList();
        
        assertAll("List assertions",
            () -> assertEquals(5, expensiveList.size(), 
                () -> "Size should be 5, but was " + expensiveList.size()),
            () -> assertTrue(expensiveList.contains("item3"), 
                () -> "List should contain 'item3' but was: " + expensiveList),
            () -> assertFalse(expensiveList.contains("nonexistent"), 
                () -> "List should not contain 'nonexistent' but was: " + expensiveList)
        );
    }
    
    // Timeout assertions
    @Test
    void testWithTimeout() {
        assertTimeout(Duration.ofSeconds(5), () -> {
            // Simulate long-running operation
            Thread.sleep(2000);
            return "completed";
        }, "Operation should complete within 5 seconds");
    }
    
    // Timeout preemption
    @Test
    void testWithTimeoutPreemptively() {
        assertTimeoutPreemptively(Duration.ofMillis(500), () -> {
            // This will be interrupted after 500ms
            Thread.sleep(1000);
            fail("This should not be reached");
        });
    }
    
    // Executable assertions
    @Test
    void testExecutable() {
        Executable executable = () -> {
            String result = performComplexOperation();
            assertNotNull(result);
            assertTrue(result.length() > 0);
        };
        
        assertAll("Executable test", executable);
    }
    
    // Assumptions with external conditions
    @Test
    void testWithAssumptions() {
        assumeTrue(System.getProperty("test.db.enabled", "false").equals("true"));
        assumeFalse(System.getProperty("test.mode", "production").equals("production"));
        
        // Test code that depends on database being available
        // This test will be skipped if assumptions are not met
    }
    
    // Assumptions with message
    @Test
    void testWithAssumptionMessage() {
        String environment = System.getProperty("test.environment", "local");
        
        assumeTrue(
            environment.equals("staging") || environment.equals("local"),
            () -> "Test requires staging or local environment, but running in: " + environment
        );
        
        // Test code
    }
    
    // Group assertions
    @Test
    void testGroupAssertions() {
        Person person = new Person("John", "Doe", 30);
        
        assertAll("Person properties",
            () -> assertAll("Name validations",
                () -> assertEquals("John", person.getFirstName()),
                () -> assertEquals("Doe", person.getLastName())
            ),
            () -> assertAll("Age validations",
                () -> assertEquals(30, person.getAge()),
                () -> assertTrue(person.getAge() > 0)
            )
        );
    }
    
    // Future assertions
    @Test
    void testFutureCompletion() throws Exception {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000);
                return "Result";
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return null;
            }
        });
        
        String result = assertTimeoutPreemptively(
            Duration.ofSeconds(5),
            () -> future.get(3, TimeUnit.SECONDS)
        );
        
        assertEquals("Result", result);
    }
    
    private List<String> getExpensiveList() {
        // Simulate expensive operation
        return Arrays.asList("item1", "item2", "item3", "item4", "item5");
    }
    
    private String performComplexOperation() {
        return "result";
    }
}

class Person {
    private String firstName;
    private String lastName;
    private int age;
    
    public Person(String firstName, String lastName, int age) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.age = age;
    }
    
    // getters
    public String getFirstName() { return firstName; }
    public String getLastName() { return lastName; }
    public int getAge() { return age; }
}
```

## Mockito Advanced Mocking Techniques

### Complex Mocking Scenarios

```java
import static org.mockito.Mockito.*;
import static org.mockito.AdditionalAnswers.*;
import static org.mockito.AdditionalMatchers.*;

import java.util.*;
import java.util.concurrent.CompletableFuture;
import java.util.function.BiFunction;
import java.util.stream.Stream;

import org.mockito.ArgumentCaptor;
import org.mockito.InOrder;
import org.mockito.MockedStatic;
import org.mockito.invocation.InvocationOnMock;
import org.mockito.stubbing.Answer;
import org.mockito.stubbing.ConsecutiveStubbing;

class AdvancedMockitoTests {
    
    @Test
    void testComplexMockingScenario() {
        // Complex service with multiple dependencies
        PaymentService paymentService = mock(PaymentService.class);
        UserService userService = mock(UserService.class);
        EmailService emailService = mock(EmailService.class);
        NotificationService notificationService = mock(NotificationService.class);
        
        // Mocking complex return values
        when(paymentService.processPayment(any(PaymentRequest.class)))
            .thenReturn(PaymentResult.success("TXN123", "Payment processed successfully"));
        
        when(userService.findById(anyLong()))
            .thenReturn(Optional.of(new User("john", "john@example.com")));
        
        // Mocking void methods
        doNothing().when(emailService).sendEmail(any(String.class), any(String.class));
        
        // Mocking with custom answers
        when(paymentService.calculateFees(any(BigDecimal.class)))
            .thenAnswer(invocation -> {
                BigDecimal amount = invocation.getArgument(0);
                if (amount.compareTo(new BigDecimal("1000")) > 0) {
                    return amount.multiply(new BigDecimal("0.02")); // 2% fee
                } else {
                    return amount.multiply(new BigDecimal("0.05")); // 5% fee
                }
            });
        
        // Testing the service under test
        OrderService orderService = new OrderService(
            paymentService, userService, emailService, notificationService);
        
        Order order = new Order(1L, new User("john", "john@example.com"), 
                               BigDecimal.valueOf(1500));
        
        OrderResult result = orderService.processOrder(order);
        
        // Verify interactions
        verify(paymentService).processPayment(any(PaymentRequest.class));
        verify(emailService).sendEmail(eq("john@example.com"), any(String.class));
        verify(notificationService, times(2)).sendNotification(any(Notification.class));
        
        // Verify specific argument values
        verify(paymentService).calculateFees(argThat(amount -> amount.compareTo(BigDecimal.valueOf(100)) > 0));
    }
    
    @Test
    void testArgumentCaptor() {
        EmailService emailService = mock(EmailService.class);
        OrderService orderService = new OrderService(
            mock(PaymentService.class),
            mock(UserService.class),
            emailService,
            mock(NotificationService.class)
        );
        
        ArgumentCaptor<String> emailCaptor = ArgumentCaptor.forClass(String.class);
        ArgumentCaptor<String> subjectCaptor = ArgumentCaptor.forClass(String.class);
        
        Order order = new Order(1L, new User("john", "john@example.com"), 
                               BigDecimal.valueOf(100));
        orderService.processOrder(order);
        
        verify(emailService, times(2)).sendEmail(
            emailCaptor.capture(), 
            subjectCaptor.capture()
        );
        
        List<String> capturedEmails = emailCaptor.getAllValues();
        List<String> capturedSubjects = subjectCaptor.getAllValues();
        
        assertEquals(2, capturedEmails.size());
        assertEquals("john@example.com", capturedEmails.get(0));
        assertEquals("john@example.com", capturedEmails.get(1));
        
        assertTrue(capturedSubjects.get(0).contains("Order Confirmation"));
        assertTrue(capturedSubjects.get(1).contains("Payment Receipt"));
    }
    
    @Test
    void testInOrderVerification() {
        EmailService emailService = mock(EmailService.class);
        NotificationService notificationService = mock(NotificationService.class);
        
        OrderService orderService = new OrderService(
            mock(PaymentService.class),
            mock(UserService.class),
            emailService,
            notificationService
        );
        
        Order order = new Order(1L, new User("john", "john@example.com"), 
                               BigDecimal.valueOf(100));
        orderService.processOrder(order);
        
        InOrder inOrder = inOrder(emailService, notificationService);
        
        // Verify order of operations
        inOrder.verify(notificationService).sendNotification(any(Notification.class));
        inOrder.verify(emailService).sendEmail(any(String.class), any(String.class));
        inOrder.verify(notificationService).sendNotification(any(Notification.class));
    }
    
    @Test
    void testConsecutiveCalls() {
        PaymentService paymentService = mock(PaymentService.class);
        
        // Mock consecutive calls with different return values
        when(paymentService.processPayment(any(PaymentRequest.class)))
            .thenReturn(PaymentResult.success("TXN001", "First call"))
            .thenReturn(PaymentResult.success("TXN002", "Second call"))
            .thenThrow(new PaymentException("Third call failed"));
        
        // First call
        PaymentResult result1 = paymentService.processPayment(new PaymentRequest());
        assertEquals("TXN001", result1.getTransactionId());
        
        // Second call
        PaymentResult result2 = paymentService.processPayment(new PaymentRequest());
        assertEquals("TXN002", result2.getTransactionId());
        
        // Third call should throw exception
        assertThrows(PaymentException.class, () -> 
            paymentService.processPayment(new PaymentRequest()));
    }
    
    @Test
    void testCustomAnswer() {
        PaymentService paymentService = mock(PaymentService.class);
        
        // Custom answer that modifies the behavior based on invocation
        when(paymentService.processPayment(any(PaymentRequest.class)))
            .thenAnswer(new Answer<PaymentResult>() {
                private int callCount = 0;
                
                @Override
                public PaymentResult answer(InvocationOnMock invocation) throws Throwable {
                    PaymentRequest request = invocation.getArgument(0);
                    callCount++;
                    
                    if (callCount % 2 == 0) {
                        // Every even call returns a special result
                        return PaymentResult.success("EVEN" + callCount, "Even call result");
                    } else {
                        // Odd calls return normal result
                        return PaymentResult.success("ODD" + callCount, 
                                                   "Amount: " + request.getAmount());
                    }
                }
            });
        
        // Test the custom behavior
        PaymentResult result1 = paymentService.processPayment(
            new PaymentRequest(BigDecimal.valueOf(100)));
        assertEquals("ODD1", result1.getTransactionId());
        
        PaymentResult result2 = paymentService.processPayment(
            new PaymentRequest(BigDecimal.valueOf(200)));
        assertEquals("EVEN2", result2.getTransactionId());
    }
    
    @Test
    void testMockingFinalClasses() {
        // Mock final class with Mockito extension
        PaymentGateway finalClassMock = mock(PaymentGateway.class);
        
        when(finalClassMock.process(any(BigDecimal.class)))
            .thenReturn("MOCKED_RESULT");
        
        String result = finalClassMock.process(BigDecimal.valueOf(100));
        assertEquals("MOCKED_RESULT", result);
    }
    
    @Test
    void testMockingStaticMethods() {
        // Mock static methods (requires Mockito 3.4+)
        try (MockedStatic<UtilityClass> mockedStatic = mockStatic(UtilityClass.class)) {
            mockedStatic.when(() -> UtilityClass.generateId())
                       .thenReturn("MOCKED_ID");
            
            String id = UtilityClass.generateId();
            assertEquals("MOCKED_ID", id);
            
            // Verify static method was called
            mockedStatic.verify(() -> UtilityClass.generateId());
        }
    }
    
    @Test
    void testMockingConstructor() {
        // Mock constructor using AdditionalAnswers
        PaymentService paymentService = mock(PaymentService.class, 
            withSettings().defaultAnswer(RETURNS_SMART_NULLS));
        
        // Use RETURNS_SMART_NULLS to prevent NullPointerExceptions
        when(paymentService.processPayment(any(PaymentRequest.class)))
            .thenReturn(null);
        
        PaymentResult result = paymentService.processPayment(new PaymentRequest());
        assertNull(result);
    }
    
    @Test
    void testDeepStubbing() {
        // Deep stubbing - mocking nested method calls
        PaymentService paymentService = mock(PaymentService.class, 
            withSettings().defaultAnswer(RETURNS_DEEP_STUBS));
        
        when(paymentService.processPayment(any(PaymentRequest.class))
            .getTransactionStatus().getCode())
            .thenReturn("SUCCESS");
        
        String status = paymentService.processPayment(new PaymentRequest())
                                    .getTransactionStatus().getCode();
        assertEquals("SUCCESS", status);
    }
}

// Mock classes for testing
class OrderService {
    private final PaymentService paymentService;
    private final UserService userService;
    private final EmailService emailService;
    private final NotificationService notificationService;
    
    public OrderService(PaymentService paymentService, UserService userService,
                       EmailService emailService, NotificationService notificationService) {
        this.paymentService = paymentService;
        this.userService = userService;
        this.emailService = emailService;
        this.notificationService = notificationService;
    }
    
    public OrderResult processOrder(Order order) {
        // Process payment
        PaymentRequest paymentRequest = new PaymentRequest(order.getAmount());
        PaymentResult paymentResult = paymentService.processPayment(paymentRequest);
        
        // Send confirmation email
        emailService.sendEmail(order.getUser().getEmail(), 
                              "Order Confirmation: " + order.getId());
        
        // Send notifications
        notificationService.sendNotification(
            new Notification("order", "Order created: " + order.getId()));
        notificationService.sendNotification(
            new Notification("payment", "Payment processed: " + paymentResult.getTransactionId()));
        
        return new OrderResult(order.getId(), paymentResult);
    }
}

class PaymentService {
    public PaymentResult processPayment(PaymentRequest request) {
        // Real implementation
        return PaymentResult.success("TXN001", "Payment processed");
    }
    
    public BigDecimal calculateFees(BigDecimal amount) {
        return amount.multiply(new BigDecimal("0.03")); // 3% fee
    }
}

class UserService {
    public Optional<User> findById(Long id) {
        return Optional.empty();
    }
}

class EmailService {
    public void sendEmail(String to, String subject) {
        // Real implementation
    }
}

class NotificationService {
    public void sendNotification(Notification notification) {
        // Real implementation
    }
}

class PaymentRequest {
    private BigDecimal amount;
    
    public PaymentRequest() {}
    
    public PaymentRequest(BigDecimal amount) {
        this.amount = amount;
    }
    
    public BigDecimal getAmount() { return amount; }
}

class PaymentResult {
    private String transactionId;
    private String message;
    private TransactionStatus transactionStatus;
    
    public static PaymentResult success(String transactionId, String message) {
        PaymentResult result = new PaymentResult();
        result.transactionId = transactionId;
        result.message = message;
        result.transactionStatus = new TransactionStatus("SUCCESS");
        return result;
    }
    
    // getters
    public String getTransactionId() { return transactionId; }
    public String getMessage() { return message; }
    public TransactionStatus getTransactionStatus() { return transactionStatus; }
}

class TransactionStatus {
    private String code;
    
    public TransactionStatus(String code) {
        this.code = code;
    }
    
    public String getCode() { return code; }
}

class Order {
    private Long id;
    private User user;
    private BigDecimal amount;
    
    public Order(Long id, User user, BigDecimal amount) {
        this.id = id;
        this.user = user;
        this.amount = amount;
    }
    
    // getters
    public Long getId() { return id; }
    public User getUser() { return user; }
    public BigDecimal getAmount() { return amount; }
}

class OrderResult {
    private Long orderId;
    private PaymentResult paymentResult;
    
    public OrderResult(Long orderId, PaymentResult paymentResult) {
        this.orderId = orderId;
        this.paymentResult = paymentResult;
    }
    
    // getters
    public Long getOrderId() { return orderId; }
    public PaymentResult getPaymentResult() { return paymentResult; }
}

class User {
    private String username;
    private String email;
    
    public User(String username, String email) {
        this.username = username;
        this.email = email;
    }
    
    // getters
    public String getUsername() { return username; }
    public String getEmail() { return email; }
}

class Notification {
    private String type;
    private String message;
    
    public Notification(String type, String message) {
        this.type = type;
        this.message = message;
    }
    
    // getters
    public String getType() { return type; }
    public String getMessage() { return message; }
}

class PaymentException extends RuntimeException {
    public PaymentException(String message) {
        super(message);
    }
}

final class PaymentGateway {
    public String process(BigDecimal amount) {
        return "PROCESSED_" + amount;
    }
}

class UtilityClass {
    public static String generateId() {
        return UUID.randomUUID().toString();
    }
}
```

## Spring Testing Advanced Features

### Comprehensive Spring Test Configuration

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.boot.test.mock.mockito.SpyBean;
import org.springframework.test.context.TestPropertySource;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.jdbc.Sql;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.json.AutoConfigureJsonTesters;
import org.springframework.boot.test.autoconfigure.web.client.AutoConfigureWebClient;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.ResultActions;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import static org.mockito.ArgumentMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
@TestPropertySource(properties = {
    "spring.datasource.url=jdbc:h2:mem:testdb",
    "spring.jpa.hibernate.ddl-auto=create-drop"
})
@ActiveProfiles("test")
class AdvancedSpringTests {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private UserRepository userRepository;
    
    @MockBean
    private PaymentService paymentService;
    
    @SpyBean
    private UserService userService;
    
    @Test
    void testUserControllerWithMockedDependencies() throws Exception {
        // Mock external service
        when(paymentService.processPayment(any(PaymentRequest.class)))
            .thenReturn(PaymentResult.success("TXN123", "Payment successful"));
        
        // Prepare test data
        User testUser = new User("john", "john@example.com");
        testUser.setId(1L);
        when(userService.createUser(any(CreateUserRequest.class)))
            .thenReturn(testUser);
        
        // Perform test
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(asJsonString(new CreateUserRequest("john", "john@example.com", "password123"))))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1L))
            .andExpect(jsonPath("$.username").value("john"))
            .andExpect(jsonPath("$.email").value("john@example.com"));
        
        // Verify interactions
        verify(userService).createUser(any(CreateUserRequest.class));
        verify(paymentService, never()).processPayment(any(PaymentRequest.class));
    }
    
    @Test
    void testUserRetrieval() throws Exception {
        // Set up test data
        User user = new User("jane", "jane@example.com");
        user.setId(2L);
        userRepository.save(user);
        
        mockMvc.perform(get("/api/users/2"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(2L))
            .andExpect(jsonPath("$.username").value("jane"));
    }
    
    @Test
    @Sql(scripts = "/test-data.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
    @Sql(scripts = "/cleanup.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
    void testWithSqlScripts() throws Exception {
        mockMvc.perform(get("/api/users"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.length()").value(3)); // Expect 3 users from test-data.sql
    }
    
    private static String asJsonString(Object obj) {
        try {
            return new ObjectMapper().writeValueAsString(obj);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}

// Data JPA tests with repository testing
@DataJpaTest
@TestPropertySource(properties = {
    "spring.datasource.url=jdbc:h2:mem:testdb",
    "spring.jpa.hibernate.ddl-auto=create-drop"
})
class UserRepositoryTest {
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testFindByEmail() {
        // Given
        User user = new User("john", "john@example.com");
        entityManager.persistAndFlush(user);
        
        // When
        Optional<User> found = userRepository.findByEmail("john@example.com");
        
        // Then
        assertThat(found).isPresent();
        assertThat(found.get().getUsername()).isEqualTo("john");
    }
    
    @Test
    void testFindByUsernameContaining() {
        // Given
        User user1 = new User("john_doe", "john@example.com");
        User user2 = new User("jane_doe", "jane@example.com");
        User user3 = new User("bob_smith", "bob@example.com");
        entityManager.persist(user1);
        entityManager.persist(user2);
        entityManager.persist(user3);
        entityManager.flush();
        
        // When
        List<User> found = userRepository.findByUsernameContaining("doe");
        
        // Then
        assertThat(found).hasSize(2);
        assertThat(found).extracting(User::getUsername)
                        .containsExactlyInAnyOrder("john_doe", "jane_doe");
    }
}

// Web client testing
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebClient
class WebClientServiceTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @LocalServerPort
    private int port;
    
    @MockBean
    private UserService userService;
    
    @Test
    void testRestClientIntegration() {
        User mockUser = new User("test", "test@example.com");
        mockUser.setId(1L);
        when(userService.findById(1L)).thenReturn(Optional.of(mockUser));
        
        String url = "http://localhost:" + port + "/api/users/1";
        ResponseEntity<User> response = restTemplate.getForEntity(url, User.class);
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().getUsername()).isEqualTo("test");
    }
}

// JSON testing
@SpringBootTest
@AutoConfigureJsonTesters
class JsonSerializationTest {
    
    @Autowired
    private JacksonTester<User> json;
    
    @Test
    void testSerializeUser() throws Exception {
        User user = new User("john", "john@example.com");
        user.setId(1L);
        
        JsonContent<User> content = json.write(user);
        
        assertThat(content).extractingJsonPathNumberValue("@.id").isEqualTo(1);
        assertThat(content).extractingJsonPathStringValue("@.username").isEqualTo("john");
        assertThat(content).extractingJsonPathStringValue("@.email").isEqualTo("john@example.com");
    }
    
    @Test
    void testDeserializeUser() throws Exception {
        String content = "{\"id\": 1, \"username\": \"john\", \"email\": \"john@example.com\"}";
        
        User user = json.parse(content).getObject();
        
        assertThat(user.getId()).isEqualTo(1L);
        assertThat(user.getUsername()).isEqualTo("john");
        assertThat(user.getEmail()).isEqualTo("john@example.com");
    }
    
    @Test
    void testJsonValidation() throws Exception {
        User user = new User("john", "john@example.com");
        
        assertThat(json.write(user)).extractingJsonPathStringValue("@.username")
                                  .isEqualTo("john");
    }
}
```

## Contract Testing with Spring Cloud Contract

### Consumer-Driven Contract Testing

```java
// Consumer side contract test
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@ExtendWith(SpringExtension.class)
@AutoConfigureStubRunner(ids = "com.example:user-service:+:8080", 
                        workOffline = true)
class UserClientContractTest {
    
    @Autowired
    private StubFinder stubFinder;
    
    private UserClient userClient;
    
    @BeforeEach
    void setUp() {
        // Use stubbed service URL
        String stubUrl = stubFinder.findStubUrl("user-service").toString();
        this.userClient = new UserClient(stubUrl);
    }
    
    @Test
    void shouldGetUserById() {
        // When
        User user = userClient.getUserById(1L);
        
        // Then
        assertThat(user.getId()).isEqualTo(1L);
        assertThat(user.getUsername()).isEqualTo("john");
        assertThat(user.getEmail()).isEqualTo("john@example.com");
    }
    
    @Test
    void shouldCreateUser() {
        // Given
        CreateUserRequest request = new CreateUserRequest("jane", "jane@example.com", "password");
        
        // When
        User createdUser = userClient.createUser(request);
        
        // Then
        assertThat(createdUser.getUsername()).isEqualTo("jane");
        assertThat(createdUser.getEmail()).isEqualTo("jane@example.com");
        assertThat(createdUser.getId()).isNotNull();
    }
}

// Contract definition (as a test)
public class UserContractTest extends ContractTestBase {
    
    @Test
    @DisplayName("Should return user by ID")
    public void validateGetUserById() throws Exception {
        // Contract definition using Spring Cloud Contract DSL
        Contract contract = Contract.make {
            request {
                method 'GET'
                url '/api/users/1'
            }
            response {
                status 200
                body([
                    id: 1,
                    username: $(regex('[a-zA-Z0-9_]+')),
                    email: $(regex('[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}')),
                    createdDate: $(regex('\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}\\.\\d{3}Z'))
                ])
                headers {
                    contentType('application/json')
                }
            }
        }
        
        // Execute contract test
        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.username").value(matchesRegex("[a-zA-Z0-9_]+")))
            .andExpect(jsonPath("$.email").value(matchesRegex("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}")));
    }
    
    @Test
    @DisplayName("Should create user successfully")
    public void validateCreateUser() throws Exception {
        Contract contract = Contract.make {
            request {
                method 'POST'
                url '/api/users'
                body([
                    username: 'jane',
                    email: 'jane@example.com',
                    password: value(regex('[a-zA-Z0-9!@#$%^&*]{8,}'))
                ])
                headers {
                    contentType('application/json')
                }
            }
            response {
                status 201
                body([
                    id: $(anyNumber()),
                    username: 'jane',
                    email: 'jane@example.com',
                    createdDate: $(regex('\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}\\.\\d{3}Z'))
                ])
                headers {
                    contentType('application/json')
                    location($(regex('.*/api/users/\\d+')))
                }
            }
        }
        
        // Execute contract test
        CreateUserRequest request = new CreateUserRequest("jane", "jane@example.com", "password123");
        
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.username").value("jane"))
            .andExpect(jsonPath("$.email").value("jane@example.com"))
            .andExpect(header().string("Location", matchesRegex(".*/api/users/\\d+")));
    }
}

// Contract test base class
@SpringBootTest
@AutoConfigureMockMvc
@TestPropertySource(properties = {
    "spring.datasource.url=jdbc:h2:mem:testdb",
    "spring.jpa.hibernate.ddl-auto=create-drop"
})
public abstract class ContractTestBase {
    
    @Autowired
    protected MockMvc mockMvc;
    
    @Autowired
    protected ObjectMapper objectMapper;
    
    @MockBean
    protected UserService userService;
}
```

## Test-Driven Development (TDD) Patterns

### TDD Implementation Example

```java
// Start with the test (red phase)
class UserServiceTddTest {
    
    private UserService userService;
    private UserRepository userRepository;
    
    @BeforeEach
    void setUp() {
        userRepository = mock(UserRepository.class);
        userService = new UserService(userRepository, new PasswordEncoder());
    }
    
    @Test
    @DisplayName("Should create user with valid input")
    void shouldCreateUserWithValidInput() {
        // Given
        CreateUserRequest request = new CreateUserRequest("john", "john@example.com", "password123");
        User expectedUser = new User("john", "john@example.com");
        
        when(userRepository.save(any(User.class))).thenReturn(expectedUser);
        when(userRepository.existsByEmail("john@example.com")).thenReturn(false);
        when(userRepository.existsByUsername("john")).thenReturn(false);
        
        // When
        User result = userService.createUser(request);
        
        // Then
        assertThat(result.getUsername()).isEqualTo("john");
        assertThat(result.getEmail()).isEqualTo("john@example.com");
        verify(userRepository).save(any(User.class));
    }
    
    @Test
    @DisplayName("Should throw exception when email already exists")
    void shouldThrowExceptionWhenEmailAlreadyExists() {
        // Given
        CreateUserRequest request = new CreateUserRequest("john", "existing@example.com", "password123");
        when(userRepository.existsByEmail("existing@example.com")).thenReturn(true);
        
        // When & Then
        assertThatThrownBy(() -> userService.createUser(request))
            .isInstanceOf(BusinessValidationException.class)
            .hasMessageContaining("Email already exists");
    }
    
    @Test
    @DisplayName("Should validate user input")
    void shouldValidateUserInput() {
        // Given
        CreateUserRequest invalidRequest = new CreateUserRequest("", "invalid", "short");
        
        // When & Then
        assertThatThrownBy(() -> userService.createUser(invalidRequest))
            .isInstanceOf(ValidationException.class);
    }
    
    @Test
    @DisplayName("Should find user by email")
    void shouldFindUserByEmail() {
        // Given
        User expectedUser = new User("john", "john@example.com");
        when(userRepository.findByEmail("john@example.com")).thenReturn(Optional.of(expectedUser));
        
        // When
        Optional<User> result = userService.findByEmail("john@example.com");
        
        // Then
        assertThat(result).isPresent();
        assertThat(result.get().getUsername()).isEqualTo("john");
    }
    
    @Test
    @DisplayName("Should update user successfully")
    void shouldUpdateUserSuccessfully() {
        // Given
        User existingUser = new User("john", "john@example.com");
        existingUser.setId(1L);
        UpdateUserRequest updateRequest = new UpdateUserRequest("newemail@example.com", "Jane");
        
        when(userRepository.findById(1L)).thenReturn(Optional.of(existingUser));
        when(userRepository.save(any(User.class))).thenReturn(existingUser);
        
        // When
        User updatedUser = userService.updateUser(1L, updateRequest);
        
        // Then
        assertThat(updatedUser.getEmail()).isEqualTo("newemail@example.com");
        assertThat(updatedUser.getUsername()).isEqualTo("Jane");
    }
}

// Implementation following TDD (green phase)
@Service
@Transactional
public class UserService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    
    public UserService(UserRepository userRepository, PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }
    
    public User createUser(CreateUserRequest request) {
        // Validate input
        validateCreateUserRequest(request);
        
        // Check for existing user
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new BusinessValidationException("Email already exists");
        }
        
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new BusinessValidationException("Username already exists");
        }
        
        // Create and save user
        User user = new User(request.getUsername(), request.getEmail());
        user.setPassword(passwordEncoder.encode(request.getPassword()));
        user.setCreatedDate(LocalDateTime.now());
        
        return userRepository.save(user);
    }
    
    public Optional<User> findByEmail(String email) {
        return userRepository.findByEmail(email);
    }
    
    public User updateUser(Long userId, UpdateUserRequest request) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        if (request.getEmail() != null) {
            user.setEmail(request.getEmail());
        }
        
        if (request.getUsername() != null) {
            user.setUsername(request.getUsername());
        }
        
        user.setLastModifiedDate(LocalDateTime.now());
        return userRepository.save(user);
    }
    
    private void validateCreateUserRequest(CreateUserRequest request) {
        if (request.getUsername() == null || request.getUsername().trim().isEmpty()) {
            throw new ValidationException("Username is required");
        }
        
        if (request.getEmail() == null || !isValidEmail(request.getEmail())) {
            throw new ValidationException("Valid email is required");
        }
        
        if (request.getPassword() == null || request.getPassword().length() < 8) {
            throw new ValidationException("Password must be at least 8 characters");
        }
    }
    
    private boolean isValidEmail(String email) {
        return email != null && email.contains("@") && email.contains(".");
    }
}

// Refactor phase (improving the implementation)
@Service
@Transactional
@Validated
public class UserServiceRefactored {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final ValidationService validationService;
    
    public UserServiceRefactored(UserRepository userRepository, 
                               PasswordEncoder passwordEncoder,
                               ValidationService validationService) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
        this.validationService = validationService;
    }
    
    @Transactional(rollbackFor = Exception.class)
    public User createUser(@Valid CreateUserRequest request) {
        validationService.validateUserRequest(request);
        validationService.checkUserUniqueness(request.getEmail(), request.getUsername());
        
        User user = mapToUser(request);
        return userRepository.save(user);
    }
    
    @Transactional(readOnly = true)
    public Optional<User> findByEmail(String email) {
        return userRepository.findByEmail(email);
    }
    
    @Transactional
    public User updateUser(Long userId, @Valid UpdateUserRequest request) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        updateUserData(user, request);
        return userRepository.save(user);
    }
    
    private User mapToUser(CreateUserRequest request) {
        User user = new User(request.getUsername(), request.getEmail());
        user.setPassword(passwordEncoder.encode(request.getPassword()));
        user.setCreatedDate(LocalDateTime.now());
        user.setLastModifiedDate(LocalDateTime.now());
        return user;
    }
    
    private void updateUserData(User user, UpdateUserRequest request) {
        if (request.getEmail() != null) {
            user.setEmail(request.getEmail());
        }
        if (request.getUsername() != null) {
            user.setUsername(request.getUsername());
        }
        if (request.getFirstName() != null) {
            user.setFirstName(request.getFirstName());
        }
        if (request.getLastName() != null) {
            user.setLastName(request.getLastName());
        }
        user.setLastModifiedDate(LocalDateTime.now());
    }
}
```

## Performance and Load Testing

### JUnit-based Performance Tests

```java
@TestMethodOrder(OrderAnnotation.class)
class PerformanceTests {
    
    private UserService userService;
    private List<User> testUsers;
    
    @BeforeEach
    void setUp() {
        // Initialize service with in-memory database for performance testing
        userService = new UserService(mock(UserRepository.class), new PasswordEncoder());
        generateTestUsers(1000); // Generate test data
    }
    
    @Test
    @Order(1)
    @PerformanceTest
    void testUserCreationPerformance() {
        long startTime = System.nanoTime();
        int operations = 1000;
        
        for (int i = 0; i < operations; i++) {
            CreateUserRequest request = new CreateUserRequest(
                "user" + i, "user" + i + "@example.com", "password123");
            userService.createUser(request);
        }
        
        long endTime = System.nanoTime();
        long duration = (endTime - startTime) / 1_000_000; // Convert to milliseconds
        
        System.out.println("Created " + operations + " users in " + duration + " ms");
        assertTrue(duration < 5000, "User creation should take less than 5 seconds for 1000 users");
    }
    
    @Test
    @Order(2)
    @PerformanceTest
    void testUserRetrievalPerformance() {
        // First, create users for retrieval test
        createUsersForTesting();
        
        long startTime = System.nanoTime();
        int operations = 10000;
        
        for (int i = 0; i < operations; i++) {
            userService.findById((long) (i % 1000)); // Cycle through 1000 users
        }
        
        long endTime = System.nanoTime();
        long duration = (endTime - startTime) / 1_000_000;
        
        System.out.println("Retrieved " + operations + " users in " + duration + " ms");
        assertTrue(duration < 2000, "User retrieval should take less than 2 seconds for 10000 operations");
    }
    
    @Test
    @Order(3)
    @PerformanceTest
    void testConcurrentUserOperations() throws InterruptedException {
        int threadCount = 10;
        int operationsPerThread = 100;
        CountDownLatch latch = new CountDownLatch(threadCount);
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        
        long startTime = System.nanoTime();
        
        for (int i = 0; i < threadCount; i++) {
            final int threadId = i;
            executor.submit(() -> {
                try {
                    for (int j = 0; j < operationsPerThread; j++) {
                        CreateUserRequest request = new CreateUserRequest(
                            "thread" + threadId + "user" + j,
                            "thread" + threadId + "user" + j + "@example.com",
                            "password123"
                        );
                        userService.createUser(request);
                    }
                } finally {
                    latch.countDown();
                }
            });
        }
        
        latch.await();
        executor.shutdown();
        
        long endTime = System.nanoTime();
        long duration = (endTime - startTime) / 1_000_000;
        
        int totalOperations = threadCount * operationsPerThread;
        System.out.println("Concurrent operations: " + totalOperations + " in " + duration + " ms");
        
        assertTrue(duration < 10000, "Concurrent operations should complete in less than 10 seconds");
    }
    
    private void generateTestUsers(int count) {
        testUsers = new ArrayList<>();
        for (int i = 0; i < count; i++) {
            testUsers.add(new User("user" + i, "user" + i + "@example.com"));
        }
    }
    
    private void createUsersForTesting() {
        // Create users for retrieval testing
        for (User user : testUsers) {
            // Simulate user creation
        }
    }
}

// Custom performance test annotation
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Test
@interface PerformanceTest {
    long maxExecutionTimeMs() default 5000;
    int maxMemoryUsageMB() default 100;
}

// Memory usage monitoring
class MemoryUsageMonitor {
    
    public static long getCurrentMemoryUsage() {
        Runtime runtime = Runtime.getRuntime();
        return (runtime.totalMemory() - runtime.freeMemory()) / (1024 * 1024); // MB
    }
    
    public static void printMemoryUsage(String label) {
        long memory = getCurrentMemoryUsage();
        System.out.println(label + ": " + memory + " MB");
    }
    
    public static boolean isMemoryUsageAcceptable(long maxMB) {
        return getCurrentMemoryUsage() <= maxMB;
    }
}
```

## Conclusion

Advanced Testing Strategies and Mocking provide:
- Comprehensive test coverage with JUnit 5 advanced features
- Effective mocking techniques for isolating components
- Spring-specific testing utilities for different scenarios
- Contract testing for API consistency
- Test-driven development practices
- Performance testing methodologies

Mastering these techniques is essential for:
- Building reliable, maintainable software
- Ensuring code quality and preventing regressions
- Creating effective test suites for complex applications
- Implementing proper TDD practices
- Performance optimization and monitoring