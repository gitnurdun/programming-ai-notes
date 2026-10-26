# Java Annotations: Deep Dive into Metadata Programming

## Introduction

Java annotations are a form of syntactic metadata that can be added to Java code to provide additional information about program elements such as classes, methods, fields, parameters, and more. Introduced in Java 5, annotations have become fundamental to modern Java development, enabling features like compile-time checking, code generation, and runtime behavior modification.

This comprehensive guide explores Java annotations in depth, covering built-in annotations, custom annotation creation, annotation processing, and practical applications in modern Java frameworks.

## Understanding Annotations

### What Are Annotations?

Annotations are special markers in Java code that provide metadata about the program elements they annotate. They don't directly affect program execution but can be used by:

- Compilers to detect errors or suppress warnings
- Development tools to generate code or documentation
- Runtime systems to process annotated elements

### Basic Annotation Syntax

```java
// Simple annotation
@AnnotationName

// Annotation with elements
@AnnotationName(element1 = "value1", element2 = "value2")

// Annotation with single element (value element)
@AnnotationName("value")

// Multiple annotations
@Annotation1
@Annotation2
public class MyClass { }

// Java 8+ type annotations
List<@NonNull String> list;
```

## Built-in Annotations

### Standard Annotations

Java provides several built-in annotations that serve different purposes:

```java
import java.lang.annotation.*;
import java.util.*;

public class BuiltInAnnotations {
    // @Override - Indicates that a method is intended to override a superclass method
    static class Parent {
        public void doSomething() {
            System.out.println("Parent method");
        }
        
        @Deprecated(since = "2.0", forRemoval = true)
        public void oldMethod() {
            System.out.println("Deprecated method");
        }
    }
    
    static class Child extends Parent {
        @Override  // Compiler will verify this actually overrides a parent method
        public void doSomething() {
            System.out.println("Child method");
        }
        
        // This would cause a compilation error if uncommented
        // @Override
        // public void nonExistentMethod() { }
    }
    
    // @Deprecated - Marks a program element as deprecated
    @Deprecated(since = "1.5", forRemoval = true)
    public static void oldApi() {
        System.out.println("This API is deprecated");
    }
    
    // @SuppressWarnings - Tells the compiler to suppress specific warnings
    @SuppressWarnings({"unchecked", "deprecation"})
    public static void suppressWarningsExample() {
        List list = new ArrayList(); // Raw type warning
        list.add("item");
        oldApi(); // Deprecation warning
    }
    
    // @FunctionalInterface - Indicates that an interface is intended to be functional
    @FunctionalInterface
    public interface Calculator {
        int calculate(int a, int b);
        
        // Can have default methods
        default void printResult(int result) {
            System.out.println("Result: " + result);
        }
        
        // Can have static methods
        static Calculator getAdder() {
            return (a, b) -> a + b;
        }
        
        // Methods from Object class don't count toward abstract method count
        String toString();
    }
    
    // @SafeVarargs - Asserts that a method or constructor uses varargs parameters safely
    public static class VarargsExample {
        @SafeVarargs
        public final void processItems(List<String>... lists) {
            for (List<String> list : lists) {
                System.out.println("Processing list: " + list);
            }
        }
    }
    
    public static void main(String[] args) {
        System.out.println("=== Built-in Annotations Demo ===");
        
        Child child = new Child();
        child.doSomething();
        child.oldMethod();
        
        oldApi();
        
        suppressWarningsExample();
        
        Calculator adder = Calculator.getAdder();
        int result = adder.calculate(5, 3);
        adder.printResult(result);
        
        VarargsExample varargs = new VarargsExample();
        varargs.processItems(
            Arrays.asList("a", "b"),
            Arrays.asList("c", "d")
        );
    }
}
```

## Meta-Annotations

### Controlling Annotation Behavior

Meta-annotations are annotations applied to other annotations to control their behavior:

```java
import java.lang.annotation.*;
import java.lang.reflect.*;

// @Target - Specifies where an annotation can be applied
@Target({
    ElementType.TYPE,           // Class, interface, enum
    ElementType.METHOD,         // Method
    ElementType.FIELD,          // Field
    ElementType.PARAMETER,      // Parameter
    ElementType.CONSTRUCTOR,    // Constructor
    ElementType.LOCAL_VARIABLE, // Local variable
    ElementType.MODULE          // Module (Java 9+)
})
@interface CustomAnnotation {
    String value() default "";
    int priority() default 1;
}

// @Retention - Specifies how long the annotation should be retained
@Retention(RetentionPolicy.SOURCE) // Retained only at source level
@interface SourceLevelAnnotation {
    String value() default "";
}

@Retention(RetentionPolicy.CLASS) // Retained in class file (default)
@interface ClassLevelAnnotation {
    String value() default "";
}

@Retention(RetentionPolicy.RUNTIME) // Retained at runtime
@interface RuntimeLevelAnnotation {
    String value() default "";
}

// @Documented - Indicates that the annotation should be documented by javadoc
@Documented
@Retention(RetentionPolicy.RUNTIME)
@interface DocumentedAnnotation {
    String description() default "";
}

// @Inherited - Allows subclasses to inherit the annotation from their superclass
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@interface InheritableAnnotation {
    String value() default "";
}

// @Repeatable - Allows an annotation to be applied multiple times to the same element (Java 8+)
@Repeatable(Reviews.class)
@Retention(RetentionPolicy.RUNTIME)
@interface Review {
    String reviewer();
    String comment() default "";
    int rating();
}

@Retention(RetentionPolicy.RUNTIME)
@interface Reviews {
    Review[] value();
}

// Example classes demonstrating meta-annotations
@SourceLevelAnnotation("Source level")
@ClassLevelAnnotation("Class level")
@RuntimeLevelAnnotation("Runtime level")
@DocumentedAnnotation(description = "This class demonstrates meta-annotations")
@InheritableAnnotation("Inherited value")
class AnnotatedClass {
    @CustomAnnotation(value = "Field annotation", priority = 5)
    private String field;
    
    @Review(reviewer = "Alice", rating = 5, comment = "Excellent work")
    @Review(reviewer = "Bob", rating = 4, comment = "Good but needs improvement")
    public void annotatedMethod() {
        System.out.println("Method with reviews");
    }
}

// Subclass that inherits annotations
class SubClass extends AnnotatedClass {
    // @InheritableAnnotation is inherited
    // @RuntimeLevelAnnotation is inherited (from parent class)
}

public class MetaAnnotationsDemo {
    public static void main(String[] args) {
        System.out.println("=== Meta-Annotations Demo ===");
        
        Class<?> clazz = AnnotatedClass.class;
        
        // Demonstrate @Retention
        System.out.println("Annotations with SOURCE retention are not available at runtime");
        System.out.println("Annotations with CLASS retention: " + 
            Arrays.toString(clazz.getAnnotationsByType(ClassLevelAnnotation.class)));
        System.out.println("Annotations with RUNTIME retention: " + 
            Arrays.toString(clazz.getAnnotationsByType(RuntimeLevelAnnotation.class)));
        
        // Demonstrate @Target
        try {
            Field field = clazz.getDeclaredField("field");
            CustomAnnotation custom = field.getAnnotation(CustomAnnotation.class);
            if (custom != null) {
                System.out.println("Custom annotation on field: value=" + custom.value() + 
                                 ", priority=" + custom.priority());
            }
        } catch (NoSuchFieldException e) {
            System.out.println("Field not found: " + e.getMessage());
        }
        
        // Demonstrate @Inherited
        Class<?> subClazz = SubClass.class;
        InheritableAnnotation inherited = subClazz.getAnnotation(InheritableAnnotation.class);
        if (inherited != null) {
            System.out.println("Inherited annotation: " + inherited.value());
        }
        
        // Demonstrate @Repeatable
        try {
            Method method = clazz.getMethod("annotatedMethod");
            Review[] reviews = method.getAnnotationsByType(Review.class);
            System.out.println("Number of reviews: " + reviews.length);
            for (Review review : reviews) {
                System.out.println("  Review by " + review.reviewer() + 
                                 ": " + review.rating() + "/5 - " + review.comment());
            }
        } catch (NoSuchMethodException e) {
            System.out.println("Method not found: " + e.getMessage());
        }
        
        // Demonstrate @Documented
        DocumentedAnnotation documented = clazz.getAnnotation(DocumentedAnnotation.class);
        if (documented != null) {
            System.out.println("Documented annotation: " + documented.description());
        }
    }
}
```

