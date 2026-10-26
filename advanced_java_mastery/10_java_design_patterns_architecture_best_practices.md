# Java Design Patterns: Mastering Software Architecture and Best Practices

## Introduction

Design patterns are reusable solutions to common software design problems. They provide proven approaches to structuring code, improving maintainability, and promoting best practices in object-oriented design. Java, being an object-oriented language, benefits greatly from the application of design patterns.

This comprehensive guide explores the most important design patterns in Java, covering creational, structural, and behavioral patterns with practical examples and real-world applications.

## Creational Patterns

### Singleton Pattern

The Singleton pattern ensures that a class has only one instance and provides a global point of access to it.

```java
import java.io.*;
import java.util.concurrent.*;

public class SingletonPattern {
    public static void main(String[] args) {
        System.out.println("=== Singleton Pattern ===");
        
        // Demonstrate different Singleton implementations
        demonstrateSingletonImplementations();
        
        // Show thread safety considerations
        demonstrateThreadSafety();
        
        // Demonstrate serialization issues
        demonstrateSerializationIssues();
    }
    
    private static void demonstrateSingletonImplementations() {
        System.out.println("\n=== Singleton Implementations ===");
        
        // Eager initialization
        EagerSingleton eager1 = EagerSingleton.getInstance();
        EagerSingleton eager2 = EagerSingleton.getInstance();
        System.out.println("Eager Singleton - Same instance: " + (eager1 == eager2));
        
        // Lazy initialization
        LazySingleton lazy1 = LazySingleton.getInstance();
        LazySingleton lazy2 = LazySingleton.getInstance();
        System.out.println("Lazy Singleton - Same instance: " + (lazy1 == lazy2));
        
        // Thread-safe lazy initialization
        ThreadSafeSingleton threadSafe1 = ThreadSafeSingleton.getInstance();
        ThreadSafeSingleton threadSafe2 = ThreadSafeSingleton.getInstance();
        System.out.println("Thread-safe Singleton - Same instance: " + (threadSafe1 == threadSafe2));
        
        // Bill Pugh Singleton (Initialization-on-demand holder)
        BillPughSingleton billPugh1 = BillPughSingleton.getInstance();
        BillPughSingleton billPugh2 = BillPughSingleton.getInstance();
        System.out.println("Bill Pugh Singleton - Same instance: " + (billPugh1 == billPugh2));
        
        // Enum Singleton
        EnumSingleton enum1 = EnumSingleton.INSTANCE;
        EnumSingleton enum2 = EnumSingleton.INSTANCE;
        System.out.println("Enum Singleton - Same instance: " + (enum1 == enum2));
    }
    
    private static void demonstrateThreadSafety() throws InterruptedException {
        System.out.println("\n=== Thread Safety Demonstration ===");
        
        // Test thread safety with multiple threads
        int threadCount = 100;
        CountDownLatch latch = new CountDownLatch(threadCount);
        ConcurrentLinkedQueue<ThreadSafeSingleton> instances = new ConcurrentLinkedQueue<>();
        
        // Create multiple threads accessing Singleton simultaneously
        for (int i = 0; i < threadCount; i++) {
            new Thread(() -> {
                try {
                    ThreadSafeSingleton instance = ThreadSafeSingleton.getInstance();
                    instances.add(instance);
                } finally {
                    latch.countDown();
                }
            }).start();
        }
        
        latch.await();
        
        // Verify all instances are the same
        ThreadSafeSingleton firstInstance = instances.peek();
        boolean allSame = instances.stream().allMatch(instance -> instance == firstInstance);
        System.out.println("All thread-safe instances are same: " + allSame);
        System.out.println("Total instances created: 1 (as expected)");
    }
    
    private static void demonstrateSerializationIssues() throws IOException, ClassNotFoundException {
        System.out.println("\n=== Serialization Issues ===");
        
        // Test serialization with regular Singleton
        RegularSingleton regular = RegularSingleton.getInstance();
        System.out.println("Regular Singleton before serialization: " + regular.hashCode());
        
        // Serialize
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(regular);
        oos.close();
        
        // Deserialize
        ByteArrayInputStream bais = new ByteArrayInputStream(baos.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(bais);
        RegularSingleton deserialized = (RegularSingleton) ois.readObject();
        ois.close();
        
        System.out.println("Regular Singleton after deserialization: " + deserialized.hashCode());
        System.out.println("Regular Singleton same instance: " + (regular == deserialized));
        
        // Test serialization with Singleton that handles readResolve
        SerializableSingleton serializable = SerializableSingleton.getInstance();
        System.out.println("Serializable Singleton before serialization: " + serializable.hashCode());
        
        // Serialize
        ByteArrayOutputStream baos2 = new ByteArrayOutputStream();
        ObjectOutputStream oos2 = new ObjectOutputStream(baos2);
        oos2.writeObject(serializable);
        oos2.close();
        
        // Deserialize
        ByteArrayInputStream bais2 = new ByteArrayInputStream(baos2.toByteArray());
        ObjectInputStream ois2 = new ObjectInputStream(bais2);
        SerializableSingleton deserialized2 = (SerializableSingleton) ois2.readObject();
        ois2.close();
        
        System.out.println("Serializable Singleton after deserialization: " + deserialized2.hashCode());
        System.out.println("Serializable Singleton same instance: " + (serializable == deserialized2));
    }
}

// Eager initialization - thread-safe by default
class EagerSingleton {
    private static final EagerSingleton INSTANCE = new EagerSingleton();
    
    private EagerSingleton() {
        // Private constructor prevents instantiation
    }
    
    public static EagerSingleton getInstance() {
        return INSTANCE;
    }
    
    public void doSomething() {
        System.out.println("Eager Singleton doing something...");
    }
}

// Lazy initialization - not thread-safe
class LazySingleton {
    private static LazySingleton instance;
    
    private LazySingleton() {
        // Private constructor
    }
    
    public static LazySingleton getInstance() {
        if (instance == null) {
            instance = new LazySingleton();
        }
        return instance;
    }
    
    public void doSomething() {
        System.out.println("Lazy Singleton doing something...");
    }
}

// Thread-safe lazy initialization
class ThreadSafeSingleton {
    private static ThreadSafeSingleton instance;
    
    private ThreadSafeSingleton() {
        // Private constructor
    }
    
    public static synchronized ThreadSafeSingleton getInstance() {
        if (instance == null) {
            instance = new ThreadSafeSingleton();
        }
        return instance;
    }
    
    // Double-checked locking optimization
    public static ThreadSafeSingleton getDoubleCheckedInstance() {
        if (instance == null) {
            synchronized (ThreadSafeSingleton.class) {
                if (instance == null) {
                    instance = new ThreadSafeSingleton();
                }
            }
        }
        return instance;
    }
    
    public void doSomething() {
        System.out.println("Thread-safe Singleton doing something...");
    }
}

// Bill Pugh Singleton - thread-safe and efficient
class BillPughSingleton {
    private BillPughSingleton() {
        // Private constructor
    }
    
    private static class SingletonHelper {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }
    
    public static BillPughSingleton getInstance() {
        return SingletonHelper.INSTANCE;
    }
    
    public void doSomething() {
        System.out.println("Bill Pugh Singleton doing something...");
    }
}

// Enum Singleton - thread-safe and serialization-safe
enum EnumSingleton {
    INSTANCE;
    
    public void doSomething() {
        System.out.println("Enum Singleton doing something...");
    }
}

// Regular Singleton that has serialization issues
class RegularSingleton implements Serializable {
    private static final long serialVersionUID = 1L;
    private static RegularSingleton instance;
    
    private RegularSingleton() {
        // Private constructor
    }
    
    public static RegularSingleton getInstance() {
        if (instance == null) {
            instance = new RegularSingleton();
        }
        return instance;
    }
    
    public void doSomething() {
        System.out.println("Regular Singleton doing something...");
    }
}

// Serializable Singleton that handles readResolve
class SerializableSingleton implements Serializable {
    private static final long serialVersionUID = 1L;
    private static SerializableSingleton instance;
    
    private SerializableSingleton() {
        // Private constructor
    }
    
    public static SerializableSingleton getInstance() {
        if (instance == null) {
            instance = new SerializableSingleton();
        }
        return instance;
    }
    
    // This method is called during deserialization
    private Object readResolve() {
        return getInstance(); // Return the singleton instance
    }
    
    public void doSomething() {
        System.out.println("Serializable Singleton doing something...");
    }
}
```

### Factory Pattern

The Factory pattern provides an interface for creating objects without specifying their exact classes.

```java
import java.util.*;

public class FactoryPattern {
    public static void main(String[] args) {
        System.out.println("=== Factory Pattern ===");
        
        // Demonstrate simple factory
        demonstrateSimpleFactory();
        
        // Demonstrate factory method
        demonstrateFactoryMethod();
        
        // Demonstrate abstract factory
        demonstrateAbstractFactory();
        
        // Show real-world factory example
        demonstrateRealWorldFactory();
    }
    
    private static void demonstrateSimpleFactory() {
        System.out.println("\n=== Simple Factory ===");
        
        // Simple factory for creating shapes
        ShapeFactory factory = new ShapeFactory();
        
        Shape circle = factory.createShape("CIRCLE");
        Shape rectangle = factory.createShape("RECTANGLE");
        Shape square = factory.createShape("SQUARE");
        
        circle.draw();
        rectangle.draw();
        square.draw();
        
        // Try invalid shape
        try {
            Shape invalid = factory.createShape("TRIANGLE");
        } catch (IllegalArgumentException e) {
            System.out.println("Caught expected exception: " + e.getMessage());
        }
    }
    
    private static void demonstrateFactoryMethod() {
        System.out.println("\n=== Factory Method ===");
        
        // Different creators with different factory methods
        ShapeCreator circleCreator = new CircleCreator();
        ShapeCreator rectangleCreator = new RectangleCreator();
        
        Shape circle = circleCreator.createShape();
        Shape rectangle = rectangleCreator.createShape();
        
        circle.draw();
        rectangle.draw();
    }
    
    private static void demonstrateAbstractFactory() {
        System.out.println("\n=== Abstract Factory ===");
        
        // Create factories for different themes
        GUIFactory windowsFactory = new WindowsGUIFactory();
        GUIFactory macFactory = new MacGUIFactory();
        
        // Create UI components using Windows factory
        Button windowsButton = windowsFactory.createButton();
        Checkbox windowsCheckbox = windowsFactory.createCheckbox();
        
        windowsButton.render();
        windowsCheckbox.render();
        
        // Create UI components using Mac factory
        Button macButton = macFactory.createButton();
        Checkbox macCheckbox = macFactory.createCheckbox();
        
        macButton.render();
        macCheckbox.render();
    }
    
    private static void demonstrateRealWorldFactory() {
        System.out.println("\n=== Real-World Factory Example ===");
        
        // Database connection factory
        DatabaseConnectionFactory factory = new DatabaseConnectionFactory();
        
        // Create different types of database connections
        DatabaseConnection mysqlConnection = factory.createConnection("MYSQL");
        DatabaseConnection postgresConnection = factory.createConnection("POSTGRES");
        DatabaseConnection oracleConnection = factory.createConnection("ORACLE");
        
        mysqlConnection.connect();
        postgresConnection.connect();
        oracleConnection.connect();
        
        // Execute queries
        mysqlConnection.executeQuery("SELECT * FROM users");
        postgresConnection.executeQuery("SELECT * FROM products");
        oracleConnection.executeQuery("SELECT * FROM orders");
        
        mysqlConnection.disconnect();
        postgresConnection.disconnect();
        oracleConnection.disconnect();
    }
}

// Simple Factory implementation
interface Shape {
    void draw();
}

class Circle implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing a Circle");
    }
}

class Rectangle implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing a Rectangle");
    }
}

class Square implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing a Square");
    }
}

class ShapeFactory {
    public Shape createShape(String shapeType) {
        if (shapeType == null) {
            throw new IllegalArgumentException("Shape type cannot be null");
        }
        
        switch (shapeType.toUpperCase()) {
            case "CIRCLE":
                return new Circle();
            case "RECTANGLE":
                return new Rectangle();
            case "SQUARE":
                return new Square();
            default:
                throw new IllegalArgumentException("Unknown shape type: " + shapeType);
        }
    }
}

// Factory Method pattern
abstract class ShapeCreator {
    // Factory method
    public abstract Shape createShape();
    
    // Other operations that use the factory method
    public void renderShape() {
        Shape shape = createShape();
        shape.draw();
    }
}

class CircleCreator extends ShapeCreator {
    @Override
    public Shape createShape() {
        return new Circle();
    }
}

class RectangleCreator extends ShapeCreator {
    @Override
    public Shape createShape() {
        return new Rectangle();
    }
}

// Abstract Factory pattern
interface GUIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

interface Button {
    void render();
}

interface Checkbox {
    void render();
}

// Windows implementations
class WindowsButton implements Button {
    @Override
    public void render() {
        System.out.println("Rendering Windows Button");
    }
}

class WindowsCheckbox implements Checkbox {
    @Override
    public void render() {
        System.out.println("Rendering Windows Checkbox");
    }
}

class WindowsGUIFactory implements GUIFactory {
    @Override
    public Button createButton() {
        return new WindowsButton();
    }
    
    @Override
    public Checkbox createCheckbox() {
        return new WindowsCheckbox();
    }
}

// Mac implementations
class MacButton implements Button {
    @Override
    public void render() {
        System.out.println("Rendering Mac Button");
    }
}

class MacCheckbox implements Checkbox {
    @Override
    public void render() {
        System.out.println("Rendering Mac Checkbox");
    }
}

class MacGUIFactory implements GUIFactory {
    @Override
    public Button createButton() {
        return new MacButton();
    }
    
    @Override
    public Checkbox createCheckbox() {
        return new MacCheckbox();
    }
}

// Real-world factory example
interface DatabaseConnection {
    void connect();
    void disconnect();
    void executeQuery(String query);
}

class MySQLConnection implements DatabaseConnection {
    @Override
    public void connect() {
        System.out.println("Connecting to MySQL database");
    }
    
    @Override
    public void disconnect() {
        System.out.println("Disconnecting from MySQL database");
    }
    
    @Override
    public void executeQuery(String query) {
        System.out.println("Executing MySQL query: " + query);
    }
}

class PostgreSQLConnection implements DatabaseConnection {
    @Override
    public void connect() {
        System.out.println("Connecting to PostgreSQL database");
    }
    
    @Override
    public void disconnect() {
        System.out.println("Disconnecting from PostgreSQL database");
    }
    
    @Override
    public void executeQuery(String query) {
        System.out.println("Executing PostgreSQL query: " + query);
    }
}

class OracleConnection implements DatabaseConnection {
    @Override
    public void connect() {
        System.out.println("Connecting to Oracle database");
    }
    
    @Override
    public void disconnect() {
        System.out.println("Disconnecting from Oracle database");
    }
    
    @Override
    public void executeQuery(String query) {
        System.out.println("Executing Oracle query: " + query);
    }
}

class DatabaseConnectionFactory {
    public DatabaseConnection createConnection(String databaseType) {
        switch (databaseType.toUpperCase()) {
            case "MYSQL":
                return new MySQLConnection();
            case "POSTGRES":
                return new PostgreSQLConnection();
            case "ORACLE":
                return new OracleConnection();
            default:
                throw new IllegalArgumentException("Unsupported database type: " + databaseType);
        }
    }
}
```

