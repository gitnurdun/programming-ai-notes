# Java Reflection: Comprehensive Guide to Runtime Introspection

## Introduction

Java Reflection is a powerful feature that allows programs to examine or modify the runtime behavior of applications running in the Java Virtual Machine (JVM). It enables programs to inspect and manipulate classes, interfaces, fields, and methods at runtime without knowing their names at compile time.

Reflection is part of the `java.lang.reflect` package and is a key component in many Java frameworks such as Spring, Hibernate, and JUnit.

## Core Concepts of Reflection

### What Is Reflection?

Reflection allows Java code to:
- Examine classes, interfaces, fields, and methods at runtime
- Instantiate objects and invoke methods dynamically
- Access private members of classes (with caution)
- Create and manipulate arrays
- Work with annotations at runtime

### The Reflection API Hierarchy

The core reflection classes include:
- `Class`: Represents classes and interfaces
- `Field`: Represents class fields (variables)
- `Method`: Represents class methods
- `Constructor`: Represents class constructors
- `Modifier`: Provides information about access modifiers

## Basic Reflection Operations

### Getting Class Objects

There are several ways to obtain a `Class` object:

```java
// Method 1: Using .class syntax
Class<String> stringClass = String.class;

// Method 2: Using Class.forName()
Class<?> listClass = Class.forName("java.util.ArrayList");

// Method 3: Using getClass() method on an object
List<String> list = new ArrayList<>();
Class<? extends List> listClass2 = list.getClass();

// Method 4: Using primitive type classes
Class<Integer> intClass = int.class;
Class<Void> voidClass = void.class;
```

### Examining Class Information

```java
import java.lang.reflect.*;
import java.util.*;

public class ClassInspector {
    public static void inspectClass(Class<?> clazz) {
        System.out.println("=== Class Information ===");
        System.out.println("Class Name: " + clazz.getName());
        System.out.println("Simple Name: " + clazz.getSimpleName());
        System.out.println("Package: " + clazz.getPackage());
        System.out.println("Is Interface: " + clazz.isInterface());
        System.out.println("Is Array: " + clazz.isArray());
        System.out.println("Is Primitive: " + clazz.isPrimitive());
        System.out.println("Is Enum: " + clazz.isEnum());
        System.out.println("Is Annotation: " + clazz.isAnnotation());
        
        // Superclass information
        Class<?> superClass = clazz.getSuperclass();
        if (superClass != null) {
            System.out.println("Superclass: " + superClass.getName());
        }
        
        // Implemented interfaces
        Class<?>[] interfaces = clazz.getInterfaces();
        System.out.println("Implemented Interfaces: " + Arrays.toString(interfaces));
        
        // Modifiers
        int modifiers = clazz.getModifiers();
        System.out.println("Modifiers: " + Modifier.toString(modifiers));
        System.out.println("Is Public: " + Modifier.isPublic(modifiers));
        System.out.println("Is Abstract: " + Modifier.isAbstract(modifiers));
        System.out.println("Is Final: " + Modifier.isFinal(modifiers));
    }
    
    public static void main(String[] args) throws ClassNotFoundException {
        inspectClass(String.class);
        System.out.println();
        inspectClass(ArrayList.class);
    }
}
```

## Working with Fields

### Getting Field Information