## Creating Custom Annotations

### Defining Custom Annotations

```java
import java.lang.annotation.*;
import java.lang.reflect.*;
import java.util.*;

// Simple custom annotation
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Test {
    String value() default "Default test";
    boolean enabled() default true;
}

// Complex custom annotation with default values
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@interface Configuration {
    String name() default "";
    String[] profiles() default {};
    boolean required() default false;
    
    // Nested annotation
    @interface Property {
        String key();
        String value();
        boolean encrypted() default false;
    }
    
    Property[] properties() default {};
    
    // Enum element
    enum LogLevel {
        DEBUG, INFO, WARN, ERROR
    }
    
    LogLevel logLevel() default LogLevel.INFO;
}

// Annotation with array elements
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Authors {
    String[] value(); // Single element annotation with array
}

// Marker annotation (no elements)
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Marker {
    // Empty - just marks classes
}

// Annotation for validation
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Validate {
    String pattern() default ".*";
    int minLength() default 0;
    int maxLength() default Integer.MAX_VALUE;
    String message() default "Validation failed";
}

// Example classes using custom annotations
@Authors({"John Doe", "Jane Smith"})
@Configuration(
    name = "DatabaseConfig",
    profiles = {"dev", "test"},
    required = true,
    logLevel = Configuration.LogLevel.DEBUG,
    properties = {
        @Configuration.Property(key = "db.url", value = "jdbc:mysql://localhost:3306/test"),
        @Configuration.Property(key = "db.username", value = "user")
    }
)
@Marker
class DatabaseService {
    @Validate(pattern = "\\w+@\\w+\\.\\w+", message = "Invalid email format")
    private String email;
    
    @Validate(minLength = 5, maxLength = 20)
    private String username;
    
    @Test(value = "Database connection test", enabled = true)
    public boolean testConnection() {
        System.out.println("Testing database connection...");
        return true;
    }
    
    @Test("User creation test")
    public void testUserCreation() {
        System.out.println("Testing user creation...");
    }
    
    @Test(enabled = false)
    public void disabledTest() {
        System.out.println("This test is disabled");
    }
}

public class CustomAnnotationsDemo {
    public static void main(String[] args) {
        System.out.println("=== Custom Annotations Demo ===");
        
        Class<?> clazz = DatabaseService.class;
        
        // Process @Authors annotation
        Authors authors = clazz.getAnnotation(Authors.class);
        if (authors != null) {
            System.out.println("Authors: " + Arrays.toString(authors.value()));
        }
        
        // Process @Configuration annotation
        Configuration config = clazz.getAnnotation(Configuration.class);
        if (config != null) {
            System.out.println("Configuration:");
            System.out.println("  Name: " + config.name());
            System.out.println("  Profiles: " + Arrays.toString(config.profiles()));
            System.out.println("  Required: " + config.required());
            System.out.println("  Log level: " + config.logLevel());
            
            System.out.println("  Properties:");
            for (Configuration.Property prop : config.properties()) {
                System.out.println("    " + prop.key() + " = " + prop.value() + 
                                 (prop.encrypted() ? " (encrypted)" : ""));
            }
        }
        
        // Process @Marker annotation
        if (clazz.isAnnotationPresent(Marker.class)) {
            System.out.println("Class is marked");
        }
        
        // Process method annotations
        Method[] methods = clazz.getDeclaredMethods();
        for (Method method : methods) {
            Test test = method.getAnnotation(Test.class);
            if (test != null) {
                System.out.println("Method " + method.getName() + ":");
                System.out.println("  Test name: " + test.value());
                System.out.println("  Enabled: " + test.enabled());
            }
        }
        
        // Process field annotations
        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields) {
            Validate validate = field.getAnnotation(Validate.class);
            if (validate != null) {
                System.out.println("Field " + field.getName() + ":");
                System.out.println("  Pattern: " + validate.pattern());
                System.out.println("  Min length: " + validate.minLength());
                System.out.println("  Max length: " + validate.maxLength());
                System.out.println("  Message: " + validate.message());
            }
        }
    }
}
```

### Advanced Custom Annotation Features