### Builder Pattern

The Builder pattern separates the construction of a complex object from its representation.

```java
import java.util.*;

public class BuilderPattern {
    public static void main(String[] args) {
        System.out.println("=== Builder Pattern ===");
        
        // Demonstrate basic builder
        demonstrateBasicBuilder();
        
        // Demonstrate fluent builder
        demonstrateFluentBuilder();
        
        // Demonstrate builder with validation
        demonstrateBuilderWithValidation();
        
        // Show real-world builder example
        demonstrateRealWorldBuilder();
    }
    
    private static void demonstrateBasicBuilder() {
        System.out.println("\n=== Basic Builder ===");
        
        // Using builder to create complex objects
        Person person = new Person.Builder()
            .setFirstName("John")
            .setLastName("Doe")
            .setAge(30)
            .setEmail("john.doe@example.com")
            .build();
        
        System.out.println("Created person: " + person);
    }
    
    private static void demonstrateFluentBuilder() {
        System.out.println("\n=== Fluent Builder ===");
        
        // Fluent interface for more readable construction
        Computer computer = new Computer.Builder()
            .setCPU("Intel i9")
            .setRAM("32GB")
            .setStorage("1TB SSD")
            .setGraphicsCard("RTX 4090")
            .setOperatingSystem("Windows 11")
            .build();
        
        System.out.println("Built computer: " + computer);
    }
    
    private static void demonstrateBuilderWithValidation() {
        System.out.println("\n=== Builder with Validation ===");
        
        try {
            // Valid construction
            User user = new User.Builder()
                .setUsername("johndoe")
                .setEmail("john@example.com")
                .setPassword("securePassword123")
                .setAge(25)
                .build();
            
            System.out.println("Created valid user: " + user);
            
            // Invalid construction (missing required fields)
            User invalidUser = new User.Builder()
                .setUsername("invalid")
                .build();
            
        } catch (IllegalStateException e) {
            System.out.println("Caught validation error: " + e.getMessage());
        }
    }
    
    private static void demonstrateRealWorldBuilder() {
        System.out.println("\n=== Real-World Builder Example ===");
        
        // HTTP request builder
        HttpRequest request = new HttpRequest.Builder()
            .setMethod("POST")
            .setUrl("https://api.example.com/users")
            .addHeader("Content-Type", "application/json")
            .addHeader("Authorization", "Bearer token123")
            .setBody("{\"name\":\"John\",\"email\":\"john@example.com\"}")
            .setTimeout(5000)
            .build();
        
        System.out.println("Built HTTP request: " + request);
    }
}

// Basic Builder example
class Person {
    private final String firstName;
    private final String lastName;
    private final int age;
    private final String email;
    
    private Person(Builder builder) {
        this.firstName = builder.firstName;
        this.lastName = builder.lastName;
        this.age = builder.age;
        this.email = builder.email;
    }
    
    public static class Builder {
        private String firstName;
        private String lastName;
        private int age;
        private String email;
        
        public Builder setFirstName(String firstName) {
            this.firstName = firstName;
            return this;
        }
        
        public Builder setLastName(String lastName) {
            this.lastName = lastName;
            return this;
        }
        
        public Builder setAge(int age) {
            this.age = age;
            return this;
        }
        
        public Builder setEmail(String email) {
            this.email = email;
            return this;
        }
        
        public Person build() {
            return new Person(this);
        }
    }
    
    @Override
    public String toString() {
        return "Person{firstName='" + firstName + "', lastName='" + lastName + 
               "', age=" + age + ", email='" + email + "'}";
    }
}

// Fluent Builder example
class Computer {
    private final String CPU;
    private final String RAM;
    private final String storage;
    private final String graphicsCard;
    private final String operatingSystem;
    
    private Computer(Builder builder) {
        this.CPU = builder.CPU;
        this.RAM = builder.RAM;
        this.storage = builder.storage;
        this.graphicsCard = builder.graphicsCard;
        this.operatingSystem = builder.operatingSystem;
    }
    
    public static class Builder {
        private String CPU;
        private String RAM;
        private String storage;
        private String graphicsCard;
        private String operatingSystem;
        
        public Builder setCPU(String CPU) {
            this.CPU = CPU;
            return this;
        }
        
        public Builder setRAM(String RAM) {
            this.RAM = RAM;
            return this;
        }
        
        public Builder setStorage(String storage) {
            this.storage = storage;
            return this;
        }
        
        public Builder setGraphicsCard(String graphicsCard) {
            this.graphicsCard = graphicsCard;
            return this;
        }
        
        public Builder setOperatingSystem(String operatingSystem) {
            this.operatingSystem = operatingSystem;
            return this;
        }
        
        public Computer build() {
            return new Computer(this);
        }
    }
    
    @Override
    public String toString() {
        return "Computer{CPU='" + CPU + "', RAM='" + RAM + "', storage='" + storage + 
               "', graphicsCard='" + graphicsCard + "', OS='" + operatingSystem + "'}";
    }
}

// Builder with validation
class User {
    private final String username;
    private final String email;
    private final String password;
    private final int age;
    
    private User(Builder builder) {
        this.username = builder.username;
        this.email = builder.email;
        this.password = builder.password;
        this.age = builder.age;
    }
    
    public static class Builder {
        private String username;
        private String email;
        private String password;
        private int age;
        
        public Builder setUsername(String username) {
            this.username = username;
            return this;
        }
        
        public Builder setEmail(String email) {
            this.email = email;
            return this;
        }
        
        public Builder setPassword(String password) {
            this.password = password;
            return this;
        }
        
        public Builder setAge(int age) {
            this.age = age;
            return this;
        }
        
        public User build() {
            validate();
            return new User(this);
        }
        
        private void validate() {
            if (username == null || username.isEmpty()) {
                throw new IllegalStateException("Username is required");
            }
            if (email == null || !email.contains("@")) {
                throw new IllegalStateException("Valid email is required");
            }
            if (password == null || password.length() < 8) {
                throw new IllegalStateException("Password must be at least 8 characters");
            }
            if (age < 0 || age > 150) {
                throw new IllegalStateException("Age must be between 0 and 150");
            }
        }
    }
    
    @Override
    public String toString() {
        return "User{username='" + username + "', email='" + email + "', age=" + age + "}";
    }
}

// Real-world builder example
class HttpRequest {
    private final String method;
    private final String url;
    private final Map<String, String> headers;
    private final String body;
    private final int timeout;
    
    private HttpRequest(Builder builder) {
        this.method = builder.method;
        this.url = builder.url;
        this.headers = new HashMap<>(builder.headers);
        this.body = builder.body;
        this.timeout = builder.timeout;
    }
    
    public static class Builder {
        private String method = "GET";
        private String url;
        private Map<String, String> headers = new HashMap<>();
        private String body;
        private int timeout = 30000; // 30 seconds default
        
        public Builder setMethod(String method) {
            this.method = method;
            return this;
        }
        
        public Builder setUrl(String url) {
            this.url = url;
            return this;
        }
        
        public Builder addHeader(String name, String value) {
            this.headers.put(name, value);
            return this;
        }
        
        public Builder setBody(String body) {
            this.body = body;
            return this;
        }
        
        public Builder setTimeout(int timeout) {
            this.timeout = timeout;
            return this;
        }
        
        public HttpRequest build() {
            if (url == null || url.isEmpty()) {
                throw new IllegalStateException("URL is required");
            }
            return new HttpRequest(this);
        }
    }
    
    @Override
    public String toString() {
        return "HttpRequest{method='" + method + "', url='" + url + 
               "', headers=" + headers.size() + ", bodyLength=" + 
               (body != null ? body.length() : 0) + ", timeout=" + timeout + "ms}";
    }
}
```

### Prototype Pattern

The Prototype pattern creates new objects by copying existing ones.

