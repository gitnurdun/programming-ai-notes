# Mastering Java Classes, Interfaces, Inheritance, and Abstraction

## Introduction

Java's object-oriented programming model is built on fundamental concepts including classes, interfaces, inheritance, and abstraction. These concepts form the backbone of Java applications and are essential for writing maintainable, scalable, and efficient code.

This comprehensive guide will explore these concepts in depth, covering everything from basic syntax to advanced patterns and best practices.

## Classes in Java

### What Is a Class?

A class in Java is a blueprint or template for creating objects. It defines the properties (fields) and behaviors (methods) that objects of that class will have.

### Class Declaration and Structure

```java
public class Person {
    // Fields (instance variables)
    private String name;
    private int age;
    private String email;
    
    // Static field (class variable)
    private static int totalPersons = 0;
    
    // Constructors
    public Person() {
        this.totalPersons++;
    }
    
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
        this.totalPersons++;
    }
    
    // Methods
    public void introduce() {
        System.out.println("Hi, I'm " + name + " and I'm " + age + " years old.");
    }
    
    // Static method
    public static int getTotalPersons() {
        return totalPersons;
    }
    
    // Getter and setter methods
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
    
    public int getAge() {
        return age;
    }
    
    public void setAge(int age) {
        if (age >= 0) {
            this.age = age;
        }
    }
}
```

### Class Members

#### Fields

Fields are variables that hold the state of an object. They can be instance variables (unique to each object) or class variables (shared among all instances).

```java
public class BankAccount {
    // Instance fields
    private String accountNumber;
    private double balance;
    
    // Class field (static)
    private static final double MIN_BALANCE = 100.0;
    private static int totalAccounts = 0;
    
    public BankAccount(String accountNumber, double initialBalance) {
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
        BankAccount.totalAccounts++; // Accessing static field
    }
    
    // Instance method accessing instance fields
    public void deposit(double amount) {
        if (amount > 0) {
            this.balance += amount;
        }
    }
    
    // Static method accessing static fields
    public static int getTotalAccounts() {
        return BankAccount.totalAccounts;
    }
    
    // Instance method accessing both instance and static fields
    public boolean isValid() {
        return this.balance >= BankAccount.MIN_BALANCE;
    }
}
```

#### Methods

Methods define the behavior of a class. They can be instance methods, static methods, or special methods like constructors.

```java
public class Calculator {
    // Instance method
    public double add(double a, double b) {
        return a + b;
    }
    
    // Static method
    public static double multiply(double a, double b) {
        return a * b;
    }
    
    // Method with variable arguments (varargs)
    public int sum(int... numbers) {
        int total = 0;
        for (int num : numbers) {
            total += num;
        }
        return total;
    }
    
    // Method with exception handling
    public double divide(double dividend, double divisor) throws ArithmeticException {
        if (divisor == 0) {
            throw new ArithmeticException("Division by zero");
        }
        return dividend / divisor;
    }
    
    // Method overriding Object.toString()
    @Override
    public String toString() {
        return "Calculator instance";
    }
}
```

#### Constructors

Constructors are special methods used to initialize objects when they are created.

```java
public class Student {
    private String name;
    private int id;
    private String[] courses;
    
    // Default constructor
    public Student() {
        this("Unknown", 0);
    }
    
    // Parameterized constructor
    public Student(String name, int id) {
        this.name = name;
        this.id = id;
        this.courses = new String[5]; // Initialize array
    }
    
    // Copy constructor
    public Student(Student other) {
        this.name = other.name;
        this.id = other.id;
        this.courses = other.courses.clone(); // Deep copy of array
    }
    
    // Constructor chaining
    public Student(String name) {
        this(name, generateId()); // Calling another constructor
    }
    
    private static int generateId() {
        return (int) (Math.random() * 10000);
    }
}
```

## Inheritance in Java

### Understanding Inheritance

Inheritance allows a class to inherit properties and methods from another class. The class that inherits is called the subclass (or child class), and the class being inherited from is called the superclass (or parent class).

### Basic Inheritance