```java
import java.lang.reflect.*;
import java.util.*;

public class FieldInspector {
    public static void inspectFields(Class<?> clazz) {
        System.out.println("=== Fields ===");
        
        // Get all public fields (including inherited)
        Field[] publicFields = clazz.getFields();
        System.out.println("Public Fields:");
        for (Field field : publicFields) {
            System.out.println("  " + field.getName() + " : " + field.getType().getName() + 
                             " [Modifiers: " + Modifier.toString(field.getModifiers()) + "]");
        }
        
        // Get all declared fields (including private, but not inherited)
        Field[] declaredFields = clazz.getDeclaredFields();
        System.out.println("\nDeclared Fields:");
        for (Field field : declaredFields) {
            System.out.println("  " + field.getName() + " : " + field.getType().getName() + 
                             " [Modifiers: " + Modifier.toString(field.getModifiers()) + "]");
        }
    }
    
    // Example class with various fields
    static class ExampleClass {
        public static final String CONSTANT = "CONSTANT_VALUE";
        public int publicField;
        protected String protectedField;
        private double privateField;
        String packageField;
        
        public ExampleClass(int publicField, String protectedField, 
                           double privateField, String packageField) {
            this.publicField = publicField;
            this.protectedField = protectedField;
            this.privateField = privateField;
            this.packageField = packageField;
        }
    }
    
    public static void main(String[] args) {
        inspectFields(ExampleClass.class);
    }
}
```

### Accessing and Modifying Field Values

```java
import java.lang.reflect.*;

public class FieldAccessor {
    static class Person {
        private String name;
        private int age;
        public static String species = "Homo sapiens";
        
        public Person(String name, int age) {
            this.name = name;
            this.age = age;
        }
        
        @Override
        public String toString() {
            return "Person{name='" + name + "', age=" + age + "}";
        }
    }
    
    public static void manipulateFields() throws Exception {
        Person person = new Person("Alice", 30);
        Class<?> clazz = person.getClass();
        
        // Access private field
        Field nameField = clazz.getDeclaredField("name");
        nameField.setAccessible(true); // Bypass private access modifier
        System.out.println("Original name: " + nameField.get(person));
        nameField.set(person, "Bob");
        System.out.println("Modified name: " + nameField.get(person));
        
        // Access private int field
        Field ageField = clazz.getDeclaredField("age");
        ageField.setAccessible(true);
        System.out.println("Original age: " + ageField.getInt(person));
        ageField.setInt(person, 25);
        System.out.println("Modified age: " + ageField.getInt(person));
        
        // Access static field
        Field speciesField = clazz.getField("species");
        System.out.println("Original species: " + speciesField.get(null));
        speciesField.set(null, "Human");
        System.out.println("Modified species: " + speciesField.get(null));
        
        System.out.println("Final person: " + person);
    }
    
    public static void main(String[] args) throws Exception {
        manipulateFields();
    }
}
```

## Working with Methods

### Getting Method Information

```java
import java.lang.reflect.*;
import java.util.*;

public class MethodInspector {
    public static void inspectMethods(Class<?> clazz) {
        System.out.println("=== Methods ===");
        
        // Get all public methods (including inherited)
        Method[] publicMethods = clazz.getMethods();
        System.out.println("Public Methods:");
        for (Method method : publicMethods) {
            System.out.println("  " + formatMethod(method));
        }
        
        // Get all declared methods (including private, but not inherited)
        Method[] declaredMethods = clazz.getDeclaredMethods();
        System.out.println("\nDeclared Methods:");
        for (Method method : declaredMethods) {
            System.out.println("  " + formatMethod(method));
        }
    }
    
    private static String formatMethod(Method method) {
        StringBuilder sb = new StringBuilder();
        sb.append(Modifier.toString(method.getModifiers()));
        if (sb.length() > 0) sb.append(" ");
        sb.append(method.getReturnType().getSimpleName()).append(" ");
        sb.append(method.getName()).append("(");
        
        Class<?>[] paramTypes = method.getParameterTypes();
        for (int i = 0; i < paramTypes.length; i++) {
            if (i > 0) sb.append(", ");
            sb.append(paramTypes[i].getSimpleName());
        }
        sb.append(")");
        
        // Add exceptions
        Class<?>[] exceptions = method.getExceptionTypes();
        if (exceptions.length > 0) {
            sb.append(" throws ");
            for (int i = 0; i < exceptions.length; i++) {
                if (i > 0) sb.append(", ");
                sb.append(exceptions[i].getSimpleName());
            }
        }
        
        return sb.toString();
    }
    
    // Example class with various methods
    static class ExampleClass {
        public void publicMethod(String param) {}
        protected int protectedMethod(int a, int b) { return a + b; }
        private void privateMethod() {}
        static void staticMethod() {}
        
        public String overloadedMethod() { return "no params"; }
        public String overloadedMethod(String param) { return "with param: " + param; }
    }
    
    public static void main(String[] args) {
        inspectMethods(ExampleClass.class);
    }
}
```