```java
import java.util.*;

public class PrototypePattern {
    public static void main(String[] args) {
        System.out.println("=== Prototype Pattern ===");
        
        // Demonstrate basic prototype
        demonstrateBasicPrototype();
        
        // Demonstrate deep copy vs shallow copy
        demonstrateCopyTypes();
        
        // Demonstrate prototype registry
        demonstratePrototypeRegistry();
        
        // Show real-world prototype example
        demonstrateRealWorldPrototype();
    }
    
    private static void demonstrateBasicPrototype() {
        System.out.println("\n=== Basic Prototype ===");
        
        // Create original objects
        Circle originalCircle = new Circle();
        originalCircle.setX(10);
        originalCircle.setY(20);
        originalCircle.setRadius(15);
        originalCircle.setColor("Red");
        
        Rectangle originalRectangle = new Rectangle();
        originalRectangle.setX(30);
        originalRectangle.setY(40);
        originalRectangle.setWidth(50);
        originalRectangle.setHeight(60);
        originalRectangle.setColor("Blue");
        
        // Clone objects
        Circle clonedCircle = (Circle) originalCircle.clone();
        Rectangle clonedRectangle = (Rectangle) originalRectangle.clone();
        
        System.out.println("Original Circle: " + originalCircle);
        System.out.println("Cloned Circle: " + clonedCircle);
        System.out.println("Same object: " + (originalCircle == clonedCircle));
        System.out.println("Equal objects: " + originalCircle.equals(clonedCircle));
        
        System.out.println("Original Rectangle: " + originalRectangle);
        System.out.println("Cloned Rectangle: " + clonedRectangle);
        System.out.println("Same object: " + (originalRectangle == clonedRectangle));
        System.out.println("Equal objects: " + originalRectangle.equals(clonedRectangle));
    }
    
    private static void demonstrateCopyTypes() {
        System.out.println("\n=== Deep Copy vs Shallow Copy ===");
        
        // Shallow copy example
        ShallowCopyObject shallowOriginal = new ShallowCopyObject();
        shallowOriginal.setName("Original");
        shallowOriginal.setTags(Arrays.asList("tag1", "tag2", "tag3"));
        
        ShallowCopyObject shallowClone = (ShallowCopyObject) shallowOriginal.clone();
        System.out.println("Shallow copy - Original tags: " + shallowOriginal.getTags());
        System.out.println("Shallow copy - Clone tags: " + shallowClone.getTags());
        
        // Modify original list - affects clone in shallow copy
        shallowOriginal.getTags().add("tag4");
        System.out.println("After modifying original - Clone tags: " + shallowClone.getTags());
        
        // Deep copy example
        DeepCopyObject deepOriginal = new DeepCopyObject();
        deepOriginal.setName("Original");
        deepOriginal.setTags(Arrays.asList("tag1", "tag2", "tag3"));
        
        DeepCopyObject deepClone = (DeepCopyObject) deepOriginal.clone();
        System.out.println("\nDeep copy - Original tags: " + deepOriginal.getTags());
        System.out.println("Deep copy - Clone tags: " + deepClone.getTags());
        
        // Modify original list - doesn't affect clone in deep copy
        deepOriginal.getTags().add("tag4");
        System.out.println("After modifying original - Clone tags: " + deepClone.getTags());
    }
    
    private static void demonstratePrototypeRegistry() {
        System.out.println("\n=== Prototype Registry ===");
        
        // Create prototype registry
        ShapeRegistry registry = new ShapeRegistry();
        
        // Register prototype shapes
        registry.registerPrototype("RedCircle", new Circle(0, 0, 10, "Red"));
        registry.registerPrototype("BlueRectangle", new Rectangle(0, 0, 20, 30, "Blue"));
        registry.registerPrototype("GreenSquare", new Square(0, 0, 15, "Green"));
        
        // Create shapes using prototypes
        Shape redCircle1 = registry.getPrototype("RedCircle");
        Shape redCircle2 = registry.getPrototype("RedCircle");
        Shape blueRectangle = registry.getPrototype("BlueRectangle");
        Shape greenSquare = registry.getPrototype("GreenSquare");
        
        redCircle1.setX(100);
        redCircle1.setY(200);
        
        blueRectangle.setX(300);
        blueRectangle.setY(400);
        
        System.out.println("Created shapes from prototypes:");
        System.out.println("Red Circle 1: " + redCircle1);
        System.out.println("Red Circle 2: " + redCircle2);
        System.out.println("Blue Rectangle: " + blueRectangle);
        System.out.println("Green Square: " + greenSquare);
        
        // Verify they are different objects but same structure
        System.out.println("Different objects: " + (redCircle1 != redCircle2));
        System.out.println("Same structure: " + 
            (((Circle) redCircle1).getRadius() == ((Circle) redCircle2).getRadius()));
    }
    
    private static void demonstrateRealWorldPrototype() {
        System.out.println("\n=== Real-World Prototype Example ===");
        
        // Configuration prototype
        Configuration configPrototype = new Configuration();
        configPrototype.setDatabaseUrl("jdbc:mysql://localhost:3306/mydb");
        configPrototype.setDatabaseUser("user");
        configPrototype.setDatabasePassword("password");
        configPrototype.setCacheEnabled(true);
        configPrototype.setCacheSize(1000);
        configPrototype.setFeatures(Arrays.asList("feature1", "feature2", "feature3"));
        
        // Create different configurations based on prototype
        Configuration devConfig = (Configuration) configPrototype.clone();
        devConfig.setEnvironment("development");
        devConfig.setDatabaseUrl("jdbc:mysql://dev-server:3306/mydb");
        devConfig.setCacheSize(100); // Smaller cache for development
        
        Configuration prodConfig = (Configuration) configPrototype.clone();
        prodConfig.setEnvironment("production");
        prodConfig.setDatabaseUrl("jdbc:mysql://prod-server:3306/mydb");
        prodConfig.setDatabaseUser("prod_user");
        prodConfig.setDatabasePassword("prod_password");
        prodConfig.setCacheSize(10000); // Larger cache for production
        
        System.out.println("Development config: " + devConfig);
        System.out.println("Production config: " + prodConfig);
    }
}

// Basic Prototype interface
interface Prototype extends Cloneable {
    Prototype clone();
}

// Abstract Shape class implementing Prototype
abstract class Shape implements Prototype {
    protected int x, y;
    protected String color;
    
    public Shape() {}
    
    public Shape(Shape source) {
        this.x = source.x;
        this.y = source.y;
        this.color = source.color;
    }
    
    public abstract void draw();
    
    // Default clone implementation
    @Override
    public Prototype clone() {
        try {
            return (Prototype) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException("Cloning failed", e);
        }
    }
    
    // Getters and setters
    public int getX() { return x; }
    public void setX(int x) { this.x = x; }
    public int getY() { return y; }
    public void setY(int y) { this.y = y; }
    public String getColor() { return color; }
    public void setColor(String color) { this.color = color; }
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Shape shape = (Shape) obj;
        return x == shape.x && y == shape.y && Objects.equals(color, shape.color);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(x, y, color);
    }
}

// Concrete Shape implementations
class Circle extends Shape {
    private int radius;
    
    public Circle() {}
    
    public Circle(int x, int y, int radius, String color) {
        super();
        this.x = x;
        this.y = y;
        this.radius = radius;
        this.color = color;
    }
    
    public Circle(Circle source) {
        super(source);
        this.radius = source.radius;
    }
    
    @Override
    public void draw() {
        System.out.println("Drawing a circle at (" + x + "," + y + ") with radius " + radius);
    }
    
    @Override
    public Prototype clone() {
        return new Circle(this);
    }
    
    public int getRadius() { return radius; }
    public void setRadius(int radius) { this.radius = radius; }
    
    @Override
    public String toString() {
        return "Circle{x=" + x + ", y=" + y + ", radius=" + radius + ", color='" + color + "'}";
    }
}

class Rectangle extends Shape {
    private int width, height;
    
    public Rectangle() {}
    
    public Rectangle(int x, int y, int width, int height, String color) {
        super();
        this.x = x;
        this.y = y;
        this.width = width;
        this.height = height;
        this.color = color;
    }
    
    public Rectangle(Rectangle source) {
        super(source);
        this.width = source.width;
        this.height = source.height;
    }
    
    @Override
    public void draw() {
        System.out.println("Drawing a rectangle at (" + x + "," + y + ") with size " + width + "x" + height);
    }
    
    @Override
    public Prototype clone() {
        return new Rectangle(this);
    }
    
    public int getWidth() { return width; }
    public void setWidth(int width) { this.width = width; }
    public int getHeight() { return height; }
    public void setHeight(int height) { this.height = height; }
    
    @Override
    public String toString() {
        return "Rectangle{x=" + x + ", y=" + y + ", width=" + width + ", height=" + height + ", color='" + color + "'}";
    }
}

class Square extends Rectangle {
    public Square() {}
    
    public Square(int x, int y, int side, String color) {
        super(x, y, side, side, color);
    }
    
    public Square(Square source) {
        super(source);
    }
    
    @Override
    public Prototype clone() {
        return new Square(this);
    }
    
    @Override
    public String toString() {
        return "Square{x=" + getX() + ", y=" + getY() + ", side=" + getWidth() + ", color='" + getColor() + "'}";
    }
}

// Shallow copy example
class ShallowCopyObject implements Prototype {
    private String name;
    private List<String> tags;
    
    public ShallowCopyObject() {}
    
    public ShallowCopyObject(ShallowCopyObject source) {
        this.name = source.name;
        this.tags = source.tags; // Shallow copy - same reference
    }
    
    @Override
    public Prototype clone() {
        return new ShallowCopyObject(this);
    }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public List<String> getTags() { return tags; }
    public void setTags(List<String> tags) { this.tags = tags; }
    
    @Override
    public String toString() {
        return "ShallowCopyObject{name='" + name + "', tags=" + tags + "}";
    }
}

// Deep copy example
class DeepCopyObject implements Prototype {
    private String name;
    private List<String> tags;
    
    public DeepCopyObject() {}
    
    public DeepCopyObject(DeepCopyObject source) {
        this.name = source.name;
        this.tags = new ArrayList<>(source.tags); // Deep copy - new list
    }
    
    @Override
    public Prototype clone() {
        return new DeepCopyObject(this);
    }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public List<String> getTags() { return tags; }
    public void setTags(List<String> tags) { this.tags = new ArrayList<>(tags); }
    
    @Override
    public String toString() {
        return "DeepCopyObject{name='" + name + "', tags=" + tags + "}";
    }
}

// Prototype Registry
class ShapeRegistry {
    private Map<String, Shape> prototypes = new HashMap<>();
    
    public void registerPrototype(String key, Shape prototype) {
        prototypes.put(key, prototype);
    }
    
    public Shape getPrototype(String key) {
        Shape prototype = prototypes.get(key);
        if (prototype == null) {
            throw new IllegalArgumentException("Prototype not found: " + key);
        }
        return (Shape) prototype.clone();
    }
    
    public Set<String> getPrototypeKeys() {
        return prototypes.keySet();
    }
}

// Real-world prototype example
class Configuration implements Prototype {
    private String environment;
    private String databaseUrl;
    private String databaseUser;
    private String databasePassword;
    private boolean cacheEnabled;
    private int cacheSize;
    private List<String> features;
    
    public Configuration() {}
    
    public Configuration(Configuration source) {
        this.environment = source.environment;
        this.databaseUrl = source.databaseUrl;
        this.databaseUser = source.databaseUser;
        this.databasePassword = source.databasePassword;
        this.cacheEnabled = source.cacheEnabled;
        this.cacheSize = source.cacheSize;
        this.features = new ArrayList<>(source.features); // Deep copy
    }
    
    @Override
    public Prototype clone() {
        return new Configuration(this);
    }
    
    // Getters and setters
    public String getEnvironment() { return environment; }
    public void setEnvironment(String environment) { this.environment = environment; }
    public String getDatabaseUrl() { return databaseUrl; }
    public void setDatabaseUrl(String databaseUrl) { this.databaseUrl = databaseUrl; }
    public String getDatabaseUser() { return databaseUser; }
    public void setDatabaseUser(String databaseUser) { this.databaseUser = databaseUser; }
    public String getDatabasePassword() { return databasePassword; }
    public void setDatabasePassword(String databasePassword) { this.databasePassword = databasePassword; }
    public boolean isCacheEnabled() { return cacheEnabled; }
    public void setCacheEnabled(boolean cacheEnabled) { this.cacheEnabled = cacheEnabled; }
    public int getCacheSize() { return cacheSize; }
    public void setCacheSize(int cacheSize) { this.cacheSize = cacheSize; }
    public List<String> getFeatures() { return features; }
    public void setFeatures(List<String> features) { this.features = new ArrayList<>(features); }
    
    @Override
    public String toString() {
        return "Configuration{environment='" + environment + "', databaseUrl='" + databaseUrl + 
               "', cacheEnabled=" + cacheEnabled + ", cacheSize=" + cacheSize + 
               ", features=" + features + "}";
    }
}
```

## Structural Patterns

### Adapter Pattern

The Adapter pattern allows incompatible interfaces to work together.