```java
// Superclass
public class Animal {
    protected String name;
    protected int age;
    
    public Animal(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public void eat() {
        System.out.println(name + " is eating");
    }
    
    public void sleep() {
        System.out.println(name + " is sleeping");
    }
    
    public void makeSound() {
        System.out.println(name + " makes a sound");
    }
    
    // Final method - cannot be overridden
    public final void species() {
        System.out.println("This is an animal");
    }
}

// Subclass
public class Dog extends Animal {
    private String breed;
    
    public Dog(String name, int age, String breed) {
        super(name, age); // Calling superclass constructor
        this.breed = breed;
    }
    
    // Overriding superclass method
    @Override
    public void makeSound() {
        System.out.println(name + " barks: Woof! Woof!");
    }
    
    // New method specific to Dog
    public void wagTail() {
        System.out.println(name + " wags tail happily");
    }
    
    // Overloading method
    public void fetch(String item) {
        System.out.println(name + " fetches " + item);
    }
    
    public void fetch(String item, int times) {
        for (int i = 0; i < times; i++) {
            System.out.println(name + " fetches " + item);
        }
    }
}
```

### Multiple Levels of Inheritance

```java
// Base class
public class Vehicle {
    protected String brand;
    protected int year;
    
    public Vehicle(String brand, int year) {
        this.brand = brand;
        this.year = year;
    }
    
    public void start() {
        System.out.println("Vehicle is starting");
    }
}

// Intermediate class
public class Car extends Vehicle {
    protected int doors;
    
    public Car(String brand, int year, int doors) {
        super(brand, year);
        this.doors = doors;
    }
    
    @Override
    public void start() {
        System.out.println(brand + " car is starting with " + doors + " doors");
    }
    
    public void honk() {
        System.out.println("Car honks: Beep! Beep!");
    }
}

// Derived class
public class ElectricCar extends Car {
    private int batteryCapacity;
    
    public ElectricCar(String brand, int year, int doors, int batteryCapacity) {
        super(brand, year, doors);
        this.batteryCapacity = batteryCapacity;
    }
    
    @Override
    public void start() {
        System.out.println(brand + " electric car is starting silently with " + 
                         batteryCapacity + "kWh battery");
    }
    
    public void charge() {
        System.out.println("Electric car is charging");
    }
}
```

### The `super` Keyword

The `super` keyword is used to refer to the immediate parent class object.

```java
public class Shape {
    protected String color;
    
    public Shape(String color) {
        this.color = color;
    }
    
    public double getArea() {
        return 0; // Default implementation
    }
    
    public void displayInfo() {
        System.out.println("Shape color: " + color);
    }
}

public class Rectangle extends Shape {
    private double width;
    private double height;
    
    public Rectangle(String color, double width, double height) {
        super(color); // Calling parent constructor
        this.width = width;
        this.height = height;
    }
    
    @Override
    public double getArea() {
        return width * height;
    }
    
    @Override
    public void displayInfo() {
        super.displayInfo(); // Calling parent method
        System.out.println("Rectangle area: " + getArea());
    }
    
    // Method that calls superclass method explicitly
    public void displayDetailedInfo() {
        System.out.println("=== Detailed Rectangle Info ===");
        super.displayInfo(); // Parent class method
        System.out.println("Width: " + width);
        System.out.println("Height: " + height);
        System.out.println("Area: " + getArea());
    }
}
```

## Abstract Classes and Methods

### Understanding Abstract Classes

Abstract classes cannot be instantiated and are meant to be extended by other classes. They can contain both abstract methods (without implementation) and concrete methods (with implementation).

```java
// Abstract class
public abstract class Animal {
    protected String name;
    protected int age;
    
    public Animal(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    // Concrete method
    public void sleep() {
        System.out.println(name + " is sleeping");
    }
    
    // Abstract method - must be implemented by subclasses
    public abstract void makeSound();
    
    // Another abstract method
    public abstract void move();
    
    // Concrete method that can be overridden
    public void displayInfo() {
        System.out.println("Name: " + name + ", Age: " + age);
    }
}

// Concrete subclass
public class Cat extends Animal {
    public Cat(String name, int age) {
        super(name, age);
    }
    
    @Override
    public void makeSound() {
        System.out.println(name + " meows: Meow! Meow!");
    }
    
    @Override
    public void move() {
        System.out.println(name + " moves gracefully on four legs");
    }
    
    // Overriding concrete method
    @Override
    public void displayInfo() {
        super.displayInfo();
        System.out.println(name + " is a cat");
    }
}

// Another concrete subclass
public class Bird extends Animal {
    public Bird(String name, int age) {
        super(name, age);
    }
    
    @Override
    public void makeSound() {
        System.out.println(name + " chirps: Tweet! Tweet!");
    }
    
    @Override
    public void move() {
        System.out.println(name + " flies in the sky");
    }
}
```

