# Advanced Java Reflection and Annotations

## Overview
Java Reflection is a powerful mechanism that allows programs to examine and modify their own runtime behavior. Combined with annotations, it enables sophisticated frameworks like Spring, Hibernate, and JUnit. Understanding these mechanisms is crucial for advanced Java development and framework creation.

## Reflection API Deep Dive

### Class Object and Reflection Fundamentals

```java
import java.lang.reflect.*;
import java.lang.annotation.*;
import java.util.*;

public class ReflectionFundamentalsDemo {
    
    // Sample class for reflection demonstration
    public static class Person {
        private String name;
        protected int age;
        public String email;
        
        public Person() {}
        
        public Person(String name, int age) {
            this.name = name;
            this.age = age;
        }
        
        public String getName() {
            return name;
        }
        
        public void setName(String name) {
            this.name = name;
        }
        
        public String getDetails(String prefix) {
            return prefix + ": " + name + ", " + age;
        }
        
        private void internalMethod() {
            System.out.println("This is a private method");
        }
    }
    
    public static void demonstrateClassObject() throws Exception {
        // Different ways to get Class object
        Class<?> class1 = Person.class;
        Class<?> class2 = Class.forName("ReflectionFundamentalsDemo$Person");
        Class<?> class3 = new Person().getClass();
        
        // Class metadata
        System.out.println("Class name: " + class1.getName());
        System.out.println("Simple name: " + class1.getSimpleName());
        System.out.println("Package: " + class1.getPackage().getName());
        System.out.println("Superclass: " + class1.getSuperclass().getName());
        System.out.println("Is interface: " + class1.isInterface());
        System.out.println("Is enum: " + class1.isEnum());
        System.out.println("Is array: " + class1.isArray());
        
        // Get all interfaces
        Class<?>[] interfaces = class1.getInterfaces();
        System.out.println("Interfaces: " + Arrays.toString(interfaces));
    }
    
    public static void demonstrateFieldAccess() throws Exception {
        Class<Person> personClass = Person.class;
        Person person = new Person("John", 30);
        
        // Get all declared fields (including private)
        Field[] declaredFields = personClass.getDeclaredFields();
        System.out.println("Declared fields:");
        for (Field field : declaredFields) {
            System.out.println("  " + field.getName() + " - " + field.getType().getSimpleName());
        }
        
        // Get all public fields (from class and superclasses)
        Field[] fields = personClass.getFields();
        System.out.println("Public fields: " + fields.length);
        
        // Access private field
        Field nameField = personClass.getDeclaredField("name");
        nameField.setAccessible(true); // Bypass access control
        System.out.println("Current name: " + nameField.get(person));
        
        // Modify private field
        nameField.set(person, "Jane");
        System.out.println("Modified name: " + nameField.get(person));
        
        // Access public field
        Field emailField = personClass.getField("email");
        emailField.set(person, "jane@example.com");
        System.out.println("Email: " + person.email);
    }
    
    public static void demonstrateMethodAccess() throws Exception {
        Class<Person> personClass = Person.class;
        Person person = new Person("John", 30);
        
        // Get all declared methods (including private)
        Method[] declaredMethods = personClass.getDeclaredMethods();
        System.out.println("Declared methods:");
        for (Method method : declaredMethods) {
            System.out.println("  " + method.getName() + 
                             "(" + Arrays.toString(method.getParameterTypes()) + ")");
        }
        
        // Invoke public method
        Method getNameMethod = personClass.getMethod("getName");
        String name = (String) getNameMethod.invoke(person);
        System.out.println("Name via reflection: " + name);
        
        // Invoke method with parameters
        Method getDetailsMethod = personClass.getMethod("getDetails", String.class);
        String details = (String) getDetailsMethod.invoke(person, "Info");
        System.out.println("Details: " + details);
        
        // Invoke private method
        Method internalMethod = personClass.getDeclaredMethod("internalMethod");
        internalMethod.setAccessible(true);
        internalMethod.invoke(person);
        
        // Get method parameters
        Parameter[] parameters = getDetailsMethod.getParameters();
        for (Parameter param : parameters) {
            System.out.println("Parameter: " + param.getName() + 
                             " - Type: " + param.getType().getSimpleName());
        }
    }
    
    public static void demonstrateConstructorAccess() throws Exception {
        Class<Person> personClass = Person.class;
        
        // Get all constructors
        Constructor<?>[] constructors = personClass.getConstructors();
        System.out.println("Public constructors:");
        for (Constructor<?> constructor : constructors) {
            System.out.println("  " + constructor.getName() + 
                             "(" + Arrays.toString(constructor.getParameterTypes()) + ")");
        }
        
        // Get specific constructor
        Constructor<Person> constructor = personClass.getConstructor(String.class, int.class);
        
        // Create new instance
        Person person = constructor.newInstance("Alice", 25);
        System.out.println("Created person: " + person.getName());
    }
}
```