```java
import java.util.*;

public class AdapterPattern {
    public static void main(String[] args) {
        System.out.println("=== Adapter Pattern ===");
        
        // Demonstrate class adapter
        demonstrateClassAdapter();
        
        // Demonstrate object adapter
        demonstrateObjectAdapter();
        
        // Demonstrate two-way adapter
        demonstrateTwoWayAdapter();
        
        // Show real-world adapter example
        demonstrateRealWorldAdapter();
    }
    
    private static void demonstrateClassAdapter() {
        System.out.println("\n=== Class Adapter ===");
        
        // Using the adapter to make incompatible interfaces work
        MediaPlayer player = new AudioPlayer();
        
        player.play("mp3", "song1.mp3");
        player.play("mp4", "video1.mp4");
        player.play("vlc", "movie1.vlc");
        player.play("avi", "clip1.avi"); // Unsupported format
    }
    
    private static void demonstrateObjectAdapter() {
        System.out.println("\n=== Object Adapter ===");
        
        // Using object adapter with composition
        EuropeanSocket europeanSocket = new EuropeanSocketImpl();
        USASocket usaSocket = new EuropeanToUSASocketAdapter(europeanSocket);
        
        // Use USA device with European socket through adapter
        usaSocket.plugIn("USA Device");
        usaSocket.provideElectricity();
    }
    
    private static void demonstrateTwoWayAdapter() {
        System.out.println("\n=== Two-Way Adapter ===");
        
        // Two-way adapter allows communication in both directions
        Database database = new MySQLDatabase();
        DatabaseAdapter adapter = new DatabaseAdapter(database);
        
        // Use as MySQL interface
        adapter.connectMySQL();
        adapter.executeMySQLQuery("SELECT * FROM users");
        adapter.disconnectMySQL();
        
        // Use as PostgreSQL interface
        adapter.connectPostgreSQL();
        adapter.executePostgreSQLQuery("SELECT * FROM users");
        adapter.disconnectPostgreSQL();
    }
    
    private static void demonstrateRealWorldAdapter() {
        System.out.println("\n=== Real-World Adapter Example ===");
        
        // Payment processing adapter
        PaymentProcessor creditCardProcessor = new CreditCardProcessor();
        PaymentProcessor paypalAdapter = new PayPalAdapter(new PayPalService());
        PaymentProcessor stripeAdapter = new StripeAdapter(new StripeService());
        
        // Process payments through different systems
        creditCardProcessor.processPayment(100.0, "Credit Card Payment");
        paypalAdapter.processPayment(150.0, "PayPal Payment");
        stripeAdapter.processPayment(200.0, "Stripe Payment");
    }
}

// Adapter pattern - Target interface
interface MediaPlayer {
    void play(String audioType, String fileName);
}

// Adaptee classes
interface AdvancedMediaPlayer {
    void playVlc(String fileName);
    void playMp4(String fileName);
}

class VlcPlayer implements AdvancedMediaPlayer {
    @Override
    public void playVlc(String fileName) {
        System.out.println("Playing VLC file: " + fileName);
    }
    
    @Override
    public void playMp4(String fileName) {
        // Do nothing
    }
}

class Mp4Player implements AdvancedMediaPlayer {
    @Override
    public void playVlc(String fileName) {
        // Do nothing
    }
    
    @Override
    public void playMp4(String fileName) {
        System.out.println("Playing MP4 file: " + fileName);
    }
}

// Class Adapter - extends MediaPlayer and implements AdvancedMediaPlayer
class AudioPlayer extends MediaPlayerAdapter {
    @Override
    public void play(String audioType, String fileName) {
        if (audioType.equalsIgnoreCase("mp3")) {
            System.out.println("Playing MP3 file: " + fileName);
        } else if (audioType.equalsIgnoreCase("mp4")) {
            advancedMusicPlayer = new Mp4Player();
            advancedMusicPlayer.playMp4(fileName);
        } else if (audioType.equalsIgnoreCase("vlc")) {
            advancedMusicPlayer = new VlcPlayer();
            advancedMusicPlayer.playVlc(fileName);
        } else {
            System.out.println("Invalid media type: " + audioType);
        }
    }
}

// Base adapter class
abstract class MediaPlayerAdapter implements MediaPlayer {
    protected AdvancedMediaPlayer advancedMusicPlayer;
    
    public abstract void play(String audioType, String fileName);
}

// Object Adapter pattern
interface EuropeanSocket {
    void provideElectricity();
}

interface USASocket {
    void plugIn(String device);
    void provideElectricity();
}

class EuropeanSocketImpl implements EuropeanSocket {
    @Override
    public void provideElectricity() {
        System.out.println("Providing 230V electricity (European standard)");
    }
}

// Object adapter using composition
class EuropeanToUSASocketAdapter implements USASocket {
    private EuropeanSocket europeanSocket;
    
    public EuropeanToUSASocketAdapter(EuropeanSocket europeanSocket) {
        this.europeanSocket = europeanSocket;
    }
    
    @Override
    public void plugIn(String device) {
        System.out.println(device + " plugged into USA socket adapter");
    }
    
    @Override
    public void provideElectricity() {
        System.out.println("Converting voltage...");
        europeanSocket.provideElectricity();
        System.out.println("Converted to 120V for USA devices");
    }
}

// Two-way adapter
interface MySQLInterface {
    void connectMySQL();
    void executeMySQLQuery(String query);
    void disconnectMySQL();
}

interface PostgreSQLInterface {
    void connectPostgreSQL();
    void executePostgreSQLQuery(String query);
    void disconnectPostgreSQL();
}

class MySQLDatabase implements MySQLInterface {
    @Override
    public void connectMySQL() {
        System.out.println("Connected to MySQL database");
    }
    
    @Override
    public void executeMySQLQuery(String query) {
        System.out.println("Executing MySQL query: " + query);
    }
    
    @Override
    public void disconnectMySQL() {
        System.out.println("Disconnected from MySQL database");
    }
}

// Two-way adapter
class DatabaseAdapter implements MySQLInterface, PostgreSQLInterface {
    private MySQLDatabase mySQLDatabase;
    
    public DatabaseAdapter(MySQLDatabase mySQLDatabase) {
        this.mySQLDatabase = mySQLDatabase;
    }
    
    // MySQL interface methods
    @Override
    public void connectMySQL() {
        mySQLDatabase.connectMySQL();
    }
    
    @Override
    public void executeMySQLQuery(String query) {
        mySQLDatabase.executeMySQLQuery(query);
    }
    
    @Override
    public void disconnectMySQL() {
        mySQLDatabase.disconnectMySQL();
    }
    
    // PostgreSQL interface methods (adapted from MySQL)
    @Override
    public void connectPostgreSQL() {
        System.out.println("Adapting MySQL connection to PostgreSQL interface");
        mySQLDatabase.connectMySQL();
    }
    
    @Override
    public void executePostgreSQLQuery(String query) {
        System.out.println("Adapting MySQL query to PostgreSQL format");
        mySQLDatabase.executeMySQLQuery(query);
    }
    
    @Override
    public void disconnectPostgreSQL() {
        System.out.println("Adapting MySQL disconnection to PostgreSQL interface");
        mySQLDatabase.disconnectMySQL();
    }
}

// Real-world adapter example
interface PaymentProcessor {
    void processPayment(double amount, String description);
}

class CreditCardProcessor implements PaymentProcessor {
    @Override
    public void processPayment(double amount, String description) {
        System.out.println("Processing credit card payment of $" + amount + " for " + description);
    }
}

// External services that need adaptation
class PayPalService {
    public void makePayment(double amount) {
        System.out.println("PayPal processing payment of $" + amount);
    }
    
    public void logTransaction(String description) {
        System.out.println("PayPal logging transaction: " + description);
    }
}

class StripeService {
    public void charge(double amount, String description) {
        System.out.println("Stripe charging $" + amount + " for " + description);
    }
    
    public boolean verifyPayment() {
        System.out.println("Stripe verifying payment");
        return true;
    }
}

// Adapters for external services
class PayPalAdapter implements PaymentProcessor {
    private PayPalService payPalService;
    
    public PayPalAdapter(PayPalService payPalService) {
        this.payPalService = payPalService;
    }
    
    @Override
    public void processPayment(double amount, String description) {
        System.out.println("Adapting to PayPal payment system");
        payPalService.makePayment(amount);
        payPalService.logTransaction(description);
    }
}

class StripeAdapter implements PaymentProcessor {
    private StripeService stripeService;
    
    public StripeAdapter(StripeService stripeService) {
        this.stripeService = stripeService;
    }
    
    @Override
    public void processPayment(double amount, String description) {
        System.out.println("Adapting to Stripe payment system");
        stripeService.charge(amount, description);
        if (stripeService.verifyPayment()) {
            System.out.println("Stripe payment verified successfully");
        }
    }
}
```

### Decorator Pattern

The Decorator pattern adds new functionality to objects dynamically without altering their structure.

```java
import java.util.*;

public class DecoratorPattern {
    public static void main(String[] args) {
        System.out.println("=== Decorator Pattern ===");
        
        // Demonstrate basic decorator
        demonstrateBasicDecorator();
        
        // Demonstrate multiple decorators
        demonstrateMultipleDecorators();
        
        // Demonstrate decorator with state
        demonstrateDecoratorWithState();
        
        // Show real-world decorator example
        demonstrateRealWorldDecorator();
    }
    
    private static void demonstrateBasicDecorator() {
        System.out.println("\n=== Basic Decorator ===");
        
        // Basic coffee
        Coffee coffee = new SimpleCoffee();
        System.out.println("Basic coffee: " + coffee.getDescription() + " - $" + coffee.getCost());
        
        // Decorated coffee with milk
        Coffee milkCoffee = new MilkDecorator(coffee);
        System.out.println("Coffee with milk: " + milkCoffee.getDescription() + " - $" + milkCoffee.getCost());
        
        // Decorated coffee with sugar
        Coffee sugarCoffee = new SugarDecorator(coffee);
        System.out.println("Coffee with sugar: " + sugarCoffee.getDescription() + " - $" + sugarCoffee.getCost());
    }
    
    private static void demonstrateMultipleDecorators() {
        System.out.println("\n=== Multiple Decorators ===");
        
        // Start with basic coffee
        Coffee coffee = new SimpleCoffee();
        System.out.println("Basic coffee: " + coffee.getDescription() + " - $" + coffee.getCost());
        
        // Add multiple decorators
        Coffee decoratedCoffee = new MilkDecorator(
            new SugarDecorator(
                new WhipDecorator(
                    new VanillaDecorator(coffee)
                )
            )
        );
        
        System.out.println("Fully decorated coffee: " + decoratedCoffee.getDescription() + 
                          " - $" + decoratedCoffee.getCost());
    }
    
    private static void demonstrateDecoratorWithState() {
        System.out.println("\n=== Decorator with State ===");
        
        // Text processor with decorators
        TextProcessor processor = new BasicTextProcessor();
        System.out.println("Basic text: " + processor.process("Hello World"));
        
        // Add decorators with state
        TextProcessor upperCaseProcessor = new UpperCaseDecorator(processor);
        TextProcessor reverseProcessor = new ReverseDecorator(upperCaseProcessor);
        TextProcessor prefixProcessor = new PrefixDecorator(reverseProcessor, ">>> ");
        
        System.out.println("Processed text: " + prefixProcessor.process("Hello World"));
        
        // Show that decorators can be combined in different orders
        TextProcessor differentOrder = new PrefixDecorator(
            new ReverseDecorator(
                new UpperCaseDecorator(new BasicTextProcessor())
            ), 
            "### "
        );
        
        System.out.println("Different order: " + differentOrder.process("Hello World"));
    }
    
    private static void demonstrateRealWorldDecorator() {
        System.out.println("\n=== Real-World Decorator Example ===");
        
        // HTTP response with decorators
        HttpResponse response = new BasicHttpResponse(200, "OK", "Hello World");
        System.out.println("Basic response: " + response.getStatusCode() + " - " + response.getBody());
        
        // Add compression
        HttpResponse compressedResponse = new CompressionDecorator(response);
        System.out.println("Compressed response body length: " + compressedResponse.getBody().length());
        
        // Add encryption
        HttpResponse encryptedResponse = new EncryptionDecorator(compressedResponse);
        System.out.println("Encrypted response body: " + encryptedResponse.getBody());
        
        // Add caching
        HttpResponse cachedResponse = new CachingDecorator(encryptedResponse);
        System.out.println("Cached response: " + cachedResponse.isCached());
        
        // Full chain
        HttpResponse fullResponse = new CachingDecorator(
            new EncryptionDecorator(
                new CompressionDecorator(
                    new BasicHttpResponse(200, "OK", "This is a long response that should be compressed")
                )
            )
        );
        
        System.out.println("Full chain response body length: " + fullResponse.getBody().length());
        System.out.println("Full chain is cached: " + fullResponse.isCached());
    }
}

// Basic Decorator pattern - Component interface
interface Coffee {
    double getCost();
    String getDescription();
}

// Concrete component
class SimpleCoffee implements Coffee {
    @Override
    public double getCost() {
        return 2.0;
    }
    
    @Override
    public String getDescription() {
        return "Simple coffee";
    }
}

// Base decorator
abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee;
    
    public CoffeeDecorator(Coffee coffee) {
        this.coffee = coffee;
    }
    
    @Override
    public abstract double getCost();
    
    @Override
    public abstract String getDescription();
}

// Concrete decorators
class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public double getCost() {
        return coffee.getCost() + 0.5;
    }
    
    @Override
    public String getDescription() {
        return coffee.getDescription() + ", milk";
    }
}

class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public double getCost() {
        return coffee.getCost() + 0.2;
    }
    
    @Override
    public String getDescription() {
        return coffee.getDescription() + ", sugar";
    }
}

class WhipDecorator extends CoffeeDecorator {
    public WhipDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public double getCost() {
        return coffee.getCost() + 0.7;
    }
    
    @Override
    public String getDescription() {
        return coffee.getDescription() + ", whip";
    }
}

class VanillaDecorator extends CoffeeDecorator {
    public VanillaDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public double getCost() {
        return coffee.getCost() + 0.3;
    }
    
    @Override
    public String getDescription() {
        return coffee.getDescription() + ", vanilla";
    }
}

// Decorator with state example
interface TextProcessor {
    String process(String text);
}

class BasicTextProcessor implements TextProcessor {
    @Override
    public String process(String text) {
        return text;
    }
}

abstract class TextProcessorDecorator implements TextProcessor {
    protected TextProcessor processor;
    
    public TextProcessorDecorator(TextProcessor processor) {
        this.processor = processor;
    }
    
    @Override
    public abstract String process(String text);
}

class UpperCaseDecorator extends TextProcessorDecorator {
    public UpperCaseDecorator(TextProcessor processor) {
        super(processor);
    }
    
    @Override
    public String process(String text) {
        return processor.process(text).toUpperCase();
    }
}

class ReverseDecorator extends TextProcessorDecorator {
    public ReverseDecorator(TextProcessor processor) {
        super(processor);
    }
    
    @Override
    public String process(String text) {
        return new StringBuilder(processor.process(text)).reverse().toString();
    }
}

class PrefixDecorator extends TextProcessorDecorator {
    private String prefix;
    
    public PrefixDecorator(TextProcessor processor, String prefix) {
        super(processor);
        this.prefix = prefix;
    }
    
    @Override
    public String process(String text) {
        return prefix + processor.process(text);
    }
}

// Real-world decorator example
interface HttpResponse {
    int getStatusCode();
    String getStatusMessage();
    String getBody();
    boolean isCached();
}

class BasicHttpResponse implements HttpResponse {
    private int statusCode;
    private String statusMessage;
    private String body;
    
    public BasicHttpResponse(int statusCode, String statusMessage, String body) {
        this.statusCode = statusCode;
        this.statusMessage = statusMessage;
        this.body = body;
    }
    
    @Override
    public int getStatusCode() {
        return statusCode;
    }
    
    @Override
    public String getStatusMessage() {
        return statusMessage;
    }
    
    @Override
    public String getBody() {
        return body;
    }
    
    @Override
    public boolean isCached() {
        return false;
    }
}

abstract class HttpResponseDecorator implements HttpResponse {
    protected HttpResponse response;
    
    public HttpResponseDecorator(HttpResponse response) {
        this.response = response;
    }
    
    @Override
    public int getStatusCode() {
        return response.getStatusCode();
    }
    
    @Override
    public String getStatusMessage() {
        return response.getStatusMessage();
    }
    
    @Override
    public String getBody() {
        return response.getBody();
    }
    
    @Override
    public boolean isCached() {
        return response.isCached();
    }
}

class CompressionDecorator extends HttpResponseDecorator {
    public CompressionDecorator(HttpResponse response) {
        super(response);
    }
    
    @Override
    public String getBody() {
        String body = response.getBody();
        // Simple compression simulation
        return "COMPRESSED:" + body.length() + ":" + body;
    }
}

class EncryptionDecorator extends HttpResponseDecorator {
    public EncryptionDecorator(HttpResponse response) {
        super(response);
    }
    
    @Override
    public String getBody() {
        String body = response.getBody();
        // Simple encryption simulation
        return "ENCRYPTED:" + Base64.getEncoder().encodeToString(body.getBytes());
    }
}

class CachingDecorator extends HttpResponseDecorator {
    public CachingDecorator(HttpResponse response) {
        super(response);
    }
    
    @Override
    public boolean isCached() {
        return true;
    }
}
```