```java
import java.lang.annotation.*;
import java.lang.reflect.*;
import java.util.*;

// Annotation with generic types and complex structures
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface ServiceDefinition {
    // Required elements
    String name();
    Class<?> serviceInterface();
    
    // Optional elements with defaults
    String version() default "1.0.0";
    String[] dependencies() default {};
    
    // Array of complex objects
    Endpoint[] endpoints() default {};
    
    // Nested annotation
    @interface Endpoint {
        String path();
        String method() default "GET";
        int timeout() default 30;
        boolean secured() default false;
    }
    
    // Enum for service types
    enum ServiceType {
        REST, SOAP, GRAPHQL, GRPC
    }
    
    ServiceType type() default ServiceType.REST;
    
    // Class reference element
    Class<? extends Exception> exceptionClass() default RuntimeException.class;
}

// Annotation for data mapping
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface DataMapping {
    String source();
    String target() default "";
    boolean required() default false;
    String defaultValue() default "";
    
    // Transformation function
    Class<? extends Function<String, Object>> transformer() default DefaultTransformer.class;
}

// Default transformer implementation
interface Function<T, R> {
    R apply(T t);
}

class DefaultTransformer implements Function<String, Object> {
    @Override
    public Object apply(String s) {
        return s;
    }
}

class IntegerTransformer implements Function<String, Object> {
    @Override
    public Object apply(String s) {
        try {
            return Integer.parseInt(s);
        } catch (NumberFormatException e) {
            return 0;
        }
    }
}

// Example service with complex annotation
@ServiceDefinition(
    name = "UserService",
    serviceInterface = UserServiceInterface.class,
    version = "2.1.0",
    dependencies = {"DatabaseService", "EmailService"},
    type = ServiceDefinition.ServiceType.REST,
    endpoints = {
        @ServiceDefinition.Endpoint(
            path = "/users",
            method = "POST",
            timeout = 60,
            secured = true
        ),
        @ServiceDefinition.Endpoint(
            path = "/users/{id}",
            method = "GET",
            timeout = 30
        )
    }
)
class UserService implements UserServiceInterface {
    @DataMapping(
        source = "user_id",
        target = "id",
        required = true
    )
    private String id;
    
    @DataMapping(
        source = "user_name",
        transformer = IntegerTransformer.class
    )
    private int age;
    
    @DataMapping(
        source = "user_email",
        defaultValue = "unknown@example.com"
    )
    private String email;
    
    // Getters and setters
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}

interface UserServiceInterface {
    void createUser(String userData);
    String getUser(String id);
}

// Annotation processor for service definitions
class ServiceAnnotationProcessor {
    public static void analyzeService(Class<?> serviceClass) {
        System.out.println("=== Analyzing Service: " + serviceClass.getSimpleName() + " ===");
        
        ServiceDefinition serviceDef = serviceClass.getAnnotation(ServiceDefinition.class);
        if (serviceDef != null) {
            System.out.println("Service Name: " + serviceDef.name());
            System.out.println("Interface: " + serviceDef.serviceInterface().getSimpleName());
            System.out.println("Version: " + serviceDef.version());
            System.out.println("Type: " + serviceDef.type());
            System.out.println("Dependencies: " + Arrays.toString(serviceDef.dependencies()));
            System.out.println("Exception Class: " + serviceDef.exceptionClass().getSimpleName());
            
            System.out.println("Endpoints:");
            for (ServiceDefinition.Endpoint endpoint : serviceDef.endpoints()) {
                System.out.println("  - " + endpoint.method() + " " + endpoint.path());
                System.out.println("    Timeout: " + endpoint.timeout() + "s");
                System.out.println("    Secured: " + endpoint.secured());
            }
        }
        
        System.out.println("\nField Mappings:");
        Field[] fields = serviceClass.getDeclaredFields();
        for (Field field : fields) {
            DataMapping mapping = field.getAnnotation(DataMapping.class);
            if (mapping != null) {
                System.out.println("  Field: " + field.getName());
                System.out.println("    Source: " + mapping.source());
                System.out.println("    Target: " + (mapping.target().isEmpty() ? field.getName() : mapping.target()));
                System.out.println("    Required: " + mapping.required());
                System.out.println("    Default: " + mapping.defaultValue());
                System.out.println("    Transformer: " + mapping.transformer().getSimpleName());
            }
        }
    }
    
    public static Object transformValue(String sourceValue, Class<? extends Function<String, Object>> transformerClass) {
        try {
            Function<String, Object> transformer = transformerClass.getDeclaredConstructor().newInstance();
            return transformer.apply(sourceValue);
        } catch (Exception e) {
            System.err.println("Error creating transformer: " + e.getMessage());
            return sourceValue;
        }
    }
}

public class AdvancedAnnotationsDemo {
    public static void main(String[] args) {
        System.out.println("=== Advanced Annotations Demo ===");
        
        // Analyze service with complex annotations
        ServiceAnnotationProcessor.analyzeService(UserService.class);
        
        System.out.println("\n=== Value Transformation Demo ===");
        
        // Demonstrate value transformation
        String ageString = "25";
        Object transformedAge = ServiceAnnotationProcessor.transformValue(
            ageString, IntegerTransformer.class);
        System.out.println("Transformed age: " + ageString + " -> " + transformedAge);
        
        String nameString = "John";
        Object transformedName = ServiceAnnotationProcessor.transformValue(
            nameString, DefaultTransformer.class);
        System.out.println("Transformed name: " + nameString + " -> " + transformedName);
        
        System.out.println("\n=== Runtime Annotation Processing ===");
        
        // Process annotations at runtime
        processAnnotationsAtRuntime();
    }
    
    private static void processAnnotationsAtRuntime() {
        Class<?> clazz = UserService.class;
        
        // Get all annotations on the class
        Annotation[] classAnnotations = clazz.getAnnotations();
        System.out.println("Class annotations:");
        for (Annotation ann : classAnnotations) {
            System.out.println("  " + ann.annotationType().getSimpleName());
        }
        
        // Get specific annotation type
        ServiceDefinition serviceDef = clazz.getAnnotation(ServiceDefinition.class);
        if (serviceDef != null) {
            System.out.println("\nService Definition Details:");
            System.out.println("  Name: " + serviceDef.name());
            System.out.println("  Type: " + serviceDef.type());
            
            // Get annotation elements as a map (Java 8+)
            System.out.println("\nAnnotation elements:");
            try {
                Method[] methods = serviceDef.annotationType().getDeclaredMethods();
                for (Method method : methods) {
                    Object value = method.invoke(serviceDef);
                    System.out.println("  " + method.getName() + ": " + value);
                }
            } catch (Exception e) {
                System.err.println("Error processing annotation elements: " + e.getMessage());
            }
        }
        
        // Process repeated annotations (if any)
        System.out.println("\nProcessing repeated annotations:");
        // This would be used with @Repeatable annotations
    }
}
```

