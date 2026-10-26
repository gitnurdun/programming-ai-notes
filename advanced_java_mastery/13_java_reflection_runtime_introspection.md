# Java Reflection: Deep Dive into Runtime Introspection

## Introduction

Java Reflection is a powerful feature that allows programs to examine or modify the runtime behavior of applications running in the Java Virtual Machine (JVM). It enables programs to inspect and manipulate classes, interfaces, fields, and methods at runtime without knowing their names at compile time.

Reflection is part of the `java.lang.reflect` package and is a key component in many Java frameworks such as Spring, Hibernate, and JUnit. This comprehensive guide explores Java reflection in depth, covering its capabilities, use cases, performance considerations, and best practices.

## Understanding Java Reflection

### What Is Reflection?

Reflection is the ability of a program to examine and modify its own structure and behavior at runtime. In Java, reflection allows:

- Examining classes, interfaces, fields, and methods at runtime
- Creating instances and invoking methods dynamically
- Accessing private members of classes (with caution)
- Working with annotations at runtime
- Creating and manipulating arrays dynamically

### Core Reflection Classes

The reflection API is built around several core classes:

- `Class`: Represents classes and interfaces
- `Field`: Represents class fields (variables)
- `Method`: Represents class methods
- `Constructor`: Represents class constructors
- `Modifier`: Provides information about access modifiers

## Basic Reflection Operations

### Getting Class Objects

There are several ways to obtain a `Class` object:

```java
import java.lang.reflect.*;
import java.util.*;

public class GettingClassObjects {
    public static void main(String[] args) throws ClassNotFoundException {
        System.out.println("=== Getting Class Objects ===");
        
        // Method 1: Using .class syntax (compile-time)
        Class<String> stringClass = String.class;
        System.out.println("String class: " + stringClass.getName());
        
        // Method 2: Using Class.forName() (runtime - requires exception handling)
        Class<?> listClass = Class.forName("java.util.ArrayList");
        System.out.println("ArrayList class: " + listClass.getName());
        
        // Method 3: Using getClass() method on an object (runtime)
        List<String> list = new ArrayList<>();
        Class<? extends List> listClass2 = list.getClass();
        System.out.println("List instance class: " + listClass2.getName());
        
        // Method 4: Using primitive type classes
        Class<Integer> intClass = int.class;
        Class<Void> voidClass = void.class;
        System.out.println("int class: " + intClass.getName());
        System.out.println("void class: " + voidClass.getName());
        
        // Method 5: Using wrapper type for primitives
        Class<Boolean> booleanClass = Boolean.TYPE;
        System.out.println("boolean class (TYPE): " + booleanClass.getName());
        
        // Method 6: For array types
        Class<int[]> intArrayClass = int[].class;
        Class<String[]> stringArrayClass = String[].class;
        System.out.println("int[] class: " + intArrayClass.getName());
        System.out.println("String[] class: " + stringArrayClass.getName());
    }
}
```

### Examining Class Information

```java
import java.lang.reflect.*;
import java.util.*;

public class ExaminingClassInfo {
    public static void main(String[] args) {
        System.out.println("=== Examining Class Information ===");
        
        Class<?> clazz = String.class;
        examineClass(clazz);
        
        System.out.println();
        Class<?> collectionClass = ArrayList.class;
        examineClass(collectionClass);
    }
    
    private static void examineClass(Class<?> clazz) {
        System.out.println("=== Class Information for: " + clazz.getName() + " ===");
        
        // Basic class information
        System.out.println("Simple name: " + clazz.getSimpleName());
        System.out.println("Package: " + clazz.getPackage());
        System.out.println("Is interface: " + clazz.isInterface());
        System.out.println("Is array: " + clazz.isArray());
        System.out.println("Is primitive: " + clazz.isPrimitive());
        System.out.println("Is enum: " + clazz.isEnum());
        System.out.println("Is annotation: " + clazz.isAnnotation());
        System.out.println("Is synthetic: " + clazz.isSynthetic());
        
        // Superclass information
        Class<?> superClass = clazz.getSuperclass();
        if (superClass != null) {
            System.out.println("Superclass: " + superClass.getName());
        } else {
            System.out.println("No superclass (Object or primitive)");
        }
        
        // Implemented interfaces
        Class<?>[] interfaces = clazz.getInterfaces();
        System.out.println("Implemented interfaces: " + Arrays.toString(interfaces));
        
        // Class modifiers
        int modifiers = clazz.getModifiers();
        System.out.println("Modifiers: " + Modifier.toString(modifiers));
        System.out.println("Is public: " + Modifier.isPublic(modifiers));
        System.out.println("Is abstract: " + Modifier.isAbstract(modifiers));
        System.out.println("Is final: " + Modifier.isFinal(modifiers));
        System.out.println("Is static: " + Modifier.isStatic(modifiers));
        System.out.println("Is final: " + Modifier.isFinal(modifiers));
        
        // Generic type information
        TypeVariable<?>[] typeParameters = clazz.getTypeParameters();
        System.out.println("Type parameters: " + Arrays.toString(typeParameters));
        
        // Component type for arrays
        if (clazz.isArray()) {
            Class<?> componentType = clazz.getComponentType();
            System.out.println("Component type: " + componentType.getName());
        }
    }
}
```

## Working with Fields

### Getting Field Information

```java
import java.lang.reflect.*;
import java.util.*;

public class FieldReflection {
    // Example class with various field types
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
        System.out.println("=== Field Reflection ===");
        
        Class<?> clazz = ExampleClass.class;
        
        // Get all public fields (including inherited)
        Field[] publicFields = clazz.getFields();
        System.out.println("Public fields (including inherited):");
        for (Field field : publicFields) {
            System.out.println("  " + field.getName() + " : " + field.getType().getName() + 
                             " [Modifiers: " + Modifier.toString(field.getModifiers()) + "]");
        }
        
        System.out.println();
        
        // Get all declared fields (including private, but not inherited)
        Field[] declaredFields = clazz.getDeclaredFields();
        System.out.println("Declared fields (including private):");
        for (Field field : declaredFields) {
            System.out.println("  " + field.getName() + " : " + field.getType().getName() + 
                             " [Modifiers: " + Modifier.toString(field.getModifiers()) + "]");
        }
        
        System.out.println();
        
        // Get specific field
        try {
            Field specificField = clazz.getDeclaredField("privateField");
            System.out.println("Specific field found: " + specificField.getName());
            System.out.println("Field type: " + specificField.getType().getName());
            System.out.println("Field modifiers: " + Modifier.toString(specificField.getModifiers()));
        } catch (NoSuchFieldException e) {
            System.out.println("Field not found: " + e.getMessage());
        }
    }
}
```