### Facade Pattern

The Facade pattern provides a simplified interface to a complex subsystem.

```java
import java.util.*;

public class FacadePattern {
    public static void main(String[] args) {
        System.out.println("=== Facade Pattern ===");
        
        // Demonstrate basic facade
        demonstrateBasicFacade();
        
        // Demonstrate facade with subsystem complexity
        demonstrateComplexFacade();
        
        // Demonstrate multiple facades
        demonstrateMultipleFacades();
        
        // Show real-world facade example
        demonstrateRealWorldFacade();
    }
    
    private static void demonstrateBasicFacade() {
        System.out.println("\n=== Basic Facade ===");
        
        // Without facade - complex subsystem
        CPU cpu = new CPU();
        Memory memory = new Memory();
        HardDrive hardDrive = new HardDrive();
        
        // Complex startup process
        cpu.freeze();
        memory.load(0, hardDrive.read(0, 1024));
        cpu.jump(0);
        cpu.execute();
        
        // With facade - simplified interface
        ComputerFacade computer = new ComputerFacade();
        computer.start();
        computer.shutdown();
    }
    
    private static void demonstrateComplexFacade() {
        System.out.println("\n=== Complex Facade ===");
        
        // Video conversion facade
        VideoConversionFacade converter = new VideoConversionFacade();
        
        // Simple interface for complex conversion process
        String result1 = converter.convertVideo("input.mp4", "avi");
        System.out.println("Conversion result 1: " + result1);
        
        String result2 = converter.convertVideo("input.avi", "mp4");
        System.out.println("Conversion result 2: " + result2);
        
        String result3 = converter.convertVideo("input.mkv", "webm");
        System.out.println("Conversion result 3: " + result3);
    }
    
    private static void demonstrateMultipleFacades() {
        System.out.println("\n=== Multiple Facades ===");
        
        // Different facades for different subsystems
        DatabaseFacade dbFacade = new DatabaseFacade();
        FileFacade fileFacade = new FileFacade();
        NetworkFacade networkFacade = new NetworkFacade();
        
        // Database operations through facade
        dbFacade.connect("localhost", "mydb", "user", "pass");
        dbFacade.executeQuery("SELECT * FROM users");
        dbFacade.disconnect();
        
        // File operations through facade
        fileFacade.createFile("test.txt");
        fileFacade.writeFile("test.txt", "Hello World");
        fileFacade.readFile("test.txt");
        fileFacade.deleteFile("test.txt");
        
        // Network operations through facade
        networkFacade.openConnection("http://example.com");
        networkFacade.sendData("GET / HTTP/1.1");
        networkFacade.receiveData();
        networkFacade.closeConnection();
    }
    
    private static void demonstrateRealWorldFacade() {
        System.out.println("\n=== Real-World Facade Example ===");
        
        // E-commerce system facade
        ECommerceFacade ecommerce = new ECommerceFacade();
        
        // Simple interface for complex e-commerce operations
        ecommerce.placeOrder("user123", Arrays.asList("item1", "item2", "item3"), 150.0);
        
        ecommerce.processPayment("order456", 150.0, "credit_card");
        
        ecommerce.shipOrder("order456", "123 Main St, City, State");
        
        ecommerce.sendNotification("user123", "Your order has been shipped");
    }
}

// Basic Facade pattern example
class CPU {
    public void freeze() {
        System.out.println("CPU: Freezing processor");
    }
    
    public void jump(long position) {
        System.out.println("CPU: Jumping to position " + position);
    }
    
    public void execute() {
        System.out.println("CPU: Executing instructions");
    }
}

class Memory {
    public void load(long position, byte[] data) {
        System.out.println("Memory: Loading data at position " + position);
    }
    
    public byte[] load(long position, int size) {
        System.out.println("Memory: Loading " + size + " bytes from position " + position);
        return new byte[size];
    }
}

class HardDrive {
    public byte[] read(long lba, int size) {
        System.out.println("HardDrive: Reading " + size + " bytes from LBA " + lba);
        return new byte[size];
    }
    
    public void write(long lba, byte[] data) {
        System.out.println("HardDrive: Writing " + data.length + " bytes to LBA " + lba);
    }
}

// Facade class
class ComputerFacade {
    private CPU cpu;
    private Memory memory;
    private HardDrive hardDrive;
    
    public ComputerFacade() {
        this.cpu = new CPU();
        this.memory = new Memory();
        this.hardDrive = new HardDrive();
    }
    
    public void start() {
        System.out.println("=== Starting computer ===");
        cpu.freeze();
        memory.load(0, hardDrive.read(0, 1024));
        cpu.jump(0);
        cpu.execute();
        System.out.println("Computer started successfully");
    }
    
    public void shutdown() {
        System.out.println("=== Shutting down computer ===");
        cpu.freeze();
        System.out.println("Computer shut down successfully");
    }
}

// Complex Facade example - Video conversion
class VideoFile {
    private String name;
    private String codecType;
    
    public VideoFile(String name) {
        this.name = name;
        this.codecType = name.substring(name.indexOf(".") + 1);
    }
    
    public String getName() {
        return name;
    }
    
    public String getCodecType() {
        return codecType;
    }
}

class Codec {
    // Codec implementation
}

class MPEG4CompressionCodec extends Codec {
    public String type = "mp4";
}

class OggCompressionCodec extends Codec {
    public String type = "ogg";
}

class VideoConverter {
    public VideoFile convert(VideoFile file, Codec codec) {
        System.out.println("VideoConverter: Converting " + file.getName() + " to " + codec.getClass().getSimpleName());
        return new VideoFile(file.getName().replace(file.getCodecType(), codec.getClass().getSimpleName().toLowerCase().replace("compressioncodec", "")));
    }
}

class AudioMixer {
    public byte[] fix(VideoFile result) {
        System.out.println("AudioMixer: Fixing audio in " + result.getName());
        return new byte[1024];
    }
}

// Complex facade
class VideoConversionFacade {
    public String convertVideo(String fileName, String format) {
        System.out.println("=== Video Conversion Facade ===");
        System.out.println("Converting " + fileName + " to " + format);
        
        VideoFile file = new VideoFile(fileName);
        Codec sourceCodec = getCodec(file);
        Codec destinationCodec = getCodec(new VideoFile("temp." + format));
        
        VideoConverter converter = new VideoConverter();
        VideoFile intermediate = converter.convert(file, sourceCodec);
        
        VideoFile result = converter.convert(intermediate, destinationCodec);
        AudioMixer mixer = new AudioMixer();
        byte[] resultBytes = mixer.fix(result);
        
        System.out.println("Conversion completed successfully");
        return result.getName();
    }
    
    private Codec getCodec(VideoFile file) {
        String type = file.getCodecType();
        if (type.equals("mp4")) {
            return new MPEG4CompressionCodec();
        } else {
            return new OggCompressionCodec();
        }
    }
}

// Multiple facades example
class DatabaseFacade {
    public void connect(String host, String database, String user, String password) {
        System.out.println("Database: Connecting to " + database + " on " + host);
        // Complex connection logic
    }
    
    public void executeQuery(String query) {
        System.out.println("Database: Executing query: " + query);
        // Query execution logic
    }
    
    public void disconnect() {
        System.out.println("Database: Disconnecting");
        // Disconnection logic
    }
}

class FileFacade {
    public void createFile(String filename) {
        System.out.println("File: Creating file " + filename);
        // File creation logic
    }
    
    public void writeFile(String filename, String content) {
        System.out.println("File: Writing to " + filename + ": " + content);
        // File writing logic
    }
    
    public String readFile(String filename) {
        System.out.println("File: Reading from " + filename);
        // File reading logic
        return "File content";
    }
    
    public void deleteFile(String filename) {
        System.out.println("File: Deleting " + filename);
        // File deletion logic
    }
}

class NetworkFacade {
    public void openConnection(String url) {
        System.out.println("Network: Opening connection to " + url);
        // Connection logic
    }
    
    public void sendData(String data) {
        System.out.println("Network: Sending data: " + data);
        // Send logic
    }
    
    public String receiveData() {
        System.out.println("Network: Receiving data");
        // Receive logic
        return "Received data";
    }
    
    public void closeConnection() {
        System.out.println("Network: Closing connection");
        // Close logic
    }
}

// Real-world facade example - E-commerce system
class ECommerceFacade {
    private OrderService orderService;
    private PaymentService paymentService;
    private InventoryService inventoryService;
    private ShippingService shippingService;
    private NotificationService notificationService;
    
    public ECommerceFacade() {
        this.orderService = new OrderService();
        this.paymentService = new PaymentService();
        this.inventoryService = new InventoryService();
        this.shippingService = new ShippingService();
        this.notificationService = new NotificationService();
    }
    
    public void placeOrder(String userId, List<String> items, double totalAmount) {
        System.out.println("=== Placing Order ===");
        
        // Check inventory
        if (!inventoryService.checkAvailability(items)) {
            System.out.println("Some items are not available");
            return;
        }
        
        // Create order
        String orderId = orderService.createOrder(userId, items, totalAmount);
        System.out.println("Order created with ID: " + orderId);
        
        // Reserve inventory
        inventoryService.reserveItems(items);
        System.out.println("Items reserved in inventory");
    }
    
    public void processPayment(String orderId, double amount, String paymentMethod) {
        System.out.println("=== Processing Payment ===");
        
        // Process payment
        boolean paymentSuccess = paymentService.processPayment(orderId, amount, paymentMethod);
        if (paymentSuccess) {
            System.out.println("Payment processed successfully");
            orderService.updateOrderStatus(orderId, "PAID");
        } else {
            System.out.println("Payment failed");
            orderService.updateOrderStatus(orderId, "PAYMENT_FAILED");
        }
    }
    
    public void shipOrder(String orderId, String shippingAddress) {
        System.out.println("=== Shipping Order ===");
        
        // Generate shipping label
        String trackingNumber = shippingService.generateShippingLabel(orderId, shippingAddress);
        System.out.println("Shipping label generated with tracking: " + trackingNumber);
        
        // Update order status
        orderService.updateOrderStatus(orderId, "SHIPPED");
        
        // Release inventory reservation
        // inventoryService.releaseReservation(orderId);
    }
    
    public void sendNotification(String userId, String message) {
        System.out.println("=== Sending Notification ===");
        
        // Send notification
        notificationService.sendEmail(userId, message);
        notificationService.sendSMS(userId, message);
        System.out.println("Notification sent to user " + userId);
    }
}

// Supporting services for e-commerce facade
class OrderService {
    public String createOrder(String userId, List<String> items, double totalAmount) {
        return "ORDER_" + System.currentTimeMillis();
    }
    
    public void updateOrderStatus(String orderId, String status) {
        System.out.println("Order " + orderId + " status updated to " + status);
    }
}

class PaymentService {
    public boolean processPayment(String orderId, double amount, String paymentMethod) {
        System.out.println("Processing " + paymentMethod + " payment of $" + amount + " for order " + orderId);
        return true; // Simulate success
    }
}

class InventoryService {
    public boolean checkAvailability(List<String> items) {
        System.out.println("Checking availability for " + items.size() + " items");
        return true; // Simulate availability
    }
    
    public void reserveItems(List<String> items) {
        System.out.println("Reserving " + items.size() + " items");
    }
}

class ShippingService {
    public String generateShippingLabel(String orderId, String address) {
        System.out.println("Generating shipping label for order " + orderId + " to " + address);
        return "TRK" + System.currentTimeMillis();
    }
}

class NotificationService {
    public void sendEmail(String userId, String message) {
        System.out.println("Sending email to user " + userId + ": " + message);
    }
    
    public void sendSMS(String userId, String message) {
        System.out.println("Sending SMS to user " + userId + ": " + message);
    }
}
```