### Invoking Methods Dynamically

```java
import java.lang.reflect.*;
import java.util.*;

public class MethodInvocationExample {
    static class Calculator {
        public int add(int a, int b) {
            return a + b;
        }
        
        public double divide(double a, double b) throws ArithmeticException {
            if (b == 0) {
                throw new ArithmeticException("Division by zero");
            }
            return a / b;
        }
        
        private String formatResult(double result) {
            return String.format("%.2f", result);
        }
        
        static String getVersion() {
            return "Calculator v1.0";
        }
    }
    
    public static void invokeMethods() throws Exception {
        Calculator calc = new Calculator();
        Class<?> clazz = calc.getClass();
        
        // Invoke public method
        Method addMethod = clazz.getMethod("add", int.class, int.class);
        Object result = addMethod.invoke(calc, 10, 5);
        System.out.println("10 + 5 = " + result);
        
        // Invoke method with exception handling
        try {
            Method divideMethod = clazz.getMethod("divide", double.class, double.class);
            Object divResult = divideMethod.invoke(calc, 10.0, 3.0);
            System.out.println("10.0 / 3.0 = " + divResult);
        } catch (InvocationTargetException e) {
            System.out.println("Caught exception: " + e.getCause().getMessage());
        }
        
        // Invoke private method
        Method formatMethod = clazz.getDeclaredMethod("formatResult", double.class);
        formatMethod.setAccessible(true);
        Object formatted = formatMethod.invoke(calc, 3.333333);
        System.out.println("Formatted result: " + formatted);
        
        // Invoke static method
        Method versionMethod = clazz.getMethod("getVersion");
        Object version = versionMethod.invoke(null); // null for static methods
        System.out.println("Version: " + version);
    }
    
    public static void main(String[] args) throws Exception {
        invokeMethods();
    }
}
```

## Working with Constructors

### Getting Constructor Information

```java
import java.lang.reflect.*;
import java.util.*;

public class ConstructorInspector {
    public static void inspectConstructors(Class<?> clazz) {
        System.out.println("=== Constructors ===");
        
        Constructor<?>[] constructors = clazz.getConstructors();
        for (Constructor<?> constructor : constructors) {
            System.out.println("  " + formatConstructor(constructor));
        }
        
        System.out.println("\nDeclared Constructors:");
        Constructor<?>[] declaredConstructors = clazz.getDeclaredConstructors();
        for (Constructor<?> constructor : declaredConstructors) {
            System.out.println("  " + formatConstructor(constructor));
        }
    }
    
    private static String formatConstructor(Constructor<?> constructor) {
        StringBuilder sb = new StringBuilder();
        sb.append(Modifier.toString(constructor.getModifiers()));
        if (sb.length() > 0) sb.append(" ");
        sb.append(constructor.getDeclaringClass().getSimpleName()).append("(");
        
        Class<?>[] paramTypes = constructor.getParameterTypes();
        for (int i = 0; i < paramTypes.length; i++) {
            if (i > 0) sb.append(", ");
            sb.append(paramTypes[i].getSimpleName());
        }
        sb.append(")");
        
        // Add exceptions
        Class<?>[] exceptions = constructor.getExceptionTypes();
        if (exceptions.length > 0) {
            sb.append(" throws ");
            for (int i = 0; i < exceptions.length; i++) {
                if (i > 0) sb.append(", ");
                sb.append(exceptions[i].getSimpleName());
            }
        }
        
        return sb.toString();
    }
    
    // Example class with various constructors
    static class ExampleClass {
        private String name;
        private int age;
        
        public ExampleClass() {}
        
        public ExampleClass(String name) {
            this.name = name;
        }
        
        public ExampleClass(String name, int age) {
            this.name = name;
            this.age = age;
        }
        
        private ExampleClass(int age) {
            this.age = age;
        }
    }
    
    public static void main(String[] args) {
        inspectConstructors(ExampleClass.class);
    }
}
```