## Annotation Processing

### Runtime Annotation Processing

```java
import java.lang.annotation.*;
import java.lang.reflect.*;
import java.util.*;

// Annotations for a simple validation framework
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface NotNull {
    String message() default "Field cannot be null";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Size {
    int min() default 0;
    int max() default Integer.MAX_VALUE;
    String message() default "Size constraint violated";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Email {
    String message() default "Invalid email format";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Range {
    long min() default Long.MIN_VALUE;
    long max() default Long.MAX_VALUE;
    String message() default "Value out of range";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Validated {
    // Marker annotation for classes that should be validated
}

// Example class with validation annotations
@Validated
class User {
    @NotNull(message = "Name is required")
    @Size(min = 2, max = 50, message = "Name must be between 2 and 50 characters")
    private String name;
    
    @Email(message = "Please provide a valid email address")
    private String email;
    
    @Range(min = 18, max = 120, message = "Age must be between 18 and 120")
    private int age;
    
    @Size(max = 1000, message = "Description is too long")
    private String description;
    
    public User(String name, String email, int age, String description) {
        this.name = name;
        this.email = email;
        this.age = age;
        this.description = description;
    }
    
    // Getters and setters
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    
    @Override
    public String toString() {
        return "User{name='" + name + "', email='" + email + "', age=" + age + 
               ", description='" + description + "'}";
    }
}

// Validation framework
class ValidationFramework {
    public static class ValidationError {
        private final String field;
        private final String message;
        
        public ValidationError(String field, String message) {
            this.field = field;
            this.message = message;
        }
        
        public String getField() { return field; }
        public String getMessage() { return message; }
        
        @Override
        public String toString() {
            return field + ": " + message;
        }
    }
    
    public static List<ValidationError> validate(Object object) {
        List<ValidationError> errors = new ArrayList<>();
        
        // Check if class is annotated with @Validated
        if (!object.getClass().isAnnotationPresent(Validated.class)) {
            return errors; // Skip validation if not annotated
        }
        
        Field[] fields = object.getClass().getDeclaredFields();
        
        for (Field field : fields) {
            field.setAccessible(true);
            
            try {
                Object value = field.get(object);
                
                // Process @NotNull
                if (field.isAnnotationPresent(NotNull.class)) {
                    NotNull notNull = field.getAnnotation(NotNull.class);
                    if (value == null) {
                        errors.add(new ValidationError(field.getName(), notNull.message()));
                    }
                }
                
                // Process @Size
                if (field.isAnnotationPresent(Size.class)) {
                    Size size = field.getAnnotation(Size.class);
                    if (value instanceof String) {
                        String strValue = (String) value;
                        if (strValue.length() < size.min() || strValue.length() > size.max()) {
                            errors.add(new ValidationError(field.getName(), size.message()));
                        }
                    }
                }
                
                // Process @Email
                if (field.isAnnotationPresent(Email.class)) {
                    Email email = field.getAnnotation(Email.class);
                    if (value instanceof String) {
                        String emailValue = (String) value;
                        if (!isValidEmail(emailValue)) {
                            errors.add(new ValidationError(field.getName(), email.message()));
                        }
                    }
                }
                
                // Process @Range
                if (field.isAnnotationPresent(Range.class)) {
                    Range range = field.getAnnotation(Range.class);
                    if (value instanceof Number) {
                        long numValue = ((Number) value).longValue();
                        if (numValue < range.min() || numValue > range.max()) {
                            errors.add(new ValidationError(field.getName(), range.message()));
                        }
                    }
                }
                
            } catch (IllegalAccessException e) {
                errors.add(new ValidationError(field.getName(), "Cannot access field: " + e.getMessage()));
            }
        }
        
        return errors;
    }
    
    private static boolean isValidEmail(String email) {
        return email != null && email.contains("@") && email.contains(".");
    }
    
    public static boolean isValid(Object object) {
        return validate(object).isEmpty();
    }
    
    public static void printValidationErrors(List<ValidationError> errors) {
        if (errors.isEmpty()) {
            System.out.println("Validation passed!");
        } else {
            System.out.println("Validation errors:");
            for (ValidationError error : errors) {
                System.out.println("  " + error);
            }
        }
    }
}

// Annotation-based configuration processor
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface ConfigurationProperties {
    String prefix() default "";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Property {
    String value();
    String defaultValue() default "";
}

@ConfigurationProperties(prefix = "app.database")
class DatabaseConfig {
    @Property(value = "url", defaultValue = "jdbc:h2:mem:testdb")
    private String url;
    
    @Property(value = "username", defaultValue = "sa")
    private String username;
    
    @Property(value = "password")
    private String password;
    
    @Property(value = "pool.size", defaultValue = "10")
    private int poolSize;
    
    // Getters and setters
    public String getUrl() { return url; }
    public void setUrl(String url) { this.url = url; }
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
    public int getPoolSize() { return poolSize; }
    public void setPoolSize(int poolSize) { this.poolSize = poolSize; }
    
    @Override
    public String toString() {
        return "DatabaseConfig{url='" + url + "', username='" + username + 
               "', password='***', poolSize=" + poolSize + "}";
    }
}

class ConfigurationProcessor {
    public static void loadConfiguration(Object config, Map<String, String> properties) {
        Class<?> clazz = config.getClass();
        
        if (!clazz.isAnnotationPresent(ConfigurationProperties.class)) {
            System.out.println("Class is not annotated with @ConfigurationProperties");
            return;
        }
        
        ConfigurationProperties configProps = clazz.getAnnotation(ConfigurationProperties.class);
        String prefix = configProps.prefix();
        
        Field[] fields = clazz.getDeclaredFields();
        
        for (Field field : fields) {
            if (field.isAnnotationPresent(Property.class)) {
                Property prop = field.getAnnotation(Property.class);
                String key = prefix.isEmpty() ? prop.value() : prefix + "." + prop.value();
                String value = properties.get(key);
                
                if (value == null || value.isEmpty()) {
                    value = prop.defaultValue();
                }
                
                if (!value.isEmpty()) {
                    setFieldValue(config, field, value);
                }
            }
        }
    }
    
    private static void setFieldValue(Object obj, Field field, String value) {
        field.setAccessible(true);
        try {
            Class<?> fieldType = field.getType();
            if (fieldType == String.class) {
                field.set(obj, value);
            } else if (fieldType == int.class || fieldType == Integer.class) {
                field.set(obj, Integer.parseInt(value));
            } else if (fieldType == long.class || fieldType == Long.class) {
                field.set(obj, Long.parseLong(value));
            } else if (fieldType == boolean.class || fieldType == Boolean.class) {
                field.set(obj, Boolean.parseBoolean(value));
            } else if (fieldType == double.class || fieldType == Double.class) {
                field.set(obj, Double.parseDouble(value));
            } else {
                field.set(obj, value); // Default to string
            }
            System.out.println("Set " + field.getName() + " = " + value);
        } catch (Exception e) {
            System.err.println("Error setting field " + field.getName() + ": " + e.getMessage());
        }
    }
}

public class RuntimeAnnotationProcessing {
    public static void main(String[] args) {
        System.out.println("=== Runtime Annotation Processing ===");
        
        // Demonstrate validation framework
        demonstrateValidation();
        
        System.out.println("\n" + "=".repeat(50));
        
        // Demonstrate configuration processing
        demonstrateConfigurationProcessing();
    }
    
    private static void demonstrateValidation() {
        System.out.println("=== Validation Framework Demo ===");
        
        // Valid user
        User validUser = new User("John Doe", "john@example.com", 25, "Software Developer");
        List<ValidationFramework.ValidationError> validErrors = ValidationFramework.validate(validUser);
        System.out.println("Valid user: " + validUser);
        ValidationFramework.printValidationErrors(validErrors);
        
        System.out.println();
        
        // Invalid user
        User invalidUser = new User("", "invalid-email", 15, "A".repeat(1001));
        List<ValidationFramework.ValidationError> invalidErrors = ValidationFramework.validate(invalidUser);
        System.out.println("Invalid user: " + invalidUser);
        ValidationFramework.printValidationErrors(invalidErrors);
        
        System.out.println();
        
        // User with null values
        User nullUser = new User(null, null, 0, null);
        List<ValidationFramework.ValidationError> nullErrors = ValidationFramework.validate(nullUser);
        System.out.println("Null user: " + nullUser);
        ValidationFramework.printValidationErrors(nullErrors);
    }
    
    private static void demonstrateConfigurationProcessing() {
        System.out.println("=== Configuration Processing Demo ===");
        
        DatabaseConfig config = new DatabaseConfig();
        System.out.println("Before configuration: " + config);
        
        // Simulate properties from a configuration file
        Map<String, String> properties = new HashMap<>();
        properties.put("app.database.url", "jdbc:postgresql://localhost:5432/mydb");
        properties.put("app.database.username", "myuser");
        properties.put("app.database.password", "mypassword");
        properties.put("app.database.pool.size", "20");
        
        ConfigurationProcessor.loadConfiguration(config, properties);
        System.out.println("After configuration: " + config);
        
        System.out.println("\nProcessing with missing properties (using defaults):");
        DatabaseConfig defaultConfig = new DatabaseConfig();
        Map<String, String> minimalProperties = new HashMap<>();
        minimalProperties.put("app.database.username", "testuser");
        // Other properties will use defaults
        
        ConfigurationProcessor.loadConfiguration(defaultConfig, minimalProperties);
        System.out.println("With defaults: " + defaultConfig);
    }
}
```