### Generic Type Information

```java
import java.lang.reflect.*;
import java.util.*;

public class GenericTypeDemo {
    // Generic class example
    public static class GenericBox<T> {
        private T content;
        
        public void setContent(T content) {
            this.content = content;
        }
        
        public T getContent() {
            return content;
        }
    }
    
    // Class that uses generics
    public static class Container {
        public List<String> stringList;
        public Map<String, Integer> stringToIntMap;
        public GenericBox<Date> dateBox;
    }
    
    public static void analyzeGenericTypes() throws Exception {
        Class<Container> containerClass = Container.class;
        
        // Analyze field with generic type
        Field stringListField = containerClass.getDeclaredField("stringList");
        Type stringListType = stringListField.getGenericType();
        
        System.out.println("Field type: " + stringListType);
        System.out.println("Type class: " + stringListType.getClass().getSimpleName());
        
        if (stringListType instanceof ParameterizedType) {
            ParameterizedType pType = (ParameterizedType) stringListType;
            System.out.println("Raw type: " + pType.getRawType());
            System.out.println("Actual type arguments:");
            for (Type arg : pType.getActualTypeArguments()) {
                System.out.println("  " + arg.getTypeName());
            }
        }
        
        // Method with generic return type
        Method method = GenericTypeDemo.class.getDeclaredMethod("createGenericBox");
        Type returnType = method.getGenericReturnType();
        System.out.println("Method return type: " + returnType);
    }
    
    public static <T> GenericBox<T> createGenericBox(T content) {
        GenericBox<T> box = new GenericBox<>();
        box.setContent(content);
        return box;
    }
    
    // Type token pattern for generic type preservation
    public static class TypeToken<T> {
        protected final Type type;
        
        protected TypeToken() {
            Type superClass = getClass().getGenericSuperclass();
            if (superClass instanceof Class) {
                throw new RuntimeException("Missing type parameter");
            }
            this.type = ((ParameterizedType) superClass).getActualTypeArguments()[0];
        }
        
        public Type getType() {
            return type;
        }
        
        @Override
        public boolean equals(Object obj) {
            return obj instanceof TypeToken && 
                   TypeToken.class != obj.getClass() &&
                   Objects.equals(type, ((TypeToken<?>) obj).type);
        }
        
        @Override
        public int hashCode() {
            return type.hashCode();
        }
    }
    
    // Usage example
    static class StringToken extends TypeToken<String> {}
    
    public static void demonstrateTypeToken() {
        TypeToken<String> token = new StringToken();
        System.out.println("Token type: " + token.getType().getTypeName());
    }
}
```

## Annotations System

### Built-in Annotations

```java
import java.lang.annotation.*;
import java.lang.reflect.*;

public class BuiltInAnnotationsDemo {
    
    @Override
    public String toString() {
        return "BuiltInAnnotationsDemo";
    }
    
    @Deprecated
    public void oldMethod() {
        System.out.println("This method is deprecated");
    }
    
    @SuppressWarnings("unchecked")
    public void demonstrateSuppressWarnings() {
        // This would normally generate a warning
        List list = new ArrayList();
        list.add("Hello");
    }
    
    @FunctionalInterface
    public interface MyFunctionalInterface {
        void doSomething();
        
        // Default methods are allowed
        default void defaultMethod() {
            System.out.println("Default method");
        }
    }
    
    public static void demonstrateAnnotationRetrieval() {
        Class<BuiltInAnnotationsDemo> clazz = BuiltInAnnotationsDemo.class;
        
        // Check for annotations
        if (clazz.isAnnotationPresent(Deprecated.class)) {
            System.out.println("Class is deprecated");
        }
        
        // Get all annotations
        Annotation[] annotations = clazz.getAnnotations();
        for (Annotation annotation : annotations) {
            System.out.println("Annotation: " + annotation.annotationType().getName());
        }
        
        // Get specific annotation
        Deprecated deprecated = clazz.getAnnotation(Deprecated.class);
        if (deprecated != null) {
            System.out.println("Found @Deprecated annotation");
        }
    }
}
```