### Abstract Class with Partial Implementation

```java
public abstract class DatabaseConnection {
    protected String connectionString;
    protected boolean isConnected;
    
    public DatabaseConnection(String connectionString) {
        this.connectionString = connectionString;
        this.isConnected = false;
    }
    
    // Concrete method
    public void connect() {
        if (!isConnected) {
            establishConnection();
            isConnected = true;
            System.out.println("Connected to database: " + connectionString);
        }
    }
    
    // Abstract method to be implemented by subclasses
    protected abstract void establishConnection();
    
    // Concrete method that uses abstract method
    public void executeQuery(String query) {
        if (isConnected) {
            System.out.println("Executing query: " + query);
            String result = performQuery(query); // Abstract method call
            System.out.println("Query result: " + result);
        } else {
            System.out.println("Not connected to database");
        }
    }
    
    // Another abstract method
    protected abstract String performQuery(String query);
    
    // Concrete method
    public void disconnect() {
        if (isConnected) {
            closeConnection();
            isConnected = false;
            System.out.println("Disconnected from database");
        }
    }
    
    // Abstract method to be implemented by subclasses
    protected abstract void closeConnection();
}

public class MySQLConnection extends DatabaseConnection {
    public MySQLConnection(String connectionString) {
        super(connectionString);
    }
    
    @Override
    protected void establishConnection() {
        System.out.println("Establishing MySQL connection...");
        // MySQL-specific connection logic
    }
    
    @Override
    protected String performQuery(String query) {
        // MySQL-specific query execution
        return "MySQL result for: " + query;
    }
    
    @Override
    protected void closeConnection() {
        System.out.println("Closing MySQL connection...");
        // MySQL-specific disconnection logic
    }
}

public class PostgreSQLConnection extends DatabaseConnection {
    public PostgreSQLConnection(String connectionString) {
        super(connectionString);
    }
    
    @Override
    protected void establishConnection() {
        System.out.println("Establishing PostgreSQL connection...");
        // PostgreSQL-specific connection logic
    }
    
    @Override
    protected String performQuery(String query) {
        // PostgreSQL-specific query execution
        return "PostgreSQL result for: " + query;
    }
    
    @Override
    protected void closeConnection() {
        System.out.println("Closing PostgreSQL connection...");
        // PostgreSQL-specific disconnection logic
    }
}
```

## Interfaces in Java

### Understanding Interfaces

Interfaces define a contract that implementing classes must follow. They contain method signatures without implementation (until Java 8+).

### Basic Interface

```java
// Interface definition
public interface Drawable {
    // Constant (implicitly public, static, final)
    int MAX_SIZE = 1000;
    
    // Abstract method (implicitly public and abstract)
    void draw();
    
    // Default method (Java 8+)
    default void display() {
        System.out.println("Displaying drawable object");
    }
    
    // Static method (Java 8+)
    static void info() {
        System.out.println("This is a drawable interface");
    }
    
    // Private method (Java 9+)
    private void log(String message) {
        System.out.println("Log: " + message);
    }
    
    // Private static method (Java 9+)
    private static void validateSize(int size) {
        if (size > MAX_SIZE) {
            throw new IllegalArgumentException("Size exceeds maximum allowed");
        }
    }
}

// Implementing class
public class Circle implements Drawable {
    private double radius;
    
    public Circle(double radius) {
        this.radius = radius;
    }
    
    @Override
    public void draw() {
        System.out.println("Drawing a circle with radius " + radius);
    }
    
    // Can override default method if needed
    @Override
    public void display() {
        System.out.println("Displaying circle with area " + (Math.PI * radius * radius));
    }
}
```

### Multiple Interface Implementation