### Creating Objects Dynamically

```java
import java.lang.reflect.*;
import java.util.*;

public class ObjectCreationExample {
    static class Person {
        private String name;
        private int age;
        
        public Person() {
            this.name = "Unknown";
            this.age = 0;
        }
        
        public Person(String name) {
            this.name = name;
            this.age = 0;
        }
        
        public Person(String name, int age) {
            this.name = name;
            this.age = age;
        }
        
        private Person(int age) {
            this.name = "Private";
            this.age = age;
        }
        
        @Override
        public String toString() {
            return "Person{name='" + name + "', age=" + age + "}";
        }
    }
    
    public static void createObjects() throws Exception {
        Class<?> clazz = Person.class;
        
        // Create object using default constructor
        Constructor<?> defaultConstructor = clazz.getConstructor();
        Object person1 = defaultConstructor.newInstance();
        System.out.println("Default constructor: " + person1);
        
        // Create object using parameterized constructor
        Constructor<?> nameConstructor = clazz.getConstructor(String.class);
        Object person2 = nameConstructor.newInstance("Alice");
        System.out.println("Name constructor: " + person2);
        
        // Create object using constructor with multiple parameters
        Constructor<?> fullConstructor = clazz.getConstructor(String.class, int.class);
        Object person3 = fullConstructor.newInstance("Bob", 25);
        System.out.println("Full constructor: " + person3);
        
        // Create object using private constructor
        Constructor<?> privateConstructor = clazz.getDeclaredConstructor(int.class);
        privateConstructor.setAccessible(true);
        Object person4 = privateConstructor.newInstance(30);
        System.out.println("Private constructor: " + person4);
    }
    
    public static void main(String[] args) throws Exception {
        createObjects();
    }
}
```

## Advanced Reflection Techniques

### Working with Annotations at Runtime

```java
import java.lang.annotation.*;
import java.lang.reflect.*;

// Custom annotations
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Column {
    String name() default "";
    boolean primaryKey() default false;
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Table {
    String name() default "";
}

// Example class with annotations
@Table(name = "users")
class User {
    @Column(name = "id", primaryKey = true)
    private int id;
    
    @Column(name = "username")
    private String username;
    
    @Column(name = "email")
    private String email;
    
    public User(int id, String username, String email) {
        this.id = id;
        this.username = username;
        this.email = email;
    }
}

// Annotation processor using reflection
public class AnnotationProcessor {
    public static void processAnnotations(Class<?> clazz) throws Exception {
        // Process class-level annotations
        if (clazz.isAnnotationPresent(Table.class)) {
            Table table = clazz.getAnnotation(Table.class);
            System.out.println("Table name: " + table.name());
        }
        
        // Process field-level annotations
        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields) {
            if (field.isAnnotationPresent(Column.class)) {
                Column column = field.getAnnotation(Column.class);
                field.setAccessible(true);
                
                System.out.println("Field: " + field.getName());
                System.out.println("  Column name: " + column.name());
                System.out.println("  Primary key: " + column.primaryKey());
            }
        }
    }
    
    public static void main(String[] args) throws Exception {
        processAnnotations(User.class);
    }
}
```

### Dynamic Proxy Creation