### Accessing and Modifying Field Values

```java
import java.lang.reflect.*;

public class FieldAccess {
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
    
    public static void main(String[] args) throws Exception {
        System.out.println("=== Field Access and Modification ===");
        
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
        
        // Demonstrate different primitive field types
        System.out.println("\n=== Different Primitive Field Types ===");
        
        // Create a class with various primitive types
        FieldExample example = new FieldExample();
        Class<?> exampleClass = example.getClass();
        
        // Boolean field
        Field booleanField = exampleClass.getDeclaredField("booleanField");
        booleanField.setAccessible(true);
        System.out.println("Boolean value: " + booleanField.getBoolean(example));
        booleanField.setBoolean(example, true);
        System.out.println("Boolean value after modification: " + booleanField.getBoolean(example));
        
        // Char field
        Field charField = exampleClass.getDeclaredField("charField");
        charField.setAccessible(true);
        System.out.println("Char value: " + charField.getChar(example));
        charField.setChar(example, 'X');
        System.out.println("Char value after modification: " + charField.getChar(example));
        
        // Float field
        Field floatField = exampleClass.getDeclaredField("floatField");
        floatField.setAccessible(true);
        System.out.println("Float value: " + floatField.getFloat(example));
        floatField.setFloat(example, 3.14f);
        System.out.println("Float value after modification: " + floatField.getFloat(example));
        
        System.out.println("Modified example: " + example);
    }
}

class FieldExample {
    private boolean booleanField = false;
    private char charField = 'A';
    private float floatField = 1.0f;
    
    @Override
    public String toString() {
        return "FieldExample{booleanField=" + booleanField + 
               ", charField=" + charField + ", floatField=" + floatField + "}";
    }
}
```

## Working with Methods

### Getting Method Information

```java
import java.lang.reflect.*;
import java.util.*;

public class MethodReflection {
    // Example class with various method types
    static class ExampleClass {
        public void publicMethod(String param) {}
        protected int protectedMethod(int a, int b) { return a + b; }
        private void privateMethod() {}
        static void staticMethod() {}
        
        public String overloadedMethod() { return "no params"; }
        public String overloadedMethod(String param) { return "with param: " + param; }
        public String overloadedMethod(String param, int num) { return "with params: " + param + ", " + num; }
        
        public void methodWithException() throws Exception {}
        public void methodWithGenericTypes(List<String> list, Map<Integer, String> map) {}
    }
    
    public static void main(String[] args) {
        System.out.println("=== Method Reflection ===");
        
        Class<?> clazz = ExampleClass.class;
        
        // Get all public methods (including inherited)
        Method[] publicMethods = clazz.getMethods();
        System.out.println("Public methods (including inherited):");
        for (Method method : publicMethods) {
            System.out.println("  " + formatMethod(method));
        }
        
        System.out.println();
        
        // Get all declared methods (including private, but not inherited)
        Method[] declaredMethods = clazz.getDeclaredMethods();
        System.out.println("Declared methods (including private):");
        for (Method method : declaredMethods) {
            System.out.println("  " + formatMethod(method));
        }
        
        System.out.println();
        
        // Get specific method by name and parameter types
        try {
            Method specificMethod = clazz.getDeclaredMethod("privateMethod");
            System.out.println("Specific method: " + formatMethod(specificMethod));
        } catch (NoSuchMethodException e) {
            System.out.println("Method not found: " + e.getMessage());
        }
        
        // Get overloaded method variants
        System.out.println("\n=== Overloaded Method Variants ===");
        try {
            Method noParamMethod = clazz.getDeclaredMethod("overloadedMethod");
            Method stringParamMethod = clazz.getDeclaredMethod("overloadedMethod", String.class);
            Method twoParamMethod = clazz.getDeclaredMethod("overloadedMethod", String.class, int.class);
            
            System.out.println("No param method: " + formatMethod(noParamMethod));
            System.out.println("String param method: " + formatMethod(stringParamMethod));
            System.out.println("Two param method: " + formatMethod(twoParamMethod));
        } catch (NoSuchMethodException e) {
            System.out.println("Method not found: " + e.getMessage());
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
}
```

### Invoking Methods Dynamically

```java
import java.lang.reflect.*;
import java.util.*;

public class MethodInvocation {
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
        
        public String process(List<String> items) {
            return "Processed: " + items.size() + " items";
        }
    }
    
    public static void main(String[] args) throws Exception {
        System.out.println("=== Method Invocation ===");
        
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
        
        // Invoke method with collection parameter
        Method processMethod = clazz.getMethod("process", List.class);
        List<String> items = Arrays.asList("item1", "item2", "item3");
        Object processResult = processMethod.invoke(calc, items);
        System.out.println("Process result: " + processResult);
        
        System.out.println("\n=== Method Signature Variations ===");
        
        // Different ways to get methods
        try {
            // Get method with exact parameter types
            Method method1 = clazz.getMethod("add", int.class, int.class);
            System.out.println("Method with exact param types: " + method1.getName());
            
            // Get method with object parameter (for reference types)
            Method method2 = clazz.getMethod("process", List.class);
            System.out.println("Method with object param: " + method2.getName());
            
            // Get method with varargs (treated as array)
            Method varargsMethod = clazz.getDeclaredMethod("process", String[].class);
            System.out.println("Method with varargs: " + varargsMethod.getName());
        } catch (NoSuchMethodException e) {
            System.out.println("Method not found: " + e.getMessage());
        }
        
        System.out.println("\n=== Method Metadata ===");
        
        Method addMethodInfo = clazz.getMethod("add", int.class, int.class);
        System.out.println("Method name: " + addMethodInfo.getName());
        System.out.println("Return type: " + addMethodInfo.getReturnType().getSimpleName());
        System.out.println("Parameter count: " + addMethodInfo.getParameterCount());
        System.out.println("Is varargs: " + addMethodInfo.isVarArgs());
        System.out.println("Is synthetic: " + addMethodInfo.isSynthetic());
        System.out.println("Is bridge: " + addMethodInfo.isBridge());
        System.out.println("Method modifiers: " + Modifier.toString(addMethodInfo.getModifiers()));
        
        // Get parameter information
        Parameter[] parameters = addMethodInfo.getParameters();
        for (int i = 0; i < parameters.length; i++) {
            Parameter param = parameters[i];
            System.out.println("  Parameter " + i + ": " + param.getType().getSimpleName() + 
                             " (name: " + param.getName() + ")");
        }
    }
}
```