## Practical Applications

### Framework Integration Examples

```java
import java.lang.annotation.*;
import java.lang.reflect.*;
import java.util.*;
import java.util.concurrent.*;

// Annotations for a simple web framework
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface WebController {
    String basePath() default "";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface RequestMapping {
    String value();
    RequestMethod method() default RequestMethod.GET;
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
@interface RequestParam {
    String value();
    boolean required() default true;
    String defaultValue() default "";
}

enum RequestMethod {
    GET, POST, PUT, DELETE, PATCH
}

// Example web controller
@WebController(basePath = "/api/users")
class UserController {
    @RequestMapping(value = "/", method = RequestMethod.GET)
    public String listUsers(@RequestParam(value = "page", defaultValue = "1") int page,
                           @RequestParam(value = "size", defaultValue = "10") int size) {
        return "Listing users - page: " + page + ", size: " + size;
    }
    
    @RequestMapping(value = "/{id}", method = RequestMethod.GET)
    public String getUser(@RequestParam("id") String id) {
        return "Getting user with id: " + id;
    }
    
    @RequestMapping(value = "/", method = RequestMethod.POST)
    public String createUser(@RequestParam("name") String name,
                            @RequestParam("email") String email) {
        return "Creating user: " + name + " (" + email + ")";
    }
}

// Simple dependency injection annotations
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Autowired {
    String value() default "";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Service {
    String value() default "";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Repository {
    String value() default "";
}

// Example services with dependency injection
@Service("userService")
class UserServiceImpl {
    @Autowired("userRepository")
    private UserRepository repository;
    
    public String getUser(String id) {
        return "User " + id + " from " + (repository != null ? "repository" : "default source");
    }
}

@Repository("userRepository")
class UserRepository {
    public String findUser(String id) {
        return "Found user " + id;
    }
}

// Transaction management annotations
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Transactional {
    String value() default "";
    boolean readOnly() default false;
    Class<? extends Throwable>[] rollbackFor() default {};
    Class<? extends Throwable>[] noRollbackFor() default {};
}

// Example service with transactions
class AccountService {
    @Transactional(rollbackFor = Exception.class)
    public void transfer(String fromAccount, String toAccount, double amount) {
        System.out.println("Transferring " + amount + " from " + fromAccount + " to " + toAccount);
        // Transaction logic here
    }
    
    @Transactional(readOnly = true)
    public double getBalance(String account) {
        System.out.println("Getting balance for " + account);
        return 1000.0; // Simulated balance
    }
}

// Security annotations
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@interface Secured {
    String[] roles() default {};
    String[] permissions() default {};
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
@interface CurrentUser {
    // Marker for current user parameter
}

// Example secured controller
@Secured(roles = {"ADMIN", "USER"})
class AdminController {
    @RequestMapping(value = "/admin/users", method = RequestMethod.GET)
    @Secured(roles = {"ADMIN"})
    public String listAllUsers(@CurrentUser String currentUser) {
        return "Listing all users for admin: " + currentUser;
    }
    
    @RequestMapping(value = "/admin/settings", method = RequestMethod.POST)
    @Secured(roles = {"ADMIN"}, permissions = {"WRITE_SETTINGS"})
    public String updateSettings(@RequestParam("setting") String setting) {
        return "Updated setting: " + setting;
    }
}

// Annotation processor for web framework
class WebFrameworkProcessor {
    public static class RouteInfo {
        private final String path;
        private final RequestMethod method;
        private final Method handlerMethod;
        private final List<ParameterInfo> parameters;
        
        public RouteInfo(String path, RequestMethod method, Method handlerMethod) {
            this.path = path;
            this.method = method;
            this.handlerMethod = handlerMethod;
            this.parameters = extractParameterInfo(handlerMethod);
        }
        
        private List<ParameterInfo> extractParameterInfo(Method method) {
            List<ParameterInfo> params = new ArrayList<>();
            Parameter[] methodParams = method.getParameters();
            
            for (int i = 0; i < methodParams.length; i++) {
                Parameter param = methodParams[i];
                if (param.isAnnotationPresent(RequestParam.class)) {
                    RequestParam requestParam = param.getAnnotation(RequestParam.class);
                    params.add(new ParameterInfo(i, requestParam.value(), requestParam.required(), 
                                              requestParam.defaultValue(), param.getType()));
                }
            }
            return params;
        }
        
        // Getters
        public String getPath() { return path; }
        public RequestMethod getMethod() { return method; }
        public Method getHandlerMethod() { return handlerMethod; }
        public List<ParameterInfo> getParameters() { return parameters; }
        
        @Override
        public String toString() {
            return method + " " + path + " -> " + handlerMethod.getName() + 
                   " (" + parameters.size() + " params)";
        }
    }
    
    public static class ParameterInfo {
        private final int index;
        private final String name;
        private final boolean required;
        private final String defaultValue;
        private final Class<?> type;
        
        public ParameterInfo(int index, String name, boolean required, String defaultValue, Class<?> type) {
            this.index = index;
            this.name = name;
            this.required = required;
            this.defaultValue = defaultValue;
            this.type = type;
        }
        
        // Getters
        public int getIndex() { return index; }
        public String getName() { return name; }
        public boolean isRequired() { return required; }
        public String getDefaultValue() { return defaultValue; }
        public Class<?> getType() { return type; }
    }
    
    public static Map<String, RouteInfo> processController(Class<?> controllerClass) {
        Map<String, RouteInfo> routes = new HashMap<>();
        
        if (!controllerClass.isAnnotationPresent(WebController.class)) {
            return routes;
        }
        
        WebController webController = controllerClass.getAnnotation(WebController.class);
        String basePath = webController.basePath();
        
        Method[] methods = controllerClass.getDeclaredMethods();
        for (Method method : methods) {
            if (method.isAnnotationPresent(RequestMapping.class)) {
                RequestMapping mapping = method.getAnnotation(RequestMapping.class);
                String fullPath = basePath + mapping.value();
                
                RouteInfo routeInfo = new RouteInfo(fullPath, mapping.method(), method);
                String routeKey = mapping.method() + ":" + fullPath;
                routes.put(routeKey, routeInfo);
                
                System.out.println("Registered route: " + routeInfo);
            }
        }
        
        return routes;
    }
    
    public static void processDependencies(Object target, Map<String, Object> beanRegistry) {
        Class<?> clazz = target.getClass();
        Field[] fields = clazz.getDeclaredFields();
        
        for (Field field : fields) {
            if (field.isAnnotationPresent(Autowired.class)) {
                Autowired autowired = field.getAnnotation(Autowired.class);
                String beanName = autowired.value();
                if (beanName.isEmpty()) {
                    beanName = field.getType().getSimpleName();
                    // Convert first letter to lowercase
                    beanName = Character.toLowerCase(beanName.charAt(0)) + beanName.substring(1);
                }
                
                Object dependency = beanRegistry.get(beanName);
                if (dependency != null) {
                    field.setAccessible(true);
                    try {
                        field.set(target, dependency);
                        System.out.println("Autowired " + beanName + " into " + 
                                         clazz.getSimpleName() + "." + field.getName());
                    } catch (IllegalAccessException e) {
                        System.err.println("Failed to autowire " + beanName + ": " + e.getMessage());
                    }
                } else {
                    System.out.println("Warning: No bean found for " + beanName);
                }
            }
        }
    }
}

public class FrameworkIntegrationDemo {
    public static void main(String[] args) {
        System.out.println("=== Framework Integration Demo ===");
        
        // Process web controller routes
        System.out.println("=== Web Framework Processing ===");
        Map<String, WebFrameworkProcessor.RouteInfo> routes = 
            WebFrameworkProcessor.processController(UserController.class);
        
        System.out.println("\nDiscovered routes:");
        routes.forEach((key, route) -> System.out.println("  " + key + " -> " + route));
        
        System.out.println("\n=== Dependency Injection Processing ===");
        
        // Set up bean registry
        Map<String, Object> beanRegistry = new HashMap<>();
        UserRepository userRepository = new UserRepository();
        UserServiceImpl userService = new UserServiceImpl();
        
        beanRegistry.put("userRepository", userRepository);
        beanRegistry.put("userService", userService);
        
        // Process dependencies
        WebFrameworkProcessor.processDependencies(userService, beanRegistry);
        
        // Test the injected dependency
        System.out.println("\nTesting dependency injection:");
        System.out.println(userService.getUser("123"));
        
        System.out.println("\n=== Security Annotation Processing ===");
        
        // Check security annotations
        checkSecurityAnnotations(AdminController.class);
        
        System.out.println("\n=== Transaction Annotation Processing ===");
        
        // Process transaction annotations
        processTransactionAnnotations();
    }
    
    private static void checkSecurityAnnotations(Class<?> clazz) {
        if (clazz.isAnnotationPresent(Secured.class)) {
            Secured classSecured = clazz.getAnnotation(Secured.class);
            System.out.println("Class " + clazz.getSimpleName() + " secured with roles: " + 
                             Arrays.toString(classSecured.roles()));
        }
        
        Method[] methods = clazz.getDeclaredMethods();
        for (Method method : methods) {
            if (method.isAnnotationPresent(Secured.class)) {
                Secured methodSecured = method.getAnnotation(Secured.class);
                System.out.println("Method " + method.getName() + " secured with:");
                System.out.println("  Roles: " + Arrays.toString(methodSecured.roles()));
                System.out.println("  Permissions: " + Arrays.toString(methodSecured.permissions()));
            }
            
            Parameter[] parameters = method.getParameters();
            for (int i = 0; i < parameters.length; i++) {
                Parameter param = parameters[i];
                if (param.isAnnotationPresent(CurrentUser.class)) {
                    System.out.println("Method " + method.getName() + " parameter " + i + 
                                     " is annotated with @CurrentUser");
                }
            }
        }
    }
    
    private static void processTransactionAnnotations() {
        AccountService accountService = new AccountService();
        Class<?> clazz = accountService.getClass();
        
        Method[] methods = clazz.getDeclaredMethods();
        for (Method method : methods) {
            if (method.isAnnotationPresent(Transactional.class)) {
                Transactional transactional = method.getAnnotation(Transactional.class);
                System.out.println("Method " + method.getName() + " is transactional:");
                System.out.println("  Read-only: " + transactional.readOnly());
                System.out.println("  Rollback for: " + Arrays.toString(transactional.rollbackFor()));
                System.out.println("  No rollback for: " + Arrays.toString(transactional.noRollbackFor()));
            }
        }
    }
}
```