```java
import java.lang.reflect.*;
import java.util.*;

// Interface to be proxied
interface Calculator {
    int add(int a, int b);
    int subtract(int a, int b);
    double divide(double a, double b);
}

// Implementation of the interface
class CalculatorImpl implements Calculator {
    @Override
    public int add(int a, int b) {
        System.out.println("Adding " + a + " + " + b);
        return a + b;
    }
    
    @Override
    public int subtract(int a, int b) {
        System.out.println("Subtracting " + a + " - " + b);
        return a - b;
    }
    
    @Override
    public double divide(double a, double b) {
        System.out.println("Dividing " + a + " / " + b);
        if (b == 0) {
            throw new ArithmeticException("Division by zero");
        }
        return a / b;
    }
}

// Invocation handler for dynamic proxy
class LoggingInvocationHandler implements InvocationHandler {
    private final Object target;
    
    public LoggingInvocationHandler(Object target) {
        this.target = target;
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        long startTime = System.nanoTime();
        System.out.println("Method " + method.getName() + " called with args: " + Arrays.toString(args));
        
        try {
            Object result = method.invoke(target, args);
            long endTime = System.nanoTime();
            System.out.println("Method " + method.getName() + " completed in " + (endTime - startTime) + " ns");
            System.out.println("Result: " + result);
            return result;
        } catch (InvocationTargetException e) {
            System.out.println("Method " + method.getName() + " threw exception: " + e.getCause().getMessage());
            throw e.getCause();
        }
    }
}

// Example usage
public class DynamicProxyExample {
    public static void main(String[] args) {
        // Create the target object
        CalculatorImpl calculator = new CalculatorImpl();
        
        // Create the proxy
        Calculator proxy = (Calculator) Proxy.newProxyInstance(
            Calculator.class.getClassLoader(),
            new Class[]{Calculator.class},
            new LoggingInvocationHandler(calculator)
        );
        
        // Use the proxy
        System.out.println("=== Using Proxy ===");
        proxy.add(10, 5);
        System.out.println();
        proxy.subtract(10, 3);
        System.out.println();
        
        try {
            proxy.divide(10, 0);
        } catch (ArithmeticException e) {
            System.out.println("Caught expected exception: " + e.getMessage());
        }
    }
}
```

### Array Manipulation with Reflection

```java
import java.lang.reflect.*;
import java.util.*;

public class ArrayReflectionExample {
    public static void demonstrateArrayReflection() throws Exception {
        // Create array using reflection
        Class<?> stringArrayClass = Class.forName("[Ljava.lang.String;");
        Object stringArray = Array.newInstance(String.class, 5);
        
        // Set values in the array
        Array.set(stringArray, 0, "First");
        Array.set(stringArray, 1, "Second");
        Array.set(stringArray, 2, "Third");
        Array.set(stringArray, 3, "Fourth");
        Array.set(stringArray, 4, "Fifth");
        
        // Get values from the array
        System.out.println("Array contents:");
        int length = Array.getLength(stringArray);
        for (int i = 0; i < length; i++) {
            System.out.println("  [" + i + "] = " + Array.get(stringArray, i));
        }
        
        // Create multi-dimensional array
        Object multiArray = Array.newInstance(int.class, 3, 4);
        System.out.println("\nMulti-dimensional array class: " + multiArray.getClass().getName());
        
        // Get component type
        Class<?> componentType = stringArrayClass.getComponentType();
        System.out.println("Component type of string array: " + componentType.getName());
        
        // Resize array using reflection
        Object resizedArray = resizeArray(stringArray, 7);
        System.out.println("\nResized array length: " + Array.getLength(resizedArray));
    }
    
    public static Object resizeArray(Object originalArray, int newSize) {
        Class<?> componentType = originalArray.getClass().getComponentType();
        Object newArray = Array.newInstance(componentType, newSize);
        
        int length = Math.min(Array.getLength(originalArray), newSize);
        System.arraycopy(originalArray, 0, newArray, 0, length);
        
        return newArray;
    }
    
    public static void main(String[] args) throws Exception {
        demonstrateArrayReflection();
    }
}
```

## Performance Considerations and Best Practices

### Performance Impact of Reflection