### Custom Annotations

```java
import java.lang.annotation.*;
import java.lang.reflect.*;

// Custom annotation with various element types
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.FIELD, ElementType.TYPE})
@interface ValidationRule {
    String name() default "default";
    String message() default "Validation failed";
    int minLength() default 0;
    int maxLength() default Integer.MAX_VALUE;
    String pattern() default "";
    Class<?>[] groups() default {};
}

// Annotation with array and enum values
enum Severity {
    LOW, MEDIUM, HIGH
}

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD})
@interface ValidationConfig {
    ValidationRule[] rules();
    Severity severity() default Severity.MEDIUM;
    boolean enabled() default true;
}

// Usage example
public class ValidationExample {
    @ValidationRule(minLength = 3, maxLength = 50, 
                   pattern = "^[a-zA-Z0-9_]+$", message = "Invalid username")
    private String username;
    
    @ValidationConfig(rules = {
        @ValidationRule(minLength = 8, message = "Password too short"),
        @ValidationRule(pattern = ".*[A-Z].*", message = "Must contain uppercase")
    }, severity = Severity.HIGH)
    private String password;
    
    @ValidationRule(name = "emailValidation", 
                   pattern = "^[A-Za-z0-9+_.-]+@(.+)$", 
                   message = "Invalid email format")
    public String getEmail() {
        return "user@example.com";
    }
    
    public static void demonstrateCustomAnnotationUsage() throws Exception {
        Class<ValidationExample> clazz = ValidationExample.class;
        
        // Process field annotations
        Field usernameField = clazz.getDeclaredField("username");
        if (usernameField.isAnnotationPresent(ValidationRule.class)) {
            ValidationRule rule = usernameField.getAnnotation(ValidationRule.class);
            System.out.println("Field: " + usernameField.getName());
            System.out.println("  Name: " + rule.name());
            System.out.println("  Message: " + rule.message());
            System.out.println("  Min length: " + rule.minLength());
            System.out.println("  Max length: " + rule.maxLength());
            System.out.println("  Pattern: " + rule.pattern());
        }
        
        // Process method annotations
        Method emailMethod = clazz.getMethod("getEmail");
        ValidationRule[] methodRules = emailMethod.getAnnotationsByType(ValidationRule.class);
        for (ValidationRule rule : methodRules) {
            System.out.println("Method: " + emailMethod.getName() + 
                             ", Rule: " + rule.message());
        }
        
        // Process complex annotation
        Field passwordField = clazz.getDeclaredField("password");
        if (passwordField.isAnnotationPresent(ValidationConfig.class)) {
            ValidationConfig config = passwordField.getAnnotation(ValidationConfig.class);
            System.out.println("Password validation - Severity: " + config.severity());
            System.out.println("Enabled: " + config.enabled());
            
            for (ValidationRule rule : config.rules()) {
                System.out.println("  Rule: " + rule.message());
            }
        }
    }
}
```

### Annotation Processing