```java
// First interface
public interface Movable {
    void moveUp();
    void moveDown();
    void moveLeft();
    void moveRight();
    
    default void moveTo(int x, int y) {
        System.out.println("Moving to coordinates (" + x + ", " + y + ")");
    }
}

// Second interface
public interface Resizable {
    void resize(double factor);
    
    default void scale(double factor) {
        System.out.println("Scaling by factor " + factor);
        resize(factor);
    }
}

// Third interface extending another
public interface Rotatable extends Movable {
    void rotate(double angle);
    
    default void spin() {
        rotate(360);
    }
}

// Class implementing multiple interfaces
public class Shape implements Drawable, Movable, Resizable {
    private int x, y;
    private double size;
    
    public Shape(int x, int y, double size) {
        this.x = x;
        this.y = y;
        this.size = size;
    }
    
    @Override
    public void draw() {
        System.out.println("Drawing shape at (" + x + ", " + y + ") with size " + size);
    }
    
    @Override
    public void moveUp() {
        y++;
        System.out.println("Moved up to (" + x + ", " + y + ")");
    }
    
    @Override
    public void moveDown() {
        y--;
        System.out.println("Moved down to (" + x + ", " + y + ")");
    }
    
    @Override
    public void moveLeft() {
        x--;
        System.out.println("Moved left to (" + x + ", " + y + ")");
    }
    
    @Override
    public void moveRight() {
        x++;
        System.out.println("Moved right to (" + x + ", " + y + ")");
    }
    
    @Override
    public void resize(double factor) {
        size *= factor;
        System.out.println("Resized to " + size);
    }
}
```

### Functional Interfaces (Java 8+)

```java
// Functional interface - single abstract method
@FunctionalInterface
public interface CalculatorOperation {
    double calculate(double a, double b);
    
    // Can have default methods
    default void printResult(double result) {
        System.out.println("Result: " + result);
    }
    
    // Can have static methods
    static CalculatorOperation getAddition() {
        return (a, b) -> a + b;
    }
}

// Another functional interface
@FunctionalInterface
public interface StringProcessor {
    String process(String input);
    
    // Default method
    default String processAndPrint(String input) {
        String result = process(input);
        System.out.println("Processed: " + result);
        return result;
    }
}

// Usage examples
public class FunctionalInterfaceDemo {
    public static void main(String[] args) {
        // Lambda expressions
        CalculatorOperation add = (a, b) -> a + b;
        CalculatorOperation subtract = (a, b) -> a - b;
        CalculatorOperation multiply = (a, b) -> a * b;
        CalculatorOperation divide = (a, b) -> {
            if (b == 0) throw new ArithmeticException("Division by zero");
            return a / b;
        };
        
        System.out.println("Addition: " + add.calculate(10, 5));
        System.out.println("Subtraction: " + subtract.calculate(10, 5));
        System.out.println("Multiplication: " + multiply.calculate(10, 5));
        System.out.println("Division: " + divide.calculate(10, 5));
        
        // Method references
        StringProcessor upperCase = String::toUpperCase;
        StringProcessor lowerCase = String::toLowerCase;
        StringProcessor trim = String::trim;
        
        System.out.println(upperCase.process("hello world"));
        System.out.println(lowerCase.process("HELLO WORLD"));
        System.out.println(trim.process("  hello world  "));
    }
}
```

## Advanced Concepts and Patterns

### Interface Segregation Principle

```java
// Bad example - fat interface
public interface Worker {
    void work();
    void eat();
    void sleep();
    void manage();
    void code();
}

// Good example - segregated interfaces
public interface Workable {
    void work();
}

public interface Eatable {
    void eat();
}

public interface Sleepable {
    void sleep();
}

public interface Manageable {
    void manage();
}

public interface Codeable {
    void code();
}

// Implementation classes
public class HumanWorker implements Workable, Eatable, Sleepable {
    @Override
    public void work() {
        System.out.println("Human is working");
    }
    
    @Override
    public void eat() {
        System.out.println("Human is eating");
    }
    
    @Override
    public void sleep() {
        System.out.println("Human is sleeping");
    }
}

public class RobotWorker implements Workable, Codeable {
    @Override
    public void work() {
        System.out.println("Robot is working");
    }
    
    @Override
    public void code() {
        System.out.println("Robot is coding");
    }
}

public class Manager implements Workable, Manageable {
    @Override
    public void work() {
        System.out.println("Manager is working");
    }
    
    @Override
    public void manage() {
        System.out.println("Manager is managing");
    }
}
```