## Performance and Best Practices

### Annotation Performance Considerations

```java
import java.lang.annotation.*;
import java.lang.reflect.*;
import java.util.*;
import java.util.concurrent.*;

// Performance testing annotations
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface PerformanceTest {
    int iterations() default 1000;
    boolean enabled() default true;
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Benchmark {
    String category() default "general";
    int warmupIterations() default 100;
}

// Example class with performance annotations
class PerformanceTestClass {
    @PerformanceTest(iterations = 5000, enabled = true)
    @Benchmark(category = "math", warmupIterations = 100)
    public int calculateSum(int n) {
        int sum = 0;
        for (int i = 1; i <= n; i++) {
            sum += i;
        }
        return sum;
    }
    
    @PerformanceTest(iterations = 10000, enabled = false)
    @Benchmark(category = "string", warmupIterations = 50)
    public String processString(String input) {
        return input.toUpperCase().toLowerCase();
    }
    
    @Benchmark(category = "io")
    public void doNothing() {
        // Empty method for testing
    }
}

// Annotation processor with caching for performance
class CachedAnnotationProcessor {
    private final Map<Class<?>, Map<String, Annotation>> annotationCache = 
        new ConcurrentHashMap<>();
    
    private final Map<Method, Map<String, Annotation>> methodAnnotationCache = 
        new ConcurrentHashMap<>();
    
    public <T extends Annotation> T getClassAnnotation(Class<?> clazz, Class<T> annotationClass) {
        String cacheKey = annotationClass.getName();
        Map<String, Annotation> classCache = annotationCache.computeIfAbsent(clazz, k -> 
            new ConcurrentHashMap<>());
        
        return (T) classCache.computeIfAbsent(cacheKey, k -> clazz.getAnnotation(annotationClass));
    }
    
    public <T extends Annotation> T getMethodAnnotation(Method method, Class<T> annotationClass) {
        String cacheKey = annotationClass.getName();
        Map<String, Annotation> methodCache = methodAnnotationCache.computeIfAbsent(method, k -> 
            new ConcurrentHashMap<>());
        
        return (T) methodCache.computeIfAbsent(cacheKey, k -> method.getAnnotation(annotationClass));
    }
    
    public void clearCache() {
        annotationCache.clear();
        methodAnnotationCache.clear();
    }
    
    public int getCacheSize() {
        return annotationCache.size() + methodAnnotationCache.size();
    }
}

// Performance benchmarking framework
class AnnotationBenchmark {
    private final CachedAnnotationProcessor processor = new CachedAnnotationProcessor();
    
    public void runBenchmark() throws Exception {
        System.out.println("=== Annotation Performance Benchmark ===");
        
        Class<?> testClass = PerformanceTestClass.class;
        Object testInstance = testClass.getDeclaredConstructor().newInstance();
        
        // Benchmark direct annotation access
        long startTime = System.nanoTime();
        for (int i = 0; i < 100000; i++) {
            testClass.getAnnotation(PerformanceTest.class);
        }
        long directTime = System.nanoTime() - startTime;
        
        // Benchmark cached annotation access
        startTime = System.nanoTime();
        for (int i = 0; i < 100000; i++) {
            processor.getClassAnnotation(testClass, PerformanceTest.class);
        }
        long cachedTime = System.nanoTime() - startTime;
        
        System.out.println("Direct annotation access: " + directTime + " ns");
        System.out.println("Cached annotation access: " + cachedTime + " ns");
        System.out.println("Performance improvement: " + (double) directTime / cachedTime + "x");
        System.out.println("Cache size: " + processor.getCacheSize());
        
        // Benchmark method annotation processing
        System.out.println("\n=== Method Annotation Processing ===");
        Method[] methods = testClass.getDeclaredMethods();
        
        startTime = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            for (Method method : methods) {
                method.getAnnotation(PerformanceTest.class);
                method.getAnnotation(Benchmark.class);
            }
        }
        long methodDirectTime = System.nanoTime() - startTime;
        
        startTime = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            for (Method method : methods) {
                processor.getMethodAnnotation(method, PerformanceTest.class);
                processor.getMethodAnnotation(method, Benchmark.class);
            }
        }
        long methodCachedTime = System.nanoTime() - startTime;
        
        System.out.println("Direct method annotation access: " + methodDirectTime + " ns");
        System.out.println("Cached method annotation access: " + methodCachedTime + " ns");
        System.out.println("Method annotation performance improvement: " + 
                         (double) methodDirectTime / methodCachedTime + "x");
    }
    
    public void processAnnotationsWithMetadata() throws Exception {
        System.out.println("\n=== Annotation Metadata Processing ===");
        
        Class<?> testClass = PerformanceTestClass.class;
        Method[] methods = testClass.getDeclaredMethods();
        
        for (Method method : methods) {
            PerformanceTest perfTest = processor.getMethodAnnotation(method, PerformanceTest.class);
            Benchmark benchmark = processor.getMethodAnnotation(method, Benchmark.class);
            
            if (perfTest != null && perfTest.enabled()) {
                System.out.println("Processing method: " + method.getName());
                System.out.println("  Performance test enabled: " + perfTest.enabled());
                System.out.println("  Iterations: " + perfTest.iterations());
                
                if (benchmark != null) {
                    System.out.println("  Benchmark category: " + benchmark.category());
                    System.out.println("  Warmup iterations: " + benchmark.warmupIterations());
                }
                
                // Simulate annotation-driven processing
                processMethodWithAnnotations(method, perfTest, benchmark);
            }
        }
    }
    
    private void processMethodWithAnnotations(Method method, PerformanceTest perfTest, Benchmark benchmark) {
        System.out.println("  Executing annotation-driven processing...");
        // This would contain the actual logic driven by annotations
    }
}

// Best practices demonstration
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface BestPracticeExample {
    String description();
    boolean recommended() default true;
    String[] alternatives() default {};
}

@BestPracticeExample(
    description = "Demonstrates proper annotation usage",
    alternatives = {"AvoidReflection", "UseCompileTimeChecks"}
)
class AnnotationBestPractices {
    // Good: Use specific target annotations
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    @interface GoodMethodAnnotation {
        String value() default "";
    }
    
    // Bad: Too broad target - avoid this
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.TYPE) // Should be more specific
    @interface BadAnnotation {
        // Too many optional elements
        String value() default "";
        int number() default 0;
        boolean flag() default false;
        String[] array() default {};
    }
    
    // Good: Well-designed annotation
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.FIELD)
    @interface ValidatedField {
        String pattern() default ".*";
        int minLength() default 0;
        String message() default "Validation failed";
    }
    
    // Example of good annotation usage
    @GoodMethodAnnotation("Process user data")
    public void processUserData(String data) {
        System.out.println("Processing: " + data);
    }
    
    @ValidatedField(pattern = "\\d+", minLength = 5, message = "Invalid ID format")
    private String userId;
}

public class AnnotationPerformanceAndBestPractices {
    public static void main(String[] args) throws Exception {
        System.out.println("=== Annotation Performance and Best Practices ===");
        
        // Run performance benchmarks
        AnnotationBenchmark benchmark = new AnnotationBenchmark();
        benchmark.runBenchmark();
        benchmark.processAnnotationsWithMetadata();
        
        System.out.println("\n" + "=".repeat(50));
        System.out.println("=== Best Practices Examples ===");
        
        // Demonstrate best practices
        demonstrateBestPractices();
        
        System.out.println("\n=== Annotation Design Patterns ===");
        
        // Show common annotation design patterns
        demonstrateDesignPatterns();
    }
    
    private static void demonstrateBestPractices() {
        System.out.println("Good annotation design principles:");
        System.out.println("1. Use specific @Target annotations");
        System.out.println("2. Choose appropriate @Retention policy");
        System.out.println("3. Provide meaningful default values");
        System.out.println("4. Keep annotations focused and simple");
        System.out.println("5. Use caching for frequently accessed annotations");
        System.out.println("6. Document annotation purpose and usage");
        
        // Example of processing annotations with error handling
        Class<?> clazz = AnnotationBestPractices.class;
        Annotation[] annotations = clazz.getAnnotations();
        
        System.out.println("\nProcessing annotations safely:");
        for (Annotation annotation : annotations) {
            System.out.println("Found annotation: " + annotation.annotationType().getSimpleName());
            
            // Process annotation elements safely
            try {
                Method[] methods = annotation.annotationType().getDeclaredMethods();
                for (Method method : methods) {
                    Object value = method.invoke(annotation);
                    System.out.println("  " + method.getName() + " = " + value);
                }
            } catch (Exception e) {
                System.err.println("Error processing annotation: " + e.getMessage());
            }
        }
    }
    
    private static void demonstrateDesignPatterns() {
        System.out.println("Common annotation design patterns:");
        System.out.println("1. Marker annotations (no elements)");
        System.out.println("2. Single-value annotations");
        System.out.println("3. Full-featured annotations with multiple elements");
        System.out.println("4. Nested annotations for complex configurations");
        System.out.println("5. Repeatable annotations for multiple instances");
        System.out.println("6. Inherited annotations for class hierarchies");
        
        // Example: Marker annotation
        @interface Marker {
            // Empty - just marks elements
        }
        
        // Example: Single-value annotation
        @interface SingleValue {
            String value();
        }
        
        // Example: Complex annotation with nested elements
        @interface ComplexConfig {
            String name();
            int version() default 1;
            
            @interface NestedConfig {
                String key();
                String value();
            }
            
            NestedConfig[] properties() default {};
        }
        
        System.out.println("\nPattern examples demonstrated in code above.");
    }
}
```