## Working with Constructors

### Getting Constructor Information

```java
import java.lang.reflect.*;
import java.util.*;

public class ConstructorReflection {
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
        
        protected ExampleClass(String name, String description) {
            this.name = name;
        }
    }
    
    public static void main(String[] args) {
        System.out.println("=== Constructor Reflection ===");
        
        Class<?> clazz = ExampleClass.class;
        
        // Get all public constructors
        Constructor<?>[] constructors = clazz.getConstructors();
        System.out.println("Public constructors:");
        for (Constructor<?> constructor : constructors) {
            System.out.println("  " + formatConstructor(constructor));
        }
        
        System.out.println();
        
        // Get all declared constructors (including private, protected)
        Constructor<?>[] declaredConstructors = clazz.getDeclaredConstructors();
        System.out.println("Declared constructors (including private/protected):");
        for (Constructor<?> constructor : declaredConstructors) {
            System.out.println("  " + formatConstructor(constructor));
        }
        
        System.out.println();
        
        // Get specific constructor by parameter types
        try {
            Constructor<?> specificConstructor = clazz.getConstructor(String.class, int.class);
            System.out.println("Specific constructor: " + formatConstructor(specificConstructor));
        } catch (NoSuchMethodException e) {
            System.out.println("Constructor not found: " + e.getMessage());
        }
        
        System.out.println("\n=== Constructor Parameters ===");
        
        // Examine constructor parameters
        for (Constructor<?> constructor : declaredConstructors) {
            System.out.println("Constructor: " + constructor.getName());
            System.out.println("  Parameter count: " + constructor.getParameterCount());
            
            Class<?>[] paramTypes = constructor.getParameterTypes();
            for (int i = 0; i < paramTypes.length; i++) {
                System.out.println("    Param " + i + ": " + paramTypes[i].getSimpleName());
            }
            
            Parameter[] parameters = constructor.getParameters();
            for (int i = 0; i < parameters.length; i++) {
                Parameter param = parameters[i];
                System.out.println("    Parameter " + i + ": " + param.getType().getSimpleName() + 
                                 " (name: " + param.getName() + ")");
            }
            
            System.out.println("  Exception types: " + Arrays.toString(constructor.getExceptionTypes()));
            System.out.println("  Modifiers: " + Modifier.toString(constructor.getModifiers()));
            System.out.println();
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
}
```

### Creating Objects Dynamically

```java
import java.lang.reflect.*;
import java.util.*;

public class ObjectCreation {
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
    
    static class ComplexObject {
        private String field1;
        private List<String> field2;
        private Map<String, Object> field3;
        
        public ComplexObject(String field1, List<String> field2, Map<String, Object> field3) {
            this.field1 = field1;
            this.field2 = field2;
            this.field3 = field3;
        }
        
        @Override
        public String toString() {
            return "ComplexObject{field1='" + field1 + "', field2=" + field2 + ", field3=" + field3 + "}";
        }
    }
    
    public static void main(String[] args) throws Exception {
        System.out.println("=== Object Creation with Reflection ===");
        
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
        
        System.out.println("\n=== Complex Object Creation ===");
        
        // Create complex object with collections
        Class<?> complexClass = ComplexObject.class;
        Constructor<?> complexConstructor = complexClass.getConstructor(
            String.class, List.class, Map.class);
        
        List<String> list = Arrays.asList("item1", "item2");
        Map<String, Object> map = new HashMap<>();
        map.put("key1", "value1");
        map.put("key2", 42);
        
        Object complexObj = complexConstructor.newInstance("field1Value", list, map);
        System.out.println("Complex object: " + complexObj);
        
        System.out.println("\n=== Constructor Exception Handling ===");
        
        // Example with constructor that might throw exceptions
        try {
            Constructor<?> problematicConstructor = clazz.getConstructor(String.class);
            Object obj = problematicConstructor.newInstance((Object) null); // Might throw exception
            System.out.println("Created with null: " + obj);
        } catch (InvocationTargetException e) {
            System.out.println("Constructor threw exception: " + e.getCause().getMessage());
        }
        
        System.out.println("\n=== Alternative Object Creation ===");
        
        // Using Class.newInstance() (deprecated since Java 9)
        // This only works with default constructor
        try {
            Object defaultPerson = clazz.newInstance();
            System.out.println("Using Class.newInstance(): " + defaultPerson);
        } catch (InstantiationException e) {
            System.out.println("Could not instantiate: " + e.getMessage());
        }
        
        // Using Unsafe (not recommended, for demonstration only)
        // This bypasses constructor calls and can create objects in invalid states
        System.out.println("\n=== Constructor Metadata ===");
        
        Constructor<?> mainConstructor = clazz.getConstructor(String.class, int.class);
        System.out.println("Constructor: " + mainConstructor.getName());
        System.out.println("Parameter count: " + mainConstructor.getParameterCount());
        System.out.println("Is synthetic: " + mainConstructor.isSynthetic());
        System.out.println("Is varargs: " + mainConstructor.isVarArgs());
        System.out.println("Declaring class: " + mainConstructor.getDeclaringClass().getName());
        System.out.println("Modifiers: " + Modifier.toString(mainConstructor.getModifiers()));
        
        // Get generic parameter types
        Type[] genericParamTypes = mainConstructor.getGenericParameterTypes();
        for (int i = 0; i < genericParamTypes.length; i++) {
            System.out.println("  Generic param " + i + ": " + genericParamTypes[i]);
        }
    }
}
```

## Advanced Reflection Techniques

### Working with Annotations