## Behavioral Patterns

### Observer Pattern

The Observer pattern defines a one-to-many dependency between objects so that when one object changes state, all dependents are notified automatically.

```java
import java.util.*;

public class ObserverPattern {
    public static void main(String[] args) {
        System.out.println("=== Observer Pattern ===");
        
        // Demonstrate basic observer
        demonstrateBasicObserver();
        
        // Demonstrate observer with different types of observers
        demonstrateDifferentObservers();
        
        // Demonstrate observer with state management
        demonstrateObserverWithState();
        
        // Show real-world observer example
        demonstrateRealWorldObserver();
    }
    
    private static void demonstrateBasicObserver() {
        System.out.println("\n=== Basic Observer ===");
        
        // Create subject and observers
        NewsAgency agency = new NewsAgency();
        
        NewsChannel channel1 = new NewsChannel("CNN");
        NewsChannel channel2 = new NewsChannel("BBC");
        NewsChannel channel3 = new NewsChannel("Al Jazeera");
        
        // Register observers
        agency.addObserver(channel1);
        agency.addObserver(channel2);
        agency.addObserver(channel3);
        
        // Publish news - all observers will be notified
        agency.setNews("Breaking: Major technology breakthrough announced!");
        
        // Remove one observer
        agency.removeObserver(channel2);
        
        // Publish more news - only remaining observers notified
        agency.setNews("Update: Stock markets rally after economic news");
    }
    
    private static void demonstrateDifferentObservers() {
        System.out.println("\n=== Different Types of Observers ===");
        
        // Weather station with different types of displays
        WeatherStation station = new WeatherStation();
        
        CurrentConditionsDisplay currentDisplay = new CurrentConditionsDisplay();
        StatisticsDisplay statsDisplay = new StatisticsDisplay();
        ForecastDisplay forecastDisplay = new ForecastDisplay();
        
        // Register all displays
        station.addObserver(currentDisplay);
        station.addObserver(statsDisplay);
        station.addObserver(forecastDisplay);
        
        // Weather changes - all displays update
        station.setMeasurements(25.0, 65.0, 30.4);
        System.out.println();
        station.setMeasurements(27.0, 70.0, 29.2);
        System.out.println();
        station.setMeasurements(23.0, 90.0, 29.8);
    }
    
    private static void demonstrateObserverWithState() {
        System.out.println("\n=== Observer with State Management ===");
        
        // Stock market with multiple observers tracking different aspects
        StockMarket market = new StockMarket();
        
        // Different types of observers with state
        PriceAlertObserver alertObserver = new PriceAlertObserver("AAPL", 150.0);
        PortfolioTracker portfolioTracker = new PortfolioTracker();
        TradingBot tradingBot = new TradingBot();
        
        market.addObserver(alertObserver);
        market.addObserver(portfolioTracker);
        market.addObserver(tradingBot);
        
        // Stock price changes
        market.updateStockPrice("AAPL", 145.0);
        market.updateStockPrice("AAPL", 155.0);
        market.updateStockPrice("GOOGL", 2800.0);
        market.updateStockPrice("MSFT", 300.0);
        
        // Show portfolio state
        System.out.println("Portfolio value: $" + portfolioTracker.getPortfolioValue());
        System.out.println("Number of trades: " + tradingBot.getTradeCount());
    }
    
    private static void demonstrateRealWorldObserver() {
        System.out.println("\n=== Real-World Observer Example ===");
        
        // Event management system
        EventManager eventManager = new EventManager();
        
        // Different types of listeners
        EmailNotificationListener emailListener = new EmailNotificationListener();
        SMSService smsService = new SMSService();
        LoggingService logger = new LoggingService();
        AnalyticsService analytics = new AnalyticsService();
        
        // Register listeners for different events
        eventManager.subscribe("user_registration", emailListener);
        eventManager.subscribe("user_registration", logger);
        eventManager.subscribe("purchase", smsService);
        eventManager.subscribe("purchase", analytics);
        eventManager.subscribe("login", logger);
        
        // Trigger events
        eventManager.notify("user_registration", new EventData("user123", "john@example.com"));
        eventManager.notify("purchase", new EventData("order456", "$99.99"));
        eventManager.notify("login", new EventData("user123", "2023-12-01 10:30:00"));
        
        // Show analytics
        System.out.println("Total events processed: " + analytics.getEventCount());
    }
}

// Basic Observer pattern implementation
interface Observer {
    void update(String news);
}

interface Subject {
    void addObserver(Observer observer);
    void removeObserver(Observer observer);
    void notifyObservers();
}

class NewsAgency implements Subject {
    private List<Observer> observers;
    private String news;
    
    public NewsAgency() {
        this.observers = new ArrayList<>();
    }
    
    @Override
    public void addObserver(Observer observer) {
        observers.add(observer);
    }
    
    @Override
    public void removeObserver(Observer observer) {
        observers.remove(observer);
    }
    
    @Override
    public void notifyObservers() {
        for (Observer observer : observers) {
            observer.update(news);
        }
    }
    
    public void setNews(String news) {
        this.news = news;
        notifyObservers();
    }
    
    public String getNews() {
        return news;
    }
}

class NewsChannel implements Observer {
    private String name;
    
    public NewsChannel(String name) {
        this.name = name;
    }
    
    @Override
    public void update(String news) {
        System.out.println(name + " received news: " + news);
    }
    
    public String getName() {
        return name;
    }
}

// Observer with different types of observers
interface WeatherObserver {
    void update(double temperature, double humidity, double pressure);
}

class WeatherStation {
    private List<WeatherObserver> observers;
    private double temperature;
    private double humidity;
    private double pressure;
    
    public WeatherStation() {
        this.observers = new ArrayList<>();
    }
    
    public void addObserver(WeatherObserver observer) {
        observers.add(observer);
    }
    
    public void removeObserver(WeatherObserver observer) {
        observers.remove(observer);
    }
    
    public void notifyObservers() {
        for (WeatherObserver observer : observers) {
            observer.update(temperature, humidity, pressure);
        }
    }
    
    public void setMeasurements(double temperature, double humidity, double pressure) {
        this.temperature = temperature;
        this.humidity = humidity;
        this.pressure = pressure;
        measurementsChanged();
    }
    
    private void measurementsChanged() {
        notifyObservers();
    }
}

class CurrentConditionsDisplay implements WeatherObserver {
    @Override
    public void update(double temperature, double humidity, double pressure) {
        System.out.println("Current Conditions:");
        System.out.println("  Temperature: " + temperature + "°C");
        System.out.println("  Humidity: " + humidity + "%");
        System.out.println("  Pressure: " + pressure + " hPa");
    }
}

class StatisticsDisplay implements WeatherObserver {
    private double maxTemp = 0.0;
    private double minTemp = 200.0;
    private double tempSum = 0.0;
    private int numReadings = 0;
    
    @Override
    public void update(double temperature, double humidity, double pressure) {
        tempSum += temperature;
        numReadings++;
        
        if (temperature > maxTemp) {
            maxTemp = temperature;
        }
        
        if (temperature < minTemp) {
            minTemp = temperature;
        }
        
        System.out.println("Weather Statistics:");
        System.out.println("  Avg/Max/Min temperature: " + (tempSum / numReadings) + 
                          "/" + maxTemp + "/" + minTemp);
    }
}

class ForecastDisplay implements WeatherObserver {
    private double currentPressure = 29.92;
    private double lastPressure;
    
    @Override
    public void update(double temperature, double humidity, double pressure) {
        lastPressure = currentPressure;
        currentPressure = pressure;
        
        System.out.print("Forecast: ");
        if (currentPressure > lastPressure) {
            System.out.println("Improving weather on the way!");
        } else if (currentPressure == lastPressure) {
            System.out.println("More of the same");
        } else if (currentPressure < lastPressure) {
            System.out.println("Watch out for cooler, rainy weather");
        }
    }
}

// Observer with state management
class StockMarket {
    private Map<String, Double> stockPrices;
    private List<StockObserver> observers;
    
    public StockMarket() {
        this.stockPrices = new HashMap<>();
        this.observers = new ArrayList<>();
    }
    
    public void addObserver(StockObserver observer) {
        observers.add(observer);
    }
    
    public void removeObserver(StockObserver observer) {
        observers.remove(observer);
    }
    
    public void notifyObservers(String stockSymbol, double price) {
        for (StockObserver observer : observers) {
            observer.update(stockSymbol, price);
        }
    }
    
    public void updateStockPrice(String stockSymbol, double price) {
        stockPrices.put(stockSymbol, price);
        System.out.println("Stock " + stockSymbol + " updated to $" + price);
        notifyObservers(stockSymbol, price);
    }
    
    public Double getStockPrice(String stockSymbol) {
        return stockPrices.get(stockSymbol);
    }
}

interface StockObserver {
    void update(String stockSymbol, double price);
}

class PriceAlertObserver implements StockObserver {
    private String targetStock;
    private double targetPrice;
    
    public PriceAlertObserver(String targetStock, double targetPrice) {
        this.targetStock = targetStock;
        this.targetPrice = targetPrice;
    }
    
    @Override
    public void update(String stockSymbol, double price) {
        if (stockSymbol.equals(targetStock) && price >= targetPrice) {
            System.out.println("ALERT: " + stockSymbol + " reached target price of $" + targetPrice + 
                             " (current: $" + price + ")");
        }
    }
}

class PortfolioTracker implements StockObserver {
    private Map<String, Integer> holdings;
    private double portfolioValue;
    
    public PortfolioTracker() {
        this.holdings = new HashMap<>();
        this.holdings.put("AAPL", 10);
        this.holdings.put("GOOGL", 5);
        this.holdings.put("MSFT", 8);
        this.portfolioValue = 0.0;
    }
    
    @Override
    public void update(String stockSymbol, double price) {
        if (holdings.containsKey(stockSymbol)) {
            int shares = holdings.get(stockSymbol);
            portfolioValue += shares * price;
            System.out.println("Portfolio updated for " + stockSymbol + ": " + shares + 
                             " shares @ $" + price);
        }
    }
    
    public double getPortfolioValue() {
        return portfolioValue;
    }
}

class TradingBot implements StockObserver {
    private int tradeCount;
    
    public TradingBot() {
        this.tradeCount = 0;
    }
    
    @Override
    public void update(String stockSymbol, double price) {
        // Simple trading logic
        if (price > 150.0 && stockSymbol.equals("AAPL")) {
            System.out.println("Trading Bot: Buying 1 share of " + stockSymbol + " @ $" + price);
            tradeCount++;
        } else if (price < 140.0 && stockSymbol.equals("AAPL")) {
            System.out.println("Trading Bot: Selling 1 share of " + stockSymbol + " @ $" + price);
            tradeCount++;
        }
    }
    
    public int getTradeCount() {
        return tradeCount;
    }
}

// Real-world observer example - Event management system
class EventManager {
    private Map<String, List<EventListener>> listeners;
    
    public EventManager() {
        this.listeners = new HashMap<>();
    }
    
    public void subscribe(String eventType, EventListener listener) {
        listeners.computeIfAbsent(eventType, k -> new ArrayList<>()).add(listener);
    }
    
    public void unsubscribe(String eventType, EventListener listener) {
        if (listeners.containsKey(eventType)) {
            listeners.get(eventType).remove(listener);
        }
    }
    
    public void notify(String eventType, EventData data) {
        if (listeners.containsKey(eventType)) {
            System.out.println("Event '" + eventType + "' triggered");
            for (EventListener listener : listeners.get(eventType)) {
                listener.update(data);
            }
        }
    }
}

interface EventListener {
    void update(EventData data);
}

class EventData {
    private String id;
    private String data;
    private long timestamp;
    
    public EventData(String id, String data) {
        this.id = id;
        this.data = data;
        this.timestamp = System.currentTimeMillis();
    }
    
    public String getId() { return id; }
    public String getData() { return data; }
    public long getTimestamp() { return timestamp; }
    
    @Override
    public String toString() {
        return "EventData{id='" + id + "', data='" + data + "', timestamp=" + timestamp + "}";
    }
}

class EmailNotificationListener implements EventListener {
    @Override
    public void update(EventData data) {
        System.out.println("Email notification sent: " + data.getData());
    }
}

class SMSService implements EventListener {
    @Override
    public void update(EventData data) {
        System.out.println("SMS sent: " + data.getData());
    }
}

class LoggingService implements EventListener {
    @Override
    public void update(EventData data) {
        System.out.println("Logged event: " + data);
    }
}

class AnalyticsService implements EventListener {
    private int eventCount;
    
    public AnalyticsService() {
        this.eventCount = 0;
    }
    
    @Override
    public void update(EventData data) {
        eventCount++;
        System.out.println("Analytics recorded event #" + eventCount);
    }
    
    public int getEventCount() {
        return eventCount;
    }
}
```