```java
import java.lang.annotation.*;
import java.lang.reflect.*;
import java.util.*;

// Annotation processor example
public class AnnotationProcessorDemo {
    
    // Meta-annotation example
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.ANNOTATION_TYPE)
    public @interface MetaAnnotation {
        String description() default "";
        boolean experimental() default false;
    }
    
    @MetaAnnotation(description = "Configuration annotation", experimental = true)
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.TYPE)
    public @interface Component {
        String value() default "";
        String name() default "";
        boolean singleton() default true;
    }
    
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.FIELD)
    public @interface Inject {
        String value() default "";
        boolean required() default true;
    }
    
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    public @interface PostConstruct {
    }
    
    // Example classes using annotations
    @Component(name = "userService", singleton = true)
    public static class UserService {
        @Inject
        private UserRepository userRepository;
        
        @Inject
        private EmailService emailService;
        
        private boolean initialized = false;
        
        @PostConstruct
        public void initialize() {
            System.out.println("UserService initialized");
            initialized = true;
        }
        
        public void processUser(String username) {
            System.out.println("Processing user: " + username);
        }
    }
    
    @Component(name = "userRepository")
    public static class UserRepository {
        public void save(String user) {
            System.out.println("Saving user: " + user);
        }
    }
    
    @Component
    public static class EmailService {
        public void sendEmail(String to, String subject) {
            System.out.println("Sending email to: " + to);
        }
    }
    
    // Simple DI container implementation
    public static class SimpleDIContainer {
        private final Map<String, Object> instances = new HashMap<>();
        private final Map<Class<?>, Object> typeInstances = new HashMap<>();
        
        public void registerComponents(Class<?>... componentClasses) {
            for (Class<?> clazz : componentClasses) {
                if (clazz.isAnnotationPresent(Component.class)) {
                    Component component = clazz.getAnnotation(Component.class);
                    String name = component.name();
                    if (name.isEmpty()) {
                        name = clazz.getSimpleName();
                        name = Character.toLowerCase(name.charAt(0)) + name.substring(1);
                    }
                    
                    try {
                        Object instance = clazz.getDeclaredConstructor().newInstance();
                        instances.put(name, instance);
                        typeInstances.put(clazz, instance);
                        
                        // Process PostConstruct methods
                        for (Method method : clazz.getDeclaredMethods()) {
                            if (method.isAnnotationPresent(PostConstruct.class)) {
                                method.setAccessible(true);
                                method.invoke(instance);
                            }
                        }
                    } catch (Exception e) {
                        throw new RuntimeException("Failed to instantiate component: " + clazz, e);
                    }
                }
            }
            
            // Process @Inject annotations
            processInjection();
        }
        
        private void processInjection() {
            for (Object instance : instances.values()) {
                Class<?> clazz = instance.getClass();
                
                for (Field field : clazz.getDeclaredFields()) {
                    if (field.isAnnotationPresent(Inject.class)) {
                        field.setAccessible(true);
                        String injectName = field.getAnnotation(Inject.class).value();
                        
                        Object dependency;
                        if (!injectName.isEmpty()) {
                            dependency = instances.get(injectName);
                        } else {
                            dependency = typeInstances.get(field.getType());
                        }
                        
                        if (dependency == null) {
                            if (field.getAnnotation(Inject.class).required()) {
                                throw new RuntimeException("Required dependency not found for field: " + field);
                            }
                        } else {
                            try {
                                field.set(instance, dependency);
                            } catch (IllegalAccessException e) {
                                throw new RuntimeException("Failed to inject dependency", e);
                            }
                        }
                    }
                }
            }
        }
        
        @SuppressWarnings("unchecked")
        public <T> T getBean(Class<T> type) {
            return (T) typeInstances.get(type);
        }
        
        public Object getBean(String name) {
            return instances.get(name);
        }
    }
    
    public static void demonstrateDIContainer() {
        SimpleDIContainer container = new SimpleDIContainer();
        container.registerComponents(UserService.class, UserRepository.class, EmailService.class);
        
        UserService userService = container.getBean(UserService.class);
        userService.processUser("John Doe");
        
        UserRepository repo = (UserRepository) container.getBean("userRepository");
        repo.save("John Doe");
    }
}
```

## Advanced Reflection Patterns

### Dynamic Proxy Creation