### Marker Interfaces

```java
// Marker interface - no methods
public interface Serializable {
    // Empty - just marks classes that can be serialized
}

public interface Cloneable {
    // Empty - just marks classes that can be cloned
}

// Example usage
public class DataObject implements Serializable, Cloneable {
    private String data;
    
    public DataObject(String data) {
        this.data = data;
    }
    
    // Override clone method from Object class
    @Override
    public Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
    
    // Custom serialization logic would go here
    // (simplified for this example)
    public void serialize() {
        System.out.println("Serializing: " + data);
    }
}
```

### Interface with Generic Types

```java
// Generic interface
public interface Repository<T, ID> {
    T save(T entity);
    T findById(ID id);
    void delete(T entity);
    List<T> findAll();
    
    // Default method with generics
    default Optional<T> findOptionalById(ID id) {
        try {
            return Optional.of(findById(id));
        } catch (Exception e) {
            return Optional.empty();
        }
    }
}

// Implementation for User entity
public class UserRepository implements Repository<User, Long> {
    private Map<Long, User> users = new HashMap<>();
    private AtomicLong idGenerator = new AtomicLong(1);
    
    @Override
    public User save(User user) {
        if (user.getId() == null) {
            user.setId(idGenerator.getAndIncrement());
        }
        users.put(user.getId(), user);
        return user;
    }
    
    @Override
    public User findById(Long id) {
        User user = users.get(id);
        if (user == null) {
            throw new EntityNotFoundException("User not found with id: " + id);
        }
        return user;
    }
    
    @Override
    public void delete(User user) {
        users.remove(user.getId());
    }
    
    @Override
    public List<User> findAll() {
        return new ArrayList<>(users.values());
    }
}

// User entity class
class User {
    private Long id;
    private String name;
    private String email;
    
    public User() {}
    
    public User(String name, String email) {
        this.name = name;
        this.email = email;
    }
    
    // Getters and setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    
    @Override
    public String toString() {
        return "User{id=" + id + ", name='" + name + "', email='" + email + "'}";
    }
}

class EntityNotFoundException extends RuntimeException {
    public EntityNotFoundException(String message) {
        super(message);
    }
}
```

## Best Practices and Design Patterns

### Composition Over Inheritance

```java
// Instead of deep inheritance hierarchies, use composition
public class Engine {
    private String type;
    private int horsePower;
    
    public Engine(String type, int horsePower) {
        this.type = type;
        this.horsePower = horsePower;
    }
    
    public void start() {
        System.out.println("Engine " + type + " started with " + horsePower + " HP");
    }
    
    public void stop() {
        System.out.println("Engine " + type + " stopped");
    }
}

public class GPS {
    public void navigate(String destination) {
        System.out.println("Navigating to " + destination);
    }
    
    public void getCurrentLocation() {
        System.out.println("Current location: 40.7128° N, 74.0060° W");
    }
}

// Car class using composition
public class ModernCar {
    private Engine engine;
    private GPS gps;
    private String model;
    
    public ModernCar(String model, Engine engine, GPS gps) {
        this.model = model;
        this.engine = engine;
        this.gps = gps;
    }
    
    public void startCar() {
        System.out.println("Starting " + model);
        engine.start();
    }
    
    public void navigateTo(String destination) {
        gps.navigate(destination);
    }
    
    public void getLocation() {
        gps.getCurrentLocation();
    }
}
```

### Template Method Pattern with Abstract Classes