### Strategy Pattern

The Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable.

```java
import java.util.*;

public class StrategyPattern {
    public static void main(String[] args) {
        System.out.println("=== Strategy Pattern ===");
        
        // Demonstrate basic strategy
        demonstrateBasicStrategy();
        
        // Demonstrate strategy with different algorithms
        demonstrateDifferentStrategies();
        
        // Demonstrate strategy with context
        demonstrateStrategyWithContext();
        
        // Show real-world strategy example
        demonstrateRealWorldStrategy();
    }
    
    private static void demonstrateBasicStrategy() {
        System.out.println("\n=== Basic Strategy ===");
        
        // Different sorting strategies
        SortStrategy bubbleSort = new BubbleSortStrategy();
        SortStrategy quickSort = new QuickSortStrategy();
        SortStrategy mergeSort = new MergeSortStrategy();
        
        int[] array1 = {64, 34, 25, 12, 22, 11, 90};
        int[] array2 = {64, 34, 25, 12, 22, 11, 90};
        int[] array3 = {64, 34, 25, 12, 22, 11, 90};
        
        System.out.println("Original array: " + Arrays.toString(array1));
        
        bubbleSort.sort(array1);
        System.out.println("Bubble sorted: " + Arrays.toString(array1));
        
        quickSort.sort(array2);
        System.out.println("Quick sorted: " + Arrays.toString(array2));
        
        mergeSort.sort(array3);
        System.out.println("Merge sorted: " + Arrays.toString(array3));
    }
    
    private static void demonstrateDifferentStrategies() {
        System.out.println("\n=== Different Strategies ===");
        
        // Compression strategies
        CompressionStrategy zipStrategy = new ZipCompressionStrategy();
        CompressionStrategy rarStrategy = new RarCompressionStrategy();
        CompressionStrategy gzipStrategy = new GzipCompressionStrategy();
        
        // Files to compress
        List<String> files = Arrays.asList("file1.txt", "file2.jpg", "file3.pdf");
        
        System.out.println("Files to compress: " + files);
        
        zipStrategy.compress(files);
        rarStrategy.compress(files);
        gzipStrategy.compress(files);
    }
    
    private static void demonstrateStrategyWithContext() {
        System.out.println("\n=== Strategy with Context ===");
        
        // Navigation context with different route strategies
        NavigationContext context = new NavigationContext();
        
        RouteStrategy carStrategy = new CarRouteStrategy();
        RouteStrategy bikeStrategy = new BikeRouteStrategy();
        RouteStrategy walkingStrategy = new WalkingRouteStrategy();
        
        String start = "Home";
        String end = "Office";
        
        System.out.println("Navigation from " + start + " to " + end + ":");
        
        context.setRouteStrategy(carStrategy);
        context.buildRoute(start, end);
        
        context.setRouteStrategy(bikeStrategy);
        context.buildRoute(start, end);
        
        context.setRouteStrategy(walkingStrategy);
        context.buildRoute(start, end);
    }
    
    private static void demonstrateRealWorldStrategy() {
        System.out.println("\n=== Real-World Strategy Example ===");
        
        // Payment processing with different strategies
        PaymentContext paymentContext = new PaymentContext();
        
        PaymentStrategy creditCardStrategy = new CreditCardStrategy("John Doe", "1234567890123456", 
                                                                   "123", "12/25");
        PaymentStrategy paypalStrategy = new PaypalStrategy("john@example.com");
        PaymentStrategy bitcoinStrategy = new BitcoinStrategy("1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa");
        
        // Process payments with different strategies
        paymentContext.setPaymentStrategy(creditCardStrategy);
        paymentContext.processPayment(100.0);
        
        paymentContext.setPaymentStrategy(paypalStrategy);
        paymentContext.processPayment(150.0);
        
        paymentContext.setPaymentStrategy(bitcoinStrategy);
        paymentContext.processPayment(200.0);
    }
}

// Basic Strategy pattern - Strategy interface
interface SortStrategy {
    void sort(int[] array);
}

// Concrete strategies
class BubbleSortStrategy implements SortStrategy {
    @Override
    public void sort(int[] array) {
        System.out.println("Sorting using Bubble Sort");
        int n = array.length;
        for (int i = 0; i < n - 1; i++) {
            for (int j = 0; j < n - i - 1; j++) {
                if (array[j] > array[j + 1]) {
                    // Swap elements
                    int temp = array[j];
                    array[j] = array[j + 1];
                    array[j + 1] = temp;
                }
            }
        }
    }
}

class QuickSortStrategy implements SortStrategy {
    @Override
    public void sort(int[] array) {
        System.out.println("Sorting using Quick Sort");
        quickSort(array, 0, array.length - 1);
    }
    
    private void quickSort(int[] array, int low, int high) {
        if (low < high) {
            int pi = partition(array, low, high);
            quickSort(array, low, pi - 1);
            quickSort(array, pi + 1, high);
        }
    }
    
    private int partition(int[] array, int low, int high) {
        int pivot = array[high];
        int i = (low - 1);
        
        for (int j = low; j < high; j++) {
            if (array[j] <= pivot) {
                i++;
                int temp = array[i];
                array[i] = array[j];
                array[j] = temp;
            }
        }
        
        int temp = array[i + 1];
        array[i + 1] = array[high];
        array[high] = temp;
        
        return i + 1;
    }
}

class MergeSortStrategy implements SortStrategy {
    @Override
    public void sort(int[] array) {
        System.out.println("Sorting using Merge Sort");
        mergeSort(array, 0, array.length - 1);
    }
    
    private void mergeSort(int[] array, int left, int right) {
        if (left < right) {
            int mid = (left + right) / 2;
            mergeSort(array, left, mid);
            mergeSort(array, mid + 1, right);
            merge(array, left, mid, right);
        }
    }
    
    private void merge(int[] array, int left, int mid, int right) {
        int n1 = mid - left + 1;
        int n2 = right - mid;
        
        int[] leftArray = new int[n1];
        int[] rightArray = new int[n2];
        
        System.arraycopy(array, left, leftArray, 0, n1);
        System.arraycopy(array, mid + 1, rightArray, 0, n2);
        
        int i = 0, j = 0, k = left;
        while (i < n1 && j < n2) {
            if (leftArray[i] <= rightArray[j]) {
                array[k] = leftArray[i];
                i++;
            } else {
                array[k] = rightArray[j];
                j++;
            }
            k++;
        }
        
        while (i < n1) {
            array[k] = leftArray[i];
            i++;
            k++;
        }
        
        while (j < n2) {
            array[k] = rightArray[j];
            j++;
            k++;
        }
    }
}

// Strategy with different algorithms
interface CompressionStrategy {
    void compress(List<String> files);
}

class ZipCompressionStrategy implements CompressionStrategy {
    @Override
    public void compress(List<String> files) {
        System.out.println("Compressing files using ZIP algorithm:");
        for (String file : files) {
            System.out.println("  Compressing " + file + " with ZIP");
        }
        System.out.println("ZIP compression completed");
    }
}

class RarCompressionStrategy implements CompressionStrategy {
    @Override
    public void compress(List<String> files) {
        System.out.println("Compressing files using RAR algorithm:");
        for (String file : files) {
            System.out.println("  Compressing " + file + " with RAR");
        }
        System.out.println("RAR compression completed");
    }
}

class GzipCompressionStrategy implements CompressionStrategy {
    @Override
    public void compress(List<String> files) {
        System.out.println("Compressing files using GZIP algorithm:");
        for (String file : files) {
            System.out.println("  Compressing " + file + " with GZIP");
        }
        System.out.println("GZIP compression completed");
    }
}

// Strategy with context
interface RouteStrategy {
    void buildRoute(String start, String end);
}

class NavigationContext {
    private RouteStrategy routeStrategy;
    
    public void setRouteStrategy(RouteStrategy routeStrategy) {
        this.routeStrategy = routeStrategy;
    }
    
    public void buildRoute(String start, String end) {
        if (routeStrategy != null) {
            routeStrategy.buildRoute(start, end);
        } else {
            System.out.println("No route strategy set");
        }
    }
}

class CarRouteStrategy implements RouteStrategy {
    @Override
    public void buildRoute(String start, String end) {
        System.out.println("Building car route from " + start + " to " + end);
        System.out.println("  Route: Highway 101 -> Main St -> Destination");
        System.out.println("  Distance: 15.2 miles");
        System.out.println("  Estimated time: 25 minutes");
    }
}

class BikeRouteStrategy implements RouteStrategy {
    @Override
    public void buildRoute(String start, String end) {
        System.out.println("Building bike route from " + start + " to " + end);
        System.out.println("  Route: Bike path -> Park Ave -> Destination");
        System.out.println("  Distance: 12.8 miles");
        System.out.println("  Estimated time: 65 minutes");
    }
}

class WalkingRouteStrategy implements RouteStrategy {
    @Override
    public void buildRoute(String start, String end) {
        System.out.println("Building walking route from " + start + " to " + end);
        System.out.println("  Route: Sidewalk -> Crosswalk -> Destination");
        System.out.println("  Distance: 10.5 miles");
        System.out.println("  Estimated time: 3 hours 10 minutes");
    }
}

// Real-world strategy example
interface PaymentStrategy {
    void pay(double amount);
}

class PaymentContext {
    private PaymentStrategy paymentStrategy;
    
    public void setPaymentStrategy(PaymentStrategy paymentStrategy) {
        this.paymentStrategy = paymentStrategy;
    }
    
    public void processPayment(double amount) {
        if (paymentStrategy != null) {
            System.out.println("Processing payment of $" + amount);
            paymentStrategy.pay(amount);
            System.out.println("Payment completed successfully");
        } else {
            System.out.println("No payment strategy set");
        }
    }
}

class CreditCardStrategy implements PaymentStrategy {
    private String name;
    private String cardNumber;
    private String cvv;
    private String dateOfExpiry;
    
    public CreditCardStrategy(String name, String cardNumber, String cvv, String dateOfExpiry) {
        this.name = name;
        this.cardNumber = cardNumber;
        this.cvv = cvv;
        this.dateOfExpiry = dateOfExpiry;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " using Credit Card");
        System.out.println("  Cardholder: " + name);
        System.out.println("  Card ending in: " + cardNumber.substring(cardNumber.length() - 4));
    }
}

class PaypalStrategy implements PaymentStrategy {
    private String email;
    
    public PaypalStrategy(String email) {
        this.email = email;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " using PayPal");
        System.out.println("  PayPal account: " + email);
    }
}

class BitcoinStrategy implements PaymentStrategy {
    private String walletAddress;
    
    public BitcoinStrategy(String walletAddress) {
        this.walletAddress = walletAddress;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " using Bitcoin");
        System.out.println("  Wallet address: " + walletAddress);
        System.out.println("  BTC equivalent: " + (amount / 20000.0) + " BTC"); // Simplified conversion
    }
}
```

### Command Pattern