```java
import java.lang.reflect.*;
import java.util.*;

interface ServiceInterface {
    void performAction(String data);
    String getData();
}

class ServiceImplementation implements ServiceInterface {
    @Override
    public void performAction(String data) {
        System.out.println("Performing action with: " + data);
    }
    
    @Override
    public String getData() {
        return "Service data";
    }
}

class LoggingInvocationHandler implements InvocationHandler {
    private final Object target;
    
    public LoggingInvocationHandler(Object target) {
        this.target = target;
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Before method: " + method.getName());
        
        long startTime = System.currentTimeMillis();
        Object result = method.invoke(target, args);
        long endTime = System.currentTimeMillis();
        
        System.out.println("After method: " + method.getName() + 
                          ", execution time: " + (endTime - startTime) + "ms");
        
        return result;
    }
    
    public static ServiceInterface createProxy(ServiceInterface target) {
        return (ServiceInterface) Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            new LoggingInvocationHandler(target)
        );
    }
}

public class DynamicProxyDemo {
    public static void demonstrateDynamicProxy() {
        ServiceInterface service = new ServiceImplementation();
        ServiceInterface proxy = LoggingInvocationHandler.createProxy(service);
        
        proxy.performAction("test data");
        String data = proxy.getData();
        System.out.println("Retrieved data: " + data);
    }
    
    // Generic proxy for any interface
    public static <T> T createGenericProxy(T target, InvocationHandler handler) {
        Class<?> targetClass = target.getClass();
        Class<?>[] interfaces = targetClass.getInterfaces();
        
        @SuppressWarnings("unchecked")
        T proxy = (T) Proxy.newProxyInstance(
            targetClass.getClassLoader(),
            interfaces,
            handler
        );
        
        return proxy;
    }
}
```

### Reflection Performance Considerations

```java
import java.lang.reflect.*;
import java.util.*;

public class ReflectionPerformanceDemo {
    private String name = "Performance Test";
    private int value = 42;
    
    public String getName() { return name; }
    public int getValue() { return value; }
    
    public void setName(String name) { this.name = name; }
    public void setValue(int value) { this.value = value; }
    
    public static void compareAccessMethods() throws Exception {
        ReflectionPerformanceDemo obj = new ReflectionPerformanceDemo();
        Class<?> clazz = ReflectionPerformanceDemo.class;
        
        // Direct field access (fastest)
        long start = System.nanoTime();
        for (int i = 0; i < 1000000; i++) {
            obj.setValue(i);
            int val = obj.getValue();
        }
        long directTime = System.nanoTime() - start;
        
        // Reflection field access
        Field valueField = clazz.getDeclaredField("value");
        valueField.setAccessible(true);
        start = System.nanoTime();
        for (int i = 0; i < 1000000; i++) {
            valueField.set(obj, i);
            int val = (int) valueField.get(obj);
        }
        long reflectionFieldTime = System.nanoTime() - start;
        
        // Reflection method access
        Method setValueMethod = clazz.getMethod("setValue", int.class);
        Method getValueMethod = clazz.getMethod("getValue");
        start = System.nanoTime();
        for (int i = 0; i < 1000000; i++) {
            setValueMethod.invoke(obj, i);
            int val = (int) getValueMethod.invoke(obj);
        }
        long reflectionMethodTime = System.nanoTime() - start;
        
        System.out.println("Direct access: " + directTime / 1_000_000 + "ms");
        System.out.println("Reflection field: " + reflectionFieldTime / 1_000_000 + "ms");
        System.out.println("Reflection method: " + reflectionMethodTime / 1_000_000 + "ms");
        
        // Caching reflection objects significantly improves performance
        start = System.nanoTime();
        for (int i = 0; i < 1000000; i++) {
            setValueMethod.invoke(obj, i);
            int val = (int) getValueMethod.invoke(obj);
        }
        long cachedTime = System.nanoTime() - start;
        
        System.out.println("Cached reflection: " + cachedTime / 1_000_000 + "ms");
    }
}
```

## Security Considerations

When using reflection, be aware of:
1. **Access control bypass**: Using `setAccessible(true)` can bypass Java security
2. **Performance impact**: Reflection is slower than direct method calls
3. **Security vulnerabilities**: Exposing internal implementation details
4. **Version compatibility**: Reflection code may break with API changes

## Conclusion

Java Reflection and Annotations provide:
- Powerful runtime introspection capabilities
- Foundation for modern frameworks
- Flexible and dynamic application behavior
- Annotation-driven configuration and validation

Understanding these mechanisms is essential for:
- Framework development
- Dependency injection systems
- Serialization libraries
- ORM frameworks
- Testing frameworks
- Code generation tools