```java
public abstract class DataProcessor {
    // Template method - defines the algorithm structure
    public final void process() {
        readData();
        processData();
        validateData();
        saveData();
        if (requiresNotification()) {
            sendNotification();
        }
    }
    
    // Abstract methods to be implemented by subclasses
    protected abstract void readData();
    protected abstract void processData();
    protected abstract void saveData();
    
    // Hook method - can be overridden but has default behavior
    protected void validateData() {
        System.out.println("Performing basic data validation");
    }
    
    // Another hook method
    protected boolean requiresNotification() {
        return false; // Default behavior
    }
    
    // Concrete method
    private void sendNotification() {
        System.out.println("Sending notification about data processing");
    }
}

public class XMLDataProcessor extends DataProcessor {
    @Override
    protected void readData() {
        System.out.println("Reading XML data from file");
    }
    
    @Override
    protected void processData() {
        System.out.println("Processing XML data");
    }
    
    @Override
    protected void saveData() {
        System.out.println("Saving processed XML data to database");
    }
    
    @Override
    protected void validateData() {
        System.out.println("Validating XML data against schema");
    }
    
    @Override
    protected boolean requiresNotification() {
        return true; // XML processing requires notification
    }
}

public class CSVDataProcessor extends DataProcessor {
    @Override
    protected void readData() {
        System.out.println("Reading CSV data from file");
    }
    
    @Override
    protected void processData() {
        System.out.println("Processing CSV data");
    }
    
    @Override
    protected void saveData() {
        System.out.println("Saving processed CSV data to database");
    }
}
```

## Practical Examples and Use Cases

### Complete Example: Payment Processing System