The Command pattern encapsulates a request as an object, thereby allowing for parameterization of clients with queues, requests, and operations.

```java
import java.util.*;

public class CommandPattern {
    public static void main(String[] args) {
        System.out.println("=== Command Pattern ===");
        
        // Demonstrate basic command
        demonstrateBasicCommand();
        
        // Demonstrate command with undo/redo
        demonstrateUndoRedoCommand();
        
        // Demonstrate command queue/macro
        demonstrateCommandQueue();
        
        // Show real-world command example
        demonstrateRealWorldCommand();
    }
    
    private static void demonstrateBasicCommand() {
        System.out.println("\n=== Basic Command ===");
        
        // Receiver
        Light light = new Light();
        
        // Commands
        Command lightOnCommand = new LightOnCommand(light);
        Command lightOffCommand = new LightOffCommand(light);
        
        // Invoker
        RemoteControl remote = new RemoteControl();
        
        // Execute commands
        remote.setCommand(lightOnCommand);
        remote.pressButton();
        
        remote.setCommand(lightOffCommand);
        remote.pressButton();
    }
    
    private static void demonstrateUndoRedoCommand() {
        System.out.println("\n=== Command with Undo/Redo ===");
        
        // Receiver
        TextEditor editor = new TextEditor();
        
        // Command manager for undo/redo
        CommandManager commandManager = new CommandManager();
        
        // Commands with undo capability
        Command typeHello = new TypeCommand(editor, "Hello ");
        Command typeWorld = new TypeCommand(editor, "World!");
        Command deleteLast = new DeleteCommand(editor, 6);
        
        // Execute commands
        commandManager.executeCommand(typeHello);
        System.out.println("After typing 'Hello ': " + editor.getText());
        
        commandManager.executeCommand(typeWorld);
        System.out.println("After typing 'World!': " + editor.getText());
        
        commandManager.executeCommand(deleteLast);
        System.out.println("After deleting last 6 chars: " + editor.getText());
        
        // Undo operations
        commandManager.undo();
        System.out.println("After undo delete: " + editor.getText());
        
        commandManager.undo();
        System.out.println("After undo 'World!': " + editor.getText());
        
        commandManager.undo();
        System.out.println("After undo 'Hello ': " + editor.getText());
        
        // Redo operations
        commandManager.redo();
        System.out.println("After redo 'Hello ': " + editor.getText());
        
        commandManager.redo();
        System.out.println("After redo 'World!': " + editor.getText());
    }
    
    private static void demonstrateCommandQueue() {
        System.out.println("\n=== Command Queue/Macro ===");
        
        // Receiver
        Television tv = new Television();
        SoundSystem soundSystem = new SoundSystem();
        
        // Individual commands
        Command tvOn = new TVOnCommand(tv);
        Command soundOn = new SoundOnCommand(soundSystem);
        Command tvOff = new TVOffCommand(tv);
        Command soundOff = new SoundOffCommand(soundSystem);
        
        // Macro commands
        Command homeTheaterOn = new MacroCommand(Arrays.asList(tvOn, soundOn));
        Command homeTheaterOff = new MacroCommand(Arrays.asList(tvOff, soundOff));
        
        // Invoker
        RemoteControl remote = new RemoteControl();
        
        System.out.println("=== Home Theater On ===");
        remote.setCommand(homeTheaterOn);
        remote.pressButton();
        
        System.out.println("\n=== Home Theater Off ===");
        remote.setCommand(homeTheaterOff);
        remote.pressButton();
        
        // Command queue example
        System.out.println("\n=== Command Queue ===");
        CommandQueue queue = new CommandQueue();
        queue.addCommand(new LightOnCommand(new Light()));
        queue.addCommand(new TVOnCommand(new Television()));
        queue.addCommand(new SoundOnCommand(new SoundSystem()));
        
        System.out.println("Executing command queue:");
        queue.executeAll();
    }
    
    private static void demonstrateRealWorldCommand() {
        System.out.println("\n=== Real-World Command Example ===");
        
        // Trading system
        TradingSystem tradingSystem = new TradingSystem();
        OrderManager orderManager = new OrderManager();
        
        // Commands for trading operations
        Command buyCommand = new BuyStockCommand(tradingSystem, "AAPL", 100, 150.0);
        Command sellCommand = new SellStockCommand(tradingSystem, "GOOGL", 50, 2800.0);
        Command cancelCommand = new CancelOrderCommand(tradingSystem, "ORDER123");
        
        // Execute trading commands
        orderManager.setCommand(buyCommand);
        orderManager.executeCommand();
        
        orderManager.setCommand(sellCommand);
        orderManager.executeCommand();
        
        orderManager.setCommand(cancelCommand);
        orderManager.executeCommand();
        
        // Show order history
        System.out.println("\nOrder History:");
        for (String order : tradingSystem.getOrderHistory()) {
            System.out.println("  " + order);
        }
    }
}

// Basic Command pattern - Command interface
interface Command {
    void execute();
}

// Receiver classes
class Light {
    public void turnOn() {
        System.out.println("Light is ON");
    }
    
    public void turnOff() {
        System.out.println("Light is OFF");
    }
}

// Concrete commands
class LightOnCommand implements Command {
    private Light light;
    
    public LightOnCommand(Light light) {
        this.light = light;
    }
    
    @Override
    public void execute() {
        light.turnOn();
    }
}

class LightOffCommand implements Command {
    private Light light;
    
    public LightOffCommand(Light light) {
        this.light = light;
    }
    
    @Override
    public void execute() {
        light.turnOff();
    }
}

// Invoker
class RemoteControl {
    private Command command;
    
    public void setCommand(Command command) {
        this.command = command;
    }
    
    public void pressButton() {
        if (command != null) {
            command.execute();
        }
    }
}

// Command with undo capability
abstract class UndoableCommand implements Command {
    public abstract void undo();
}

class TextEditor {
    private StringBuilder text = new StringBuilder();
    
    public void addText(String newText) {
        text.append(newText);
    }
    
    public String deleteText(int count) {
        int start = Math.max(0, text.length() - count);
        String deleted = text.substring(start);
        text.delete(start, text.length());
        return deleted;
    }
    
    public String getText() {
        return text.toString();
    }
    
    public int length() {
        return text.length();
    }
}

class TypeCommand extends UndoableCommand {
    private TextEditor editor;
    private String text;
    
    public TypeCommand(TextEditor editor, String text) {
        this.editor = editor;
        this.text = text;
    }
    
    @Override
    public void execute() {
        editor.addText(text);
    }
    
    @Override
    public void undo() {
        editor.deleteText(text.length());
    }
}

class DeleteCommand extends UndoableCommand {
    private TextEditor editor;
    private int count;
    private String deletedText;
    
    public DeleteCommand(TextEditor editor, int count) {
        this.editor = editor;
        this.count = count;
    }
    
    @Override
    public void execute() {
        deletedText = editor.deleteText(count);
    }
    
    @Override
    public void undo() {
        if (deletedText != null) {
            editor.addText(deletedText);
        }
    }
}

// Command manager for undo/redo
class CommandManager {
    private Stack<UndoableCommand> commandHistory = new Stack<>();
    private Stack<UndoableCommand> redoHistory = new Stack<>();
    
    public void executeCommand(UndoableCommand command) {
        command.execute();
        commandHistory.push(command);
        redoHistory.clear(); // Clear redo history when new command is executed
    }
    
    public void undo() {
        if (!commandHistory.isEmpty()) {
            UndoableCommand command = commandHistory.pop();
            command.undo();
            redoHistory.push(command);
        }
    }
    
    public void redo() {
        if (!redoHistory.isEmpty()) {
            UndoableCommand command = redoHistory.pop();
            command.execute();
            commandHistory.push(command);
        }
    }
}

// Macro command
class MacroCommand implements Command {
    private List<Command> commands;
    
    public MacroCommand(List<Command> commands) {
        this.commands = new ArrayList<>(commands);
    }
    
    @Override
    public void execute() {
        for (Command command : commands) {
            command.execute();
        }
    }
}

// Additional receiver classes for macro example
class Television {
    public void turnOn() {
        System.out.println("Television is ON");
    }
    
    public void turnOff() {
        System.out.println("Television is OFF");
    }
}

class SoundSystem {
    public void turnOn() {
        System.out.println("Sound System is ON");
    }
    
    public void turnOff() {
        System.out.println("Sound System is OFF");
    }
}

// Additional commands for macro example
class TVOnCommand implements Command {
    private Television tv;
    
    public TVOnCommand(Television tv) {
        this.tv = tv;
    }
    
    @Override
    public void execute() {
        tv.turnOn();
    }
}

class TVOffCommand implements Command {
    private Television tv;
    
    public TVOffCommand(Television tv) {
        this.tv = tv;
    }
    
    @Override
    public void execute() {
        tv.turnOff();
    }
}

class SoundOnCommand implements Command {
    private SoundSystem soundSystem;
    
    public SoundOnCommand(SoundSystem soundSystem) {
        this.soundSystem = soundSystem;
    }
    
    @Override
    public void execute() {
        soundSystem.turnOn();
    }
}

class SoundOffCommand implements Command {
    private SoundSystem soundSystem;
    
    public SoundOffCommand(SoundSystem soundSystem) {
        this.soundSystem = soundSystem;
    }
    
    @Override
    public void execute() {
        soundSystem.turnOff();
    }
}

// Command queue
class CommandQueue {
    private Queue<Command> commands = new LinkedList<>();
    
    public void addCommand(Command command) {
        commands.offer(command);
    }
    
    public void executeAll() {
        while (!commands.isEmpty()) {
            Command command = commands.poll();
            command.execute();
        }
    }
}

// Real-world command example - Trading system
class TradingSystem {
    private List<String> orderHistory = new ArrayList<>();
    
    public void buyStock(String symbol, int quantity, double price) {
        String order = "BUY: " + quantity + " shares of " + symbol + " at $" + price;
        orderHistory.add(order);
        System.out.println("Executing: " + order);
    }
    
    public void sellStock(String symbol, int quantity, double price) {
        String order = "SELL: " + quantity + " shares of " + symbol + " at $" + price;
        orderHistory.add(order);
        System.out.println("Executing: " + order);
    }
    
    public void cancelOrder(String orderId) {
        String order = "CANCEL: Order " + orderId;
        orderHistory.add(order);
        System.out.println("Executing: " + order);
    }
    
    public List<String> getOrderHistory() {
        return new ArrayList<>(orderHistory);
    }
}

// Commands for trading system
class BuyStockCommand implements Command {
    private TradingSystem tradingSystem;
    private String symbol;
    private int quantity;
    private double price;
    
    public BuyStockCommand(TradingSystem tradingSystem, String symbol, int quantity, double price) {
        this.tradingSystem = tradingSystem;
        this.symbol = symbol;
        this.quantity = quantity;
        this.price = price;
    }
    
    @Override
    public void execute() {
        tradingSystem.buyStock(symbol, quantity, price);
    }
}

class SellStockCommand implements Command {
    private TradingSystem tradingSystem;
    private String symbol;
    private int quantity;
    private double price;
    
    public SellStockCommand(TradingSystem tradingSystem, String symbol, int quantity, double price) {
        this.tradingSystem = tradingSystem;
        this.symbol = symbol;
        this.quantity = quantity;
        this.price = price;
    }
    
    @Override
    public void execute() {
        tradingSystem.sellStock(symbol, quantity, price);
    }
}

class CancelOrderCommand implements Command {
    private TradingSystem tradingSystem;
    private String orderId;
    
    public CancelOrderCommand(TradingSystem tradingSystem, String orderId) {
        this.tradingSystem = tradingSystem;
        this.orderId = orderId;
    }
    
    @Override
    public void execute() {
        tradingSystem.cancelOrder(orderId);
    }
}

// Invoker for trading system
class OrderManager {
    private Command command;
    
    public void setCommand(Command command) {
        this.command = command;
    }
    
    public void executeCommand() {
        if (command != null) {
            command.execute();
        }
    }
}
```

## Summary

Java Design Patterns provide proven solutions to common software design problems. This guide covered:

1. **Creational Patterns**: Singleton, Factory, Builder, and Prototype patterns for object creation
2. **Structural Patterns**: Adapter, Decorator, and Facade patterns for object composition and structure
3. **Behavioral Patterns**: Observer, Strategy, and Command patterns for object interaction and communication

Key takeaways:
- Design patterns promote code reusability, maintainability, and flexibility
- Each pattern solves specific design problems with proven solutions
- Understanding when and how to apply patterns is crucial for effective software design
- Patterns should be used judiciously - not every problem requires a pattern
- Modern Java features like lambda expressions and streams can simplify some pattern implementations

Mastering design patterns enables developers to write more robust, scalable, and maintainable Java applications while following established best practices in object-oriented design.