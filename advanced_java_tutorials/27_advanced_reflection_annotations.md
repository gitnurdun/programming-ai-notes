# Advanced Java Reflection and Annotations

## Table of Contents
1. [Introduction to Java Reflection](#introduction-to-java-reflection)
2. [Class Inspection and Metadata](#class-inspection-and-metadata)
3. [Dynamic Method Invocation](#dynamic-method-invocation)
4. [Field Access and Modification](#field-access-and-modification)
5. [Constructor Manipulation](#constructor-manipulation)
6. [Annotations Processing](#annotations-processing)
7. [Custom Annotation Creation](#custom-annotation-creation)
8. [Dynamic Proxy Implementation](#dynamic-proxy-implementation)
9. [Reflection Performance Considerations](#reflection-performance-considerations)
10. [Security and Best Practices](#security-and-best-practices)

## Introduction to Java Reflection

Java Reflection is a powerful feature that allows programs to examine or modify the runtime behavior of applications running in the Java Virtual Machine (JVM). It provides the ability to inspect classes, interfaces, fields, and methods at runtime, without knowing the names of the classes, methods, etc. at compile time.

### Reflection Fundamentals

```java
// Basic reflection operations and class inspection
public class ReflectionBasics {
    
    // Sample class for reflection demonstration
    public static class SampleClass {
        private String privateField = "private value";
        protected String protectedField = "protected value";
        public String publicField = "public value";
        public static String staticField = "static value";
        
        public SampleClass() {}
        
        public SampleClass(String value) {
            this.publicField = value;
        }
        
        private void privateMethod() {
            System.out.println("Private method called");
        }
        
        protected void protectedMethod() {
            System.out.println("Protected method called");
        }
        
        public void publicMethod(String param) {
            System.out.println("Public method called with: " + param);
        }
        
        public static void staticMethod() {
            System.out.println("Static method called");
        }
    }
    
    public static void demonstrateClassInspection() {
        System.out.println("=== Class Inspection ===");
        
        try {
            // Getting Class object in different ways
            Class<?> clazz1 = SampleClass.class;
            Class<?> clazz2 = Class.forName("ReflectionBasics$SampleClass");
            Class<?> clazz3 = new SampleClass().getClass();
            
            System.out.println("Class name: " + clazz1.getName());
            System.out.println("Simple name: " + clazz1.getSimpleName());
            System.out.println("Canonical name: " + clazz1.getCanonicalName());
            System.out.println("Package: " + clazz1.getPackage().getName());
            System.out.println("Is interface: " + clazz1.isInterface());
            System.out.println("Is array: " + clazz1.isArray());
            System.out.println("Is primitive: " + clazz1.isPrimitive());
            System.out.println("Modifiers: " + java.lang.reflect.Modifier.toString(clazz1.getModifiers()));
            
            // Getting superclass and interfaces
            System.out.println("Superclass: " + clazz1.getSuperclass().getName());
            Class<?>[] interfaces = clazz1.getInterfaces();
            System.out.println("Interfaces count: " + interfaces.length);
            
            // Getting class loader
            System.out.println("Class loader: " + clazz1.getClassLoader());
            
        } catch (ClassNotFoundException e) {
            System.err.println("Class not found: " + e.getMessage());
        }
    }
    
    // Inspecting class members
    public static void demonstrateMemberInspection() {
        System.out.println("\n=== Member Inspection ===");
        
        Class<?> clazz = SampleClass.class;
        
        // Getting all fields
        System.out.println("--- Fields ---");
        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields) {
            System.out.println("Field: " + field.getName() + 
                             " | Type: " + field.getType().getSimpleName() +
                             " | Modifiers: " + java.lang.reflect.Modifier.toString(field.getModifiers()));
        }
        
        // Getting all methods
        System.out.println("\n--- Methods ---");
        Method[] methods = clazz.getDeclaredMethods();
        for (Method method : methods) {
            System.out.print("Method: " + method.getName() + 
                           " | Return type: " + method.getReturnType().getSimpleName() +
                           " | Parameters: ");
            
            Class<?>[] paramTypes = method.getParameterTypes();
            for (int i = 0; i < paramTypes.length; i++) {
                if (i > 0) System.out.print(", ");
                System.out.print(paramTypes[i].getSimpleName());
            }
            System.out.println(" | Modifiers: " + java.lang.reflect.Modifier.toString(method.getModifiers()));
        }
        
        // Getting all constructors
        System.out.println("\n--- Constructors ---");
        Constructor<?>[] constructors = clazz.getDeclaredConstructors();
        for (Constructor<?> constructor : constructors) {
            System.out.print("Constructor: ");
            Class<?>[] paramTypes = constructor.getParameterTypes();
            for (int i = 0; i < paramTypes.length; i++) {
                if (i > 0) System.out.print(", ");
                System.out.print(paramTypes[i].getSimpleName());
            }
            System.out.println(" | Modifiers: " + java.lang.reflect.Modifier.toString(constructor.getModifiers()));
        }
    }
    
    // Type inspection and generic information
    public static void demonstrateTypeInspection() {
        System.out.println("\n=== Type Inspection ===");
        
        class GenericClass<T extends Number, U> {
            private T genericField;
            private List<U> listField;
            
            public void genericMethod(List<? extends T> param) {}
        }
        
        Class<?> clazz = GenericClass.class;
        
        // Getting generic superclass (if any)
        Type genericSuperclass = clazz.getGenericSuperclass();
        System.out.println("Generic superclass: " + genericSuperclass);
        
        // Getting generic interfaces
        Type[] genericInterfaces = clazz.getGenericInterfaces();
        System.out.println("Generic interfaces count: " + genericInterfaces.length);
        
        // Getting generic type information for fields
        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields) {
            System.out.println("Field: " + field.getName() + 
                             " | Generic type: " + field.getGenericType());
        }
        
        // Getting generic type information for methods
        Method[] methods = clazz.getDeclaredMethods();
        for (Method method : methods) {
            System.out.println("Method: " + method.getName());
            System.out.println("  Generic return type: " + method.getGenericReturnType());
            
            Type[] paramTypes = method.getGenericParameterTypes();
            for (int i = 0; i < paramTypes.length; i++) {
                System.out.println("  Parameter " + i + " generic type: " + paramTypes[i]);
            }
        }
    }
    
    public static void main(String[] args) {
        demonstrateClassInspection();
        demonstrateMemberInspection();
        demonstrateTypeInspection();
    }
}
```

### Advanced Class and Interface Inspection

```java
// Advanced class and interface inspection techniques
public class AdvancedClassInspection {
    
    // Interface for demonstration
    interface SampleInterface {
        void interfaceMethod();
        default void defaultMethod() {
            System.out.println("Default method implementation");
        }
    }
    
    // Abstract class for demonstration
    abstract static class AbstractClass implements SampleInterface {
        protected abstract void abstractMethod();
        
        public void concreteMethod() {
            System.out.println("Concrete method in abstract class");
        }
    }
    
    // Concrete implementation
    static class ConcreteClass extends AbstractClass {
        private String instanceField = "instance value";
        private static String staticField = "static value";
        
        @Override
        public void interfaceMethod() {
            System.out.println("Interface method implementation");
        }
        
        @Override
        protected void abstractMethod() {
            System.out.println("Abstract method implementation");
        }
        
        public void instanceMethod(String param) {
            System.out.println("Instance method with param: " + param);
        }
        
        public static void staticMethod() {
            System.out.println("Static method");
        }
    }
    
    public static void demonstrateInheritanceInspection() {
        System.out.println("=== Inheritance Chain Inspection ===");
        
        Class<?> clazz = ConcreteClass.class;
        
        // Complete class hierarchy
        System.out.println("Class hierarchy:");
        Class<?> current = clazz;
        while (current != null) {
            System.out.println("  " + current.getName());
            current = current.getSuperclass();
        }
        
        // Interface hierarchy
        System.out.println("\nImplemented interfaces:");
        Class<?>[] interfaces = clazz.getInterfaces();
        for (Class<?> iface : interfaces) {
            System.out.println("  " + iface.getName());
        }
        
        // Getting all methods including inherited ones
        System.out.println("\nAll public methods (including inherited):");
        Method[] allMethods = clazz.getMethods();
        for (Method method : allMethods) {
            // Filter to show only methods from our classes
            if (method.getDeclaringClass().getName().startsWith("AdvancedClassInspection")) {
                System.out.println("  " + method.getDeclaringClass().getSimpleName() + 
                                 "." + method.getName());
            }
        }
        
        // Getting declared methods only (not inherited)
        System.out.println("\nDeclared methods (not inherited):");
        Method[] declaredMethods = clazz.getDeclaredMethods();
        for (Method method : declaredMethods) {
            System.out.println("  " + method.getName() + 
                             " | Declared in: " + method.getDeclaringClass().getSimpleName());
        }
    }
    
    // Method resolution and overriding inspection
    public static void demonstrateMethodResolution() {
        System.out.println("\n=== Method Resolution Analysis ===");
        
        Class<?> clazz = ConcreteClass.class;
        
        try {
            // Finding methods by name and parameter types
            Method interfaceMethod = clazz.getMethod("interfaceMethod");
            Method defaultMethod = clazz.getMethod("defaultMethod");
            Method instanceMethod = clazz.getMethod("instanceMethod", String.class);
            
            System.out.println("Interface method:");
            System.out.println("  Declaring class: " + interfaceMethod.getDeclaringClass().getName());
            System.out.println("  Is abstract: " + java.lang.reflect.Modifier.isAbstract(interfaceMethod.getModifiers()));
            
            System.out.println("Default method:");
            System.out.println("  Declaring class: " + defaultMethod.getDeclaringClass().getName());
            System.out.println("  Is default: " + defaultMethod.isDefault());
            
            System.out.println("Instance method:");
            System.out.println("  Declaring class: " + instanceMethod.getDeclaringClass().getName());
            System.out.println("  Parameter count: " + instanceMethod.getParameterCount());
            
            // Getting method exceptions
            System.out.println("\nMethod exception types:");
            Class<?>[] exceptionTypes = instanceMethod.getExceptionTypes();
            System.out.println("  Instance method exceptions: " + Arrays.toString(exceptionTypes));
            
        } catch (NoSuchMethodException e) {
            System.err.println("Method not found: " + e.getMessage());
        }
    }
    
    // Annotation inspection
    @Deprecated
    @SuppressWarnings("unused")
    static class AnnotatedClass {
        @Deprecated
        private String deprecatedField;
        
        @SuppressWarnings("unchecked")
        public void annotatedMethod(@Deprecated String param) {
            // Method with various annotations
        }
    }
    
    public static void demonstrateAnnotationInspection() {
        System.out.println("\n=== Annotation Inspection ===");
        
        Class<?> clazz = AnnotatedClass.class;
        
        // Class-level annotations
        System.out.println("Class annotations:");
        Annotation[] classAnnotations = clazz.getAnnotations();
        for (Annotation annotation : classAnnotations) {
            System.out.println("  " + annotation.annotationType().getName());
        }
        
        // Getting specific annotation
        Deprecated classDeprecated = clazz.getAnnotation(Deprecated.class);
        if (classDeprecated != null) {
            System.out.println("  Class is deprecated");
        }
        
        // Field annotations
        System.out.println("\nField annotations:");
        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields) {
            Annotation[] fieldAnnotations = field.getAnnotations();
            if (fieldAnnotations.length > 0) {
                System.out.println("  Field: " + field.getName());
                for (Annotation annotation : fieldAnnotations) {
                    System.out.println("    " + annotation.annotationType().getName());
                }
            }
        }
        
        // Method annotations
        System.out.println("\nMethod annotations:");
        Method[] methods = clazz.getDeclaredMethods();
        for (Method method : methods) {
            Annotation[] methodAnnotations = method.getAnnotations();
            if (methodAnnotations.length > 0) {
                System.out.println("  Method: " + method.getName());
                for (Annotation annotation : methodAnnotations) {
                    System.out.println("    " + annotation.annotationType().getName());
                }
            }
            
            // Parameter annotations
            Annotation[][] paramAnnotations = method.getParameterAnnotations();
            for (int i = 0; i < paramAnnotations.length; i++) {
                if (paramAnnotations[i].length > 0) {
                    System.out.println("    Parameter " + i + " annotations:");
                    for (Annotation annotation : paramAnnotations[i]) {
                        System.out.println("      " + annotation.annotationType().getName());
                    }
                }
            }
        }
    }
    
    // Generic type inspection with ParameterizedType
    public static void demonstrateGenericTypeInspection() {
        System.out.println("\n=== Generic Type Inspection ===");
        
        class GenericContainer<T, U extends Number> {
            private List<T> items;
            private Map<String, U> mappings;
            
            public void process(List<? extends T> input) {}
        }
        
        Class<?> clazz = GenericContainer.class;
        
        // Inspecting field generic types
        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields) {
            Type genericType = field.getGenericType();
            System.out.println("Field: " + field.getName() + " | Generic type: " + genericType);
            
            if (genericType instanceof ParameterizedType) {
                ParameterizedType paramType = (ParameterizedType) genericType;
                System.out.println("  Raw type: " + paramType.getRawType());
                Type[] actualTypes = paramType.getActualTypeArguments();
                for (int i = 0; i < actualTypes.length; i++) {
                    System.out.println("  Type argument " + i + ": " + actualTypes[i]);
                }
            }
        }
        
        // Inspecting method generic types
        Method[] methods = clazz.getDeclaredMethods();
        for (Method method : methods) {
            System.out.println("Method: " + method.getName());
            
            Type[] paramTypes = method.getGenericParameterTypes();
            for (int i = 0; i < paramTypes.length; i++) {
                System.out.println("  Parameter " + i + " type: " + paramTypes[i]);
                
                if (paramTypes[i] instanceof ParameterizedType) {
                    ParameterizedType paramType = (ParameterizedType) paramTypes[i];
                    System.out.println("    Raw type: " + paramType.getRawType());
                    Type[] actualTypes = paramType.getActualTypeArguments();
                    for (int j = 0; j < actualTypes.length; j++) {
                        System.out.println("    Type argument " + j + ": " + actualTypes[j]);
                    }
                }
            }
        }
        
        // Getting actual type parameters of the class
        TypeVariable<?>[] typeParameters = clazz.getTypeParameters();
        System.out.println("\nClass type parameters:");
        for (TypeVariable<?> typeParam : typeParameters) {
            System.out.println("  " + typeParam.getName());
            Type[] bounds = typeParam.getBounds();
            for (Type bound : bounds) {
                System.out.println("    Bound: " + bound);
            }
        }
    }
    
    public static void main(String[] args) {
        demonstrateInheritanceInspection();
        demonstrateMethodResolution();
        demonstrateAnnotationInspection();
        demonstrateGenericTypeInspection();
    }
}
```

## Dynamic Method Invocation

Reflection allows for dynamic method invocation, enabling flexible and powerful runtime behavior modification.

### Method Invocation Techniques

```java
// Dynamic method invocation and advanced reflection operations
public class DynamicMethodInvocation {
    
    // Sample class for method invocation demonstrations
    public static class MethodInvocationDemo {
        private String privateField = "private value";
        public String publicField = "public value";
        
        private String privateMethod(String param) {
            return "Private method called with: " + param;
        }
        
        protected String protectedMethod(String param1, int param2) {
            return "Protected method called with: " + param1 + ", " + param2;
        }
        
        public String publicMethod(String param) {
            return "Public method called with: " + param;
        }
        
        public static String staticMethod(String param) {
            return "Static method called with: " + param;
        }
        
        public void voidMethod(String message) {
            System.out.println("Void method output: " + message);
        }
        
        // Method that throws exception for error handling demonstration
        public void throwingMethod() throws IllegalArgumentException {
            throw new IllegalArgumentException("This is a test exception");
        }
    }
    
    public static void demonstrateBasicMethodInvocation() {
        System.out.println("=== Basic Method Invocation ===");
        
        try {
            Class<?> clazz = MethodInvocationDemo.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            
            // Invoking public method
            Method publicMethod = clazz.getMethod("publicMethod", String.class);
            String result = (String) publicMethod.invoke(instance, "test parameter");
            System.out.println("Public method result: " + result);
            
            // Invoking static method
            Method staticMethod = clazz.getMethod("staticMethod", String.class);
            String staticResult = (String) staticMethod.invoke(null, "static test");
            System.out.println("Static method result: " + staticResult);
            
            // Invoking void method
            Method voidMethod = clazz.getMethod("voidMethod", String.class);
            voidMethod.invoke(instance, "void method test");
            
        } catch (Exception e) {
            System.err.println("Error during method invocation: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Invoking private and protected methods
    public static void demonstrateAccessControlBypass() {
        System.out.println("\n=== Access Control Bypass ===");
        
        try {
            Class<?> clazz = MethodInvocationDemo.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            
            // Invoking private method
            Method privateMethod = clazz.getDeclaredMethod("privateMethod", String.class);
            privateMethod.setAccessible(true); // Bypass access control
            String privateResult = (String) privateMethod.invoke(instance, "private test");
            System.out.println("Private method result: " + privateResult);
            
            // Invoking protected method
            Method protectedMethod = clazz.getDeclaredMethod("protectedMethod", String.class, int.class);
            protectedMethod.setAccessible(true);
            String protectedResult = (String) protectedMethod.invoke(instance, "protected test", 42);
            System.out.println("Protected method result: " + protectedResult);
            
        } catch (Exception e) {
            System.err.println("Error during access control bypass: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Method invocation with parameter handling
    public static void demonstrateParameterHandling() {
        System.out.println("\n=== Parameter Handling ===");
        
        class ParameterDemo {
            public String methodWithMultipleParams(String str, int num, boolean flag, double decimal) {
                return String.format("String: %s, Int: %d, Boolean: %b, Double: %.2f", 
                                   str, num, flag, decimal);
            }
            
            public String methodWithArray(String[] array) {
                return "Array length: " + array.length + ", First element: " + 
                       (array.length > 0 ? array[0] : "null");
            }
            
            public String methodWithVarargs(String... args) {
                return "Varargs count: " + args.length + ", Args: " + Arrays.toString(args);
            }
        }
        
        try {
            Class<?> clazz = ParameterDemo.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            
            // Method with multiple parameters
            Method multiParamMethod = clazz.getMethod("methodWithMultipleParams", 
                                                    String.class, int.class, boolean.class, double.class);
            String multiParamResult = (String) multiParamMethod.invoke(instance, 
                                                                     "test", 123, true, 3.14);
            System.out.println("Multi-param method result: " + multiParamResult);
            
            // Method with array parameter
            Method arrayMethod = clazz.getMethod("methodWithArray", String[].class);
            String[] testArray = {"first", "second", "third"};
            String arrayResult = (String) arrayMethod.invoke(instance, (Object) testArray);
            System.out.println("Array method result: " + arrayResult);
            
            // Method with varargs
            Method varargsMethod = clazz.getMethod("methodWithVarargs", String[].class);
            String varargsResult = (String) varargsMethod.invoke(instance, 
                                                               (Object) new String[]{"a", "b", "c"});
            System.out.println("Varargs method result: " + varargsResult);
            
        } catch (Exception e) {
            System.err.println("Error during parameter handling: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Error handling during method invocation
    public static void demonstrateErrorHandling() {
        System.out.println("\n=== Error Handling ===");
        
        try {
            Class<?> clazz = MethodInvocationDemo.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            
            // Method that throws exception
            Method throwingMethod = clazz.getMethod("throwingMethod");
            
            try {
                throwingMethod.invoke(instance);
            } catch (InvocationTargetException e) {
                // This is how exceptions thrown by the invoked method are wrapped
                System.out.println("Caught InvocationTargetException: " + e.getCause().getMessage());
                System.out.println("Original exception type: " + e.getCause().getClass().getName());
            }
            
            // Handling NoSuchMethodException
            try {
                Method nonExistentMethod = clazz.getMethod("nonExistentMethod");
            } catch (NoSuchMethodException e) {
                System.out.println("NoSuchMethodException caught: " + e.getMessage());
            }
            
            // Handling IllegalAccessException
            Method privateMethod = clazz.getDeclaredMethod("privateMethod", String.class);
            try {
                privateMethod.invoke(instance, "test"); // Will fail without setAccessible(true)
            } catch (IllegalAccessException e) {
                System.out.println("IllegalAccessException caught (expected): " + e.getMessage());
            }
            
            // Correct way to invoke private method
            privateMethod.setAccessible(true);
            String result = (String) privateMethod.invoke(instance, "test");
            System.out.println("Successfully invoked private method: " + result);
            
        } catch (Exception e) {
            System.err.println("Unexpected error: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Performance considerations and caching
    public static void demonstratePerformanceOptimization() {
        System.out.println("\n=== Performance Optimization ===");
        
        class PerformanceDemo {
            public String simpleMethod(String input) {
                return input.toUpperCase();
            }
        }
        
        try {
            Class<?> clazz = PerformanceDemo.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            Method method = clazz.getMethod("simpleMethod", String.class);
            
            // Without caching - look up method every time (inefficient)
            long startTime = System.nanoTime();
            for (int i = 0; i < 100000; i++) {
                Method tempMethod = clazz.getMethod("simpleMethod", String.class);
                tempMethod.invoke(instance, "test");
            }
            long withoutCachingTime = System.nanoTime() - startTime;
            
            // With caching - reuse Method object (efficient)
            startTime = System.nanoTime();
            for (int i = 0; i < 100000; i++) {
                method.invoke(instance, "test");
            }
            long withCachingTime = System.nanoTime() - startTime;
            
            System.out.println("Without caching: " + withoutCachingTime / 1_000_000 + " ms");
            System.out.println("With caching: " + withCachingTime / 1_000_000 + " ms");
            System.out.println("Performance improvement: " + 
                             (double) withoutCachingTime / withCachingTime + "x");
            
        } catch (Exception e) {
            System.err.println("Error during performance test: " + e.getMessage());
        }
    }
    
    // Method invocation with dynamic parameter resolution
    public static void demonstrateDynamicInvocation() {
        System.out.println("\n=== Dynamic Method Invocation ===");
        
        class DynamicDemo {
            public String method1(String param) {
                return "Method1: " + param;
            }
            
            public String method2(int param) {
                return "Method2: " + param;
            }
            
            public String method3(String param1, int param2) {
                return "Method3: " + param1 + ", " + param2;
            }
        }
        
        try {
            Class<?> clazz = DynamicDemo.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            
            // Dynamic method invocation based on method name and parameters
            Map<String, Object[]> methodCalls = new HashMap<>();
            methodCalls.put("method1", new Object[]{"test"});
            methodCalls.put("method2", new Object[]{42});
            methodCalls.put("method3", new Object[]{"hello", 123});
            
            for (Map.Entry<String, Object[]> entry : methodCalls.entrySet()) {
                String methodName = entry.getKey();
                Object[] params = entry.getValue();
                
                // Dynamically determine parameter types
                Class<?>[] paramTypes = new Class<?>[params.length];
                for (int i = 0; i < params.length; i++) {
                    paramTypes[i] = params[i].getClass();
                    // Handle primitive types
                    if (paramTypes[i] == Integer.class) paramTypes[i] = int.class;
                    else if (paramTypes[i] == Double.class) paramTypes[i] = double.class;
                    else if (paramTypes[i] == Boolean.class) paramTypes[i] = boolean.class;
                    // Add more primitive type handling as needed
                }
                
                try {
                    Method method = clazz.getMethod(methodName, paramTypes);
                    Object result = method.invoke(instance, params);
                    System.out.println("Dynamic invocation result: " + result);
                } catch (NoSuchMethodException e) {
                    System.out.println("Method not found: " + methodName + " with params: " + 
                                     Arrays.toString(params));
                }
            }
            
        } catch (Exception e) {
            System.err.println("Error during dynamic invocation: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    public static void main(String[] args) {
        demonstrateBasicMethodInvocation();
        demonstrateAccessControlBypass();
        demonstrateParameterHandling();
        demonstrateErrorHandling();
        demonstratePerformanceOptimization();
        demonstrateDynamicInvocation();
    }
}
```

### Advanced Method Invocation Patterns

```java
// Advanced method invocation patterns and utilities
public class AdvancedMethodInvocation {
    
    // Method invocation with security manager considerations
    public static void demonstrateSecurityConsiderations() {
        System.out.println("=== Security Considerations ===");
        
        class SecurityDemo {
            private String sensitiveData = "sensitive information";
            
            private void privateMethod() {
                System.out.println("Accessing sensitive data: " + sensitiveData);
            }
        }
        
        try {
            // Check if security manager is present
            SecurityManager sm = System.getSecurityManager();
            if (sm != null) {
                System.out.println("Security manager is active");
            } else {
                System.out.println("No security manager active");
            }
            
            Class<?> clazz = SecurityDemo.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            
            // Attempting to access private members
            Method privateMethod = clazz.getDeclaredMethod("privateMethod");
            privateMethod.setAccessible(true);
            privateMethod.invoke(instance);
            
            System.out.println("Private method accessed successfully");
            
        } catch (SecurityException e) {
            System.out.println("Security exception: " + e.getMessage());
        } catch (Exception e) {
            System.err.println("Error: " + e.getMessage());
        }
    }
    
    // Method invocation with custom invocation handler
    public static void demonstrateCustomInvocationHandler() {
        System.out.println("\n=== Custom Invocation Handler ===");
        
        // Interface for dynamic proxy
        interface ServiceInterface {
            String process(String input);
            int calculate(int a, int b);
        }
        
        // Implementation
        class ServiceImpl implements ServiceInterface {
            @Override
            public String process(String input) {
                return "Processed: " + input;
            }
            
            @Override
            public int calculate(int a, int b) {
                return a + b;
            }
        }
        
        // Custom invocation handler with logging and timing
        class LoggingInvocationHandler implements InvocationHandler {
            private final Object target;
            private final Map<String, Long> callCounts = new ConcurrentHashMap<>();
            
            public LoggingInvocationHandler(Object target) {
                this.target = target;
            }
            
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                long startTime = System.nanoTime();
                
                // Log method call
                String methodName = method.getName();
                callCounts.merge(methodName, 1L, Long::sum);
                
                System.out.println("Calling method: " + methodName + 
                                 " with args: " + Arrays.toString(args));
                
                try {
                    // Invoke actual method
                    Object result = method.invoke(target, args);
                    
                    long duration = System.nanoTime() - startTime;
                    System.out.println("Method " + methodName + " completed in " + 
                                     duration / 1000 + " μs, result: " + result);
                    
                    return result;
                } catch (InvocationTargetException e) {
                    System.err.println("Method " + methodName + " threw exception: " + 
                                     e.getCause().getMessage());
                    throw e.getCause();
                }
            }
            
            public Map<String, Long> getCallCounts() {
                return new HashMap<>(callCounts);
            }
        }
        
        try {
            // Create proxy with custom handler
            ServiceInterface impl = new ServiceImpl();
            ServiceInterface proxy = (ServiceInterface) Proxy.newProxyInstance(
                ServiceInterface.class.getClassLoader(),
                new Class<?>[]{ServiceInterface.class},
                new LoggingInvocationHandler(impl)
            );
            
            // Use the proxy
            String result1 = proxy.process("test data");
            int result2 = proxy.calculate(10, 20);
            String result3 = proxy.process("more data");
            
            LoggingInvocationHandler handler = (LoggingInvocationHandler) 
                Proxy.getInvocationHandler(proxy);
            System.out.println("Call counts: " + handler.getCallCounts());
            
        } catch (Exception e) {
            System.err.println("Error in custom invocation handler: " + e.getMessage());
        }
    }
    
    // Batch method invocation utility
    public static void demonstrateBatchInvocation() {
        System.out.println("\n=== Batch Method Invocation ===");
        
        class BatchDemo {
            private int counter = 0;
            
            public synchronized int increment() {
                return ++counter;
            }
            
            public synchronized int decrement() {
                return --counter;
            }
            
            public synchronized int getValue() {
                return counter;
            }
            
            public synchronized void reset() {
                counter = 0;
            }
        }
        
        try {
            Class<?> clazz = BatchDemo.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            
            // Define batch operations
            List<MethodInvocation> batchOperations = Arrays.asList(
                new MethodInvocation("increment", new Class<?>[0], new Object[0]),
                new MethodInvocation("increment", new Class<?>[0], new Object[0]),
                new MethodInvocation("increment", new Class<?>[0], new Object[0]),
                new MethodInvocation("getValue", new Class<?>[0], new Object[0]),
                new MethodInvocation("decrement", new Class<?>[0], new Object[0]),
                new MethodInvocation("getValue", new Class<?>[0], new Object[0])
            );
            
            // Execute batch
            List<Object> results = executeBatch(instance, batchOperations);
            System.out.println("Batch results: " + results);
            
        } catch (Exception e) {
            System.err.println("Error in batch invocation: " + e.getMessage());
        }
    }
    
    // Helper class for batch invocation
    static class MethodInvocation {
        final String methodName;
        final Class<?>[] paramTypes;
        final Object[] args;
        
        MethodInvocation(String methodName, Class<?>[] paramTypes, Object[] args) {
            this.methodName = methodName;
            this.paramTypes = paramTypes != null ? paramTypes : new Class<?>[0];
            this.args = args != null ? args : new Object[0];
        }
    }
    
    // Batch execution utility
    public static List<Object> executeBatch(Object target, List<MethodInvocation> invocations) 
            throws Exception {
        List<Object> results = new ArrayList<>();
        
        for (MethodInvocation invocation : invocations) {
            Method method = target.getClass().getMethod(invocation.methodName, 
                                                      invocation.paramTypes);
            Object result = method.invoke(target, invocation.args);
            results.add(result);
        }
        
        return results;
    }
    
    // Method invocation with validation and type conversion
    public static void demonstrateValidatedInvocation() {
        System.out.println("\n=== Validated Method Invocation ===");
        
        class ValidationDemo {
            public String processString(String input) {
                if (input == null || input.isEmpty()) {
                    throw new IllegalArgumentException("Input cannot be null or empty");
                }
                return "Processed: " + input.toUpperCase();
            }
            
            public int processNumber(int number) {
                if (number < 0) {
                    throw new IllegalArgumentException("Number must be non-negative");
                }
                return number * 2;
            }
        }
        
        try {
            Class<?> clazz = ValidationDemo.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            
            // Valid invocations
            Method stringMethod = clazz.getMethod("processString", String.class);
            String validResult = (String) validatedInvoke(instance, stringMethod, "hello");
            System.out.println("Valid string result: " + validResult);
            
            Method numberMethod = clazz.getMethod("processNumber", int.class);
            Integer validNumberResult = (Integer) validatedInvoke(instance, numberMethod, 42);
            System.out.println("Valid number result: " + validNumberResult);
            
            // Invalid invocations
            try {
                validatedInvoke(instance, stringMethod, "");
            } catch (Exception e) {
                System.out.println("Caught validation error: " + e.getCause().getMessage());
            }
            
            try {
                validatedInvoke(instance, numberMethod, -5);
            } catch (Exception e) {
                System.out.println("Caught validation error: " + e.getCause().getMessage());
            }
            
        } catch (Exception e) {
            System.err.println("Error in validated invocation: " + e.getMessage());
        }
    }
    
    // Utility method with validation
    public static Object validatedInvoke(Object target, Method method, Object... args) 
            throws Exception {
        try {
            return method.invoke(target, args);
        } catch (InvocationTargetException e) {
            if (e.getCause() instanceof IllegalArgumentException) {
                throw e; // Re-throw validation errors
            }
            throw e; // Re-throw other errors
        }
    }
    
    // Asynchronous method invocation
    public static void demonstrateAsyncInvocation() {
        System.out.println("\n=== Asynchronous Method Invocation ===");
        
        class AsyncDemo {
            public String slowOperation(String input) {
                try {
                    Thread.sleep(1000); // Simulate slow operation
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                return "Processed: " + input;
            }
        }
        
        try {
            Class<?> clazz = AsyncDemo.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            Method method = clazz.getMethod("slowOperation", String.class);
            
            ExecutorService executor = Executors.newFixedThreadPool(3);
            
            // Submit multiple async invocations
            List<Future<Object>> futures = new ArrayList<>();
            
            long startTime = System.currentTimeMillis();
            
            for (int i = 0; i < 5; i++) {
                final int taskId = i;
                Future<Object> future = executor.submit(() -> {
                    try {
                        return method.invoke(instance, "task-" + taskId);
                    } catch (Exception e) {
                        throw new RuntimeException(e);
                    }
                });
                futures.add(future);
            }
            
            // Collect results
            List<String> results = new ArrayList<>();
            for (Future<Object> future : futures) {
                try {
                    results.add((String) future.get());
                } catch (ExecutionException e) {
                    System.err.println("Task failed: " + e.getCause().getMessage());
                }
            }
            
            long endTime = System.currentTimeMillis();
            
            System.out.println("Async results: " + results);
            System.out.println("Total time: " + (endTime - startTime) + " ms");
            
            executor.shutdown();
            
        } catch (Exception e) {
            System.err.println("Error in async invocation: " + e.getMessage());
        }
    }
    
    public static void main(String[] args) {
        demonstrateSecurityConsiderations();
        demonstrateCustomInvocationHandler();
        demonstrateBatchInvocation();
        demonstrateValidatedInvocation();
        demonstrateAsyncInvocation();
    }
}
```

## Field Access and Modification

Reflection provides powerful capabilities for accessing and modifying object fields at runtime.

### Field Access and Modification Techniques

```java
// Field access and modification using reflection
public class FieldAccessAndModification {
    
    // Sample class with various field types for demonstration
    public static class FieldDemo {
        private String privateString = "private value";
        protected int protectedInt = 42;
        public double publicDouble = 3.14;
        public static String staticString = "static value";
        private final String finalString = "final value";
        private int[] intArray = {1, 2, 3, 4, 5};
        private List<String> stringList = new ArrayList<>(Arrays.asList("a", "b", "c"));
        
        // For demonstrating nested object access
        private NestedObject nested = new NestedObject("nested value");
        
        @Override
        public String toString() {
            return String.format("FieldDemo{privateString='%s', protectedInt=%d, publicDouble=%.2f, " +
                               "staticString='%s', finalString='%s', intArray=%s, stringList=%s, nested=%s}",
                               privateString, protectedInt, publicDouble, staticString, 
                               finalString, Arrays.toString(intArray), stringList, nested);
        }
        
        static class NestedObject {
            private String nestedValue;
            
            public NestedObject(String value) {
                this.nestedValue = value;
            }
            
            @Override
            public String toString() {
                return "NestedObject{nestedValue='" + nestedValue + "'}";
            }
        }
    }
    
    public static void demonstrateBasicFieldAccess() {
        System.out.println("=== Basic Field Access ===");
        
        try {
            Class<?> clazz = FieldDemo.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            
            // Accessing public field
            Field publicField = clazz.getField("publicDouble");
            double publicValue = publicField.getDouble(instance);
            System.out.println("Public field value: " + publicValue);
            
            // Modifying public field
            publicField.setDouble(instance, 2.71);
            System.out.println("Modified public field: " + publicField.getDouble(instance));
            
            // Accessing static field
            Field staticField = clazz.getField("staticString");
            String staticValue = (String) staticField.get(null); // null for static fields
            System.out.println("Static field value: " + staticValue);
            
            // Modifying static field
            staticField.set(null, "modified static");
            System.out.println("Modified static field: " + staticField.get(null));
            
        } catch (Exception e) {
            System.err.println("Error in basic field access: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Accessing private and protected fields
    public static void demonstrateAccessControlBypass() {
        System.out.println("\n=== Access Control Bypass ===");
        
        try {
            Class<?> clazz = FieldDemo.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            
            // Accessing private field
            Field privateField = clazz.getDeclaredField("privateString");
            privateField.setAccessible(true); // Bypass access control
            String privateValue = (String) privateField.get(instance);
            System.out.println("Private field value: " + privateValue);
            
            // Modifying private field
            privateField.set(instance, "modified private value");
            System.out.println("Modified private field: " + privateField.get(instance));
            
            // Accessing protected field
            Field protectedField = clazz.getDeclaredField("protectedInt");
            protectedField.setAccessible(true);
            int protectedValue = protectedField.getInt(instance);
            System.out.println("Protected field value: " + protectedValue);
            
            // Modifying protected field
            protectedField.setInt(instance, 99);
            System.out.println("Modified protected field: " + protectedField.getInt(instance));
            
            System.out.println("Instance after modifications: " + instance);
            
        } catch (Exception e) {
            System.err.println("Error in access control bypass: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Working with arrays and collections
    public static void demonstrateArrayAndCollectionAccess() {
        System.out.println("\n=== Array and Collection Access ===");
        
        try {
            Class<?> clazz = FieldDemo.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            
            // Accessing array field
            Field arrayField = clazz.getDeclaredField("intArray");
            arrayField.setAccessible(true);
            int[] array = (int[]) arrayField.get(instance);
            System.out.println("Original array: " + Arrays.toString(array));
            
            // Modifying array elements
            array[0] = 100;
            array[2] = 300;
            System.out.println("Modified array: " + Arrays.toString(array));
            
            // Note: Changes to the array reference are reflected in the object
            System.out.println("Array in object: " + Arrays.toString((int[]) arrayField.get(instance)));
            
            // Accessing collection field
            Field listField = clazz.getDeclaredField("stringList");
            listField.setAccessible(true);
            List<String> list = (List<String>) listField.get(instance);
            System.out.println("Original list: " + list);
            
            // Modifying collection
            list.add("d");
            list.remove("a");
            System.out.println("Modified list: " + list);
            
            System.out.println("List in object: " + listField.get(instance));
            
        } catch (Exception e) {
            System.err.println("Error in array/collection access: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Working with nested objects
    public static void demonstrateNestedObjectAccess() {
        System.out.println("\n=== Nested Object Access ===");
        
        try {
            Class<?> clazz = FieldDemo.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            
            // Accessing nested object field
            Field nestedField = clazz.getDeclaredField("nested");
            nestedField.setAccessible(true);
            Object nestedObject = nestedField.get(instance);
            
            System.out.println("Nested object: " + nestedObject);
            
            // Accessing field within nested object
            Class<?> nestedClass = nestedObject.getClass();
            Field nestedValueField = nestedClass.getDeclaredField("nestedValue");
            nestedValueField.setAccessible(true);
            String nestedValue = (String) nestedValueField.get(nestedObject);
            System.out.println("Nested value: " + nestedValue);
            
            // Modifying nested object field
            nestedValueField.set(nestedObject, "modified nested value");
            System.out.println("Modified nested value: " + nestedValueField.get(nestedObject));
            
            System.out.println("Updated nested object: " + nestedField.get(instance));
            
        } catch (Exception e) {
            System.err.println("Error in nested object access: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Field access with type conversion and validation
    public static void demonstrateTypeSafeFieldAccess() {
        System.out.println("\n=== Type-Safe Field Access ===");
        
        try {
            Class<?> clazz = FieldDemo.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            
            // Generic field getter with type safety
            String privateString = getField(instance, "privateString", String.class);
            System.out.println("Type-safe private string: " + privateString);
            
            Integer protectedInt = getField(instance, "protectedInt", Integer.class);
            System.out.println("Type-safe protected int: " + protectedInt);
            
            Double publicDouble = getField(instance, "publicDouble", Double.class);
            System.out.println("Type-safe public double: " + publicDouble);
            
            // Generic field setter with type safety
            setField(instance, "privateString", "type-safe modified value");
            System.out.println("After type-safe modification: " + 
                             getField(instance, "privateString", String.class));
            
        } catch (Exception e) {
            System.err.println("Error in type-safe field access: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Generic field getter with type safety
    public static <T> T getField(Object target, String fieldName, Class<T> expectedType) 
            throws Exception {
        Class<?> clazz = target.getClass();
        Field field = clazz.getDeclaredField(fieldName);
        field.setAccessible(true);
        
        Object value = field.get(target);
        if (value != null && !expectedType.isAssignableFrom(value.getClass())) {
            throw new IllegalArgumentException("Field " + fieldName + " is not of type " + 
                                             expectedType.getName());
        }
        
        return expectedType.cast(value);
    }
    
    // Generic field setter with type safety
    public static void setField(Object target, String fieldName, Object value) throws Exception {
        Class<?> clazz = target.getClass();
        Field field = clazz.getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(target, value);
    }
    
    // Performance comparison of field access methods
    public static void demonstratePerformanceComparison() {
        System.out.println("\n=== Field Access Performance ===");
        
        try {
            Class<?> clazz = FieldDemo.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            Field field = clazz.getDeclaredField("publicDouble");
            
            // Direct field access (baseline)
            long startTime = System.nanoTime();
            FieldDemo demo = new FieldDemo();
            for (int i = 0; i < 1000000; i++) {
                double value = demo.publicDouble;
                demo.publicDouble = value + 0.1;
            }
            long directTime = System.nanoTime() - startTime;
            
            // Reflection field access
            field.setAccessible(true);
            startTime = System.nanoTime();
            for (int i = 0; i < 1000000; i++) {
                double value = field.getDouble(instance);
                field.setDouble(instance, value + 0.1);
            }
            long reflectionTime = System.nanoTime() - startTime;
            
            System.out.println("Direct access time: " + directTime / 1_000_000 + " ms");
            System.out.println("Reflection access time: " + reflectionTime / 1_000_000 + " ms");
            System.out.println("Reflection overhead: " + (double) reflectionTime / directTime + "x");
            
        } catch (Exception e) {
            System.err.println("Error in performance comparison: " + e.getMessage());
        }
    }
    
    // Field access with caching for performance
    public static void demonstrateCachedFieldAccess() {
        System.out.println("\n=== Cached Field Access ===");
        
        // Field cache to avoid repeated field lookup
        class FieldCache {
            private final Map<String, Field> fieldMap = new ConcurrentHashMap<>();
            
            public Field getField(Class<?> clazz, String fieldName) throws NoSuchFieldException {
                String key = clazz.getName() + "." + fieldName;
                return fieldMap.computeIfAbsent(key, k -> {
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
        
        try {
            Class<?> clazz = FieldDemo.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            FieldCache cache = new FieldCache();
            
            // Using cached field access
            Field field = cache.getField(clazz, "publicDouble");
            double value = field.getDouble(instance);
            System.out.println("Cached field access value: " + value);
            
            field.setDouble(instance, 9.99);
            System.out.println("Modified via cached access: " + field.getDouble(instance));
            
        } catch (Exception e) {
            System.err.println("Error in cached field access: " + e.getMessage());
        }
    }
    
    public static void main(String[] args) {
        demonstrateBasicFieldAccess();
        demonstrateAccessControlBypass();
        demonstrateArrayAndCollectionAccess();
        demonstrateNestedObjectAccess();
        demonstrateTypeSafeFieldAccess();
        demonstratePerformanceComparison();
        demonstrateCachedFieldAccess();
    }
}
```

### Advanced Field Manipulation Techniques

```java
// Advanced field manipulation and utilities
public class AdvancedFieldManipulation {
    
    // Utility class for deep field access and manipulation
    public static class DeepFieldAccessor {
        private final Map<String, Field> fieldCache = new ConcurrentHashMap<>();
        
        // Get field value by path (e.g., "nested.inner.value")
        public Object getFieldValue(Object target, String fieldPath) throws Exception {
            String[] fields = fieldPath.split("\\.");
            Object current = target;
            
            for (String fieldName : fields) {
                if (current == null) return null;
                
                Class<?> clazz = current.getClass();
                String cacheKey = clazz.getName() + "." + fieldName;
                Field field = fieldCache.computeIfAbsent(cacheKey, k -> {
                    try {
                        Field f = clazz.getDeclaredField(fieldName);
                        f.setAccessible(true);
                        return f;
                    } catch (NoSuchFieldException e) {
                        throw new RuntimeException(e);
                    }
                });
                
                current = field.get(current);
            }
            
            return current;
        }
        
        // Set field value by path
        public void setFieldValue(Object target, String fieldPath, Object value) throws Exception {
            String[] fields = fieldPath.split("\\.");
            Object current = target;
            
            // Navigate to the parent object
            for (int i = 0; i < fields.length - 1; i++) {
                String fieldName = fields[i];
                Class<?> clazz = current.getClass();
                String cacheKey = clazz.getName() + "." + fieldName;
                Field field = fieldCache.computeIfAbsent(cacheKey, k -> {
                    try {
                        Field f = clazz.getDeclaredField(fieldName);
                        f.setAccessible(true);
                        return f;
                    } catch (NoSuchFieldException e) {
                        throw new RuntimeException(e);
                    }
                });
                
                current = field.get(current);
                if (current == null) {
                    throw new IllegalArgumentException("Field " + fieldName + " is null");
                }
            }
            
            // Set the final field
            String finalFieldName = fields[fields.length - 1];
            Class<?> clazz = current.getClass();
            String cacheKey = clazz.getName() + "." + finalFieldName;
            Field field = fieldCache.computeIfAbsent(cacheKey, k -> {
                try {
                    Field f = clazz.getDeclaredField(finalFieldName);
                    f.setAccessible(true);
                    return f;
                } catch (NoSuchFieldException e) {
                    throw new RuntimeException(e);
                }
            });
            
            field.set(current, value);
        }
        
        // Get all field values as a map
        public Map<String, Object> getAllFieldValues(Object target) throws Exception {
            Map<String, Object> fieldValues = new HashMap<>();
            Class<?> clazz = target.getClass();
            
            while (clazz != null && clazz != Object.class) {
                Field[] fields = clazz.getDeclaredFields();
                for (Field field : fields) {
                    field.setAccessible(true);
                    fieldValues.put(field.getName(), field.get(target));
                }
                clazz = clazz.getSuperclass();
            }
            
            return fieldValues;
        }
    }
    
    // Demonstrate deep field access
    public static void demonstrateDeepFieldAccess() {
        System.out.println("=== Deep Field Access ===");
        
        class Level1 {
            private String level1Value = "level1";
            private Level2 level2 = new Level2();
        }
        
        class Level2 {
            private String level2Value = "level2";
            private Level3 level3 = new Level3();
        }
        
        class Level3 {
            private String level3Value = "level3";
        }
        
        try {
            Level1 root = new Level1();
            DeepFieldAccessor accessor = new DeepFieldAccessor();
            
            // Access nested fields
            Object value1 = accessor.getFieldValue(root, "level1Value");
            Object value2 = accessor.getFieldValue(root, "level2.level2Value");
            Object value3 = accessor.getFieldValue(root, "level2.level3.level3Value");
            
            System.out.println("Level1 value: " + value1);
            System.out.println("Level2 value: " + value2);
            System.out.println("Level3 value: " + value3);
            
            // Modify nested fields
            accessor.setFieldValue(root, "level2.level3.level3Value", "modified level3");
            Object modifiedValue = accessor.getFieldValue(root, "level2.level3.level3Value");
            System.out.println("Modified Level3 value: " + modifiedValue);
            
        } catch (Exception e) {
            System.err.println("Error in deep field access: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Field validation and constraint checking
    public static void demonstrateFieldValidation() {
        System.out.println("\n=== Field Validation ===");
        
        // Annotation for field validation
        @Retention(RetentionPolicy.RUNTIME)
        @Target(ElementType.FIELD)
        @interface NotNull {
            String message() default "Field cannot be null";
        }
        
        @Retention(RetentionPolicy.RUNTIME)
        @Target(ElementType.FIELD)
        @interface Range {
            int min() default Integer.MIN_VALUE;
            int max() default Integer.MAX_VALUE;
            String message() default "Value out of range";
        }
        
        class ValidatedClass {
            @NotNull
            private String name;
            
            @Range(min = 0, max = 150)
            private int age;
            
            private String email;
            
            public ValidatedClass(String name, int age) {
                this.name = name;
                this.age = age;
            }
        }
        
        // Validation utility
        class FieldValidator {
            public List<String> validate(Object target) throws Exception {
                List<String> errors = new ArrayList<>();
                Class<?> clazz = target.getClass();
                
                Field[] fields = clazz.getDeclaredFields();
                for (Field field : fields) {
                    field.setAccessible(true);
                    Object value = field.get(target);
                    
                    // Check @NotNull
                    NotNull notNull = field.getAnnotation(NotNull.class);
                    if (notNull != null && value == null) {
                        errors.add(notNull.message() + " (" + field.getName() + ")");
                    }
                    
                    // Check @Range for integer fields
                    Range range = field.getAnnotation(Range.class);
                    if (range != null && value instanceof Integer) {
                        int intValue = (Integer) value;
                        if (intValue < range.min() || intValue > range.max()) {
                            errors.add(range.message() + " (" + field.getName() + 
                                     ": " + intValue + ", range: " + range.min() + "-" + range.max() + ")");
                        }
                    }
                }
                
                return errors;
            }
        }
        
        try {
            ValidatedClass validInstance = new ValidatedClass("John", 25);
            ValidatedClass invalidInstance = new ValidatedClass(null, 200);
            
            FieldValidator validator = new FieldValidator();
            
            List<String> validErrors = validator.validate(validInstance);
            System.out.println("Valid instance errors: " + validErrors);
            
            List<String> invalidErrors = validator.validate(invalidInstance);
            System.out.println("Invalid instance errors: " + invalidErrors);
            
        } catch (Exception e) {
            System.err.println("Error in field validation: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Field copying and cloning utilities
    public static void demonstrateFieldCopying() {
        System.out.println("\n=== Field Copying and Cloning ===");
        
        class SourceClass {
            private String stringField = "source string";
            private int intField = 42;
            private double doubleField = 3.14;
            private List<String> listField = Arrays.asList("a", "b", "c");
        }
        
        class TargetClass {
            private String stringField;
            private int intField;
            private double doubleField;
            private List<String> listField;
        }
        
        // Field copying utility
        class FieldCopier {
            public void copyFields(Object source, Object target) throws Exception {
                Class<?> sourceClass = source.getClass();
                Class<?> targetClass = target.getClass();
                
                Field[] sourceFields = sourceClass.getDeclaredFields();
                for (Field sourceField : sourceFields) {
                    sourceField.setAccessible(true);
                    Object value = sourceField.get(source);
                    
                    try {
                        Field targetField = targetClass.getDeclaredField(sourceField.getName());
                        targetField.setAccessible(true);
                        
                        // Handle type compatibility
                        if (isAssignable(sourceField.getType(), targetField.getType())) {
                            targetField.set(target, value);
                        }
                    } catch (NoSuchFieldException e) {
                        // Skip fields that don't exist in target
                        System.out.println("Field " + sourceField.getName() + " not found in target");
                    }
                }
            }
            
            private boolean isAssignable(Class<?> sourceType, Class<?> targetType) {
                return targetType.isAssignableFrom(sourceType) || 
                       (sourceType.isPrimitive() && 
                        getWrapperClass(sourceType).isAssignableFrom(getWrapperClass(targetType)));
            }
            
            private Class<?> getWrapperClass(Class<?> primitiveType) {
                if (primitiveType == int.class) return Integer.class;
                if (primitiveType == double.class) return Double.class;
                if (primitiveType == boolean.class) return Boolean.class;
                // Add more primitive types as needed
                return primitiveType;
            }
        }
        
        try {
            SourceClass source = new SourceClass();
            TargetClass target = new TargetClass();
            
            FieldCopier copier = new FieldCopier();
            copier.copyFields(source, target);
            
            System.out.println("Copied fields:");
            System.out.println("  String field: " + target.stringField);
            System.out.println("  Int field: " + target.intField);
            System.out.println("  Double field: " + target.doubleField);
            System.out.println("  List field: " + target.listField);
            
        } catch (Exception e) {
            System.err.println("Error in field copying: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Field monitoring and change tracking
    public static void demonstrateFieldMonitoring() {
        System.out.println("\n=== Field Monitoring and Change Tracking ===");
        
        // Change tracking utility
        class FieldMonitor {
            private final Map<Object, Map<String, Object>> snapshots = new WeakHashMap<>();
            
            public void takeSnapshot(Object target) throws Exception {
                Map<String, Object> fieldValues = new HashMap<>();
                Class<?> clazz = target.getClass();
                
                while (clazz != null && clazz != Object.class) {
                    Field[] fields = clazz.getDeclaredFields();
                    for (Field field : fields) {
                        field.setAccessible(true);
                        fieldValues.put(field.getName(), field.get(target));
                    }
                    clazz = clazz.getSuperclass();
                }
                
                snapshots.put(target, fieldValues);
            }
            
            public List<FieldChange> getChanges(Object target) throws Exception {
                Map<String, Object> oldValues = snapshots.get(target);
                if (oldValues == null) {
                    throw new IllegalStateException("No snapshot found for object");
                }
                
                List<FieldChange> changes = new ArrayList<>();
                Class<?> clazz = target.getClass();
                
                while (clazz != null && clazz != Object.class) {
                    Field[] fields = clazz.getDeclaredFields();
                    for (Field field : fields) {
                        field.setAccessible(true);
                        Object newValue = field.get(target);
                        Object oldValue = oldValues.get(field.getName());
                        
                        if (!Objects.equals(oldValue, newValue)) {
                            changes.add(new FieldChange(field.getName(), oldValue, newValue));
                        }
                    }
                    clazz = clazz.getSuperclass();
                }
                
                return changes;
            }
        }
        
        class FieldChange {
            private final String fieldName;
            private final Object oldValue;
            private final Object newValue;
            
            public FieldChange(String fieldName, Object oldValue, Object newValue) {
                this.fieldName = fieldName;
                this.oldValue = oldValue;
                this.newValue = newValue;
            }
            
            @Override
            public String toString() {
                return fieldName + ": " + oldValue + " -> " + newValue;
            }
        }
        
        class MonitorDemo {
            private String name = "initial";
            private int value = 0;
            
            public void setName(String name) { this.name = name; }
            public void setValue(int value) { this.value = value; }
        }
        
        try {
            MonitorDemo demo = new MonitorDemo();
            FieldMonitor monitor = new FieldMonitor();
            
            // Take initial snapshot
            monitor.takeSnapshot(demo);
            
            // Make changes
            demo.setName("changed");
            demo.setValue(100);
            
            // Get changes
            List<FieldChange> changes = monitor.getChanges(demo);
            System.out.println("Field changes:");
            for (FieldChange change : changes) {
                System.out.println("  " + change);
            }
            
        } catch (Exception e) {
            System.err.println("Error in field monitoring: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Field serialization and deserialization
    public static void demonstrateFieldSerialization() {
        System.out.println("\n=== Field Serialization/Deserialization ===");
        
        class SerializableDemo {
            private String name = "test";
            private int age = 25;
            private double score = 95.5;
            private List<String> tags = Arrays.asList("tag1", "tag2");
        }
        
        // Simple field-based serialization
        class FieldSerializer {
            public Map<String, Object> serialize(Object obj) throws Exception {
                Map<String, Object> data = new HashMap<>();
                Class<?> clazz = obj.getClass();
                
                while (clazz != null && clazz != Object.class) {
                    Field[] fields = clazz.getDeclaredFields();
                    for (Field field : fields) {
                        field.setAccessible(true);
                        data.put(field.getName(), field.get(obj));
                    }
                    clazz = clazz.getSuperclass();
                }
                
                return data;
            }
            
            public <T> T deserialize(Map<String, Object> data, Class<T> clazz) throws Exception {
                T instance = clazz.getDeclaredConstructor().newInstance();
                
                for (Map.Entry<String, Object> entry : data.entrySet()) {
                    try {
                        Field field = clazz.getDeclaredField(entry.getKey());
                        field.setAccessible(true);
                        field.set(instance, entry.getValue());
                    } catch (NoSuchFieldException e) {
                        System.out.println("Field " + entry.getKey() + " not found in class");
                    }
                }
                
                return instance;
            }
        }
        
        try {
            SerializableDemo original = new SerializableDemo();
            FieldSerializer serializer = new FieldSerializer();
            
            // Serialize
            Map<String, Object> serialized = serializer.serialize(original);
            System.out.println("Serialized data: " + serialized);
            
            // Deserialize
            SerializableDemo deserialized = serializer.deserialize(serialized, SerializableDemo.class);
            Map<String, Object> deserializedData = serializer.serialize(deserialized);
            System.out.println("Deserialized data: " + deserializedData);
            
            System.out.println("Serialization successful: " + serialized.equals(deserializedData));
            
        } catch (Exception e) {
            System.err.println("Error in field serialization: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    public static void main(String[] args) {
        demonstrateDeepFieldAccess();
        demonstrateFieldValidation();
        demonstrateFieldCopying();
        demonstrateFieldMonitoring();
        demonstrateFieldSerialization();
    }
}
```

## Constructor Manipulation

Reflection allows for dynamic constructor invocation and object creation at runtime.

### Constructor Manipulation Techniques

```java
// Constructor manipulation and dynamic object creation
public class ConstructorManipulation {
    
    // Sample class with various constructors for demonstration
    public static class ConstructorDemo {
        private String name;
        private int age;
        private double score;
        private List<String> tags;
        private boolean initialized = false;
        
        // Default constructor
        public ConstructorDemo() {
            this.name = "default";
            this.age = 0;
            this.score = 0.0;
            this.tags = new ArrayList<>();
            this.initialized = true;
        }
        
        // Parameterized constructor
        public ConstructorDemo(String name, int age) {
            this.name = name;
            this.age = age;
            this.score = 0.0;
            this.tags = new ArrayList<>();
            this.initialized = true;
        }
        
        // Constructor with multiple parameters
        public ConstructorDemo(String name, int age, double score, List<String> tags) {
            this.name = name;
            this.age = age;
            this.score = score;
            this.tags = new ArrayList<>(tags);
            this.initialized = true;
        }
        
        // Private constructor
        private ConstructorDemo(String name) {
            this.name = name;
            this.age = 0;
            this.score = 0.0;
            this.tags = new ArrayList<>();
            this.initialized = true;
        }
        
        // Constructor that throws exception
        public ConstructorDemo(int riskyValue) throws IllegalArgumentException {
            if (riskyValue < 0) {
                throw new IllegalArgumentException("Risky value cannot be negative: " + riskyValue);
            }
            this.name = "risky";
            this.age = riskyValue;
            this.score = 0.0;
            this.tags = new ArrayList<>();
            this.initialized = true;
        }
        
        @Override
        public String toString() {
            return String.format("ConstructorDemo{name='%s', age=%d, score=%.1f, tags=%s, initialized=%b}",
                               name, age, score, tags, initialized);
        }
    }
    
    public static void demonstrateBasicConstructorAccess() {
        System.out.println("=== Basic Constructor Access ===");
        
        try {
            Class<?> clazz = ConstructorDemo.class;
            
            // Getting all constructors
            Constructor<?>[] constructors = clazz.getDeclaredConstructors();
            System.out.println("Available constructors:");
            for (int i = 0; i < constructors.length; i++) {
                Constructor<?> constructor = constructors[i];
                System.out.print("  " + i + ": ");
                Class<?>[] paramTypes = constructor.getParameterTypes();
                for (int j = 0; j < paramTypes.length; j++) {
                    if (j > 0) System.out.print(", ");
                    System.out.print(paramTypes[j].getSimpleName());
                }
                System.out.println(" | Modifiers: " + 
                                 java.lang.reflect.Modifier.toString(constructor.getModifiers()));
            }
            
            // Using default constructor
            Constructor<?> defaultConstructor = clazz.getConstructor();
            Object instance1 = defaultConstructor.newInstance();
            System.out.println("Default constructor result: " + instance1);
            
            // Using parameterized constructor
            Constructor<?> paramConstructor = clazz.getConstructor(String.class, int.class);
            Object instance2 = paramConstructor.newInstance("John", 25);
            System.out.println("Parameterized constructor result: " + instance2);
            
            // Using constructor with multiple parameters
            Constructor<?> multiParamConstructor = clazz.getConstructor(
                String.class, int.class, double.class, List.class);
            List<String> tags = Arrays.asList("tag1", "tag2");
            Object instance3 = multiParamConstructor.newInstance("Alice", 30, 95.5, tags);
            System.out.println("Multi-parameter constructor result: " + instance3);
            
        } catch (Exception e) {
            System.err.println("Error in basic constructor access: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Accessing private constructors
    public static void demonstratePrivateConstructorAccess() {
        System.out.println("\n=== Private Constructor Access ===");
        
        try {
            Class<?> clazz = ConstructorDemo.class;
            
            // Accessing private constructor
            Constructor<?> privateConstructor = clazz.getDeclaredConstructor(String.class);
            privateConstructor.setAccessible(true); // Bypass access control
            
            Object instance = privateConstructor.newInstance("Private Name");
            System.out.println("Private constructor result: " + instance);
            
        } catch (Exception e) {
            System.err.println("Error in private constructor access: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Constructor with exception handling
    public static void demonstrateConstructorExceptionHandling() {
        System.out.println("\n=== Constructor Exception Handling ===");
        
        try {
            Class<?> clazz = ConstructorDemo.class;
            
            // Constructor that throws exception
            Constructor<?> riskyConstructor = clazz.getConstructor(int.class);
            
            try {
                Object instance = riskyConstructor.newInstance(-5); // This should throw exception
                System.out.println("Unexpected success: " + instance);
            } catch (InvocationTargetException e) {
                System.out.println("Caught InvocationTargetException: " + e.getCause().getMessage());
                System.out.println("Original exception type: " + e.getCause().getClass().getName());
            }
            
            // Successful invocation
            Object instance = riskyConstructor.newInstance(10);
            System.out.println("Successful risky constructor: " + instance);
            
        } catch (Exception e) {
            System.err.println("Error in constructor exception handling: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Dynamic constructor selection based on parameters
    public static void demonstrateDynamicConstructorSelection() {
        System.out.println("\n=== Dynamic Constructor Selection ===");
        
        try {
            Class<?> clazz = ConstructorDemo.class;
            
            // Test cases for dynamic constructor selection
            Object[][] testCases = {
                {}, // Default constructor
                {"John", 25}, // String, int constructor
                {"Alice", 30, 95.5, Arrays.asList("tag1", "tag2")}, // Full constructor
                {"Private", "unused"}, // Should fail - no matching constructor
                {"OnlyName"} // Private constructor (String)
            };
            
            for (int i = 0; i < testCases.length; i++) {
                Object[] params = testCases[i];
                Class<?>[] paramTypes = new Class<?>[params.length];
                
                // Determine parameter types
                for (int j = 0; j < params.length; j++) {
                    paramTypes[j] = params[j].getClass();
                    // Handle primitive types and nulls
                    if (paramTypes[j] == Integer.class) paramTypes[j] = int.class;
                    else if (paramTypes[j] == Double.class) paramTypes[j] = double.class;
                }
                
                try {
                    Constructor<?> constructor = clazz.getConstructor(paramTypes);
                    Object instance = constructor.newInstance(params);
                    System.out.println("Test case " + i + " success: " + instance);
                } catch (NoSuchMethodException e) {
                    // Try declared constructors for private access
                    try {
                        Constructor<?> constructor = clazz.getDeclaredConstructor(paramTypes);
                        constructor.setAccessible(true);
                        Object instance = constructor.newInstance(params);
                        System.out.println("Test case " + i + " success (private): " + instance);
                    } catch (Exception ex) {
                        System.out.println("Test case " + i + " failed: No matching constructor");
                    }
                } catch (Exception e) {
                    System.out.println("Test case " + i + " failed: " + e.getMessage());
                }
            }
            
        } catch (Exception e) {
            System.err.println("Error in dynamic constructor selection: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Constructor performance comparison
    public static void demonstrateConstructorPerformance() {
        System.out.println("\n=== Constructor Performance Comparison ===");
        
        try {
            Class<?> clazz = ConstructorDemo.class;
            
            // Direct instantiation (baseline)
            long startTime = System.nanoTime();
            for (int i = 0; i < 100000; i++) {
                ConstructorDemo direct = new ConstructorDemo("test", 25);
            }
            long directTime = System.nanoTime() - startTime;
            
            // Reflection-based instantiation
            Constructor<?> constructor = clazz.getConstructor(String.class, int.class);
            startTime = System.nanoTime();
            for (int i = 0; i < 100000; i++) {
                Object reflected = constructor.newInstance("test", 25);
            }
            long reflectionTime = System.nanoTime() - startTime;
            
            System.out.println("Direct instantiation time: " + directTime / 1_000_000 + " ms");
            System.out.println("Reflection instantiation time: " + reflectionTime / 1_000_000 + " ms");
            System.out.println("Reflection overhead: " + (double) reflectionTime / directTime + "x");
            
        } catch (Exception e) {
            System.err.println("Error in constructor performance test: " + e.getMessage());
        }
    }
    
    // Factory pattern using reflection constructors
    public static void demonstrateReflectionFactory() {
        System.out.println("\n=== Reflection-Based Factory Pattern ===");
        
        // Generic factory using reflection
        class ReflectionFactory {
            private final Map<String, Constructor<?>> constructorCache = new ConcurrentHashMap<>();
            
            public <T> T createInstance(Class<T> clazz, Object... args) throws Exception {
                Class<?>[] paramTypes = new Class<?>[args.length];
                for (int i = 0; i < args.length; i++) {
                    paramTypes[i] = args[i].getClass();
                    // Handle primitive types
                    if (paramTypes[i] == Integer.class) paramTypes[i] = int.class;
                    else if (paramTypes[i] == Double.class) paramTypes[i] = double.class;
                    else if (paramTypes[i] == Boolean.class) paramTypes[i] = boolean.class;
                }
                
                String cacheKey = clazz.getName() + Arrays.toString(paramTypes);
                Constructor<?> constructor = constructorCache.computeIfAbsent(cacheKey, k -> {
                    try {
                        return clazz.getConstructor(paramTypes);
                    } catch (NoSuchMethodException e) {
                        try {
                            // Try declared constructors for private access
                            Constructor<?> declared = clazz.getDeclaredConstructor(paramTypes);
                            declared.setAccessible(true);
                            return declared;
                        } catch (NoSuchMethodException ex) {
                            throw new RuntimeException(ex);
                        }
                    }
                });
                
                @SuppressWarnings("unchecked")
                T instance = (T) constructor.newInstance(args);
                return instance;
            }
        }
        
        try {
            ReflectionFactory factory = new ReflectionFactory();
            
            // Create instances using factory
            ConstructorDemo instance1 = factory.createInstance(ConstructorDemo.class);
            System.out.println("Factory default instance: " + instance1);
            
            ConstructorDemo instance2 = factory.createInstance(ConstructorDemo.class, "Factory", 40);
            System.out.println("Factory parameterized instance: " + instance2);
            
            ConstructorDemo instance3 = factory.createInstance(ConstructorDemo.class, 
                                                             "Factory Full", 35, 88.8, 
                                                             Arrays.asList("factory", "test"));
            System.out.println("Factory full instance: " + instance3);
            
        } catch (Exception e) {
            System.err.println("Error in reflection factory: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Constructor parameter validation
    public static void demonstrateConstructorValidation() {
        System.out.println("\n=== Constructor Parameter Validation ===");
        
        // Annotation for constructor parameter validation
        @Retention(RetentionPolicy.RUNTIME)
        @Target(ElementType.PARAMETER)
        @interface NotEmpty {
            String message() default "Parameter cannot be empty";
        }
        
        @Retention(RetentionPolicy.RUNTIME)
        @Target(ElementType.PARAMETER)
        @interface Range {
            int min() default Integer.MIN_VALUE;
            int max() default Integer.MAX_VALUE;
            String message() default "Value out of range";
        }
        
        class ValidatedConstructorDemo {
            private final String name;
            private final int age;
            
            public ValidatedConstructorDemo(
                    @NotEmpty(message = "Name cannot be empty") String name,
                    @Range(min = 0, max = 150, message = "Age must be between 0-150") int age) {
                this.name = name;
                this.age = age;
            }
            
            @Override
            public String toString() {
                return String.format("ValidatedConstructorDemo{name='%s', age=%d}", name, age);
            }
        }
        
        // Constructor validation utility
        class ConstructorValidator {
            public List<String> validateConstructor(Constructor<?> constructor, Object... args) {
                List<String> errors = new ArrayList<>();
                Annotation[][] paramAnnotations = constructor.getParameterAnnotations();
                
                for (int i = 0; i < paramAnnotations.length && i < args.length; i++) {
                    Object arg = args[i];
                    Annotation[] annotations = paramAnnotations[i];
                    
                    for (Annotation annotation : annotations) {
                        if (annotation instanceof NotEmpty && arg instanceof String) {
                            NotEmpty notEmpty = (NotEmpty) annotation;
                            if (((String) arg).isEmpty()) {
                                errors.add(notEmpty.message() + " (parameter " + i + ")");
                            }
                        } else if (annotation instanceof Range && arg instanceof Integer) {
                            Range range = (Range) annotation;
                            int value = (Integer) arg;
                            if (value < range.min() || value > range.max()) {
                                errors.add(range.message() + " (parameter " + i + 
                                         ": " + value + ", range: " + range.min() + "-" + range.max() + ")");
                            }
                        }
                    }
                }
                
                return errors;
            }
        }
        
        try {
            Class<?> clazz = ValidatedConstructorDemo.class;
            Constructor<?> constructor = clazz.getConstructor(String.class, int.class);
            ConstructorValidator validator = new ConstructorValidator();
            
            // Valid parameters
            List<String> validErrors = validator.validateConstructor(constructor, "John", 25);
            System.out.println("Valid parameters errors: " + validErrors);
            
            // Invalid parameters
            List<String> invalidErrors = validator.validateConstructor(constructor, "", 200);
            System.out.println("Invalid parameters errors: " + invalidErrors);
            
            // Still create instances
            if (validErrors.isEmpty()) {
                Object validInstance = constructor.newInstance("Alice", 30);
                System.out.println("Valid instance: " + validInstance);
            }
            
            if (invalidErrors.isEmpty()) {
                Object invalidInstance = constructor.newInstance("", 200);
                System.out.println("Invalid instance: " + invalidInstance);
            } else {
                System.out.println("Skipping invalid instance creation due to validation errors");
            }
            
        } catch (Exception e) {
            System.err.println("Error in constructor validation: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    public static void main(String[] args) {
        demonstrateBasicConstructorAccess();
        demonstratePrivateConstructorAccess();
        demonstrateConstructorExceptionHandling();
        demonstrateDynamicConstructorSelection();
        demonstrateConstructorPerformance();
        demonstrateReflectionFactory();
        demonstrateConstructorValidation();
    }
}
```

## Annotations Processing

Annotations provide metadata about code and can be processed at runtime using reflection.

### Runtime Annotation Processing

```java
// Runtime annotation processing and analysis
public class AnnotationsProcessing {
    
    // Custom annotations for demonstration
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.TYPE, ElementType.METHOD, ElementType.FIELD})
    @interface Entity {
        String name() default "";
        String table() default "";
    }
    
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.FIELD, ElementType.PARAMETER})
    @interface Column {
        String name() default "";
        boolean nullable() default true;
        int length() default 255;
    }
    
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    @interface Query {
        String value();
        int timeout() default 30;
    }
    
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.PARAMETER)
    @interface Param {
        String value();
    }
    
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.TYPE, ElementType.METHOD})
    @interface Security {
        String[] roles() default {};
        boolean authenticated() default false;
    }
    
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    @interface Cacheable {
        String key() default "";
        long ttl() default 3600; // 1 hour default
        boolean evictOnUpdate() default true;
    }
    
    // Sample class with various annotations
    @Entity(name = "User", table = "users")
    @Security(roles = {"USER", "ADMIN"}, authenticated = true)
    public static class User {
        @Column(name = "id", nullable = false)
        private Long id;
        
        @Column(name = "username", nullable = false, length = 50)
        private String username;
        
        @Column(name = "email", nullable = true, length = 100)
        private String email;
        
        @Column(name = "age", nullable = true)
        private Integer age;
        
        public User() {}
        
        public User(Long id, String username, String email, Integer age) {
            this.id = id;
            this.username = username;
            this.email = email;
            this.age = age;
        }
        
        @Query(value = "SELECT * FROM users WHERE username = :username", timeout = 10)
        @Cacheable(key = "userByUsername", ttl = 1800)
        @Security(roles = {"USER", "ADMIN"})
        public User findByUsername(@Param("username") String username) {
            // Implementation would go here
            return new User(1L, username, username + "@example.com", 25);
        }
        
        @Query(value = "UPDATE users SET email = :email WHERE id = :id")
        @Security(roles = {"ADMIN"})
        public void updateEmail(@Param("id") Long id, @Param("email") String email) {
            // Implementation would go here
        }
        
        @Query(value = "DELETE FROM users WHERE id = :id", timeout = 5)
        @Security(roles = {"ADMIN"})
        public void delete(@Param("id") Long id) {
            // Implementation would go here
        }
        
        // Getters and setters
        public Long getId() { return id; }
        public void setId(Long id) { this.id = id; }
        
        public String getUsername() { return username; }
        public void setUsername(String username) { this.username = username; }
        
        public String getEmail() { return email; }
        public void setEmail(String email) { this.email = email; }
        
        public Integer getAge() { return age; }
        public void setAge(Integer age) { this.age = age; }
        
        @Override
        public String toString() {
            return String.format("User{id=%d, username='%s', email='%s', age=%s}", 
                               id, username, email, age);
        }
    }
    
    public static void demonstrateBasicAnnotationProcessing() {
        System.out.println("=== Basic Annotation Processing ===");
        
        try {
            Class<?> clazz = User.class;
            
            // Class-level annotations
            System.out.println("Class annotations:");
            Entity entity = clazz.getAnnotation(Entity.class);
            if (entity != null) {
                System.out.println("  @Entity(name='" + entity.name() + "', table='" + entity.table() + "')");
            }
            
            Security classSecurity = clazz.getAnnotation(Security.class);
            if (classSecurity != null) {
                System.out.println("  @Security(roles=" + Arrays.toString(classSecurity.roles()) + 
                                 ", authenticated=" + classSecurity.authenticated() + ")");
            }
            
            // Field annotations
            System.out.println("\nField annotations:");
            Field[] fields = clazz.getDeclaredFields();
            for (Field field : fields) {
                Column column = field.getAnnotation(Column.class);
                if (column != null) {
                    System.out.println("  " + field.getName() + ":");
                    System.out.println("    @Column(name='" + column.name() + 
                                     "', nullable=" + column.nullable() + 
                                     ", length=" + column.length() + ")");
                }
            }
            
            // Method annotations
            System.out.println("\nMethod annotations:");
            Method[] methods = clazz.getDeclaredMethods();
            for (Method method : methods) {
                Query query = method.getAnnotation(Query.class);
                Cacheable cacheable = method.getAnnotation(Cacheable.class);
                Security methodSecurity = method.getAnnotation(Security.class);
                
                if (query != null || cacheable != null || methodSecurity != null) {
                    System.out.println("  " + method.getName() + "():");
                    
                    if (query != null) {
                        System.out.println("    @Query(value='" + query.value() + 
                                         "', timeout=" + query.timeout() + ")");
                    }
                    
                    if (cacheable != null) {
                        System.out.println("    @Cacheable(key='" + cacheable.key() + 
                                         "', ttl=" + cacheable.ttl() + 
                                         ", evictOnUpdate=" + cacheable.evictOnUpdate() + ")");
                    }
                    
                    if (methodSecurity != null) {
                        System.out.println("    @Security(roles=" + 
                                         Arrays.toString(methodSecurity.roles()) + 
                                         ", authenticated=" + methodSecurity.authenticated() + ")");
                    }
                    
                    // Parameter annotations
                    Annotation[][] paramAnnotations = method.getParameterAnnotations();
                    Class<?>[] paramTypes = method.getParameterTypes();
                    for (int i = 0; i < paramAnnotations.length; i++) {
                        for (Annotation annotation : paramAnnotations[i]) {
                            if (annotation instanceof Param) {
                                Param param = (Param) annotation;
                                System.out.println("      Parameter " + i + " (" + 
                                                 paramTypes[i].getSimpleName() + "): @Param('" + 
                                                 param.value() + "')");
                            }
                        }
                    }
                }
            }
            
        } catch (Exception e) {
            System.err.println("Error in basic annotation processing: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Annotation processing for metadata extraction
    public static void demonstrateMetadataExtraction() {
        System.out.println("\n=== Metadata Extraction ===");
        
        try {
            Class<?> clazz = User.class;
            
            // Extract entity metadata
            Entity entity = clazz.getAnnotation(Entity.class);
            if (entity != null) {
                System.out.println("Entity Metadata:");
                System.out.println("  Name: " + entity.name());
                System.out.println("  Table: " + entity.table());
            }
            
            // Extract column metadata
            System.out.println("\nColumn Metadata:");
            Map<String, ColumnInfo> columnInfo = new HashMap<>();
            Field[] fields = clazz.getDeclaredFields();
            for (Field field : fields) {
                Column column = field.getAnnotation(Column.class);
                if (column != null) {
                    columnInfo.put(field.getName(), new ColumnInfo(
                        column.name().isEmpty() ? field.getName() : column.name(),
                        field.getType(),
                        column.nullable(),
                        column.length()
                    ));
                }
            }
            
            for (Map.Entry<String, ColumnInfo> entry : columnInfo.entrySet()) {
                System.out.println("  " + entry.getKey() + ": " + entry.getValue());
            }
            
            // Extract query metadata
            System.out.println("\nQuery Metadata:");
            Method[] methods = clazz.getDeclaredMethods();
            for (Method method : methods) {
                Query query = method.getAnnotation(Query.class);
                if (query != null) {
                    System.out.println("  " + method.getName() + "():");
                    System.out.println("    SQL: " + query.value());
                    System.out.println("    Timeout: " + query.timeout() + " seconds");
                    
                    // Extract parameter metadata
                    Annotation[][] paramAnnotations = method.getParameterAnnotations();
                    for (int i = 0; i < paramAnnotations.length; i++) {
                        for (Annotation annotation : paramAnnotations[i]) {
                            if (annotation instanceof Param) {
                                Param param = (Param) annotation;
                                System.out.println("    Parameter " + i + ": :" + param.value());
                            }
                        }
                    }
                }
            }
            
        } catch (Exception e) {
            System.err.println("Error in metadata extraction: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Helper class for column information
    static class ColumnInfo {
        private final String columnName;
        private final Class<?> type;
        private final boolean nullable;
        private final int length;
        
        public ColumnInfo(String columnName, Class<?> type, boolean nullable, int length) {
            this.columnName = columnName;
            this.type = type;
            this.nullable = nullable;
            this.length = length;
        }
        
        @Override
        public String toString() {
            return String.format("ColumnInfo{name='%s', type=%s, nullable=%b, length=%d}",
                               columnName, type.getSimpleName(), nullable, length);
        }
    }
    
    // Security annotation processing
    public static void demonstrateSecurityProcessing() {
        System.out.println("\n=== Security Annotation Processing ===");
        
        class SecurityContext {
            private final Set<String> userRoles;
            private final boolean isAuthenticated;
            
            public SecurityContext(Set<String> userRoles, boolean isAuthenticated) {
                this.userRoles = userRoles;
                this.isAuthenticated = isAuthenticated;
            }
            
            public boolean hasRole(String role) {
                return userRoles.contains(role);
            }
            
            public boolean isAuthenticated() {
                return isAuthenticated;
            }
        }
        
        // Security validator
        class SecurityValidator {
            public boolean canAccess(SecurityContext context, Class<?> clazz) {
                Security security = clazz.getAnnotation(Security.class);
                if (security == null) return true; // No security restrictions
                
                if (security.authenticated() && !context.isAuthenticated()) {
                    return false;
                }
                
                if (security.roles().length > 0) {
                    boolean hasAnyRole = false;
                    for (String role : security.roles()) {
                        if (context.hasRole(role)) {
                            hasAnyRole = true;
                            break;
                        }
                    }
                    if (!hasAnyRole) return false;
                }
                
                return true;
            }
            
            public boolean canAccess(SecurityContext context, Method method) {
                Security security = method.getAnnotation(Security.class);
                if (security == null) return true; // No security restrictions
                
                if (security.authenticated() && !context.isAuthenticated()) {
                    return false;
                }
                
                if (security.roles().length > 0) {
                    boolean hasAnyRole = false;
                    for (String role : security.roles()) {
                        if (context.hasRole(role)) {
                            hasAnyRole = true;
                            break;
                        }
                    }
                    if (!hasAnyRole) return false;
                }
                
                return true;
            }
        }
        
        try {
            Class<?> clazz = User.class;
            SecurityValidator validator = new SecurityValidator();
            
            // Test different security contexts
            SecurityContext adminContext = new SecurityContext(
                new HashSet<>(Arrays.asList("USER", "ADMIN")), true);
            SecurityContext userContext = new SecurityContext(
                new HashSet<>(Arrays.asList("USER")), true);
            SecurityContext guestContext = new SecurityContext(
                new HashSet<>(), false);
            
            System.out.println("Class access checks:");
            System.out.println("  Admin can access: " + validator.canAccess(adminContext, clazz));
            System.out.println("  User can access: " + validator.canAccess(userContext, clazz));
            System.out.println("  Guest can access: " + validator.canAccess(guestContext, clazz));
            
            // Method access checks
            Method findByUsername = clazz.getMethod("findByUsername", String.class);
            Method updateEmail = clazz.getMethod("updateEmail", Long.class, String.class);
            Method delete = clazz.getMethod("delete", Long.class);
            
            System.out.println("\nMethod access checks for findByUsername:");
            System.out.println("  Admin can access: " + validator.canAccess(adminContext, findByUsername));
            System.out.println("  User can access: " + validator.canAccess(userContext, findByUsername));
            System.out.println("  Guest can access: " + validator.canAccess(guestContext, findByUsername));
            
            System.out.println("\nMethod access checks for updateEmail:");
            System.out.println("  Admin can access: " + validator.canAccess(adminContext, updateEmail));
            System.out.println("  User can access: " + validator.canAccess(userContext, updateEmail));
            System.out.println("  Guest can access: " + validator.canAccess(guestContext, updateEmail));
            
            System.out.println("\nMethod access checks for delete:");
            System.out.println("  Admin can access: " + validator.canAccess(adminContext, delete));
            System.out.println("  User can access: " + validator.canAccess(userContext, delete));
            System.out.println("  Guest can access: " + validator.canAccess(guestContext, delete));
            
        } catch (Exception e) {
            System.err.println("Error in security processing: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Cache annotation processing
    public static void demonstrateCacheProcessing() {
        System.out.println("\n=== Cache Annotation Processing ===");
        
        // Simple cache implementation
        class SimpleCache {
            private final Map<String, CacheEntry> cache = new ConcurrentHashMap<>();
            
            public Object get(String key) {
                CacheEntry entry = cache.get(key);
                if (entry != null && !entry.isExpired()) {
                    System.out.println("Cache HIT for key: " + key);
                    return entry.getValue();
                } else {
                    if (entry != null) {
                        System.out.println("Cache EXPIRED for key: " + key);
                    } else {
                        System.out.println("Cache MISS for key: " + key);
                    }
                    cache.remove(key);
                    return null;
                }
            }
            
            public void put(String key, Object value, long ttl) {
                cache.put(key, new CacheEntry(value, System.currentTimeMillis() + ttl * 1000));
                System.out.println("Cached value for key: " + key + " (TTL: " + ttl + "s)");
            }
            
            public void evict(String key) {
                cache.remove(key);
                System.out.println("Evicted cache entry for key: " + key);
            }
            
            static class CacheEntry {
                private final Object value;
                private final long expiryTime;
                
                public CacheEntry(Object value, long expiryTime) {
                    this.value = value;
                    this.expiryTime = expiryTime;
                }
                
                public Object getValue() { return value; }
                public boolean isExpired() { return System.currentTimeMillis() > expiryTime; }
            }
        }
        
        // Cache processor
        class CacheProcessor {
            private final SimpleCache cache = new SimpleCache();
            
            public Object processCachedMethod(Method method, Object target, Object... args) 
                    throws Exception {
                Cacheable cacheable = method.getAnnotation(Cacheable.class);
                if (cacheable == null) {
                    // Not cacheable, invoke directly
                    return method.invoke(target, args);
                }
                
                // Generate cache key
                String cacheKey = generateCacheKey(cacheable.key(), method, args);
                
                // Try to get from cache
                Object cachedResult = cache.get(cacheKey);
                if (cachedResult != null) {
                    return cachedResult;
                }
                
                // Not in cache, invoke method
                Object result = method.invoke(target, args);
                
                // Cache the result
                cache.put(cacheKey, result, cacheable.ttl());
                
                return result;
            }
            
            private String generateCacheKey(String baseKey, Method method, Object... args) {
                if (!baseKey.isEmpty()) {
                    return baseKey;
                }
                
                StringBuilder key = new StringBuilder();
                key.append(method.getDeclaringClass().getName())
                   .append(".").append(method.getName());
                
                for (Object arg : args) {
                    key.append(".").append(arg != null ? arg.toString() : "null");
                }
                
                return key.toString();
            }
        }
        
        try {
            Class<?> clazz = User.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            CacheProcessor processor = new CacheProcessor();
            
            Method findByUsername = clazz.getMethod("findByUsername", String.class);
            
            // First call - should cache the result
            System.out.println("First call to findByUsername:");
            Object result1 = processor.processCachedMethod(findByUsername, instance, "john");
            System.out.println("Result: " + result1);
            
            // Second call - should use cache
            System.out.println("\nSecond call to findByUsername:");
            Object result2 = processor.processCachedMethod(findByUsername, instance, "john");
            System.out.println("Result: " + result2);
            
            // Different parameter - should not use cache
            System.out.println("\nCall with different parameter:");
            Object result3 = processor.processCachedMethod(findByUsername, instance, "alice");
            System.out.println("Result: " + result3);
            
        } catch (Exception e) {
            System.err.println("Error in cache processing: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Annotation inheritance and processing
    public static void demonstrateAnnotationInheritance() {
        System.out.println("\n=== Annotation Inheritance ===");
        
        @Entity(name = "BaseEntity")
        @Security(roles = {"BASE_USER"})
        static class BaseEntity {
            @Column(name = "base_id")
            protected Long baseId;
            
            @Query(value = "SELECT * FROM base WHERE id = :id")
            public BaseEntity findById(@Param("id") Long id) {
                return null;
            }
        }
        
        @Entity(name = "ExtendedEntity", table = "extended_table")
        @Security(roles = {"EXTENDED_USER", "ADMIN"}, authenticated = true)
        static class ExtendedEntity extends BaseEntity {
            @Column(name = "extended_field", length = 1000)
            private String extendedField;
            
            @Override
            @Cacheable(key = "extendedById", ttl = 7200)
            public ExtendedEntity findById(@Param("id") Long id) {
                return null;
            }
            
            @Query(value = "SELECT * FROM extended WHERE field = :field")
            @Security(roles = {"EXTENDED_USER"})
            public List<ExtendedEntity> findByField(@Param("field") String field) {
                return new ArrayList<>();
            }
        }
        
        try {
            Class<?> clazz = ExtendedEntity.class;
            
            System.out.println("ExtendedEntity annotations:");
            
            // Class annotations (including inherited)
            System.out.println("Class annotations:");
            Entity entity = clazz.getAnnotation(Entity.class);
            if (entity != null) {
                System.out.println("  @Entity(name='" + entity.name() + "', table='" + entity.table() + "')");
            }
            
            // Note: @Inherited is required for annotation inheritance
            // Security annotation is not inherited by default
            Security classSecurity = clazz.getAnnotation(Security.class);
            if (classSecurity != null) {
                System.out.println("  @Security(roles=" + Arrays.toString(classSecurity.roles()) + 
                                 ", authenticated=" + classSecurity.authenticated() + ")");
            } else {
                System.out.println("  No @Security annotation on ExtendedEntity");
            }
            
            // Field annotations
            System.out.println("\nField annotations:");
            Field[] fields = clazz.getDeclaredFields();
            for (Field field : fields) {
                Column column = field.getAnnotation(Column.class);
                if (column != null) {
                    System.out.println("  " + field.getName() + ": @Column(name='" + 
                                     column.name() + "', length=" + column.length() + ")");
                }
            }
            
            // Method annotations
            System.out.println("\nMethod annotations:");
            Method[] methods = clazz.getDeclaredMethods();
            for (Method method : methods) {
                Query query = method.getAnnotation(Query.class);
                Cacheable cacheable = method.getAnnotation(Cacheable.class);
                Security methodSecurity = method.getAnnotation(Security.class);
                
                if (query != null || cacheable != null || methodSecurity != null) {
                    System.out.println("  " + method.getName() + "():");
                    if (query != null) {
                        System.out.println("    @Query(value='" + query.value() + "')");
                    }
                    if (cacheable != null) {
                        System.out.println("    @Cacheable(key='" + cacheable.key() + 
                                         "', ttl=" + cacheable.ttl() + ")");
                    }
                    if (methodSecurity != null) {
                        System.out.println("    @Security(roles=" + 
                                         Arrays.toString(methodSecurity.roles()) + ")");
                    }
                }
            }
            
            // Inherited methods
            System.out.println("\nInherited method annotations:");
            Method inheritedMethod = clazz.getMethod("findById", Long.class);
            if (inheritedMethod.getDeclaringClass() != clazz) {
                System.out.println("  findById() is inherited from " + 
                                 inheritedMethod.getDeclaringClass().getSimpleName());
                
                Query query = inheritedMethod.getAnnotation(Query.class);
                if (query != null) {
                    System.out.println("    Inherited @Query: " + query.value());
                }
                
                Cacheable cacheable = inheritedMethod.getAnnotation(Cacheable.class);
                if (cacheable != null) {
                    System.out.println("    Overridden @Cacheable: " + cacheable.key());
                }
            }
            
        } catch (Exception e) {
            System.err.println("Error in annotation inheritance: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    public static void main(String[] args) {
        demonstrateBasicAnnotationProcessing();
        demonstrateMetadataExtraction();
        demonstrateSecurityProcessing();
        demonstrateCacheProcessing();
        demonstrateAnnotationInheritance();
    }
}
```

## Custom Annotation Creation

Creating custom annotations and their processors for specialized functionality.

### Custom Annotation Implementation

```java
// Custom annotation creation and processing
public class CustomAnnotationCreation {
    
    // Basic custom annotation
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    @interface Timed {
        String value() default "";
        boolean logExecution() default true;
    }
    
    // Validation annotation
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.FIELD, ElementType.PARAMETER})
    @interface Validated {
        String pattern() default "";
        int minLength() default 0;
        int maxLength() default Integer.MAX_VALUE;
        String message() default "Validation failed";
    }
    
    // Configuration annotation
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.TYPE)
    @interface Configuration {
        String name() default "";
        String version() default "1.0";
        String[] profiles() default {};
    }
    
    // Event handling annotation
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    @interface EventHandler {
        String event() default "";
        int priority() default 0;
        boolean async() default false;
    }
    
    // Dependency injection annotation
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.FIELD, ElementType.CONSTRUCTOR, ElementType.METHOD})
    @interface Inject {
        String name() default "";
        boolean required() default true;
    }
    
    // Transaction annotation
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    @interface Transactional {
        String value() default "REQUIRED";
        String[] rollbackFor() default {};
        String[] noRollbackFor() default {};
        int timeout() default 30;
    }
    
    // Sample class using custom annotations
    @Configuration(name = "UserServiceConfig", version = "2.0", profiles = {"production", "staging"})
    public static class UserService {
        @Inject(name = "userRepository")
        private UserRepository userRepository;
        
        @Inject(name = "emailService")
        private EmailService emailService;
        
        public UserService() {}
        
        @Inject
        public UserService(UserRepository userRepository) {
            this.userRepository = userRepository;
        }
        
        @Timed(value = "userCreation", logExecution = true)
        @Transactional(timeout = 60, rollbackFor = {"ValidationException"})
        public User createUser(@Validated(minLength = 3, maxLength = 50, 
                                        pattern = "^[a-zA-Z0-9_]+$") String username,
                              @Validated(minLength = 5, maxLength = 100) String email) 
                throws ValidationException {
            
            // Validate inputs
            validateUsername(username);
            validateEmail(email);
            
            // Create user
            User user = new User(username, email);
            userRepository.save(user);
            
            // Send welcome email
            emailService.sendWelcomeEmail(user);
            
            return user;
        }
        
        @EventHandler(event = "user.login", priority = 10)
        public void handleUserLogin(User user) {
            System.out.println("Handling user login for: " + user.getUsername());
            // Log user login, update last login time, etc.
        }
        
        @EventHandler(event = "user.logout", priority = 5, async = true)
        public void handleUserLogout(User user) {
            System.out.println("Handling user logout for: " + user.getUsername());
            // Async cleanup, session invalidation, etc.
        }
        
        @Timed(value = "userSearch")
        public List<User> searchUsers(@Validated(minLength = 1) String query) {
            return userRepository.findByUsernameContaining(query);
        }
        
        private void validateUsername(String username) throws ValidationException {
            if (username == null || username.length() < 3 || username.length() > 50) {
                throw new ValidationException("Username must be between 3-50 characters");
            }
            if (!username.matches("^[a-zA-Z0-9_]+$")) {
                throw new ValidationException("Username contains invalid characters");
            }
        }
        
        private void validateEmail(String email) throws ValidationException {
            if (email == null || email.length() < 5 || email.length() > 100) {
                throw new ValidationException("Email must be between 5-100 characters");
            }
            if (!email.contains("@")) {
                throw new ValidationException("Invalid email format");
            }
        }
    }
    
    // Supporting classes
    static class User {
        private String username;
        private String email;
        
        public User(String username, String email) {
            this.username = username;
            this.email = email;
        }
        
        public String getUsername() { return username; }
        public String getEmail() { return email; }
        
        @Override
        public String toString() {
            return "User{username='" + username + "', email='" + email + "'}";
        }
    }
    
    static class UserRepository {
        public void save(User user) {
            System.out.println("Saving user: " + user);
        }
        
        public List<User> findByUsernameContaining(String query) {
            System.out.println("Searching for users containing: " + query);
            return Arrays.asList(new User("john_doe", "john@example.com"));
        }
    }
    
    static class EmailService {
        public void sendWelcomeEmail(User user) {
            System.out.println("Sending welcome email to: " + user.getEmail());
        }
    }
    
    static class ValidationException extends Exception {
        public ValidationException(String message) {
            super(message);
        }
    }
    
    public static void demonstrateBasicCustomAnnotations() {
        System.out.println("=== Basic Custom Annotations ===");
        
        try {
            Class<?> clazz = UserService.class;
            
            // Configuration annotation
            Configuration config = clazz.getAnnotation(Configuration.class);
            if (config != null) {
                System.out.println("@Configuration:");
                System.out.println("  Name: " + config.name());
                System.out.println("  Version: " + config.version());
                System.out.println("  Profiles: " + Arrays.toString(config.profiles()));
            }
            
            // Field annotations
            System.out.println("\nField annotations:");
            Field[] fields = clazz.getDeclaredFields();
            for (Field field : fields) {
                Inject inject = field.getAnnotation(Inject.class);
                if (inject != null) {
                    System.out.println("  " + field.getName() + ": @Inject(name='" + 
                                     inject.name() + "', required=" + inject.required() + ")");
                }
            }
            
            // Constructor annotations
            System.out.println("\nConstructor annotations:");
            Constructor<?>[] constructors = clazz.getDeclaredConstructors();
            for (Constructor<?> constructor : constructors) {
                Inject inject = constructor.getAnnotation(Inject.class);
                if (inject != null) {
                    System.out.print("  Constructor with params: ");
                    Class<?>[] paramTypes = constructor.getParameterTypes();
                    for (int i = 0; i < paramTypes.length; i++) {
                        if (i > 0) System.out.print(", ");
                        System.out.print(paramTypes[i].getSimpleName());
                    }
                    System.out.println(" - @Inject");
                }
            }
            
            // Method annotations
            System.out.println("\nMethod annotations:");
            Method[] methods = clazz.getDeclaredMethods();
            for (Method method : methods) {
                Timed timed = method.getAnnotation(Timed.class);
                Transactional transactional = method.getAnnotation(Transactional.class);
                EventHandler eventHandler = method.getAnnotation(EventHandler.class);
                
                if (timed != null || transactional != null || eventHandler != null) {
                    System.out.println("  " + method.getName() + "():");
                    
                    if (timed != null) {
                        System.out.println("    @Timed(value='" + timed.value() + 
                                         "', logExecution=" + timed.logExecution() + ")");
                    }
                    
                    if (transactional != null) {
                        System.out.println("    @Transactional(value='" + transactional.value() + 
                                         "', timeout=" + transactional.timeout() + 
                                         ", rollbackFor=" + Arrays.toString(transactional.rollbackFor()) + ")");
                    }
                    
                    if (eventHandler != null) {
                        System.out.println("    @EventHandler(event='" + eventHandler.event() + 
                                         "', priority=" + eventHandler.priority() + 
                                         ", async=" + eventHandler.async() + ")");
                    }
                    
                    // Parameter annotations
                    Annotation[][] paramAnnotations = method.getParameterAnnotations();
                    Class<?>[] paramTypes = method.getParameterTypes();
                    for (int i = 0; i < paramAnnotations.length; i++) {
                        for (Annotation annotation : paramAnnotations[i]) {
                            if (annotation instanceof Validated) {
                                Validated validated = (Validated) annotation;
                                System.out.println("      Parameter " + i + " (" + 
                                                 paramTypes[i].getSimpleName() + "):");
                                System.out.println("        @Validated(pattern='" + validated.pattern() + 
                                                 "', minLength=" + validated.minLength() + 
                                                 ", maxLength=" + validated.maxLength() + 
                                                 ", message='" + validated.message() + "')");
                            }
                        }
                    }
                }
            }
            
        } catch (Exception e) {
            System.err.println("Error in basic custom annotations: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Annotation processor for timing operations
    public static void demonstrateTimingProcessor() {
        System.out.println("\n=== Timing Annotation Processor ===");
        
        class TimingProcessor {
            public Object processTimedMethod(Method method, Object target, Object... args) 
                    throws Exception {
                Timed timed = method.getAnnotation(Timed.class);
                if (timed == null) {
                    // Not timed, invoke directly
                    return method.invoke(target, args);
                }
                
                long startTime = System.nanoTime();
                String operationName = timed.value().isEmpty() ? 
                    method.getName() : timed.value();
                
                try {
                    Object result = method.invoke(target, args);
                    
                    long duration = System.nanoTime() - startTime;
                    if (timed.logExecution()) {
                        System.out.println("Operation '" + operationName + 
                                         "' completed in " + duration / 1_000_000 + " ms");
                    }
                    
                    return result;
                } catch (InvocationTargetException e) {
                    long duration = System.nanoTime() - startTime;
                    if (timed.logExecution()) {
                        System.out.println("Operation '" + operationName + 
                                         "' failed after " + duration / 1_000_000 + " ms");
                    }
                    throw e.getCause();
                }
            }
        }
        
        try {
            Class<?> clazz = UserService.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            
            // Set up dependencies (simplified for demo)
            Field userRepoField = clazz.getDeclaredField("userRepository");
            userRepoField.setAccessible(true);
            userRepoField.set(instance, new UserRepository());
            
            Field emailServiceField = clazz.getDeclaredField("emailService");
            emailServiceField.setAccessible(true);
            emailServiceField.set(instance, new EmailService());
            
            TimingProcessor processor = new TimingProcessor();
            
            // Process timed methods
            Method createUser = clazz.getMethod("createUser", String.class, String.class);
            Method searchUsers = clazz.getMethod("searchUsers", String.class);
            
            System.out.println("Processing timed createUser method:");
            try {
                Object result = processor.processTimedMethod(createUser, instance, "john_doe", "john@example.com");
                System.out.println("Create user result: " + result);
            } catch (ValidationException e) {
                System.out.println("Validation error: " + e.getMessage());
            }
            
            System.out.println("\nProcessing timed searchUsers method:");
            Object result2 = processor.processTimedMethod(searchUsers, instance, "john");
            System.out.println("Search users result: " + result2);
            
        } catch (Exception e) {
            System.err.println("Error in timing processor: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Annotation processor for validation
    public static void demonstrateValidationProcessor() {
        System.out.println("\n=== Validation Annotation Processor ===");
        
        class ValidationProcessor {
            public void validateParameters(Method method, Object... args) throws ValidationException {
                Annotation[][] paramAnnotations = method.getParameterAnnotations();
                
                for (int i = 0; i < paramAnnotations.length && i < args.length; i++) {
                    Object arg = args[i];
                    if (arg == null) continue;
                    
                    for (Annotation annotation : paramAnnotations[i]) {
                        if (annotation instanceof Validated) {
                            Validated validated = (Validated) annotation;
                            validateArgument(arg, validated, i);
                        }
                    }
                }
            }
            
            private void validateArgument(Object arg, Validated validated, int paramIndex) 
                    throws ValidationException {
                if (!(arg instanceof String)) {
                    // Only validate strings in this example
                    return;
                }
                
                String value = (String) arg;
                
                // Length validation
                if (value.length() < validated.minLength()) {
                    throw new ValidationException("Parameter " + paramIndex + 
                                                ": " + validated.message() + 
                                                " - minimum length " + validated.minLength());
                }
                
                if (value.length() > validated.maxLength()) {
                    throw new ValidationException("Parameter " + paramIndex + 
                                                ": " + validated.message() + 
                                                " - maximum length " + validated.maxLength());
                }
                
                // Pattern validation
                if (!validated.pattern().isEmpty()) {
                    if (!value.matches(validated.pattern())) {
                        throw new ValidationException("Parameter " + paramIndex + 
                                                    ": " + validated.message() + 
                                                    " - pattern mismatch");
                    }
                }
            }
        }
        
        try {
            Class<?> clazz = UserService.class;
            Method createUser = clazz.getMethod("createUser", String.class, String.class);
            ValidationProcessor processor = new ValidationProcessor();
            
            System.out.println("Validating valid parameters:");
            try {
                processor.validateParameters(createUser, "john_doe", "john@example.com");
                System.out.println("Validation passed for valid parameters");
            } catch (ValidationException e) {
                System.out.println("Unexpected validation error: " + e.getMessage());
            }
            
            System.out.println("\nValidating invalid username:");
            try {
                processor.validateParameters(createUser, "jo", "john@example.com");
                System.out.println("Validation should have failed");
            } catch (ValidationException e) {
                System.out.println("Validation correctly failed: " + e.getMessage());
            }
            
            System.out.println("\nValidating invalid email:");
            try {
                processor.validateParameters(createUser, "john_doe", "invalid-email");
                System.out.println("Validation should have failed");
            } catch (ValidationException e) {
                System.out.println("Validation correctly failed: " + e.getMessage());
            }
            
            System.out.println("\nValidating invalid characters in username:");
            try {
                processor.validateParameters(createUser, "john doe!", "john@example.com");
                System.out.println("Validation should have failed");
            } catch (ValidationException e) {
                System.out.println("Validation correctly failed: " + e.getMessage());
            }
            
        } catch (Exception e) {
            System.err.println("Error in validation processor: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Event handling with annotations
    public static void demonstrateEventHandling() {
        System.out.println("\n=== Event Handling with Annotations ===");
        
        // Event system
        class EventSystem {
            private final Map<String, List<EventHandlerInfo>> eventHandlers = new HashMap<>();
            private final ExecutorService executor = Executors.newCachedThreadPool();
            
            public void registerHandlers(Object handlerInstance) throws Exception {
                Class<?> clazz = handlerInstance.getClass();
                Method[] methods = clazz.getDeclaredMethods();
                
                for (Method method : methods) {
                    EventHandler eventHandler = method.getAnnotation(EventHandler.class);
                    if (eventHandler != null) {
                        method.setAccessible(true);
                        String event = eventHandler.event().isEmpty() ? 
                            method.getName() : eventHandler.event();
                        
                        eventHandlers.computeIfAbsent(event, k -> new ArrayList<>())
                            .add(new EventHandlerInfo(method, handlerInstance, eventHandler));
                        
                        System.out.println("Registered handler for event: " + event + 
                                         " (priority: " + eventHandler.priority() + 
                                         ", async: " + eventHandler.async() + ")");
                    }
                }
                
                // Sort handlers by priority
                for (List<EventHandlerInfo> handlers : eventHandlers.values()) {
                    handlers.sort(Comparator.comparingInt(h -> h.annotation.priority()));
                }
            }
            
            public void fireEvent(String event, Object... args) {
                List<EventHandlerInfo> handlers = eventHandlers.get(event);
                if (handlers == null || handlers.isEmpty()) {
                    System.out.println("No handlers registered for event: " + event);
                    return;
                }
                
                System.out.println("Firing event: " + event + " with " + handlers.size() + " handlers");
                
                for (EventHandlerInfo handlerInfo : handlers) {
                    if (handlerInfo.annotation.async()) {
                        // Handle asynchronously
                        executor.submit(() -> {
                            try {
                                invokeHandler(handlerInfo, args);
                            } catch (Exception e) {
                                System.err.println("Async handler error: " + e.getMessage());
                            }
                        });
                    } else {
                        // Handle synchronously
                        try {
                            invokeHandler(handlerInfo, args);
                        } catch (Exception e) {
                            System.err.println("Sync handler error: " + e.getMessage());
                        }
                    }
                }
            }
            
            private void invokeHandler(EventHandlerInfo handlerInfo, Object... args) throws Exception {
                Class<?>[] paramTypes = handlerInfo.method.getParameterTypes();
                if (paramTypes.length != args.length) {
                    throw new IllegalArgumentException("Parameter count mismatch");
                }
                
                // Simple type checking (in real implementation, would be more sophisticated)
                handlerInfo.method.invoke(handlerInfo.instance, args);
            }
            
            public void shutdown() {
                executor.shutdown();
            }
        }
        
        class EventHandlerInfo {
            final Method method;
            final Object instance;
            final EventHandler annotation;
            
            EventHandlerInfo(Method method, Object instance, EventHandler annotation) {
                this.method = method;
                this.instance = instance;
                this.annotation = annotation;
            }
        }
        
        try {
            UserService userService = new UserService();
            EventSystem eventSystem = new EventSystem();
            
            // Register event handlers
            eventSystem.registerHandlers(userService);
            
            // Fire events
            System.out.println("\nFiring user.login event:");
            eventSystem.fireEvent("user.login", new User("john_doe", "john@example.com"));
            
            System.out.println("\nFiring user.logout event:");
            eventSystem.fireEvent("user.logout", new User("john_doe", "john@example.com"));
            
            // Give async handlers time to complete
            Thread.sleep(100);
            
            eventSystem.shutdown();
            
        } catch (Exception e) {
            System.err.println("Error in event handling: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Dependency injection with annotations
    public static void demonstrateDependencyInjection() {
        System.out.println("\n=== Dependency Injection with Annotations ===");
        
        // Simple DI container
        class DIContainer {
            private final Map<String, Object> instances = new HashMap<>();
            private final Map<Class<?>, Object> typeInstances = new HashMap<>();
            
            public void registerInstance(String name, Object instance) {
                instances.put(name, instance);
                typeInstances.put(instance.getClass(), instance);
                System.out.println("Registered instance: " + name + " -> " + instance.getClass().getSimpleName());
            }
            
            public void injectDependencies(Object target) throws Exception {
                Class<?> clazz = target.getClass();
                
                // Inject fields
                Field[] fields = clazz.getDeclaredFields();
                for (Field field : fields) {
                    Inject inject = field.getAnnotation(Inject.class);
                    if (inject != null) {
                        field.setAccessible(true);
                        Object dependency = resolveDependency(field.getType(), inject.name());
                        if (dependency != null) {
                            field.set(target, dependency);
                            System.out.println("Injected dependency into " + field.getName() + 
                                             ": " + dependency.getClass().getSimpleName());
                        } else if (inject.required()) {
                            throw new RuntimeException("Required dependency not found: " + 
                                                     field.getName());
                        }
                    }
                }
                
                // Inject constructor parameters
                Constructor<?>[] constructors = clazz.getDeclaredConstructors();
                for (Constructor<?> constructor : constructors) {
                    Inject inject = constructor.getAnnotation(Inject.class);
                    if (inject != null) {
                        System.out.println("Constructor injection not implemented in this example");
                        break;
                    }
                }
            }
            
            private Object resolveDependency(Class<?> type, String name) {
                if (!name.isEmpty()) {
                    return instances.get(name);
                }
                return typeInstances.get(type);
            }
        }
        
        try {
            // Create container and register dependencies
            DIContainer container = new DIContainer();
            container.registerInstance("userRepository", new UserRepository());
            container.registerInstance("emailService", new EmailService());
            
            // Create service and inject dependencies
            UserService userService = new UserService();
            container.injectDependencies(userService);
            
            System.out.println("Dependencies injected successfully");
            
        } catch (Exception e) {
            System.err.println("Error in dependency injection: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    public static void main(String[] args) {
        demonstrateBasicCustomAnnotations();
        demonstrateTimingProcessor();
        demonstrateValidationProcessor();
        demonstrateEventHandling();
        demonstrateDependencyInjection();
    }
}
```

## Dynamic Proxy Implementation

Dynamic proxies provide a way to create proxy instances that implement specified interfaces at runtime.

### Dynamic Proxy Fundamentals

```java
// Dynamic proxy implementation and advanced usage
public class DynamicProxyImplementation {
    
    // Interfaces for proxy demonstration
    interface UserService {
        User createUser(String username, String email);
        User findUserById(Long id);
        List<User> findAllUsers();
        void deleteUser(Long id);
    }
    
    interface EmailService {
        void sendWelcomeEmail(User user);
        void sendNotificationEmail(User user, String message);
        void sendPasswordResetEmail(User user);
    }
    
    interface LoggingService {
        void logInfo(String message);
        void logWarning(String message);
        void logError(String message);
    }
    
    // Implementation classes
    static class UserServiceImpl implements UserService {
        @Override
        public User createUser(String username, String email) {
            System.out.println("Creating user: " + username + " (" + email + ")");
            return new User(username, email);
        }
        
        @Override
        public User findUserById(Long id) {
            System.out.println("Finding user by ID: " + id);
            return new User("user" + id, "user" + id + "@example.com");
        }
        
        @Override
        public List<User> findAllUsers() {
            System.out.println("Finding all users");
            return Arrays.asList(
                new User("user1", "user1@example.com"),
                new User("user2", "user2@example.com")
            );
        }
        
        @Override
        public void deleteUser(Long id) {
            System.out.println("Deleting user with ID: " + id);
        }
    }
    
    static class EmailServiceImpl implements EmailService {
        @Override
        public void sendWelcomeEmail(User user) {
            System.out.println("Sending welcome email to: " + user.getEmail());
        }
        
        @Override
        public void sendNotificationEmail(User user, String message) {
            System.out.println("Sending notification to " + user.getEmail() + ": " + message);
        }
        
        @Override
        public void sendPasswordResetEmail(User user) {
            System.out.println("Sending password reset email to: " + user.getEmail());
        }
    }
    
    static class LoggingServiceImpl implements LoggingService {
        @Override
        public void logInfo(String message) {
            System.out.println("[INFO] " + message);
        }
        
        @Override
        public void logWarning(String message) {
            System.out.println("[WARNING] " + message);
        }
        
        @Override
        public void logError(String message) {
            System.out.println("[ERROR] " + message);
        }
    }
    
    // Supporting classes
    static class User {
        private String username;
        private String email;
        
        public User(String username, String email) {
            this.username = username;
            this.email = email;
        }
        
        public String getUsername() { return username; }
        public String getEmail() { return email; }
        
        @Override
        public String toString() {
            return "User{username='" + username + "', email='" + email + "'}";
        }
    }
    
    // Basic dynamic proxy example
    public static void demonstrateBasicProxy() {
        System.out.println("=== Basic Dynamic Proxy ===");
        
        // Create invocation handler
        class SimpleInvocationHandler implements InvocationHandler {
            private final Object target;
            
            public SimpleInvocationHandler(Object target) {
                this.target = target;
            }
            
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                System.out.println("Before method: " + method.getName());
                
                Object result = method.invoke(target, args);
                
                System.out.println("After method: " + method.getName());
                return result;
            }
        }
        
        try {
            // Create proxy for UserService
            UserService userService = new UserServiceImpl();
            UserService proxy = (UserService) Proxy.newProxyInstance(
                UserService.class.getClassLoader(),
                new Class<?>[]{UserService.class},
                new SimpleInvocationHandler(userService)
            );
            
            // Use the proxy
            User user = proxy.createUser("john", "john@example.com");
            User foundUser = proxy.findUserById(1L);
            List<User> users = proxy.findAllUsers();
            
            System.out.println("Proxy results:");
            System.out.println("  Created user: " + user);
            System.out.println("  Found user: " + foundUser);
            System.out.println("  All users count: " + users.size());
            
        } catch (Exception e) {
            System.err.println("Error in basic proxy: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Advanced proxy with multiple cross-cutting concerns
    public static void demonstrateAdvancedProxy() {
        System.out.println("\n=== Advanced Dynamic Proxy ===");
        
        // Multi-purpose invocation handler
        class AdvancedInvocationHandler implements InvocationHandler {
            private final Object target;
            private final Map<String, Long> callCounts = new ConcurrentHashMap<>();
            private final long startTime;
            
            public AdvancedInvocationHandler(Object target) {
                this.target = target;
                this.startTime = System.currentTimeMillis();
            }
            
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                String methodName = method.getName();
                long callStart = System.nanoTime();
                
                // Increment call count
                callCounts.merge(methodName, 1L, Long::sum);
                
                // Logging
                System.out.println("[" + Thread.currentThread().getName() + "] " +
                                 "Calling " + target.getClass().getSimpleName() + "." + methodName +
                                 " with args: " + Arrays.toString(args));
                
                try {
                    // Method execution
                    Object result = method.invoke(target, args);
                    
                    long duration = System.nanoTime() - callStart;
                    System.out.println("[" + Thread.currentThread().getName() + "] " +
                                     methodName + " completed in " + duration / 1000 + " μs" +
                                     ", result: " + (result != null ? result.toString() : "void"));
                    
                    return result;
                } catch (InvocationTargetException e) {
                    long duration = System.nanoTime() - callStart;
                    System.err.println("[" + Thread.currentThread().getName() + "] " +
                                     methodName + " failed after " + duration / 1000 + " μs: " +
                                     e.getCause().getMessage());
                    throw e.getCause();
                }
            }
            
            public Map<String, Long> getCallCounts() {
                return new HashMap<>(callCounts);
            }
            
            public long getUptime() {
                return System.currentTimeMillis() - startTime;
            }
        }
        
        try {
            // Create advanced proxy for UserService
            UserService userService = new UserServiceImpl();
            AdvancedInvocationHandler handler = new AdvancedInvocationHandler(userService);
            
            UserService proxy = (UserService) Proxy.newProxyInstance(
                UserService.class.getClassLoader(),
                new Class<?>[]{UserService.class},
                handler
            );
            
            // Use the proxy with various operations
            System.out.println("=== Proxy Operations ===");
            User user1 = proxy.createUser("alice", "alice@example.com");
            User user2 = proxy.createUser("bob", "bob@example.com");
            
            User foundUser = proxy.findUserById(1L);
            List<User> allUsers = proxy.findAllUsers();
            proxy.deleteUser(2L);
            
            // Get proxy statistics
            System.out.println("\n=== Proxy Statistics ===");
            System.out.println("Call counts: " + handler.getCallCounts());
            System.out.println("Proxy uptime: " + handler.getUptime() + " ms");
            
        } catch (Exception e) {
            System.err.println("Error in advanced proxy: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Security-aware proxy
    public static void demonstrateSecurityProxy() {
        System.out.println("\n=== Security-Aware Proxy ===");
        
        // Security context
        class SecurityContext {
            private final Set<String> roles;
            private final boolean authenticated;
            
            public SecurityContext(Set<String> roles, boolean authenticated) {
                this.roles = roles;
                this.authenticated = authenticated;
            }
            
            public boolean hasRole(String role) { return roles.contains(role); }
            public boolean isAuthenticated() { return authenticated; }
        }
        
        // Security annotation
        @Retention(RetentionPolicy.RUNTIME)
        @Target(ElementType.METHOD)
        @interface Secured {
            String[] roles() default {};
            boolean authenticated() default true;
        }
        
        // Secure UserService interface
        interface SecureUserService {
            User createUser(String username, String email);
            
            @Secured(roles = {"ADMIN"})
            User findUserById(Long id);
            
            @Secured(roles = {"USER", "ADMIN"})
            List<User> findAllUsers();
            
            @Secured(roles = {"ADMIN"})
            void deleteUser(Long id);
        }
        
        // Secure UserService implementation
        class SecureUserServiceImpl implements SecureUserService {
            @Override
            public User createUser(String username, String email) {
                System.out.println("Creating user: " + username);
                return new User(username, email);
            }
            
            @Override
            public User findUserById(Long id) {
                System.out.println("Finding user by ID: " + id);
                return new User("user" + id, "user" + id + "@example.com");
            }
            
            @Override
            public List<User> findAllUsers() {
                System.out.println("Finding all users");
                return Arrays.asList(new User("user1", "user1@example.com"));
            }
            
            @Override
            public void deleteUser(Long id) {
                System.out.println("Deleting user: " + id);
            }
        }
        
        // Security-aware invocation handler
        class SecurityInvocationHandler implements InvocationHandler {
            private final Object target;
            private final SecurityContext securityContext;
            
            public SecurityInvocationHandler(Object target, SecurityContext securityContext) {
                this.target = target;
                this.securityContext = securityContext;
            }
            
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                // Check security annotations
                Secured secured = method.getAnnotation(Secured.class);
                if (secured != null) {
                    if (secured.authenticated() && !securityContext.isAuthenticated()) {
                        throw new SecurityException("Authentication required for method: " + method.getName());
                    }
                    
                    if (secured.roles().length > 0) {
                        boolean hasAccess = false;
                        for (String role : secured.roles()) {
                            if (securityContext.hasRole(role)) {
                                hasAccess = true;
                                break;
                            }
                        }
                        if (!hasAccess) {
                            throw new SecurityException("Insufficient privileges for method: " + method.getName() +
                                                      ". Required roles: " + Arrays.toString(secured.roles()));
                        }
                    }
                }
                
                System.out.println("Security check passed for: " + method.getName());
                return method.invoke(target, args);
            }
        }
        
        try {
            SecureUserService userService = new SecureUserServiceImpl();
            
            // Test with admin user
            System.out.println("=== Testing with Admin User ===");
            SecurityContext adminContext = new SecurityContext(
                new HashSet<>(Arrays.asList("USER", "ADMIN")), true);
            
            SecureUserService adminProxy = (SecureUserService) Proxy.newProxyInstance(
                SecureUserService.class.getClassLoader(),
                new Class<?>[]{SecureUserService.class},
                new SecurityInvocationHandler(userService, adminContext)
            );
            
            User adminUser = adminProxy.createUser("admin_user", "admin@example.com");
            User foundUser = adminProxy.findUserById(1L);
            List<User> allUsers = adminProxy.findAllUsers();
            adminProxy.deleteUser(1L);
            
            // Test with regular user
            System.out.println("\n=== Testing with Regular User ===");
            SecurityContext userContext = new SecurityContext(
                new HashSet<>(Arrays.asList("USER")), true);
            
            SecureUserService userProxy = (SecureUserService) Proxy.newProxyInstance(
                SecureUserService.class.getClassLoader(),
                new Class<?>[]{SecureUserService.class},
                new SecurityInvocationHandler(userService, userContext)
            );
            
            User regularUser = userProxy.createUser("regular_user", "user@example.com");
            User regularFound = userProxy.findUserById(1L); // Should fail
        } catch (SecurityException e) {
            System.out.println("Security exception caught: " + e.getMessage());
        } catch (Exception e) {
            System.err.println("Error in security proxy: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Caching proxy
    public static void demonstrateCachingProxy() {
        System.out.println("\n=== Caching Proxy ===");
        
        // Cache annotation
        @Retention(RetentionPolicy.RUNTIME)
        @Target(ElementType.METHOD)
        @interface Cached {
            String key() default "";
            long ttl() default 3600; // 1 hour default
            boolean evictOnUpdate() default true;
        }
        
        // Cache-aware UserService
        interface CacheableUserService {
            @Cached(key = "user.create", ttl = 60)
            User createUser(String username, String email);
            
            @Cached(key = "user.findById", ttl = 300)
            User findUserById(Long id);
            
            @Cached(key = "user.findAll", ttl = 600)
            List<User> findAllUsers();
            
            void deleteUser(Long id); // Not cached
        }
        
        // Simple cache implementation
        class SimpleCache {
            private final Map<String, CacheEntry> cache = new ConcurrentHashMap<>();
            
            public Object get(String key) {
                CacheEntry entry = cache.get(key);
                if (entry != null && !entry.isExpired()) {
                    System.out.println("Cache HIT for key: " + key);
                    return entry.getValue();
                } else {
                    if (entry != null) {
                        System.out.println("Cache EXPIRED for key: " + key);
                    } else {
                        System.out.println("Cache MISS for key: " + key);
                    }
                    cache.remove(key);
                    return null;
                }
            }
            
            public void put(String key, Object value, long ttl) {
                cache.put(key, new CacheEntry(value, System.currentTimeMillis() + ttl * 1000));
                System.out.println("Cached value for key: " + key + " (TTL: " + ttl + "s)");
            }
            
            public void evict(String key) {
                cache.remove(key);
                System.out.println("Evicted cache entry for key: " + key);
            }
            
            public void clear() {
                cache.clear();
                System.out.println("Cache cleared");
            }
            
            static class CacheEntry {
                private final Object value;
                private final long expiryTime;
                
                public CacheEntry(Object value, long expiryTime) {
                    this.value = value;
                    this.expiryTime = expiryTime;
                }
                
                public Object getValue() { return value; }
                public boolean isExpired() { return System.currentTimeMillis() > expiryTime; }
            }
        }
        
        // Caching UserService implementation
        class CacheableUserServiceImpl implements CacheableUserService {
            @Override
            public User createUser(String username, String email) {
                System.out.println("Creating user in service: " + username);
                return new User(username, email);
            }
            
            @Override
            public User findUserById(Long id) {
                System.out.println("Finding user by ID in service: " + id);
                return new User("user" + id, "user" + id + "@example.com");
            }
            
            @Override
            public List<User> findAllUsers() {
                System.out.println("Finding all users in service");
                return Arrays.asList(
                    new User("user1", "user1@example.com"),
                    new User("user2", "user2@example.com")
                );
            }
            
            @Override
            public void deleteUser(Long id) {
                System.out.println("Deleting user in service: " + id);
            }
        }
        
        // Caching invocation handler
        class CachingInvocationHandler implements InvocationHandler {
            private final Object target;
            private final SimpleCache cache;
            
            public CachingInvocationHandler(Object target, SimpleCache cache) {
                this.target = target;
                this.cache = cache;
            }
            
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                Cached cached = method.getAnnotation(Cached.class);
                if (cached == null) {
                    // Not cached, invoke directly
                    System.out.println("Direct invocation: " + method.getName());
                    return method.invoke(target, args);
                }
                
                // Generate cache key
                String cacheKey = generateCacheKey(cached.key(), method, args);
                
                // Try to get from cache
                Object cachedResult = cache.get(cacheKey);
                if (cachedResult != null) {
                    return cachedResult;
                }
                
                // Not in cache, invoke method
                System.out.println("Cache miss, invoking method: " + method.getName());
                Object result = method.invoke(target, args);
                
                // Cache the result
                cache.put(cacheKey, result, cached.ttl());
                
                return result;
            }
            
            private String generateCacheKey(String baseKey, Method method, Object... args) {
                if (!baseKey.isEmpty()) {
                    StringBuilder key = new StringBuilder(baseKey);
                    for (Object arg : args) {
                        key.append(".").append(arg != null ? arg.toString() : "null");
                    }
                    return key.toString();
                }
                
                StringBuilder key = new StringBuilder();
                key.append(method.getDeclaringClass().getName())
                   .append(".").append(method.getName());
                
                for (Object arg : args) {
                    key.append(".").append(arg != null ? arg.toString() : "null");
                }
                
                return key.toString();
            }
        }
        
        try {
            CacheableUserService userService = new CacheableUserServiceImpl();
            SimpleCache cache = new SimpleCache();
            
            CacheableUserService proxy = (CacheableUserService) Proxy.newProxyInstance(
                CacheableUserService.class.getClassLoader(),
                new Class<?>[]{CacheableUserService.class},
                new CachingInvocationHandler(userService, cache)
            );
            
            System.out.println("=== Caching Proxy Operations ===");
            
            // First call - should cache
            System.out.println("\n1. First createUser call:");
            User user1 = proxy.createUser("alice", "alice@example.com");
            
            System.out.println("\n2. Second createUser call with same params:");
            User user2 = proxy.createUser("alice", "alice@example.com");
            
            System.out.println("\n3. createUser with different params:");
            User user3 = proxy.createUser("bob", "bob@example.com");
            
            System.out.println("\n4. First findUserById call:");
            User found1 = proxy.findUserById(1L);
            
            System.out.println("\n5. Second findUserById call:");
            User found2 = proxy.findUserById(1L);
            
            System.out.println("\n6. findUserById with different ID:");
            User found3 = proxy.findUserById(2L);
            
            System.out.println("\n7. Non-cached method:");
            proxy.deleteUser(1L);
            
            System.out.println("\n=== Cache Contents ===");
            // Note: In a real implementation, we might expose cache statistics
            
        } catch (Exception e) {
            System.err.println("Error in caching proxy: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Composite proxy with multiple handlers
    public static void demonstrateCompositeProxy() {
        System.out.println("\n=== Composite Proxy ===");
        
        // Multiple invocation handlers
        class LoggingHandler implements InvocationHandler {
            private final InvocationHandler next;
            
            public LoggingHandler(InvocationHandler next) {
                this.next = next;
            }
            
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                System.out.println("[LOG] Before: " + method.getName());
                try {
                    Object result = next.invoke(proxy, method, args);
                    System.out.println("[LOG] After: " + method.getName());
                    return result;
                } catch (Exception e) {
                    System.out.println("[LOG] Exception in: " + method.getName() + " - " + e.getMessage());
                    throw e;
                }
            }
        }
        
        class TimingHandler implements InvocationHandler {
            private final InvocationHandler next;
            
            public TimingHandler(InvocationHandler next) {
                this.next = next;
            }
            
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                long start = System.nanoTime();
                try {
                    Object result = next.invoke(proxy, method, args);
                    long duration = System.nanoTime() - start;
                    System.out.println("[TIME] " + method.getName() + " took " + duration / 1000 + " μs");
                    return result;
                } catch (Exception e) {
                    long duration = System.nanoTime() - start;
                    System.out.println("[TIME] " + method.getName() + " failed after " + duration / 1000 + " μs");
                    throw e;
                }
            }
        }
        
        class ExceptionHandlingHandler implements InvocationHandler {
            private final InvocationHandler next;
            
            public ExceptionHandlingHandler(InvocationHandler next) {
                this.next = next;
            }
            
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                try {
                    return next.invoke(proxy, method, args);
                } catch (InvocationTargetException e) {
                    System.out.println("[ERROR] Method " + method.getName() + " threw: " + e.getCause().getMessage());
                    // Re-throw or handle as needed
                    throw e.getCause();
                }
            }
        }
        
        try {
            UserService userService = new UserServiceImpl();
            
            // Create composite handler chain
            InvocationHandler baseHandler = (proxy, method, args) -> method.invoke(userService, args);
            InvocationHandler exceptionHandler = new ExceptionHandlingHandler(baseHandler);
            InvocationHandler timingHandler = new TimingHandler(exceptionHandler);
            InvocationHandler loggingHandler = new LoggingHandler(timingHandler);
            
            UserService proxy = (UserService) Proxy.newProxyInstance(
                UserService.class.getClassLoader(),
                new Class<?>[]{UserService.class},
                loggingHandler
            );
            
            System.out.println("=== Composite Proxy Operations ===");
            User user = proxy.createUser("composite", "composite@example.com");
            User found = proxy.findUserById(1L);
            List<User> users = proxy.findAllUsers();
            
        } catch (Exception e) {
            System.err.println("Error in composite proxy: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    public static void main(String[] args) {
        demonstrateBasicProxy();
        demonstrateAdvancedProxy();
        demonstrateSecurityProxy();
        demonstrateCachingProxy();
        demonstrateCompositeProxy();
    }
}
```

## Reflection Performance Considerations

Understanding and optimizing the performance implications of reflection usage.

### Performance Analysis and Optimization

```java
// Reflection performance analysis and optimization techniques
public class ReflectionPerformanceConsiderations {
    
    // Test class for performance comparison
    public static class PerformanceTestObject {
        private String field1 = "value1";
        private String field2 = "value2";
        private String field3 = "value3";
        private int numberField = 42;
        private double doubleField = 3.14;
        
        public String getField1() { return field1; }
        public void setField1(String field1) { this.field1 = field1; }
        
        public String getField2() { return field2; }
        public void setField2(String field2) { this.field2 = field2; }
        
        public String getField3() { return field3; }
        public void setField3(String field3) { this.field3 = field3; }
        
        public int getNumberField() { return numberField; }
        public void setNumberField(int numberField) { this.numberField = numberField; }
        
        public double getDoubleField() { return doubleField; }
        public void setDoubleField(double doubleField) { this.doubleField = doubleField; }
        
        public String complexOperation(String input, int multiplier) {
            return input + "_" + (numberField * multiplier);
        }
        
        public void voidOperation(String message) {
            // Do nothing, just for testing void methods
        }
    }
    
    public static void demonstrateFieldAccessPerformance() {
        System.out.println("=== Field Access Performance Analysis ===");
        
        try {
            Class<?> clazz = PerformanceTestObject.class;
            PerformanceTestObject instance = new PerformanceTestObject();
            int iterations = 1_000_000;
            
            // Direct field access (baseline)
            long startTime = System.nanoTime();
            for (int i = 0; i < iterations; i++) {
                String value = instance.field1;
                instance.field1 = value + "x";
            }
            long directTime = System.nanoTime() - startTime;
            
            // Reflection field access without caching
            startTime = System.nanoTime();
            for (int i = 0; i < iterations; i++) {
                Field field = clazz.getDeclaredField("field1");
                field.setAccessible(true);
                String value = (String) field.get(instance);
                field.set(instance, value + "x");
            }
            long uncachedReflectionTime = System.nanoTime() - startTime;
            
            // Reflection field access with caching
            Field cachedField = clazz.getDeclaredField("field1");
            cachedField.setAccessible(true);
            startTime = System.nanoTime();
            for (int i = 0; i < iterations; i++) {
                String value = (String) cachedField.get(instance);
                cachedField.set(instance, value + "x");
            }
            long cachedReflectionTime = System.nanoTime() - startTime;
            
            System.out.println("Iterations: " + iterations);
            System.out.println("Direct access time: " + directTime / 1_000_000 + " ms");
            System.out.println("Uncached reflection time: " + uncachedReflectionTime / 1_000_000 + " ms");
            System.out.println("Cached reflection time: " + cachedReflectionTime / 1_000_000 + " ms");
            
            System.out.println("Performance ratios:");
            System.out.println("  Uncached vs Direct: " + (double) uncachedReflectionTime / directTime + "x slower");
            System.out.println("  Cached vs Direct: " + (double) cachedReflectionTime / directTime + "x slower");
            System.out.println("  Cached vs Uncached: " + (double) cachedReflectionTime / uncachedReflectionTime + "x faster");
            
        } catch (Exception e) {
            System.err.println("Error in field access performance test: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    public static void demonstrateMethodInvocationPerformance() {
        System.out.println("\n=== Method Invocation Performance Analysis ===");
        
        try {
            Class<?> clazz = PerformanceTestObject.class;
            PerformanceTestObject instance = new PerformanceTestObject();
            int iterations = 1_000_000;
            
            // Direct method invocation (baseline)
            long startTime = System.nanoTime();
            for (int i = 0; i < iterations; i++) {
                String result = instance.getField1();
                instance.setField1(result + "x");
            }
            long directTime = System.nanoTime() - startTime;
            
            // Reflection method invocation without caching
            startTime = System.nanoTime();
            for (int i = 0; i < iterations; i++) {
                Method getter = clazz.getMethod("getField1");
                Method setter = clazz.getMethod("setField1", String.class);
                String result = (String) getter.invoke(instance);
                setter.invoke(instance, result + "x");
            }
            long uncachedMethodTime = System.nanoTime() - startTime;
            
            // Reflection method invocation with caching
            Method cachedGetter = clazz.getMethod("getField1");
            Method cachedSetter = clazz.getMethod("setField1", String.class);
            startTime = System.nanoTime();
            for (int i = 0; i < iterations; i++) {
                String result = (String) cachedGetter.invoke(instance);
                cachedSetter.invoke(instance, result + "x");
            }
            long cachedMethodTime = System.nanoTime() - startTime;
            
            System.out.println("Iterations: " + iterations);
            System.out.println("Direct method time: " + directTime / 1_000_000 + " ms");
            System.out.println("Uncached method time: " + uncachedMethodTime / 1_000_000 + " ms");
            System.out.println("Cached method time: " + cachedMethodTime / 1_000_000 + " ms");
            
            System.out.println("Performance ratios:");
            System.out.println("  Uncached vs Direct: " + (double) uncachedMethodTime / directTime + "x slower");
            System.out.println("  Cached vs Direct: " + (double) cachedMethodTime / directTime + "x slower");
            System.out.println("  Cached vs Uncached: " + (double) cachedMethodTime / uncachedMethodTime + "x faster");
            
        } catch (Exception e) {
            System.err.println("Error in method invocation performance test: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Performance-optimized reflection utilities
    public static void demonstrateOptimizedReflection() {
        System.out.println("\n=== Optimized Reflection Utilities ===");
        
        // Field cache for avoiding repeated lookups
        class FieldCache {
            private final Map<String, Field> cache = new ConcurrentHashMap<>();
            
            public Field getField(Class<?> clazz, String fieldName) throws NoSuchFieldException {
                String key = clazz.getName() + "." + fieldName;
                return cache.computeIfAbsent(key, k -> {
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
        
        // Method cache for avoiding repeated lookups
        class MethodCache {
            private final Map<String, Method> cache = new ConcurrentHashMap<>();
            
            public Method getMethod(Class<?> clazz, String methodName, Class<?>... paramTypes) 
                    throws NoSuchMethodException {
                String key = clazz.getName() + "." + methodName + Arrays.toString(paramTypes);
                return cache.computeIfAbsent(key, k -> {
                    try {
                        Method method = clazz.getMethod(methodName, paramTypes);
                        method.setAccessible(true);
                        return method;
                    } catch (NoSuchMethodException e) {
                        throw new RuntimeException(e);
                    }
                });
            }
        }
        
        // Fast field accessor using MethodHandles (Java 7+)
        class FastFieldAccessor {
            private final Map<String, VarHandle> handleCache = new ConcurrentHashMap<>();
            
            public Object getFieldValue(Object target, String fieldName) throws Throwable {
                Class<?> clazz = target.getClass();
                String key = clazz.getName() + "." + fieldName;
                VarHandle handle = handleCache.computeIfAbsent(key, k -> {
                    try {
                        return MethodHandles.privateLookupIn(clazz, MethodHandles.lookup())
                                .findVarHandle(clazz, fieldName, Object.class);
                    } catch (Exception e) {
                        throw new RuntimeException(e);
                    }
                });
                
                return handle.get(target);
            }
            
            public void setFieldValue(Object target, String fieldName, Object value) throws Throwable {
                Class<?> clazz = target.getClass();
                String key = clazz.getName() + "." + fieldName;
                VarHandle handle = handleCache.computeIfAbsent(key, k -> {
                    try {
                        return MethodHandles.privateLookupIn(clazz, MethodHandles.lookup())
                                .findVarHandle(clazz, fieldName, Object.class);
                    } catch (Exception e) {
                        throw new RuntimeException(e);
                    }
                });
                
                handle.set(target, value);
            }
        }
        
        try {
            PerformanceTestObject instance = new PerformanceTestObject();
            FieldCache fieldCache = new FieldCache();
            MethodCache methodCache = new MethodCache();
            FastFieldAccessor fastAccessor = new FastFieldAccessor();
            int iterations = 100_000;
            
            System.out.println("Testing optimized utilities with " + iterations + " iterations:");
            
            // Test field cache
            long startTime = System.nanoTime();
            for (int i = 0; i < iterations; i++) {
                Field field = fieldCache.getField(PerformanceTestObject.class, "field1");
                String value = (String) field.get(instance);
                field.set(instance, value + "x");
            }
            long fieldCacheTime = System.nanoTime() - startTime;
            
            // Test method cache
            startTime = System.nanoTime();
            for (int i = 0; i < iterations; i++) {
                Method getter = methodCache.getMethod(PerformanceTestObject.class, "getField1");
                Method setter = methodCache.getMethod(PerformanceTestObject.class, "setField1", String.class);
                String result = (String) getter.invoke(instance);
                setter.invoke(instance, result + "x");
            }
            long methodCacheTime = System.nanoTime() - startTime;
            
            // Test fast field accessor (if available)
            if (System.getProperty("java.version").startsWith("1.")) {
                System.out.println("Fast field accessor test skipped (requires Java 9+)");
            } else {
                startTime = System.nanoTime();
                for (int i = 0; i < iterations; i++) {
                    Object value = fastAccessor.getFieldValue(instance, "field1");
                    fastAccessor.setFieldValue(instance, "field1", value.toString() + "x");
                }
                long fastAccessorTime = System.nanoTime() - startTime;
                System.out.println("Fast field accessor time: " + fastAccessorTime / 1_000_000 + " ms");
            }
            
            System.out.println("Field cache time: " + fieldCacheTime / 1_000_000 + " ms");
            System.out.println("Method cache time: " + methodCacheTime / 1_000_000 + " ms");
            
        } catch (Exception e) {
            System.err.println("Error in optimized reflection test: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // Memory usage analysis
    public static void demonstrateMemoryUsageAnalysis() {
        System.out.println("\n=== Memory Usage Analysis ===");
        
        // Monitor memory usage during reflection operations
        class MemoryMonitor {
            public long getUsedMemory() {
                Runtime runtime = Runtime.getRuntime();
                return runtime.totalMemory() - runtime.freeMemory();
            }
            
            public void printMemoryUsage(String operation) {
                System.gc(); // Suggest garbage collection
                try {
                    Thread.sleep(100); // Give GC time to work
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                
                long usedMemory = getUsedMemory();
                System.out.println(operation + " - Memory usage: " + usedMemory / (1024 * 1024) + " MB");
            }
        }
        
        try {
            MemoryMonitor monitor = new MemoryMonitor();
            int iterations = 100_000;
            
            System.out.println("Memory usage analysis with " + iterations + " operations:");
            
            // Baseline memory usage
            monitor.printMemoryUsage("Baseline");
            
            // Create many Method objects without caching
            monitor.printMemoryUsage("Before uncached methods");
            List<Method> uncachedMethods = new ArrayList<>();
            for (int i = 0; i < iterations; i++) {
                Method method = PerformanceTestObject.class.getMethod("getField1");
                uncachedMethods.add(method);
            }
            monitor.printMemoryUsage("After uncached methods");
            
            // Clear and test cached approach
            uncachedMethods.clear();
            System.gc();
            Thread.sleep(100);
            monitor.printMemoryUsage("After clearing uncached methods");
            
            // Create cached Method objects
            monitor.printMemoryUsage("Before cached methods");
            Map<String, Method> methodCache = new HashMap<>();
            for (int i = 0; i < iterations; i++) {
                String key = "getField1";
                Method method = methodCache.computeIfAbsent(key, k -> {
                    try {
                        return PerformanceTestObject.class.getMethod(k);
                    } catch (NoSuchMethodException e) {
                        throw new RuntimeException(e);
                    }
                });
                // Use the method to prevent it from being optimized away
                if (i == 0) {
                    method.setAccessible(true);
                }
            }
            monitor.printMemoryUsage("After cached methods");
            
            // Test Field objects
            monitor.printMemoryUsage("Before fields");
            List<Field> fields = new ArrayList<>();
            for (int i = 0; i < iterations; i++) {
                Field field = PerformanceTestObject.class.getDeclaredField("field1");
                fields.add(field);
            }
            monitor.printMemoryUsage("After fields");
            
        } catch (Exception e) {
            System.err.println("Error in memory usage analysis: " + e.getMessage());
            e.printStackTrace();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    // Performance benchmark with different scenarios
    public static void demonstrateRealWorldPerformance() {
        System.out.println("\n=== Real-World Performance Scenarios ===");
        
        // Simulate a real-world scenario: copying properties between objects
        class Person {
            private String name;
            private int age;
            private String email;
            private double salary;
            
            public Person() {}
            
            public Person(String name, int age, String email, double salary) {
                this.name = name;
                this.age = age;
                this.email = email;
                this.salary = salary;
            }
            
            // Getters and setters
            public String getName() { return name; }
            public void setName(String name) { this.name = name; }
            
            public int getAge() { return age; }
            public void setAge(int age) { this.age = age; }
            
            public String getEmail() { return email; }
            public void setEmail(String email) { this.email = email; }
            
            public double getSalary() { return salary; }
            public void setSalary(double salary) { this.salary = salary; }
        }
        
        // Property copier using different approaches
        class PropertyCopier {
            
            // Direct copying (baseline)
            public void copyDirect(Person source, Person target) {
                target.setName(source.getName());
                target.setAge(source.getAge());
                target.setEmail(source.getEmail());
                target.setSalary(source.getSalary());
            }
            
            // Reflection-based copying without caching
            public void copyReflectionUncached(Person source, Person target) throws Exception {
                Class<?> clazz = Person.class;
                Field[] fields = clazz.getDeclaredFields();
                
                for (Field field : fields) {
                    field.setAccessible(true);
                    Object value = field.get(source);
                    field.set(target, value);
                }
            }
            
            // Reflection-based copying with caching
            private final Map<String, Field> fieldCache = new ConcurrentHashMap<>();
            
            public void copyReflectionCached(Person source, Person target) throws Exception {
                Class<?> clazz = Person.class;
                Field[] fields = clazz.getDeclaredFields();
                
                for (Field field : fields) {
                    String key = clazz.getName() + "." + field.getName();
                    Field cachedField = fieldCache.computeIfAbsent(key, k -> {
                        field.setAccessible(true);
                        return field;
                    });
                    
                    Object value = cachedField.get(source);
                    cachedField.set(target, value);
                }
            }
            
            // Method-based copying
            public void copyMethods(Person source, Person target) throws Exception {
                Class<?> clazz = Person.class;
                
                // Copy name
                Method getName = clazz.getMethod("getName");
                Method setName = clazz.getMethod("setName", String.class);
                setName.invoke(target, getName.invoke(source));
                
                // Copy age
                Method getAge = clazz.getMethod("getAge");
                Method setAge = clazz.getMethod("setAge", int.class);
                setAge.invoke(target, getAge.invoke(source));
                
                // Copy email
                Method getEmail = clazz.getMethod("getEmail");
                Method setEmail = clazz.getMethod("setEmail", String.class);
                setEmail.invoke(target, getEmail.invoke(source));
                
                // Copy salary
                Method getSalary = clazz.getMethod("getSalary");
                Method setSalary = clazz.getMethod("setSalary", double.class);
                setSalary.invoke(target, getSalary.invoke(source));
            }
        }
        
        try {
            PropertyCopier copier = new PropertyCopier();
            Person source = new Person("John Doe", 30, "john@example.com", 75000.0);
            int iterations = 100_000;
            
            System.out.println("Copying " + iterations + " Person objects:");
            
            // Direct copying
            long startTime = System.nanoTime();
            for (int i = 0; i < iterations; i++) {
                Person target = new Person();
                copier.copyDirect(source, target);
            }
            long directTime = System.nanoTime() - startTime;
            
            // Uncached reflection
            startTime = System.nanoTime();
            for (int i = 0; i < iterations; i++) {
                Person target = new Person();
                copier.copyReflectionUncached(source, target);
            }
            long uncachedTime = System.nanoTime() - startTime;
            
            // Cached reflection
            startTime = System.nanoTime();
            for (int i = 0; i < iterations; i++) {
                Person target = new Person();
                copier.copyReflectionCached(source, target);
            }
            long cachedTime = System.nanoTime() - startTime;
            
            // Method-based copying
            startTime = System.nanoTime();
            for (int i = 0; i < iterations; i++) {
                Person target = new Person();
                copier.copyMethods(source, target);
            }
            long methodTime = System.nanoTime() - startTime;
            
            System.out.println("Direct copying time: " + directTime / 1_000_000 + " ms");
            System.out.println("Uncached reflection time: " + uncachedTime / 1_000_000 + " ms");
            System.out.println("Cached reflection time: " + cachedTime / 1_000_000 + " ms");
            System.out.println("Method-based time: " + methodTime / 1_000_000 + " ms");
            
            System.out.println("Performance ratios compared to direct:");
            System.out.println("  Uncached reflection: " + (double) uncachedTime / directTime + "x slower");
            System.out.println("  Cached reflection: " + (double) cachedTime / directTime + "x slower");
            System.out.println("  Method-based: " + (double) methodTime / directTime + "x slower");
            
        } catch (Exception e) {
            System.err.println("Error in real-world performance test: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // JVM warm-up and accurate benchmarking
    public static void demonstrateAccurateBenchmarking() {
        System.out.println("\n=== Accurate Benchmarking ===");
        
        try {
            PerformanceTestObject instance = new PerformanceTestObject();
            Class<?> clazz = PerformanceTestObject.class;
            int warmupIterations = 50_000;
            int benchmarkIterations = 100_000;
            
            System.out.println("Performing accurate benchmark with warmup...");
            
            // Warm up the JVM
            System.out.println("Warming up JVM...");
            Method method = clazz.getMethod("getField1");
            Field field = clazz.getDeclaredField("field1");
            field.setAccessible(true);
            
            // Warm up reflection
            for (int i = 0; i < warmupIterations; i++) {
                method.invoke(instance);
                field.get(instance);
            }
            
            // Warm up direct access
            for (int i = 0; i < warmupIterations; i++) {
                instance.getField1();
                String value = instance.field1;
            }
            
            System.out.println("Warmup completed. Starting benchmark...");
            
            // Benchmark direct access
            long startTime = System.nanoTime();
            for (int i = 0; i < benchmarkIterations; i++) {
                String value = instance.getField1();
                instance.setField1(value + "x");
            }
            long directTime = System.nanoTime() - startTime;
            
            // Benchmark reflection access
            startTime = System.nanoTime();
            for (int i = 0; i < benchmarkIterations; i++) {
                String value = (String) method.invoke(instance);
                method.invoke(instance, value + "x");
            }
            long reflectionTime = System.nanoTime() - startTime;
            
            System.out.println("Benchmark results (" + benchmarkIterations + " iterations):");
            System.out.println("Direct access: " + directTime / 1_000_000 + " ms");
            System.out.println("Reflection access: " + reflectionTime / 1_000_000 + " ms");
            System.out.println("Reflection overhead: " + (double) reflectionTime / directTime + "x");
            
            // Multiple runs for statistical accuracy
            System.out.println("\nMultiple benchmark runs for statistical analysis:");
            long[] directTimes = new long[5];
            long[] reflectionTimes = new long[5];
            
            for (int run = 0; run < 5; run++) {
                // Direct access
                startTime = System.nanoTime();
                for (int i = 0; i < benchmarkIterations; i++) {
                    String value = instance.getField1();
                    instance.setField1(value + "x");
                }
                directTimes[run] = System.nanoTime() - startTime;
                
                // Reflection access
                startTime = System.nanoTime();
                for (int i = 0; i < benchmarkIterations; i++) {
                    String value = (String) method.invoke(instance);
                    method.invoke(instance, value + "x");
                }
                reflectionTimes[run] = System.nanoTime() - startTime;
            }
            
            double avgDirect = Arrays.stream(directTimes).average().orElse(0) / 1_000_000;
            double avgReflection = Arrays.stream(reflectionTimes).average().orElse(0) / 1_000_000;
            
            System.out.println("Average direct access time: " + String.format("%.2f", avgDirect) + " ms");
            System.out.println("Average reflection time: " + String.format("%.2f", avgReflection) + " ms");
            System.out.println("Average overhead: " + String.format("%.2f", avgReflection / avgDirect) + "x");
            
        } catch (Exception e) {
            System.err.println("Error in accurate benchmarking: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    public static void main(String[] args) {
        demonstrateFieldAccessPerformance();
        demonstrateMethodInvocationPerformance();
        demonstrateOptimizedReflection();
        demonstrateMemoryUsageAnalysis();
        demonstrateRealWorldPerformance();
        demonstrateAccurateBenchmarking();
    }
}
```

## Security and Best Practices

Important security considerations and best practices when using reflection.

### Security Guidelines and Best Practices

```java
// Security considerations and best practices for reflection
public class ReflectionSecurityBestPractices {
    
    // Secure class with sensitive data
    public static class SecureClass {
        private String sensitiveData = "SECRET_INFORMATION";
        private String password = "SUPER_SECRET_PASSWORD";
        private int internalCounter = 0;
        
        public String getPublicInfo() {
            return "This is public information";
        }
        
        private void sensitiveOperation() {
            System.out.println("Performing sensitive operation with: " + sensitiveData);
        }
        
        public void publicMethod() {
            System.out.println("Public method called");
        }
    }
    
    // Security manager demonstration
    public static void demonstrateSecurityManager() {
        System.out.println("=== Security Manager Considerations ===");
        
        try {
            // Check if security manager is active
            SecurityManager sm = System.getSecurityManager();
            if (sm != null) {
                System.out.println("Security manager is active");
            } else {
                System.out.println("No security manager active");
            }
            
            // Test reflection access with security manager
            Class<?> clazz = SecureClass.class;
            Object instance = clazz.getDeclaredConstructor().newInstance();
            
            // Attempt to access private field
            Field sensitiveField = clazz.getDeclaredField("sensitiveData");
            sensitiveField.setAccessible(true);
            
            String sensitiveValue = (String) sensitiveField.get(instance);
            System.out.println("Accessed sensitive data: " + sensitiveValue);
            
        } catch (SecurityException e) {
            System.out.println("Security exception caught: " + e.getMessage());
            System.out.println("Security manager prevented access to sensitive data");
        } catch (Exception e) {
            System.err.println("Error in security manager test: " + e.getMessage());
        }
    }
    
    // Access control and permission checking
    public static void demonstrateAccessControl() {
        System.out.println("\n=== Access Control and Permissions ===");
        
        class AccessControlChecker {
            public boolean canAccessField(Field field) {
                int modifiers = field.getModifiers();
                
                // Check if field is public
                if (java.lang.reflect.Modifier.isPublic(modifiers)) {
                    return true;
                }
                
                // Check if we're in the same package
                String fieldPackage = field.getDeclaringClass().getPackage().getName();
                String callerPackage = this.getClass().getPackage().getName();
                
                if (fieldPackage.equals(callerPackage)) {
                    // Package-private access
                    return !java.lang.reflect.Modifier.isPrivate(modifiers);
                }
                
                // Protected or private access - requires setAccessible(true)
                return false;
            }
            
            public boolean canAccessMethod(Method method) {
                int modifiers = method.getModifiers();
                
                // Check if method is public
                if (java.lang.reflect.Modifier.isPublic(modifiers)) {
                    return true;
                }
                
                // Similar package and access level checks
                String methodPackage = method.getDeclaringClass().getPackage().getName();
                String callerPackage = this.getClass().getPackage().getName();
                
                if (methodPackage.equals(callerPackage)) {
                    return !java.lang.reflect.Modifier.isPrivate(modifiers);
                }
                
                return false;
            }
        }
        
        try {
            Class<?> clazz = SecureClass.class;
            AccessControlChecker checker = new AccessControlChecker();
            
            System.out.println("Access control analysis for SecureClass:");
            
            // Check fields
            Field[] fields = clazz.getDeclaredFields();
            for (Field field : fields) {
                boolean canAccess = checker.canAccessField(field);
                System.out.println("  Field " + field.getName() + 
                                 " (modifiers: " + java.lang.reflect.Modifier.toString(field.getModifiers()) + ")" +
                                 " - Can access: " + canAccess);
            }
            
            // Check methods
            Method[] methods = clazz.getDeclaredMethods();
            for (Method method : methods) {
                boolean canAccess = checker.canAccessMethod(method);
                System.out.println("  Method " + method.getName() + 
                                 " (modifiers: " + java.lang.reflect.Modifier.toString(method.getModifiers()) + ")" +
                                 " - Can access: " + canAccess);
            }
            
        } catch (Exception e) {
            System.err.println("Error in access control analysis: " + e.getMessage());
        }
    }
    
    // Secure reflection wrapper
    public static void demonstrateSecureReflectionWrapper() {
        System.out.println("\n=== Secure Reflection Wrapper ===");
        
        // Secure reflection utility that enforces access controls
        class SecureReflectionUtil {
            private final Set<String> allowedPackages;
            private final Set<String> sensitiveFieldNames;
            private final Set<String> sensitiveMethodNames;
            
            public SecureReflectionUtil() {
                // Define allowed packages
                allowedPackages = new HashSet<>(Arrays.asList(
                    "java.lang", "java.util", "java.io"
                ));
                
                // Define sensitive field and method names
                sensitiveFieldNames = new HashSet<>(Arrays.asList(
                    "password", "secret", "key", "token", "credential"
                ));
                
                sensitiveMethodNames = new HashSet<>(Arrays.asList(
                    "delete", "shutdown", "destroy", "sensitiveOperation"
                ));
            }
            
            public Object getFieldValue(Object target, String fieldName) throws Exception {
                // Security checks
                if (!isAccessAllowed(target.getClass(), fieldName)) {
                    throw new SecurityException("Access denied to field: " + fieldName);
                }
                
                Field field = target.getClass().getDeclaredField(fieldName);
                field.setAccessible(true);
                return field.get(target);
            }
            
            public void setFieldValue(Object target, String fieldName, Object value) throws Exception {
                // Security checks
                if (!isAccessAllowed(target.getClass(), fieldName)) {
                    throw new SecurityException("Access denied to field: " + fieldName);
                }
                
                Field field = target.getClass().getDeclaredField(fieldName);
                field.setAccessible(true);
                field.set(target, value);
            }
            
            public Object invokeMethod(Object target, String methodName, Object... args) throws Exception {
                // Security checks
                if (!isMethodAccessAllowed(target.getClass(), methodName)) {
                    throw new SecurityException("Access denied to method: " + methodName);
                }
                
                Class<?>[] paramTypes = getParameterTypes(args);
                Method method = target.getClass().getDeclaredMethod(methodName, paramTypes);
                method.setAccessible(true);
                return method.invoke(target, args);
            }
            
            private boolean isAccessAllowed(Class<?> clazz, String fieldName) {
                // Check package access
                String packageName = clazz.getPackage().getName();
                if (!allowedPackages.contains(packageName) && !packageName.startsWith("ReflectionSecurityBestPractices")) {
                    System.out.println("Package access denied: " + packageName);
                    return false;
                }
                
                // Check for sensitive field names
                String lowerFieldName = fieldName.toLowerCase();
                for (String sensitive : sensitiveFieldNames) {
                    if (lowerFieldName.contains(sensitive)) {
                        System.out.println("Sensitive field access denied: " + fieldName);
                        return false;
                    }
                }
                
                return true;
            }
            
            private boolean isMethodAccessAllowed(Class<?> clazz, String methodName) {
                // Check package access
                String packageName = clazz.getPackage().getName();
                if (!allowedPackages.contains(packageName) && !packageName.startsWith("ReflectionSecurityBestPractices")) {
                    System.out.println("Package access denied: " + packageName);
                    return false;
                }
                
                // Check for sensitive method names
                String lowerMethodName = methodName.toLowerCase();
                for (String sensitive : sensitiveMethodNames) {
                    if (lowerMethodName.contains(sensitive)) {
                        System.out.println("Sensitive method access denied: " + methodName);
                        return false;
                    }
                }
                
                return true;
            }
            
            private Class<?>[] getParameterTypes(Object... args) {
                Class<?>[] types = new Class<?>[args.length];
                for (int i = 0; i < args.length; i++) {
                    types[i] = args[i] != null ? args[i].getClass() : Object.class;
                }
                return types;
            }
        }
        
        try {
            SecureClass secureInstance = new SecureClass();
            SecureReflectionUtil secureUtil = new SecureReflectionUtil();
            
            System.out.println("Testing secure reflection wrapper:");
            
            // Test allowed access
            try {
                Object publicInfo = secureUtil.getFieldValue(secureInstance, "internalCounter");
                System.out.println("Allowed field access - internalCounter: " + publicInfo);
                
                secureUtil.invokeMethod(secureInstance, "publicMethod");
                System.out.println("Allowed method access - publicMethod called");
                
            } catch (SecurityException e) {
                System.out.println("Security exception (expected): " + e.getMessage());
            }
            
            // Test denied access
            try {
                Object sensitiveData = secureUtil.getFieldValue(secureInstance, "sensitiveData");
                System.out.println("This should not be printed: " + sensitiveData);
            } catch (SecurityException e) {
                System.out.println("Security exception (expected): " + e.getMessage());
            }
            
            try {
                Object password = secureUtil.getFieldValue(secureInstance, "password");
                System.out.println("This should not be printed: " + password);
            } catch (SecurityException e) {
                System.out.println("Security exception (expected): " + e.getMessage());
            }
            
            try {
                secureUtil.invokeMethod(secureInstance, "sensitiveOperation");
                System.out.println("This should not be printed");
            } catch (SecurityException e) {
                System.out.println("Security exception (expected): " + e.getMessage());
            }
            
        } catch (Exception e) {
            System.err.println("Error in secure reflection wrapper: " + e.getMessage());
        }
    }
    
    // Input validation for reflection operations
    public static void demonstrateInputValidation() {
        System.out.println("\n=== Input Validation for Reflection ===");
        
        class ReflectionInputValidator {
            private final Set<String> blacklistedClasses = new HashSet<>(Arrays.asList(
                "java.lang.Runtime", "java.lang.ProcessBuilder", "java.lang.System"
            ));
            
            private final Set<String> blacklistedMethods = new HashSet<>(Arrays.asList(
                "exit", "halt", "exec", "start", "destroy"
            ));
            
            private final Set<String> blacklistedFields = new HashSet<>(Arrays.asList(
                "out", "err", "in", "security"
            ));
            
            public Class<?> validateAndLoadClass(String className) throws ClassNotFoundException {
                // Validate class name
                if (className == null || className.trim().isEmpty()) {
                    throw new IllegalArgumentException("Class name cannot be null or empty");
                }
                
                // Check for blacklisted classes
                if (blacklistedClasses.contains(className)) {
                    throw new SecurityException("Access to class " + className + " is denied");
                }
                
                // Prevent loading of classes from certain packages
                if (className.startsWith("java.lang.reflect") || 
                    className.startsWith("sun.") ||
                    className.startsWith("com.sun.")) {
                    throw new SecurityException("Access to system reflection classes is denied");
                }
                
                return Class.forName(className);
            }
            
            public void validateMethodName(String methodName) {
                if (methodName == null || methodName.trim().isEmpty()) {
                    throw new IllegalArgumentException("Method name cannot be null or empty");
                }
                
                if (methodName.contains("$") || methodName.contains("<")) {
                    throw new SecurityException("Invalid method name: " + methodName);
                }
                
                String lowerMethodName = methodName.toLowerCase();
                for (String blacklisted : blacklistedMethods) {
                    if (lowerMethodName.contains(blacklisted)) {
                        throw new SecurityException("Access to method " + methodName + " is denied");
                    }
                }
            }
            
            public void validateFieldName(String fieldName) {
                if (fieldName == null || fieldName.trim().isEmpty()) {
                    throw new IllegalArgumentException("Field name cannot be null or empty");
                }
                
                if (fieldName.contains("$") || fieldName.contains("<")) {
                    throw new SecurityException("Invalid field name: " + fieldName);
                }
                
                String lowerFieldName = fieldName.toLowerCase();
                for (String blacklisted : blacklistedFields) {
                    if (lowerFieldName.equals(blacklisted)) {
                        throw new SecurityException("Access to field " + fieldName + " is denied");
                    }
                }
            }
            
            public Object safeInvoke(Object target, String methodName, Object... args) throws Exception {
                validateMethodName(methodName);
                
                Class<?>[] paramTypes = new Class<?>[args.length];
                for (int i = 0; i < args.length; i++) {
                    paramTypes[i] = args[i] != null ? args[i].getClass() : Object.class;
                    // Prevent passing of sensitive objects
                    validateParameter(args[i]);
                }
                
                Method method = target.getClass().getMethod(methodName, paramTypes);
                return method.invoke(target, args);
            }
            
            private void validateParameter(Object param) {
                if (param != null) {
                    String paramClass = param.getClass().getName();
                    if (param instanceof Class || paramClass.contains("ClassLoader")) {
                        throw new SecurityException("Passing Class or ClassLoader objects is not allowed");
                    }
                }
            }
        }
        
        try {
            ReflectionInputValidator validator = new ReflectionInputValidator();
            
            System.out.println("Testing input validation:");
            
            // Valid class loading
            try {
                Class<?> validClass = validator.validateAndLoadClass("java.lang.String");
                System.out.println("Valid class loaded: " + validClass.getName());
            } catch (Exception e) {
                System.out.println("Validation error: " + e.getMessage());
            }
            
            // Invalid class loading
            try {
                Class<?> invalidClass = validator.validateAndLoadClass("java.lang.Runtime");
                System.out.println("This should not be printed");
            } catch (SecurityException e) {
                System.out.println("Security exception (expected): " + e.getMessage());
            }
            
            // Valid method name
            try {
                validator.validateMethodName("toString");
                System.out.println("Valid method name accepted");
            } catch (Exception e) {
                System.out.println("Validation error: " + e.getMessage());
            }
            
            // Invalid method name
            try {
                validator.validateMethodName("exit");
                System.out.println("This should not be printed");
            } catch (SecurityException e) {
                System.out.println("Security exception (expected): " + e.getMessage());
            }
            
            // Safe method invocation
            String testString = "Hello, World!";
            try {
                Object result = validator.safeInvoke(testString, "substring", 0, 5);
                System.out.println("Safe invocation result: " + result);
            } catch (Exception e) {
                System.out.println("Invocation error: " + e.getMessage());
            }
            
        } catch (Exception e) {
            System.err.println("Error in input validation: " + e.getMessage());
        }
    }
    
    // Exception handling and error reporting
    public static void demonstrateExceptionHandling() {
        System.out.println("\n=== Exception Handling and Error Reporting ===");
        
        class SecureReflectionHandler {
            public Object handleReflectionOperation(ReflectionOperation operation) {
                try {
                    return operation.execute();
                } catch (InvocationTargetException e) {
                    // Unwrap the actual exception
                    Throwable cause = e.getCause();
                    System.err.println("Reflection operation failed: " + cause.getMessage());
                    // Log the full stack trace for debugging
                    cause.printStackTrace();
                    // Re-throw or handle appropriately
                    throw new RuntimeException("Operation failed: " + cause.getMessage(), cause);
                } catch (IllegalAccessException e) {
                    System.err.println("Access denied: " + e.getMessage());
                    // This might indicate a security issue
                    throw new SecurityException("Insufficient privileges for reflection operation", e);
                } catch (NoSuchMethodException e) {
                    System.err.println("Method not found: " + e.getMessage());
                    throw new IllegalArgumentException("Requested method does not exist", e);
                } catch (NoSuchFieldException e) {
                    System.err.println("Field not found: " + e.getMessage());
                    throw new IllegalArgumentException("Requested field does not exist", e);
                } catch (Exception e) {
                    System.err.println("Unexpected error in reflection operation: " + e.getMessage());
                    e.printStackTrace();
                    throw new RuntimeException("Reflection operation failed", e);
                }
            }
            
            @FunctionalInterface
            interface ReflectionOperation {
                Object execute() throws Exception;
            }
        }
        
        try {
            SecureClass instance = new SecureClass();
            SecureReflectionHandler handler = new SecureReflectionHandler();
            
            System.out.println("Testing exception handling:");
            
            // Successful operation
            Object result = handler.handleReflectionOperation(() -> {
                Method method = instance.getClass().getMethod("getPublicInfo");
                return method.invoke(instance);
            });
            System.out.println("Successful operation result: " + result);
            
            // Operation that throws exception
            try {
                handler.handleReflectionOperation(() -> {
                    Method method = instance.getClass().getMethod("nonExistentMethod");
                    return method.invoke(instance);
                });
            } catch (Exception e) {
                System.out.println("Handled exception: " + e.getMessage());
            }
            
            // Access denied operation
            try {
                handler.handleReflectionOperation(() -> {
                    Field field = instance.getClass().getDeclaredField("sensitiveData");
                    // Don't setAccessible(true) - this will cause IllegalAccessException
                    return field.get(instance);
                });
            } catch (SecurityException e) {
                System.out.println("Security exception handled: " + e.getMessage());
            }
            
        } catch (Exception e) {
            System.err.println("Error in exception handling test: " + e.getMessage());
        }
    }
    
    // Best practices summary and recommendations
    public static void demonstrateBestPractices() {
        System.out.println("\n=== Reflection Best Practices Summary ===");
        
        System.out.println("1. CACHING:");
        System.out.println("   - Cache Field, Method, and Constructor objects");
        System.out.println("   - Use ConcurrentHashMap for thread-safe caching");
        System.out.println("   - Example: private static final Map<String, Method> methodCache = new ConcurrentHashMap<>()");
        
        System.out.println("\n2. SECURITY:");
        System.out.println("   - Validate all input parameters");
        System.out.println("   - Implement access control checks");
        System.out.println("   - Use SecurityManager when available");
        System.out.println("   - Avoid accessing sensitive fields/methods");
        
        System.out.println("\n3. PERFORMANCE:");
        System.out.println("   - Prefer direct access over reflection when possible");
        System.out.println("   - Use MethodHandles for better performance (Java 7+)");
        System.out.println("   - Warm up the JVM before performance-critical reflection");
        System.out.println("   - Consider code generation alternatives for hot paths");
        
        System.out.println("\n4. ERROR HANDLING:");
        System.out.println("   - Handle InvocationTargetException properly");
        System.out.println("   - Distinguish between different reflection exceptions");
        System.out.println("   - Provide meaningful error messages");
        System.out.println("   - Log security-related exceptions appropriately");
        
        System.out.println("\n5. CODE QUALITY:");
        System.out.println("   - Use generics to avoid casting");
        System.out.println("   - Prefer method references over reflection when possible");
        System.out.println("   - Document reflection usage clearly");
        System.out.println("   - Test reflection code thoroughly");
        
        System.out.println("\n6. ALTERNATIVES TO CONSIDER:");
        System.out.println("   - Code generation (ASM, Javassist)");
        System.out.println("   - Dynamic proxies for interface-based operations");
        System.out.println("   - MethodHandles for high-performance scenarios");
        System.out.println("   - Annotation processing at compile time");
        
        // Demonstrate some of these practices
        System.out.println("\nDemonstrating best practices in code:");
        
        // Good: Caching with proper error handling
        class BestPracticeExample {
            private static final Map<String, Method> methodCache = new ConcurrentHashMap<>();
            
            public static Object invokeCached(Object target, String methodName, Object... args) 
                    throws Exception {
                Class<?> clazz = target.getClass();
                String cacheKey = clazz.getName() + "." + methodName + "." + args.length;
                
                Method method = methodCache.computeIfAbsent(cacheKey, k -> {
                    try {
                        Class<?>[] paramTypes = Arrays.stream(args)
                            .map(arg -> arg != null ? arg.getClass() : Object.class)
                            .toArray(Class<?>[]::new);
                        Method m = clazz.getMethod(methodName, paramTypes);
                        m.setAccessible(true);
                        return m;
                    } catch (NoSuchMethodException e) {
                        throw new RuntimeException("Method not found: " + methodName, e);
                    }
                });
                
                try {
                    return method.invoke(target, args);
                } catch (InvocationTargetException e) {
                    throw new RuntimeException("Method execution failed: " + e.getCause().getMessage(), 
                                             e.getCause());
                }
            }
        }
        
        try {
            String testString = "Hello, World!";
            Object result = BestPracticeExample.invokeCached(testString, "substring", 0, 5);
            System.out.println("Best practice example result: " + result);
        } catch (Exception e) {
            System.err.println("Error in best practice example: " + e.getMessage());
        }
    }
    
    public static void main(String[] args) {
        demonstrateSecurityManager();
        demonstrateAccessControl();
        demonstrateSecureReflectionWrapper();
        demonstrateInputValidation();
        demonstrateExceptionHandling();
        demonstrateBestPractices();
    }
}
```

## Conclusion

Java Reflection and Annotations provide powerful mechanisms for building flexible and dynamic applications. Key takeaways include:

1. **Reflection Fundamentals**: Understanding how to inspect classes, methods, and fields at runtime
2. **Dynamic Invocation**: Executing methods and accessing fields dynamically with proper error handling
3. **Annotations Processing**: Creating and processing custom annotations for metadata and configuration
4. **Dynamic Proxies**: Building proxy instances that implement interfaces at runtime
5. **Performance Considerations**: Optimizing reflection usage for better performance
6. **Security Best Practices**: Implementing proper security measures when using reflection

These features enable advanced programming techniques such as dependency injection, aspect-oriented programming, and framework development while requiring careful consideration of performance and security implications.