```java
import java.lang.reflect.*;

public class ReflectionPerformance {
    static class TestClass {
        private int value = 42;
        
        public int getValue() {
            return value;
        }
        
        public void setValue(int value) {
            this.value = value;
        }
    }
    
    public static void performanceComparison() throws Exception {
        TestClass obj = new TestClass();
        Class<?> clazz = obj.getClass();
        
        // Direct access
        long startTime = System.nanoTime();
        for (int i = 0; i < 1000000; i++) {
            int value = obj.getValue();
            obj.setValue(value + 1);
        }
        long directTime = System.nanoTime() - startTime;
        
        // Reflection access
        Method getValueMethod = clazz.getMethod("getValue");
        Method setValueMethod = clazz.getMethod("setValue", int.class);
        Field valueField = clazz.getDeclaredField("value");
        valueField.setAccessible(true);
        
        startTime = System.nanoTime();
        for (int i = 0; i < 1000000; i++) {
            int value = (Integer) getValueMethod.invoke(obj);
            setValueMethod.invoke(obj, value + 1);
        }
        long reflectionTime = System.nanoTime() - startTime;
        
        // Reflection field access
        startTime = System.nanoTime();
        for (int i = 0; i < 1000000; i++) {
            int value = valueField.getInt(obj);
            valueField.setInt(obj, value + 1);
        }
        long fieldTime = System.nanoTime() - startTime;
        
        System.out.println("Direct access time: " + directTime + " ns");
        System.out.println("Reflection method time: " + reflectionTime + " ns");
        System.out.println("Reflection field time: " + fieldTime + " ns");
        System.out.println("Reflection method overhead: " + (reflectionTime / (double) directTime) + "x");
        System.out.println("Reflection field overhead: " + (fieldTime / (double) directTime) + "x");
    }
    
    public static void main(String[] args) throws Exception {
        performanceComparison();
    }
}
```

### Security Considerations

```java
import java.lang.reflect.*;
import java.security.*;

public class SecurityExample {
    static class SecureClass {
        private String secret = "Secret Value";
        
        private void privateMethod() {
            System.out.println("Private method executed");
        }
    }
    
    public static void demonstrateSecurityManager() {
        // Note: SecurityManager is deprecated in Java 17+
        // This is for demonstration purposes only
        
        try {
            // Create a security manager that restricts reflection
            SecurityManager securityManager = new SecurityManager() {
                @Override
                public void checkPermission(Permission perm) {
                    // Allow basic permissions
                    if (perm.getName().equals("setSecurityManager")) {
                        throw new SecurityException("Changing security manager not allowed");
                    }
                }
                
                @Override
                public void checkPermission(Permission perm, Object context) {
                    checkPermission(perm);
                }
            };
            
            System.setSecurityManager(securityManager);
            
            // Now try to access private members
            SecureClass obj = new SecureClass();
            Class<?> clazz = obj.getClass();
            
            try {
                Field secretField = clazz.getDeclaredField("secret");
                secretField.setAccessible(true);
                String secret = (String) secretField.get(obj);
                System.out.println("Accessed secret: " + secret);
            } catch (SecurityException e) {
                System.out.println("Security manager prevented field access: " + e.getMessage());
            }
            
        } catch (SecurityException e) {
            System.out.println("Security manager restrictions in place: " + e.getMessage());
        } finally {
            // Reset security manager
            System.setSecurityManager(null);
        }
    }
    
    public static void main(String[] args) {
        demonstrateSecurityManager();
    }
}
```

## Practical Applications

### Generic Object Serializer