```java
import java.lang.annotation.*;
import java.lang.reflect.*;
import java.util.*;

// Custom annotations
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Column {
    String name() default "";
    boolean primaryKey() default false;
    boolean nullable() default true;
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Table {
    String name() default "";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Validation {
    String message() default "Validation failed";
    String[] allowedValues() default {};
}

// Example class with annotations
@Table(name = "users")
class User {
    @Column(name = "id", primaryKey = true, nullable = false)
    private int id;
    
    @Column(name = "username", nullable = false)
    private String username;
    
    @Column(name = "email")
    private String email;
    
    @Column
    private int age;
    
    public User() {}
    
    public User(int id, String username, String email, int age) {
        this.id = id;
        this.username = username;
        this.email = email;
        this.age = age;
    }
    
    @Validation(message = "Username must not be empty", allowedValues = {"admin", "user", "guest"})
    public void setUsername(String username) {
        this.username = username;
    }
    
    public String getUsername() {
        return username;
    }
    
    @Override
    public String toString() {
        return "User{id=" + id + ", username='" + username + "', email='" + email + "', age=" + age + "}";
    }
}

public class AnnotationReflection {
    public static void main(String[] args) throws Exception {
        System.out.println("=== Annotation Reflection ===");
        
        Class<?> clazz = User.class;
        
        // Process class-level annotations
        if (clazz.isAnnotationPresent(Table.class)) {
            Table table = clazz.getAnnotation(Table.class);
            System.out.println("Table name: " + table.name());
        }
        
        // Process field-level annotations
        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields) {
            System.out.println("\nField: " + field.getName());
            
            if (field.isAnnotationPresent(Column.class)) {
                Column column = field.getAnnotation(Column.class);
                System.out.println("  Column name: " + column.name());
                System.out.println("  Primary key: " + column.primaryKey());
                System.out.println("  Nullable: " + column.nullable());
                
                // Get default value if name is empty
                if (column.name().isEmpty()) {
                    System.out.println("  Using field name as column name: " + field.getName());
                }
            }
        }
        
        // Process method-level annotations
        Method[] methods = clazz.getDeclaredMethods();
        for (Method method : methods) {
            if (method.isAnnotationPresent(Validation.class)) {
                Validation validation = method.getAnnotation(Validation.class);
                System.out.println("\nMethod: " + method.getName());
                System.out.println("  Validation message: " + validation.message());
                System.out.println("  Allowed values: " + Arrays.toString(validation.allowedValues()));
            }
        }
        
        System.out.println("\n=== Annotation Processing Utility ===");
        
        // Create annotation processor
        AnnotationProcessor processor = new AnnotationProcessor();
        User user = new User(1, "john_doe", "john@example.com", 25);
        
        // Validate user object based on annotations
        processor.validateObject(user);
        
        // Generate SQL from annotations
        String createTableSQL = processor.generateCreateTableSQL(User.class);
        System.out.println("Generated SQL:\n" + createTableSQL);
        
        System.out.println("\n=== Meta-Annotation Processing ===");
        
        // Process annotation itself
        Class<? extends Annotation> annotationClass = Table.class;
        Annotation[] annotations = annotationClass.getAnnotations();
        System.out.println("Annotations on @Table annotation:");
        for (Annotation ann : annotations) {
            System.out.println("  " + ann.annotationType().getName());
        }
        
        // Check retention policy
        if (annotationClass.isAnnotationPresent(Retention.class)) {
            Retention retention = annotationClass.getAnnotation(Retention.class);
            System.out.println("Retention policy: " + retention.value());
        }
        
        // Check target specification
        if (annotationClass.isAnnotationPresent(Target.class)) {
            Target target = annotationClass.getAnnotation(Target.class);
            System.out.println("Target: " + Arrays.toString(target.value()));
        }
    }
}

class AnnotationProcessor {
    public void validateObject(Object obj) throws IllegalAccessException {
        Class<?> clazz = obj.getClass();
        Field[] fields = clazz.getDeclaredFields();
        
        for (Field field : fields) {
            field.setAccessible(true);
            Object value = field.get(obj);
            
            if (field.isAnnotationPresent(Column.class)) {
                Column column = field.getAnnotation(Column.class);
                
                if (!column.nullable() && (value == null || 
                    (value instanceof String && ((String) value).isEmpty()))) {
                    throw new IllegalStateException("Field " + field.getName() + " cannot be null or empty");
                }
            }
        }
        
        System.out.println("Object validation passed!");
    }
    
    public String generateCreateTableSQL(Class<?> clazz) {
        if (!clazz.isAnnotationPresent(Table.class)) {
            throw new IllegalArgumentException("Class must be annotated with @Table");
        }
        
        Table table = clazz.getAnnotation(Table.class);
        StringBuilder sql = new StringBuilder();
        
        sql.append("CREATE TABLE ").append(table.name()).append(" (\n");
        
        Field[] fields = clazz.getDeclaredFields();
        for (int i = 0; i < fields.length; i++) {
            Field field = fields[i];
            sql.append("  ");
            
            if (field.isAnnotationPresent(Column.class)) {
                Column column = field.getAnnotation(Column.class);
                String columnName = column.name().isEmpty() ? field.getName() : column.name();
                sql.append(columnName).append(" ");
                
                // Map Java types to SQL types
                if (field.getType() == int.class || field.getType() == Integer.class) {
                    sql.append("INTEGER");
                } else if (field.getType() == String.class) {
                    sql.append("VARCHAR(255)");
                } else {
                    sql.append("TEXT");
                }
                
                if (column.primaryKey()) {
                    sql.append(" PRIMARY KEY");
                }
                
                if (!column.nullable()) {
                    sql.append(" NOT NULL");
                }
            }
            
            if (i < fields.length - 1) {
                sql.append(",");
            }
            sql.append("\n");
        }
        
        sql.append(");");
        return sql.toString();
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
    private final List<String> callLog = new ArrayList<>();
    
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
            
            // Log the call
            callLog.add(method.getName() + " called at " + System.currentTimeMillis());
            
            return result;
        } catch (InvocationTargetException e) {
            System.out.println("Method " + method.getName() + " threw exception: " + e.getCause().getMessage());
            throw e.getCause(); // Re-throw the actual exception
        }
    }
    
    public List<String> getCallLog() {
        return new ArrayList<>(callLog);
    }
    
    public void clearCallLog() {
        callLog.clear();
    }
}

// Performance measurement handler
class PerformanceInvocationHandler implements InvocationHandler {
    private final Object target;
    private final Map<String, Long> performanceStats = new HashMap<>();
    
    public PerformanceInvocationHandler(Object target) {
        this.target = target;
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        long startTime = System.nanoTime();
        
        try {
            Object result = method.invoke(target, args);
            long endTime = System.nanoTime();
            
            String methodName = method.getName();
            long duration = endTime - startTime;
            performanceStats.merge(methodName, duration, Long::sum);
            
            return result;
        } catch (InvocationTargetException e) {
            throw e.getCause();
        }
    }
    
    public Map<String, Long> getPerformanceStats() {
        return new HashMap<>(performanceStats);
    }
    
    public void printPerformanceStats() {
        System.out.println("Performance Statistics:");
        performanceStats.forEach((method, time) -> 
            System.out.println("  " + method + ": " + time + " ns total"));
    }
}

public class DynamicProxyExample {
    public static void main(String[] args) {
        System.out.println("=== Dynamic Proxy Creation ===");
        
        // Create the target object
        CalculatorImpl calculator = new CalculatorImpl();
        
        // Create the logging proxy
        Calculator loggingProxy = (Calculator) Proxy.newProxyInstance(
            Calculator.class.getClassLoader(),
            new Class[]{Calculator.class},
            new LoggingInvocationHandler(calculator)
        );
        
        // Use the logging proxy
        System.out.println("=== Using Logging Proxy ===");
        loggingProxy.add(10, 5);
        loggingProxy.subtract(10, 3);
        
        try {
            loggingProxy.divide(10, 0);
        } catch (ArithmeticException e) {
            System.out.println("Caught expected exception: " + e.getMessage());
        }
        
        // Create the performance proxy
        Calculator performanceProxy = (Calculator) Proxy.newProxyInstance(
            Calculator.class.getClassLoader(),
            new Class[]{Calculator.class},
            new PerformanceInvocationHandler(calculator)
        );
        
        // Use the performance proxy
        System.out.println("\n=== Using Performance Proxy ===");
        for (int i = 0; i < 1000; i++) {
            performanceProxy.add(1, 1);
            performanceProxy.subtract(5, 3);
        }
        
        // Get performance statistics
        if (performanceProxy instanceof Calculator) {
            // We need to access the handler to get stats
            InvocationHandler handler = Proxy.getInvocationHandler(performanceProxy);
            if (handler instanceof PerformanceInvocationHandler) {
                ((PerformanceInvocationHandler) handler).printPerformanceStats();
            }
        }
        
        System.out.println("\n=== Multiple Proxies ===");
        
        // Chain proxies: first logging, then performance
        Calculator chainedProxy = (Calculator) Proxy.newProxyInstance(
            Calculator.class.getClassLoader(),
            new Class[]{Calculator.class},
            new LoggingInvocationHandler(
                Proxy.newProxyInstance(
                    Calculator.class.getClassLoader(),
                    new Class[]{Calculator.class},
                    new PerformanceInvocationHandler(calculator)
                )
            )
        );
        
        chainedProxy.add(5, 3);
        chainedProxy.divide(20, 4);
        
        System.out.println("\n=== Proxy Limitations ===");
        
        // Note: Dynamic proxies only work with interfaces, not classes
        System.out.println("Proxy class: " + loggingProxy.getClass().getName());
        System.out.println("Is proxy: " + Proxy.isProxyClass(loggingProxy.getClass()));
        System.out.println("Proxy interfaces: " + Arrays.toString(loggingProxy.getClass().getInterfaces()));
        
        // Get the original target object (not possible with standard proxies)
        InvocationHandler handler = Proxy.getInvocationHandler(loggingProxy);
        if (handler instanceof LoggingInvocationHandler) {
            System.out.println("Handler is LoggingInvocationHandler");
        }
    }
}
```