## Summary

Java annotations are a powerful feature that enables metadata-driven programming. Key takeaways include:

1. **Built-in Annotations**: `@Override`, `@Deprecated`, `@SuppressWarnings`, `@FunctionalInterface`, `@SafeVarargs`
2. **Meta-Annotations**: `@Target`, `@Retention`, `@Documented`, `@Inherited`, `@Repeatable`
3. **Custom Annotations**: Define your own annotations with appropriate metadata
4. **Runtime Processing**: Use reflection to process annotations at runtime
5. **Framework Integration**: Annotations are essential for modern Java frameworks
6. **Performance Considerations**: Cache frequently accessed annotations for better performance
7. **Best Practices**: Design focused, well-documented annotations with appropriate targets and retention

Important principles for effective annotation usage:
- Use specific `@Target` annotations to limit where annotations can be applied
- Choose appropriate `@Retention` policies based on usage (SOURCE, CLASS, RUNTIME)
- Provide meaningful default values for optional elements
- Keep annotations focused on single responsibilities
- Use caching when processing annotations frequently
- Handle exceptions properly in annotation processing code
- Document annotation purpose and usage clearly

Mastering annotations is essential for building modern, maintainable Java applications and frameworks. They enable declarative programming, reduce boilerplate code, and provide powerful extension mechanisms.