```java
import java.lang.reflect.*;
import java.util.*;

public class ObjectSerializer {
    public static String serialize(Object obj) throws Exception {
        if (obj == null) {
            return "null";
        }
        
        Class<?> clazz = obj.getClass();
        StringBuilder sb = new StringBuilder();
        sb.append(clazz.getSimpleName()).append("{");
        
        Field[] fields = clazz.getDeclaredFields();
        boolean first = true;
        
        for (Field field : fields) {
            if (!first) sb.append(", ");
            first = false;
            
            field.setAccessible(true);
            sb.append(field.getName()).append("=");
            
            Object value = field.get(obj);
            if (value == null) {
                sb.append("null");
            } else if (value instanceof String) {
                sb.append("\"").append(value).append("\"");
            } else {
                sb.append(value);
            }
        }
        
        sb.append("}");
        return sb.toString();
    }
    
    static class Person {
        private String name;
        private int age;
        private String email;
        
        public Person(String name, int age, String email) {
            this.name = name;
            this.age = age;
            this.email = email;
        }
    }
    
    public static void main(String[] args) throws Exception {
        Person person = new Person("Alice", 30, "alice@example.com");
        String serialized = serialize(person);
        System.out.println("Serialized: " + serialized);
    }
}
```

### Dependency Injection Framework

```java
import java.lang.annotation.*;
import java.lang.reflect.*;
import java.util.*;

// Custom annotations
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Inject {
    String value() default "";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Component {
    String value() default "";
}

// Example components
@Component("databaseService")
class DatabaseService {
    public void save(String data) {
        System.out.println("Saving to database: " + data);
    }
}

@Component("emailService")
class EmailService {
    public void sendEmail(String to, String message) {
        System.out.println("Sending email to " + to + ": " + message);
    }
}

// Class that uses dependency injection
class UserService {
    @Inject("databaseService")
    private DatabaseService databaseService;
    
    @Inject("emailService")
    private EmailService emailService;
    
    public void createUser(String name, String email) {
        databaseService.save("User: " + name);
        emailService.sendEmail(email, "Welcome " + name + "!");
    }
}

// Simple dependency injection container
public class DIContainer {
    private Map<String, Object> components = new HashMap<>();
    
    public void registerComponent(String name, Object instance) {
        components.put(name, instance);
    }
    
    public void injectDependencies(Object target) throws Exception {
        Class<?> clazz = target.getClass();
        Field[] fields = clazz.getDeclaredFields();
        
        for (Field field : fields) {
            if (field.isAnnotationPresent(Inject.class)) {
                Inject inject = field.getAnnotation(Inject.class);
                String componentName = inject.value();
                
                Object dependency = components.get(componentName);
                if (dependency != null) {
                    field.setAccessible(true);
                    field.set(target, dependency);
                    System.out.println("Injected " + componentName + " into " + 
                                     clazz.getSimpleName() + "." + field.getName());
                } else {
                    System.out.println("Component " + componentName + " not found");
                }
            }
        }
    }
    
    public static void main(String[] args) throws Exception {
        DIContainer container = new DIContainer();
        
        // Register components
        container.registerComponent("databaseService", new DatabaseService());
        container.registerComponent("emailService", new EmailService());
        
        // Create and inject dependencies
        UserService userService = new UserService();
        container.injectDependencies(userService);
        
        // Use the service
        userService.createUser("Alice", "alice@example.com");
    }
}
```

## Summary

Java Reflection is a powerful feature that enables runtime introspection and manipulation of Java objects. Key takeaways include:

1. **Class Inspection**: You can examine classes, fields, methods, and constructors at runtime
2. **Dynamic Object Creation**: Objects can be instantiated and methods invoked without compile-time knowledge
3. **Field Access**: Private fields can be accessed, though this should be done carefully
4. **Annotation Processing**: Annotations can be read and processed at runtime
5. **Dynamic Proxies**: Proxy classes can be created to intercept method calls
6. **Performance Considerations**: Reflection has performance overhead compared to direct access
7. **Security**: Reflection can bypass access controls, which has security implications

While reflection is a powerful tool, it should be used judiciously. It can make code harder to understand, debug, and maintain. It's best used in frameworks and libraries rather than application code, and performance-critical code should avoid reflection when possible.