### Array Reflection

```java
import java.lang.reflect.*;
import java.util.*;

public class ArrayReflection {
    public static void main(String[] args) throws Exception {
        System.out.println("=== Array Reflection ===");
        
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
        
        System.out.println("\n=== Multi-dimensional Arrays ===");
        
        // Create multi-dimensional array
        Object multiArray = Array.newInstance(int.class, 3, 4);
        System.out.println("Multi-dimensional array class: " + multiArray.getClass().getName());
        System.out.println("Dimensions: " + getArrayDimensions(multiArray.getClass()));
        
        // Fill multi-dimensional array
        for (int i = 0; i < 3; i++) {
            Object row = Array.get(multiArray, i);
            for (int j = 0; j < 4; j++) {
                Array.setInt(row, j, i * 4 + j);
            }
        }
        
        // Print multi-dimensional array
        System.out.println("Multi-dimensional array contents:");
        for (int i = 0; i < 3; i++) {
            Object row = Array.get(multiArray, i);
            for (int j = 0; j < 4; j++) {
                System.out.print(Array.getInt(row, j) + " ");
            }
            System.out.println();
        }
        
        System.out.println("\n=== Array Component Types ===");
        
        // Get component type
        Class<?> componentType = stringArrayClass.getComponentType();
        System.out.println("Component type of string array: " + componentType.getName());
        
        // Different array types
        System.out.println("int[] component type: " + int[].class.getComponentType().getName());
        System.out.println("double[] component type: " + double[].class.getComponentType().getName());
        System.out.println("String[] component type: " + String[].class.getComponentType().getName());
        System.out.println("Object[][].componentType: " + Object[][].class.getComponentType().getName());
        
        System.out.println("\n=== Array Manipulation ===");
        
        // Resize array using reflection
        Object resizedArray = resizeArray(stringArray, 7);
        System.out.println("Original array length: " + Array.getLength(stringArray));
        System.out.println("Resized array length: " + Array.getLength(resizedArray));
        
        // Copy array elements
        Object copiedArray = copyArray(stringArray);
        System.out.println("Copied array: " + Arrays.toString((String[]) copiedArray));
        
        System.out.println("\n=== Primitive Array Reflection ===");
        
        // Work with primitive arrays
        Object intArray = Array.newInstance(int.class, 5);
        for (int i = 0; i < 5; i++) {
            Array.setInt(intArray, i, i * 10);
        }
        
        System.out.println("Primitive int array:");
        for (int i = 0; i < Array.getLength(intArray); i++) {
            System.out.print(Array.getInt(intArray, i) + " ");
        }
        System.out.println();
        
        // Work with other primitive types
        Object booleanArray = Array.newInstance(boolean.class, 3);
        Array.setBoolean(booleanArray, 0, true);
        Array.setBoolean(booleanArray, 1, false);
        Array.setBoolean(booleanArray, 2, true);
        
        System.out.println("Boolean array:");
        for (int i = 0; i < Array.getLength(booleanArray); i++) {
            System.out.print(Array.getBoolean(booleanArray, i) + " ");
        }
        System.out.println();
        
        System.out.println("\n=== Array Utility Functions ===");
        
        // Generic array operations
        Integer[] numbers = {1, 2, 3, 4, 5};
        Object reversedNumbers = reverseArray(numbers);
        System.out.println("Original: " + Arrays.toString(numbers));
        System.out.println("Reversed: " + Arrays.toString((Integer[]) reversedNumbers));
        
        // Filter array
        Object filteredNumbers = filterArray(numbers, n -> n % 2 == 0);
        System.out.println("Filtered (even numbers): " + Arrays.toString((Integer[]) filteredNumbers));
    }
    
    private static int getArrayDimensions(Class<?> arrayClass) {
        int dimensions = 0;
        Class<?> componentType = arrayClass.getComponentType();
        while (componentType != null) {
            dimensions++;
            componentType = componentType.getComponentType();
        }
        return dimensions;
    }
    
    private static Object resizeArray(Object originalArray, int newSize) {
        Class<?> componentType = originalArray.getClass().getComponentType();
        Object newArray = Array.newInstance(componentType, newSize);
        
        int length = Math.min(Array.getLength(originalArray), newSize);
        System.arraycopy(originalArray, 0, newArray, 0, length);
        
        return newArray;
    }
    
    private static Object copyArray(Object originalArray) {
        int length = Array.getLength(originalArray);
        Class<?> componentType = originalArray.getClass().getComponentType();
        Object newArray = Array.newInstance(componentType, length);
        
        System.arraycopy(originalArray, 0, newArray, 0, length);
        return newArray;
    }
    
    private static Object reverseArray(Object array) {
        int length = Array.getLength(array);
        Class<?> componentType = array.getClass().getComponentType();
        Object reversedArray = Array.newInstance(componentType, length);
        
        for (int i = 0; i < length; i++) {
            Array.set(reversedArray, i, Array.get(array, length - 1 - i));
        }
        
        return reversedArray;
    }
    
    private static Object filterArray(Object array, Predicate<Object> predicate) {
        int length = Array.getLength(array);
        Class<?> componentType = array.getClass().getComponentType();
        List<Object> filteredList = new ArrayList<>();
        
        for (int i = 0; i < length; i++) {
            Object element = Array.get(array, i);
            if (predicate.test(element)) {
                filteredList.add(element);
            }
        }
        
        Object filteredArray = Array.newInstance(componentType, filteredList.size());
        for (int i = 0; i < filteredList.size(); i++) {
            Array.set(filteredArray, i, filteredList.get(i));
        }
        
        return filteredArray;
    }
}

@FunctionalInterface
interface Predicate<T> {
    boolean test(T t);
}
```