```java
// Abstract base class
public abstract class PaymentMethod {
    protected String accountNumber;
    protected String holderName;
    
    public PaymentMethod(String accountNumber, String holderName) {
        this.accountNumber = accountNumber;
        this.holderName = holderName;
    }
    
    // Common behavior
    public void validateAccount() {
        if (accountNumber == null || accountNumber.isEmpty()) {
            throw new IllegalArgumentException("Account number is required");
        }
        System.out.println("Validating account: " + accountNumber);
    }
    
    // Abstract methods
    public abstract boolean processPayment(double amount);
    public abstract String getPaymentType();
    
    // Getters
    public String getAccountNumber() { return accountNumber; }
    public String getHolderName() { return holderName; }
}

// Interface for additional capabilities
public interface Refundable {
    boolean processRefund(double amount);
    double getRefundLimit();
}

// Concrete implementation
public class CreditCardPayment extends PaymentMethod implements Refundable {
    private String cardType;
    private String cvv;
    private LocalDate expiryDate;
    
    public CreditCardPayment(String accountNumber, String holderName, 
                           String cardType, String cvv, LocalDate expiryDate) {
        super(accountNumber, holderName);
        this.cardType = cardType;
        this.cvv = cvv;
        this.expiryDate = expiryDate;
    }
    
    @Override
    public boolean processPayment(double amount) {
        validateAccount();
        if (isExpired()) {
            System.out.println("Card is expired");
            return false;
        }
        System.out.println("Processing credit card payment of $" + amount);
        return true;
    }
    
    @Override
    public boolean processRefund(double amount) {
        if (amount <= getRefundLimit()) {
            System.out.println("Processing refund of $" + amount + " to credit card");
            return true;
        }
        System.out.println("Refund amount exceeds limit");
        return false;
    }
    
    @Override
    public double getRefundLimit() {
        return 1000.0; // $1000 limit
    }
    
    @Override
    public String getPaymentType() {
        return "Credit Card (" + cardType + ")";
    }
    
    private boolean isExpired() {
        return LocalDate.now().isAfter(expiryDate);
    }
}

// Another implementation
public class BankTransferPayment extends PaymentMethod {
    private String bankCode;
    private String routingNumber;
    
    public BankTransferPayment(String accountNumber, String holderName, 
                             String bankCode, String routingNumber) {
        super(accountNumber, holderName);
        this.bankCode = bankCode;
        this.routingNumber = routingNumber;
    }
    
    @Override
    public boolean processPayment(double amount) {
        validateAccount();
        System.out.println("Processing bank transfer of $" + amount);
        System.out.println("Bank code: " + bankCode + ", Routing: " + routingNumber);
        return true;
    }
    
    @Override
    public String getPaymentType() {
        return "Bank Transfer";
    }
}

// Interface for digital payments
public interface DigitalPayment {
    boolean authenticateUser(String credentials);
    boolean isSecure();
}

// Implementation combining inheritance and interface
public class PayPalPayment extends PaymentMethod implements DigitalPayment, Refundable {
    private String email;
    private boolean twoFactorEnabled;
    
    public PayPalPayment(String accountNumber, String holderName, String email) {
        super(accountNumber, holderName);
        this.email = email;
        this.twoFactorEnabled = false;
    }
    
    @Override
    public boolean processPayment(double amount) {
        validateAccount();
        System.out.println("Processing PayPal payment of $" + amount);
        System.out.println("Email: " + email);
        return true;
    }
    
    @Override
    public boolean authenticateUser(String credentials) {
        System.out.println("Authenticating PayPal user with: " + credentials);
        return true; // Simplified
    }
    
    @Override
    public boolean isSecure() {
        return true;
    }
    
    @Override
    public boolean processRefund(double amount) {
        System.out.println("Processing PayPal refund of $" + amount);
        return true;
    }
    
    @Override
    public double getRefundLimit() {
        return 5000.0; // Higher limit for PayPal
    }
    
    @Override
    public String getPaymentType() {
        return "PayPal";
    }
    
    public void enableTwoFactor() {
        this.twoFactorEnabled = true;
        System.out.println("Two-factor authentication enabled");
    }
}

// Payment processor service
public class PaymentProcessor {
    public void processPayment(PaymentMethod paymentMethod, double amount) {
        System.out.println("=== Processing Payment ===");
        System.out.println("Payment Type: " + paymentMethod.getPaymentType());
        System.out.println("Holder: " + paymentMethod.getHolderName());
        
        boolean success = paymentMethod.processPayment(amount);
        if (success) {
            System.out.println("Payment processed successfully");
        } else {
            System.out.println("Payment failed");
        }
        
        // Check if payment method supports refunds
        if (paymentMethod instanceof Refundable) {
            Refundable refundable = (Refundable) paymentMethod;
            System.out.println("Refund limit: $" + refundable.getRefundLimit());
        }
        
        // Check if payment method is digital
        if (paymentMethod instanceof DigitalPayment) {
            DigitalPayment digital = (DigitalPayment) paymentMethod;
            System.out.println("Is secure: " + digital.isSecure());
        }
        
        System.out.println();
    }
}

// Example usage
public class PaymentSystemDemo {
    public static void main(String[] args) {
        PaymentProcessor processor = new PaymentProcessor();
        
        // Different payment methods
        CreditCardPayment creditCard = new CreditCardPayment(
            "1234-5678-9012-3456", "John Doe", "Visa", "123", 
            LocalDate.of(2025, 12, 31));
            
        BankTransferPayment bankTransfer = new BankTransferPayment(
            "987654321", "Jane Smith", "BANK123", "ROUTING456");
            
        PayPalPayment paypal = new PayPalPayment(
            "paypal-789", "Bob Johnson", "bob@example.com");
            
        // Process payments
        processor.processPayment(creditCard, 150.00);
        processor.processPayment(bankTransfer, 250.00);
        processor.processPayment(paypal, 75.50);
        
        // Additional PayPal features
        paypal.enableTwoFactor();
        paypal.authenticateUser("password123");
        
        // Process refund for credit card
        if (creditCard.processRefund(50.00)) {
            System.out.println("Refund processed successfully");
        }
    }
}
```

## Summary

This comprehensive guide covered the fundamental concepts of Java classes, interfaces, inheritance, and abstraction:

1. **Classes**: The building blocks of Java applications with fields, methods, and constructors
2. **Inheritance**: The mechanism for creating class hierarchies and reusing code
3. **Abstract Classes**: Classes that cannot be instantiated and may contain abstract methods
4. **Interfaces**: Contracts that define method signatures and can include default and static methods
5. **Advanced Patterns**: Best practices like composition over inheritance and design patterns
6. **Practical Applications**: Real-world examples showing how these concepts work together

Key takeaways:
- Use inheritance judiciously and prefer composition when possible
- Interfaces provide flexibility and support multiple inheritance of type
- Abstract classes are useful when you want to share code among related classes
- Java 8+ interfaces with default and static methods provide more flexibility
- Understanding these concepts is crucial for writing maintainable, scalable Java applications

Mastering these concepts will make you proficient in Java object-oriented programming and prepare you for advanced topics like design patterns, frameworks, and enterprise development.