## Performance Considerations and Best Practices

### Reflection Performance Analysis

```java
import java.lang.reflect.*;
import java.util.*;

public class ReflectionPerformance {
    static class TestClass {
        private int value = 42;
        private String text = "Hello";
        
        public int getValue() {
            return value;
        }
        
        public void setValue(int value) {
            this.value = value;
        }
        
        public String getText() {
            return text;
        }
        
        public void setText(String text) {
            this.text = text;
        }
    }
    
    public static void main(String[] args) throws Exception {
        System.out.println("=== Reflection Performance Analysis ===");
        
        TestClass obj = new TestClass();
        Class<?> clazz = obj.getClass();
        
        // Direct access timing
        long startTime = System.nanoTime();
        for (int i = 0; i < 1000000; i++) {
            int value = obj.getValue();
            obj.setValue(value + 1);
        }
        long directTime = System.nanoTime() - startTime;
        
        // Reflection method access timing
        Method getValueMethod = clazz.getMethod("getValue");
        Method setValueMethod = clazz.getMethod("setValue", int.class);
        startTime = System.nanoTime();
        for (int i = 0; i < 1000000; i++) {
            int value = (Integer) getValueMethod.invoke(obj);
            setValueMethod.invoke(obj, value + 1);
        }
        long reflectionMethodTime = System.nanoTime() - startTime;
        
        // Reflection field access timing
        Field valueField = clazz.getDeclaredField("value");
        valueField.setAccessible(true);
        startTime = System.nanoTime();
        for (int i = 0; i < 1000000; i++) {
            int value = valueField.getInt(obj);
            valueField.setInt(obj, value + 1);
        }
        long reflectionFieldTime = System.nanoTime() - startTime;
        
        System.out.println("Performance Results (1,000,000 operations):");
        System.out.println("Direct access time: " + directTime + " ns");
        System.out.println("Reflection method time: " + reflectionMethodTime + " ns");
        System.out.println("Reflection field time: " + reflectionFieldTime + " ns");
        System.out.println("Method reflection overhead: " + (double) reflectionMethodTime / directTime + "x");
        System.out.println("Field reflection overhead: " + (double) reflectionFieldTime / directTime + "x");
        
        System.out.println("\n=== Method Caching ===");
        
        // Demonstrate method caching for better performance
        MethodCache cache = new MethodCache();
        
        startTime = System.nanoTime();
        for (int i = 0; i < 1000000; i++) {
            int value = (Integer) cache.invokeMethod(obj, "getValue");
            cache.invokeMethod(obj, "setValue", value + 1);
        }
        long cachedTime = System.nanoTime() - startTime;
        
        System.out.println("Cached reflection time: " + cachedTime + " ns");
        System.out.println("Cache improvement over reflection: " + (double) reflectionMethodTime / cachedTime + "x");
        
        System.out.println("\n=== Constructor Performance ===");
        
        // Constructor performance
        startTime = System.nanoTime();
        for (int i = 0; i < 100000; i++) {
            new TestClass();
        }
        long directConstructorTime = System.nanoTime() - startTime;
        
        Constructor<?> constructor = TestClass.class.getConstructor();
        startTime = System.nanoTime();
        for (int i = 0; i < 100000; i++) {
            constructor.newInstance();
        }
        long reflectionConstructorTime = System.nanoTime() - startTime;
        
        System.out.println("Direct constructor: " + directConstructorTime + " ns (100,000 ops)");
        System.out.println("Reflection constructor: " + reflectionConstructorTime + " ns (100,000 ops)");
        System.out.println("Constructor overhead: " + (double) reflectionConstructorTime / directConstructorTime + "x");
    }
}

class MethodCache {
    private final Map<String, Method> methodCache = new HashMap<>();
    private final Object lock = new Object();
    
    public Object invokeMethod(Object obj, String methodName, Object... args) throws Exception {
        Class<?> clazz = obj.getClass();
        String cacheKey = clazz.getName() + "#" + methodName;
        
        Method method = methodCache.get(cacheKey);
        if (method == null) {
            synchronized (lock) {
                method = methodCache.get(cacheKey);
                if (method == null) {
                    Class<?>[] paramTypes = new Class[args.length];
                    for (int i = 0; i < args.length; i++) {
                        paramTypes[i] = args[i].getClass();
                    }
                    method = clazz.getMethod(methodName, paramTypes);
                    methodCache.put(cacheKey, method);
                }
            }
        }
        
        return method.invoke(obj, args);
    }
}
```

### Best Practices and Security Considerations

```java
import java.lang.reflect.*;
import java.util.*;

public class ReflectionBestPractices {
    public static void main(String[] args) {
        System.out.println("=== Reflection Best Practices ===");
        
        // Demonstrate safe reflection usage
        demonstrateSafeReflection();
        
        // Show performance optimization techniques
        demonstratePerformanceOptimizations();
        
        // Show security considerations
        demonstrateSecurityConsiderations();
    }
    
    private static void demonstrateSafeReflection() {
        System.out.println("\n=== Safe Reflection Usage ===");
        
        // Always validate input when using reflection
        String className = "java.lang.String"; // Example - in real code, this would come from user input
        try {
            // Validate class name to prevent malicious class loading
            if (!isValidClassName(className)) {
                throw new IllegalArgumentException("Invalid class name: " + className);
            }
            
            Class<?> clazz = Class.forName(className);
            System.out.println("Successfully loaded class: " + clazz.getName());
        } catch (ClassNotFoundException e) {
            System.out.println("Class not found: " + e.getMessage());
        }
        
        // Use try-catch for reflection operations
        Object obj = "Hello World";
        Class<?> stringClass = obj.getClass();
        
        try {
            Method method = stringClass.getMethod("length");
            Object result = method.invoke(obj);
            System.out.println("Result of length(): " + result);
        } catch (NoSuchMethodException e) {
            System.out.println("Method not found: " + e.getMessage());
        } catch (IllegalAccessException e) {
            System.out.println("Access denied: " + e.getMessage());
        } catch (InvocationTargetException e) {
            System.out.println("Method threw exception: " + e.getCause().getMessage());
        }
        
        // Safe field access
        try {
            Field field = stringClass.getDeclaredField("value");
            // Don't allow access to private fields without proper validation
            if (Modifier.isPrivate(field.getModifiers())) {
                System.out.println("Private field access restricted: " + field.getName());
            }
        } catch (NoSuchFieldException e) {
            System.out.println("Field not found: " + e.getMessage());
        }
    }
    
    private static void demonstratePerformanceOptimizations() {
        System.out.println("\n=== Performance Optimizations ===");
        
        // Cache reflection objects
        ReflectionCache cache = new ReflectionCache();
        
        // Example: Repeated field access
        PerformanceTestObject obj = new PerformanceTestObject();
        
        // Without caching
        long startTime = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            try {
                Field field = PerformanceTestObject.class.getDeclaredField("value");
                field.setAccessible(true);
                field.set(obj, i);
                field.get(obj);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        long withoutCache = System.nanoTime() - startTime;
        
        // With caching
        startTime = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            cache.setFieldValue(obj, "value", i);
            cache.getFieldValue(obj, "value");
        }
        long withCache = System.nanoTime() - startTime;
        
        System.out.println("Without cache: " + withoutCache + " ns");
        System.out.println("With cache: " + withCache + " ns");
        System.out.println("Improvement: " + (double) withoutCache / withCache + "x");
    }
    
    private static void demonstrateSecurityConsiderations() {
        System.out.println("\n=== Security Considerations ===");
        
        // Never expose reflection methods directly to untrusted input
        System.out.println("Security principles:");
        System.out.println("1. Validate all input before using in reflection");
        System.out.println("2. Restrict access to sensitive methods and fields");
        System.out.println("3. Use SecurityManager in sensitive environments");
        System.out.println("4. Be cautious with setAccessible(true)");
        System.out.println("5. Consider using modules to restrict access");
        
        // Example of restricted access
        RestrictedObject obj = new RestrictedObject();
        try {
            Field sensitiveField = RestrictedObject.class.getDeclaredField("sensitiveData");
            System.out.println("Field modifiers: " + Modifier.toString(sensitiveField.getModifiers()));
            
            // Only allow access if we have proper authorization
            if (Modifier.isPrivate(sensitiveField.getModifiers())) {
                System.out.println("Sensitive field access blocked");
            } else {
                sensitiveField.setAccessible(true);
                System.out.println("Accessing sensitive field: " + sensitiveField.get(obj));
            }
        } catch (Exception e) {
            System.out.println("Error accessing field: " + e.getMessage());
        }
    }
    
    private static boolean isValidClassName(String className) {
        // Basic validation - in production, use more comprehensive validation
        return className != null && 
               className.matches("^[a-zA-Z_][a-zA-Z0-9_]*(\\.[a-zA-Z_][a-zA-Z0-9_]*)*$") &&
               !className.startsWith("java.lang.reflect") && // Prevent access to reflection classes
               !className.startsWith("java.util.concurrent") && // Prevent access to sensitive classes
               !className.startsWith("sun.") && // Prevent access to internal classes
               !className.startsWith("jdk."); // Prevent access to internal classes
    }
}

class PerformanceTestObject {
    public int value;
}

class ReflectionCache {
    private final Map<String, Field> fieldCache = new HashMap<>();
    private final Map<String, Method> methodCache = new HashMap<>();
    
    public void setFieldValue(Object obj, String fieldName, Object value) {
        try {
            Field field = getCachedField(obj.getClass(), fieldName);
            field.set(obj, value);
        } catch (Exception e) {
            throw new RuntimeException("Error setting field", e);
        }
    }
    
    public Object getFieldValue(Object obj, String fieldName) {
        try {
            Field field = getCachedField(obj.getClass(), fieldName);
            return field.get(obj);
        } catch (Exception e) {
            throw new RuntimeException("Error getting field", e);
        }
    }
    
    private Field getCachedField(Class<?> clazz, String fieldName) throws NoSuchFieldException {
        String key = clazz.getName() + "#" + fieldName;
        return fieldCache.computeIfAbsent(key, k -> {
            try {
                Field field = clazz.getDeclaredField(fieldName);
                field.setAccessible(true);
                return field;
            } catch (NoSuchFieldException e) {
                throw new RuntimeException(e);
            }
        });
    }
}

class RestrictedObject {
    private String sensitiveData = "This is sensitive";
    public String publicData = "This is public";
}
```

## Practical Applications

### Dependency Injection Framework Example

```java
import java.lang.annotation.*;
import java.lang.reflect.*;
import java.util.*;

// Custom annotations for dependency injection
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

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.CONSTRUCTOR)
@interface InjectConstructor {
    // Marker annotation for constructor injection
}

// Example services
@Component("databaseService")
class DatabaseService {
    public String save(String data) {
        return "Saved to database: " + data;
    }
}

@Component("emailService")
class EmailService {
    public void sendEmail(String to, String message) {
        System.out.println("Sending email to " + to + ": " + message);
    }
}

@Component("loggingService")
class LoggingService {
    public void log(String message) {
        System.out.println("LOG: " + message);
    }
}

// Class that uses dependency injection
class UserService {
    @Inject("databaseService")
    private DatabaseService dbService;
    
    @Inject("emailService")
    private EmailService emailService;
    
    @Inject("loggingService")
    private LoggingService logService;
    
    public void createUser(String name, String email) {
        logService.log("Creating user: " + name);
        String result = dbService.save("User: " + name + ", Email: " + email);
        System.out.println(result);
        emailService.sendEmail(email, "Welcome " + name + "!");
    }
}

// Simple dependency injection container
class DIContainer {
    private final Map<String, Object> instances = new HashMap<>();
    private final Map<Class<?>, Object> classInstances = new HashMap<>();
    
    public void registerComponent(String name, Object instance) {
        instances.put(name, instance);
        classInstances.put(instance.getClass(), instance);
    }
    
    public void registerComponent(Class<?> clazz) throws Exception {
        if (clazz.isAnnotationPresent(Component.class)) {
            Component component = clazz.getAnnotation(Component.class);
            String name = component.value();
            if (name.isEmpty()) {
                name = clazz.getSimpleName();
            }
            
            // Find constructor to use
            Constructor<?> constructor = findInjectableConstructor(clazz);
            Object[] params = resolveConstructorParameters(constructor);
            Object instance = constructor.newInstance(params);
            
            registerComponent(name, instance);
        }
    }
    
    private Constructor<?> findInjectableConstructor(Class<?> clazz) throws NoSuchMethodException {
        // First, look for @InjectConstructor
        Constructor<?>[] constructors = clazz.getConstructors();
        for (Constructor<?> constructor : constructors) {
            if (constructor.isAnnotationPresent(InjectConstructor.class)) {
                return constructor;
            }
        }
        
        // If no @InjectConstructor, use default constructor or first constructor
        return clazz.getConstructor();
    }
    
    private Object[] resolveConstructorParameters(Constructor<?> constructor) throws Exception {
        Class<?>[] paramTypes = constructor.getParameterTypes();
        Object[] params = new Object[paramTypes.length];
        
        for (int i = 0; i < paramTypes.length; i++) {
            Object instance = classInstances.get(paramTypes[i]);
            if (instance == null) {
                // Try to create the dependency if it's a component
                registerComponent(paramTypes[i]);
                instance = classInstances.get(paramTypes[i]);
            }
            params[i] = instance;
        }
        
        return params;
    }
    
    public void injectDependencies(Object target) throws Exception {
        Class<?> clazz = target.getClass();
        Field[] fields = clazz.getDeclaredFields();
        
        for (Field field : fields) {
            if (field.isAnnotationPresent(Inject.class)) {
                Inject inject = field.getAnnotation(Inject.class);
                String componentName = inject.value();
                
                Object dependency = instances.get(componentName);
                if (dependency == null) {
                    // Try to find by type
                    dependency = classInstances.get(field.getType());
                }
                
                if (dependency != null) {
                    field.setAccessible(true);
                    field.set(target, dependency);
                    System.out.println("Injected " + componentName + " into " + 
                                     clazz.getSimpleName() + "." + field.getName());
                } else {
                    System.out.println("Warning: Could not find dependency for " + componentName);
                }
            }
        }
    }
    
    public <T> T getBean(Class<T> type) {
        return (T) classInstances.get(type);
    }
    
    public <T> T getBean(String name) {
        return (T) instances.get(name);
    }
}

public class DependencyInjectionExample {
    public static void main(String[] args) throws Exception {
        System.out.println("=== Dependency Injection Framework Example ===");
        
        DIContainer container = new DIContainer();
        
        // Register components
        container.registerComponent(DatabaseService.class);
        container.registerComponent(EmailService.class);
        container.registerComponent(LoggingService.class);
        
        // Create and inject dependencies into UserService
        UserService userService = new UserService();
        container.injectDependencies(userService);
        
        // Use the service
        userService.createUser("John Doe", "john@example.com");
        
        System.out.println("\nRegistered beans:");
        container.classInstances.forEach((clazz, instance) -> 
            System.out.println("  " + clazz.getSimpleName() + " -> " + instance));
    }
}
```

## Summary

Java Reflection is a powerful feature that enables runtime introspection and manipulation of Java objects. Key takeaways include:

1. **Class Information**: Use `Class` objects to examine type information at runtime
2. **Field Access**: Access and modify object fields dynamically using `Field` class
3. **Method Invocation**: Call methods dynamically using `Method.invoke()`
4. **Constructor Creation**: Create objects using reflection with `Constructor.newInstance()`
5. **Annotation Processing**: Read and process annotations at runtime
6. **Dynamic Proxies**: Create proxy objects for AOP and other patterns
7. **Array Manipulation**: Work with arrays dynamically using `Array` class
8. **Performance Considerations**: Cache reflection objects for better performance
9. **Security**: Validate inputs and restrict access to sensitive members

Best practices for using reflection:
- Cache reflection objects when used repeatedly
- Validate all inputs to prevent security issues
- Use reflection judiciously - prefer compile-time solutions when possible
- Handle exceptions properly in reflection code
- Consider the performance implications of reflection usage

Reflection is essential for building frameworks, but should be used carefully in application code where performance and security are critical.