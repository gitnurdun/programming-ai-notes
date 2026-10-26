# Advanced Java Functional Programming and Streams

## Table of Contents
1. [Introduction to Functional Programming in Java](#introduction-to-functional-programming-in-java)
2. [Functional Interfaces and Lambda Expressions](#functional-interfaces-and-lambda-expressions)
3. [Stream API Fundamentals](#stream-api-fundamentals)
4. [Stream Operations and Pipelines](#stream-operations-and-pipelines)
5. [Advanced Stream Operations](#advanced-stream-operations)
6. [Parallel Streams and Performance](#parallel-streams-and-performance)
7. [Collectors and Custom Collectors](#collectors-and-custom-collectors)
8. [Functional Design Patterns](#functional-design-patterns)
9. [Performance Optimization Techniques](#performance-optimization-techniques)
10. [Functional Programming Best Practices](#functional-programming-best-practices)

## Introduction to Functional Programming in Java

Functional programming in Java emphasizes the use of functions as first-class citizens, immutability, and the avoidance of changing state. Java 8 introduced functional programming concepts through lambda expressions, method references, and the Stream API, revolutionizing how developers approach data processing and manipulation.

### Core Principles of Functional Programming

```java
// Demonstrating core functional programming principles
public class FunctionalProgrammingPrinciples {
    
    // Immutability example
    public static class ImmutablePerson {
        private final String name;
        private final int age;
        private final String email;
        
        public ImmutablePerson(String name, int age, String email) {
            this.name = name;
            this.age = age;
            this.email = email;
        }
        
        // No setter methods - immutable object
        public String getName() { return name; }
        public int getAge() { return age; }
        public String getEmail() { return email; }
        
        // Return new object with modified value (functional approach)
        public ImmutablePerson withName(String newName) {
            return new ImmutablePerson(newName, this.age, this.email);
        }
        
        public ImmutablePerson withAge(int newAge) {
            return new ImmutablePerson(this.name, newAge, this.email);
        }
        
        @Override
        public String toString() {
            return "ImmutablePerson{name='" + name + "', age=" + age + ", email='" + email + "'}";
        }
    }
    
    // Function as first-class citizen
    public static class FunctionAsFirstClass {
        // Higher-order function that takes a function as parameter
        public static <T, R> List<R> map(List<T> list, Function<T, R> mapper) {
            List<R> result = new ArrayList<>();
            for (T item : list) {
                result.add(mapper.apply(item));
            }
            return result;
        }
        
        // Function that returns another function
        public static Function<String, Integer> createLengthFunction() {
            return String::length;
        }
    }
    
    // Pure functions (no side effects, same input always produces same output)
    public static class PureFunctions {
        // Pure function example
        public static int add(int a, int b) {
            return a + b; // No side effects, deterministic
        }
        
        // Impure function example (has side effects)
        public static int addWithLogging(int a, int b) {
            System.out.println("Adding " + a + " and " + b); // Side effect
            return a + b;
        }
    }
    
    public static void demonstratePrinciples() {
        System.out.println("=== Functional Programming Principles ===");
        
        // Immutability example
        ImmutablePerson person = new ImmutablePerson("John Doe", 30, "john@example.com");
        System.out.println("Original: " + person);
        
        // Create new object with modified value
        ImmutablePerson updatedPerson = person.withAge(31);
        System.out.println("Updated: " + updatedPerson);
        System.out.println("Original unchanged: " + person);
        
        // Function as first-class citizen
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
        List<Integer> lengths = FunctionAsFirstClass.map(names, String::length);
        System.out.println("Name lengths: " + lengths);
        
        // Function that returns function
        Function<String, Integer> lengthFunction = FunctionAsFirstClass.createLengthFunction();
        System.out.println("Length of 'Hello': " + lengthFunction.apply("Hello"));
        
        // Pure vs impure functions
        System.out.println("Pure function result: " + PureFunctions.add(5, 3));
        System.out.println("Impure function result: " + PureFunctions.addWithLogging(5, 3));
    }
    
    public static void main(String[] args) {
        demonstratePrinciples();
    }
}
```

### Functional vs Imperative Programming Comparison

```java
// Comparing functional and imperative approaches
public class FunctionalVsImperativeComparison {
    
    // Data model
    static class Employee {
        private String name;
        private int age;
        private double salary;
        private String department;
        
        public Employee(String name, int age, double salary, String department) {
            this.name = name;
            this.age = age;
            this.salary = salary;
            this.department = department;
        }
        
        // Getters
        public String getName() { return name; }
        public int getAge() { return age; }
        public double getSalary() { return salary; }
        public String getDepartment() { return department; }
        
        @Override
        public String toString() {
            return String.format("Employee{name='%s', age=%d, salary=%.2f, department='%s'}", 
                               name, age, salary, department);
        }
    }
    
    // Imperative approach - focusing on HOW to do things
    public static class ImperativeApproach {
        
        public static List<String> getHighEarnersImperative(List<Employee> employees, double salaryThreshold) {
            List<String> highEarners = new ArrayList<>();
            
            // Explicit loop and conditional logic
            for (Employee employee : employees) {
                if (employee.getSalary() > salaryThreshold) {
                    String upperName = employee.getName().toUpperCase();
                    if (!highEarners.contains(upperName)) { // Avoid duplicates
                        highEarners.add(upperName);
                    }
                }
            }
            
            // Sort the results
            Collections.sort(highEarners);
            
            return highEarners;
        }
        
        public static double calculateAverageAgeImperative(List<Employee> employees) {
            if (employees.isEmpty()) return 0.0;
            
            int totalAge = 0;
            int count = 0;
            
            for (Employee employee : employees) {
                if (employee.getAge() > 25) { // Only consider employees over 25
                    totalAge += employee.getAge();
                    count++;
                }
            }
            
            return count > 0 ? (double) totalAge / count : 0.0;
        }
    }
    
    // Functional approach - focusing on WHAT to do
    public static class FunctionalApproach {
        
        public static List<String> getHighEarnersFunctional(List<Employee> employees, double salaryThreshold) {
            return employees.stream()
                    .filter(employee -> employee.getSalary() > salaryThreshold)
                    .map(employee -> employee.getName().toUpperCase())
                    .distinct()
                    .sorted()
                    .collect(Collectors.toList());
        }
        
        public static double calculateAverageAgeFunctional(List<Employee> employees) {
            return employees.stream()
                    .filter(employee -> employee.getAge() > 25)
                    .mapToInt(Employee::getAge)
                    .average()
                    .orElse(0.0);
        }
        
        // More complex functional example
        public static Map<String, List<Employee>> groupByDepartmentFunctional(List<Employee> employees) {
            return employees.stream()
                    .collect(Collectors.groupingBy(Employee::getDepartment));
        }
        
        public static OptionalDouble getAverageSalaryByDepartmentFunctional(List<Employee> employees, String department) {
            return employees.stream()
                    .filter(emp -> emp.getDepartment().equals(department))
                    .mapToDouble(Employee::getSalary)
                    .average();
        }
    }
    
    public static void demonstrateComparison() {
        System.out.println("=== Functional vs Imperative Comparison ===");
        
        // Sample data
        List<Employee> employees = Arrays.asList(
            new Employee("Alice Johnson", 30, 75000.0, "Engineering"),
            new Employee("Bob Smith", 28, 65000.0, "Marketing"),
            new Employee("Charlie Brown", 35, 85000.0, "Engineering"),
            new Employee("Diana Prince", 29, 70000.0, "Sales"),
            new Employee("Eve Wilson", 32, 80000.0, "Engineering"),
            new Employee("Frank Miller", 27, 60000.0, "Marketing"),
            new Employee("Grace Lee", 31, 78000.0, "Sales")
        );
        
        System.out.println("Sample employees:");
        employees.forEach(System.out::println);
        
        // Compare approaches for getting high earners
        long startTime = System.nanoTime();
        List<String> highEarnersImperative = ImperativeApproach.getHighEarnersImperative(employees, 70000.0);
        long imperativeTime = System.nanoTime() - startTime;
        
        startTime = System.nanoTime();
        List<String> highEarnersFunctional = FunctionalApproach.getHighEarnersFunctional(employees, 70000.0);
        long functionalTime = System.nanoTime() - startTime;
        
        System.out.println("\nHigh earners (salary > 70000):");
        System.out.println("Imperative: " + highEarnersImperative);
        System.out.println("Functional: " + highEarnersFunctional);
        System.out.println("Imperative time: " + imperativeTime / 1000 + " μs");
        System.out.println("Functional time: " + functionalTime / 1000 + " μs");
        
        // Compare approaches for average age calculation
        startTime = System.nanoTime();
        double avgAgeImperative = ImperativeApproach.calculateAverageAgeImperative(employees);
        long imperativeTime2 = System.nanoTime() - startTime;
        
        startTime = System.nanoTime();
        double avgAgeFunctional = FunctionalApproach.calculateAverageAgeFunctional(employees);
        long functionalTime2 = System.nanoTime() - startTime;
        
        System.out.println("\nAverage age (employees > 25):");
        System.out.println("Imperative: " + avgAgeImperative);
        System.out.println("Functional: " + avgAgeFunctional);
        System.out.println("Imperative time: " + imperativeTime2 / 1000 + " μs");
        System.out.println("Functional time: " + functionalTime2 / 1000 + " μs");
        
        // Functional approach for more complex operations
        System.out.println("\nGrouped by department:");
        Map<String, List<Employee>> grouped = FunctionalApproach.groupByDepartmentFunctional(employees);
        grouped.forEach((dept, empList) -> {
            System.out.println(dept + ": " + empList.size() + " employees");
        });
        
        OptionalDouble engineeringAvg = FunctionalApproach.getAverageSalaryByDepartmentFunctional(employees, "Engineering");
        System.out.println("\nAverage salary in Engineering: " + 
                         engineeringAvg.map(avg -> String.format("%.2f", avg)).orElse("N/A"));
    }
    
    public static void main(String[] args) {
        demonstrateComparison();
    }
}
```

## Functional Interfaces and Lambda Expressions

Java provides a rich set of functional interfaces in the `java.util.function` package, along with powerful lambda expressions for concise functional programming.

### Core Functional Interfaces

```java
// Core functional interfaces and their usage
public class CoreFunctionalInterfaces {
    
    // Demonstrating the main functional interfaces
    public static void demonstrateCoreInterfaces() {
        System.out.println("=== Core Functional Interfaces ===");
        
        // Predicate<T> - takes T, returns boolean
        System.out.println("\n--- Predicate ---");
        Predicate<Integer> isEven = x -> x % 2 == 0;
        Predicate<Integer> isPositive = x -> x > 0;
        
        System.out.println("Is 4 even? " + isEven.test(4));
        System.out.println("Is -3 positive? " + isPositive.test(-3));
        
        // Combining predicates
        Predicate<Integer> isPositiveEven = isPositive.and(isEven);
        System.out.println("Is 4 positive and even? " + isPositiveEven.test(4));
        System.out.println("Is -2 positive and even? " + isPositiveEven.test(-2));
        
        // Function<T, R> - takes T, returns R
        System.out.println("\n--- Function ---");
        Function<String, Integer> stringToLength = String::length;
        Function<Integer, String> numberToString = Object::toString;
        Function<String, String> toUpperCase = String::toUpperCase;
        
        System.out.println("Length of 'Hello': " + stringToLength.apply("Hello"));
        System.out.println("Number to string: " + numberToString.apply(42));
        
        // Function composition
        Function<String, Integer> composed = toUpperCase.andThen(stringToLength);
        System.out.println("Length of 'Hello' in uppercase: " + composed.apply("Hello"));
        
        // Using andThen and compose
        Function<String, Integer> composed2 = stringToLength.andThen(x -> x * 2);
        System.out.println("Double the length of 'Hi': " + composed2.apply("Hi"));
        
        // Consumer<T> - takes T, returns void
        System.out.println("\n--- Consumer ---");
        Consumer<String> printer = System.out::println;
        Consumer<Integer> squareAndPrint = x -> System.out.println(x * x);
        
        printer.accept("Hello from consumer!");
        squareAndPrint.accept(5);
        
        // Chaining consumers
        Consumer<String> firstConsumer = s -> System.out.print("Processing: ");
        Consumer<String> secondConsumer = System.out::println;
        Consumer<String> combinedConsumer = firstConsumer.andThen(secondConsumer);
        combinedConsumer.accept("Hello");
        
        // Supplier<T> - takes no parameters, returns T
        System.out.println("\n--- Supplier ---");
        Supplier<String> randomString = () -> "Random-" + System.nanoTime();
        Supplier<Integer> randomInt = () -> ThreadLocalRandom.current().nextInt(100);
        
        System.out.println("Random string: " + randomString.get());
        System.out.println("Random int: " + randomInt.get());
        
        // UnaryOperator<T> - special case of Function<T, T>
        System.out.println("\n--- UnaryOperator ---");
        UnaryOperator<Integer> square = x -> x * x;
        UnaryOperator<String> addExclamation = s -> s + "!";
        
        System.out.println("Square of 5: " + square.apply(5));
        System.out.println("With exclamation: " + addExclamation.apply("Hello"));
        
        // BinaryOperator<T> - special case of BiFunction<T, T, T>
        System.out.println("\n--- BinaryOperator ---");
        BinaryOperator<Integer> max = Integer::max;
        BinaryOperator<String> concatenate = String::concat;
        
        System.out.println("Max of 10 and 20: " + max.apply(10, 20));
        System.out.println("Concatenated: " + concatenate.apply("Hello ", "World"));
    }
    
    // Primitive specializations
    public static void demonstratePrimitiveSpecializations() {
        System.out.println("\n=== Primitive Specializations ===");
        
        // IntFunction, LongFunction, DoubleFunction
        IntFunction<String> intToString = i -> "Number: " + i;
        System.out.println("Int to string: " + intToString.apply(42));
        
        // ToIntFunction, ToLongFunction, ToDoubleFunction
        ToIntFunction<String> stringToInt = Integer::parseInt;
        try {
            System.out.println("String to int: " + stringToInt.applyAsInt("123"));
        } catch (NumberFormatException e) {
            System.out.println("Number format exception: " + e.getMessage());
        }
        
        // IntConsumer, LongConsumer, DoubleConsumer
        IntConsumer intPrinter = i -> System.out.println("Int: " + i);
        intPrinter.accept(42);
        
        // IntPredicate, LongPredicate, DoublePredicate
        IntPredicate isGreaterThanTen = x -> x > 10;
        System.out.println("Is 15 > 10? " + isGreaterThanTen.test(15));
        
        // IntUnaryOperator, LongUnaryOperator, DoubleUnaryOperator
        IntUnaryOperator doubleValue = x -> x * 2;
        System.out.println("Double 25: " + doubleValue.applyAsInt(25));
        
        // IntBinaryOperator, LongBinaryOperator, DoubleBinaryOperator
        IntBinaryOperator multiply = (x, y) -> x * y;
        System.out.println("Multiply 6 and 7: " + multiply.applyAsInt(6, 7));
    }
    
    // Bi-functional interfaces
    public static void demonstrateBiFunctionalInterfaces() {
        System.out.println("\n--- Bi-Functional Interfaces ---");
        
        // BiPredicate<T, U>
        BiPredicate<String, Integer> stringLengthMatch = (str, len) -> str.length() == len;
        System.out.println("Does 'Hello' have length 5? " + stringLengthMatch.test("Hello", 5));
        
        // BiFunction<T, U, R>
        BiFunction<String, String, Integer> stringDistance = (s1, s2) -> {
            if (s1.length() != s2.length()) return -1;
            int distance = 0;
            for (int i = 0; i < s1.length(); i++) {
                if (s1.charAt(i) != s2.charAt(i)) distance++;
            }
            return distance;
        };
        
        System.out.println("String distance between 'cat' and 'bat': " + 
                         stringDistance.apply("cat", "bat"));
        
        // BiConsumer<T, U>
        BiConsumer<String, Integer> keyValueProcessor = (key, value) -> 
            System.out.println(key + ": " + value);
        keyValueProcessor.accept("Age", 25);
        
        // BinaryOperator as special case of BiFunction
        BinaryOperator<String> stringCombiner = (s1, s2) -> s1 + " " + s2;
        System.out.println("Combined string: " + stringCombiner.apply("Hello", "World"));
    }
    
    // Custom functional interface
    public static void demonstrateCustomFunctionalInterface() {
        System.out.println("\n--- Custom Functional Interface ---");
        
        // Define and use a custom functional interface
        @FunctionalInterface
        interface TriFunction<T, U, V, R> {
            R apply(T t, U u, V v);
            
            // Default method
            default TriFunction<T, U, V, R> andThen(Function<R, R> after) {
                return (t, u, v) -> after.apply(apply(t, u, v));
            }
        }
        
        TriFunction<Integer, Integer, Integer, Integer> sumThree = (a, b, c) -> a + b + c;
        System.out.println("Sum of 1, 2, 3: " + sumThree.apply(1, 2, 3));
        
        // Using the default method
        TriFunction<Integer, Integer, Integer, Integer> sumThreeSquared = 
            sumThree.andThen(x -> x * x);
        System.out.println("Sum of 1, 2, 3 squared: " + sumThreeSquared.apply(1, 2, 3));
    }
    
    public static void main(String[] args) {
        demonstrateCoreInterfaces();
        demonstratePrimitiveSpecializations();
        demonstrateBiFunctionalInterfaces();
        demonstrateCustomFunctionalInterface();
    }
}
```

### Advanced Lambda Expression Techniques

```java
// Advanced lambda expression techniques and patterns
public class AdvancedLambdaExpressions {
    
    // Method references and their various forms
    public static void demonstrateMethodReferences() {
        System.out.println("=== Method References ===");
        
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "Diana");
        
        // Reference to static method
        System.out.println("\n--- Static Method Reference ---");
        List<Integer> lengths = names.stream()
                .map(String::length) // Same as x -> x.length()
                .collect(Collectors.toList());
        System.out.println("String lengths: " + lengths);
        
        // Reference to instance method of particular object
        System.out.println("\n--- Instance Method Reference ---");
        String delimiter = "-";
        String result = names.stream()
                .collect(Collectors.joining(delimiter::toString)); // This won't work as expected
        // Better approach:
        result = names.stream()
                .collect(Collectors.joining("-", "[", "]")); // Using String instance method
        System.out.println("Joined with instance method: " + result);
        
        // Reference to instance method of arbitrary object of particular type
        System.out.println("\n--- Arbitrary Object Instance Method Reference ---");
        List<String> upperNames = names.stream()
                .map(String::toUpperCase) // Same as s -> s.toUpperCase()
                .collect(Collectors.toList());
        System.out.println("Uppercase names: " + upperNames);
        
        // Reference to constructor
        System.out.println("\n--- Constructor Reference ---");
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
        Set<Integer> numberSet = numbers.stream()
                .collect(Collectors.toCollection(TreeSet::new)); // Constructor reference
        System.out.println("TreeSet from constructor reference: " + numberSet);
        
        // Generic constructor reference
        List<String> stringList = names.stream()
                .collect(Collectors.toCollection(ArrayList::new));
        System.out.println("ArrayList from constructor reference: " + stringList);
    }
    
    // Complex lambda expressions with multiple parameters
    public static void demonstrateComplexLambdas() {
        System.out.println("\n=== Complex Lambda Expressions ===");
        
        // Multi-line lambda with complex logic
        System.out.println("\n--- Multi-line Lambda ---");
        Function<List<Integer>, OptionalInt> findMaxWithLogging = list -> {
            System.out.println("Processing list of size: " + list.size());
            if (list.isEmpty()) {
                System.out.println("List is empty, returning empty OptionalInt");
                return OptionalInt.empty();
            }
            
            int max = list.get(0);
            for (int i = 1; i < list.size(); i++) {
                if (list.get(i) > max) {
                    max = list.get(i);
                    System.out.println("New maximum found: " + max);
                }
            }
            
            return OptionalInt.of(max);
        };
        
        OptionalInt maxResult = findMaxWithLogging.apply(Arrays.asList(3, 1, 4, 1, 5, 9, 2, 6));
        System.out.println("Maximum value: " + (maxResult.isPresent() ? maxResult.getAsInt() : "None"));
        
        // Lambda with local variables (effectively final)
        System.out.println("\n--- Lambda with Local Variables ---");
        int multiplier = 10; // effectively final
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
        
        List<Integer> multiplied = numbers.stream()
                .map(x -> x * multiplier) // Using local variable
                .collect(Collectors.toList());
        System.out.println("Multiplied by " + multiplier + ": " + multiplied);
        
        // Lambda with exception handling
        System.out.println("\n--- Lambda with Exception Handling ---");
        List<String> numberStrings = Arrays.asList("1", "2", "invalid", "4", "5");
        
        List<Integer> parsedNumbers = numberStrings.stream()
                .map(s -> {
                    try {
                        return Integer.parseInt(s);
                    } catch (NumberFormatException e) {
                        System.out.println("Invalid number: " + s);
                        return 0; // or return null, or skip
                    }
                })
                .filter(n -> n != 0) // Remove invalid numbers
                .collect(Collectors.toList());
        System.out.println("Parsed valid numbers: " + parsedNumbers);
    }
    
    // Functional interfaces with generics and type inference
    public static void demonstrateGenericFunctionalInterfaces() {
        System.out.println("\n=== Generic Functional Interfaces ===");
        
        // Higher-order functions that return other functions
        System.out.println("\n--- Function Factories ---");
        Function<Integer, Function<Integer, Integer>> createMultiplier = factor -> 
            operand -> operand * factor;
        
        Function<Integer, Integer> multiplyBy5 = createMultiplier.apply(5);
        System.out.println("10 * 5 = " + multiplyBy5.apply(10));
        
        // Function that composes other functions
        Function<String, Function<String, String>> createConcatenator = prefix ->
            suffix -> prefix + suffix;
        
        Function<String, String> addHello = createConcatenator.apply("Hello ");
        System.out.println(addHello.apply("World"));
        
        // Generic method working with functional interfaces
        System.out.println("\n--- Generic Functional Methods ---");
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        // Generic filter method
        List<Integer> evenNumbers = filterList(numbers, x -> x % 2 == 0);
        System.out.println("Even numbers: " + evenNumbers);
        
        // Generic transform method
        List<String> numberWords = transformList(numbers, 
            num -> num % 2 == 0 ? "even" : "odd");
        System.out.println("Number types: " + numberWords);
    }
    
    // Generic utility methods
    public static <T> List<T> filterList(List<T> list, Predicate<T> predicate) {
        return list.stream()
                .filter(predicate)
                .collect(Collectors.toList());
    }
    
    public static <T, R> List<R> transformList(List<T> list, Function<T, R> transformer) {
        return list.stream()
                .map(transformer)
                .collect(Collectors.toList());
    }
    
    // Recursive lambda expressions (challenging due to final restrictions)
    public static void demonstrateRecursiveLambdas() {
        System.out.println("\n--- Recursive Lambda Expressions ---");
        
        // For recursive lambdas, we need to declare the variable first
        class RecursiveFactorial {
            Function<Integer, Integer> factorial;
            
            {
                factorial = n -> n <= 1 ? 1 : n * factorial.apply(n - 1);
            }
        }
        
        RecursiveFactorial recFact = new RecursiveFactorial();
        System.out.println("Factorial of 5: " + recFact.factorial.apply(5));
        
        // Fibonacci with memoization
        class MemoizedFibonacci {
            private final Map<Integer, Integer> cache = new HashMap<>();
            
            Function<Integer, Integer> fibonacci = n -> {
                if (n <= 1) return n;
                
                Integer cached = cache.get(n);
                if (cached != null) {
                    return cached;
                }
                
                int result = fibonacci.apply(n - 1) + fibonacci.apply(n - 2);
                cache.put(n, result);
                return result;
            };
        }
        
        MemoizedFibonacci fib = new MemoizedFibonacci();
        System.out.println("Fibonacci of 10: " + fib.fibonacci.apply(10));
    }
    
    public static void main(String[] args) {
        demonstrateMethodReferences();
        demonstrateComplexLambdas();
        demonstrateGenericFunctionalInterfaces();
        demonstrateRecursiveLambdas();
    }
}
```

## Stream API Fundamentals

The Stream API provides a powerful and expressive way to process sequences of elements with functional-style operations.

### Stream Creation and Basic Operations

```java
// Stream API fundamentals and creation methods
public class StreamAPIFundamentals {
    
    // Various ways to create streams
    public static void demonstrateStreamCreation() {
        System.out.println("=== Stream Creation Methods ===");
        
        // From collections
        System.out.println("\n--- From Collections ---");
        List<String> list = Arrays.asList("apple", "banana", "cherry");
        list.stream()
            .forEach(System.out::println);
        
        Set<Integer> set = new HashSet<>(Arrays.asList(1, 2, 3, 4, 5));
        set.stream()
            .filter(x -> x % 2 == 0)
            .forEach(System.out::println);
        
        // From arrays
        System.out.println("\n--- From Arrays ---");
        String[] array = {"one", "two", "three", "four"};
        Arrays.stream(array)
            .map(String::toUpperCase)
            .forEach(System.out::println);
        
        // Using Stream.of()
        System.out.println("\n--- Stream.of() ---");
        Stream.of("first", "second", "third")
            .map(String::toUpperCase)
            .forEach(System.out::println);
        
        // Using Stream.iterate()
        System.out.println("\n--- Stream.iterate() ---");
        Stream.iterate(1, n -> n + 1)  // Java 9+ has better iterate
            .limit(5)
            .forEach(System.out::println);
        
        // Using Stream.generate()
        System.out.println("\n--- Stream.generate() ---");
        Stream.generate(Math::random)
            .limit(5)
            .forEach(System.out::println);
        
        // From I/O streams
        System.out.println("\n--- From I/O Streams ---");
        try (Stream<String> lines = Files.lines(Paths.get("example.txt"))) {
            // Process lines
            lines.filter(line -> !line.isEmpty())
                 .forEach(System.out::println);
        } catch (IOException e) {
            System.out.println("File not found, creating example:");
            // Create a temporary file for demonstration
            try {
                Path tempFile = Files.createTempFile("example", ".txt");
                Files.write(tempFile, Arrays.asList("Line 1", "Line 2", "Line 3", ""));
                try (Stream<String> lines = Files.lines(tempFile)) {
                    lines.filter(line -> !line.isEmpty())
                         .forEach(System.out::println);
                }
                Files.delete(tempFile);
            } catch (IOException ioException) {
                ioException.printStackTrace();
            }
        }
        
        // Numeric streams
        System.out.println("\n--- Numeric Streams ---");
        IntStream.range(1, 6)
            .forEach(System.out::println);
        
        LongStream.rangeClosed(10, 15)
            .forEach(System.out::println);
        
        DoubleStream.of(1.1, 2.2, 3.3)
            .forEach(System.out::println);
    }
    
    // Stream intermediate operations
    public static void demonstrateIntermediateOperations() {
        System.out.println("\n=== Stream Intermediate Operations ===");
        
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        System.out.println("\n--- filter() ---");
        numbers.stream()
            .filter(n -> n % 2 == 0)
            .forEach(System.out::print);
        System.out.println();
        
        System.out.println("\n--- map() ---");
        numbers.stream()
            .map(n -> n * n)
            .forEach(System.out::print);
        System.out.println();
        
        System.out.println("\n--- flatMap() ---");
        List<List<Integer>> nestedList = Arrays.asList(
            Arrays.asList(1, 2),
            Arrays.asList(3, 4),
            Arrays.asList(5, 6)
        );
        
        nestedList.stream()
            .flatMap(List::stream)
            .forEach(System.out::print);
        System.out.println();
        
        System.out.println("\n--- distinct() ---");
        List<Integer> withDuplicates = Arrays.asList(1, 2, 2, 3, 3, 3, 4);
        withDuplicates.stream()
            .distinct()
            .forEach(System.out::print);
        System.out.println();
        
        System.out.println("\n--- sorted() ---");
        List<String> words = Arrays.asList("banana", "apple", "cherry", "date");
        words.stream()
            .sorted()
            .forEach(System.out::print);
        System.out.print(" ");
        words.stream()
            .sorted((a, b) -> b.compareTo(a)) // Reverse order
            .forEach(System.out::print);
        System.out.println();
        
        System.out.println("\n--- limit() and skip() ---");
        numbers.stream()
            .skip(3)  // Skip first 3 elements
            .limit(4) // Take next 4 elements
            .forEach(System.out::print);
        System.out.println();
        
        System.out.println("\n--- peek() ---");
        List<Integer> result = numbers.stream()
            .filter(n -> n % 2 == 0)
            .peek(n -> System.out.println("Processing: " + n))
            .map(n -> n * 2)
            .collect(Collectors.toList());
        System.out.println("Final result: " + result);
    }
    
    // Stream chain optimization
    public static void demonstrateStreamOptimization() {
        System.out.println("\n=== Stream Chain Optimization ===");
        
        List<String> words = Arrays.asList("apple", "banana", "cherry", "date", "elderberry", "fig", "grape");
        
        System.out.println("\n--- Optimized Chain ---");
        List<String> result = words.stream()
            .filter(s -> s.length() > 4)  // Filter first to reduce data early
            .map(String::toUpperCase)     // Then transform
            .sorted()                     // Finally sort
            .limit(3)                     // Limit the result
            .collect(Collectors.toList());
        
        System.out.println("Result: " + result);
        
        System.out.println("\n--- Short-circuiting operations ---");
        boolean hasLongWord = words.stream()
            .anyMatch(s -> s.length() > 10); // Short-circuits on first match
        System.out.println("Has word longer than 10 chars: " + hasLongWord);
        
        Optional<String> firstLongWord = words.stream()
            .filter(s -> s.length() > 10)
            .findFirst(); // Short-circuits, finds first match
        System.out.println("First long word: " + firstLongWord.orElse("None found"));
        
        // Infinite stream with short-circuiting
        System.out.println("\n--- Infinite Stream with Short-circuiting ---");
        List<Integer> evenNumbers = Stream.iterate(1, n -> n + 1)
            .filter(n -> n % 2 == 0)
            .limit(5)  // Essential to avoid infinite processing
            .collect(Collectors.toList());
        System.out.println("First 5 even numbers: " + evenNumbers);
    }
    
    // Stream debugging and monitoring
    public static void demonstrateStreamDebugging() {
        System.out.println("\n=== Stream Debugging and Monitoring ===");
        
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        System.out.println("\n--- Using peek() for debugging ---");
        List<Integer> processed = numbers.stream()
            .peek(n -> System.out.println("Before filter: " + n))
            .filter(n -> n % 2 == 0)
            .peek(n -> System.out.println("After filter: " + n))
            .map(n -> n * n)
            .peek(n -> System.out.println("After map (squared): " + n))
            .collect(Collectors.toList());
        
        System.out.println("Final processed list: " + processed);
        
        System.out.println("\n--- Measuring stream performance ---");
        List<Integer> largeList = IntStream.range(0, 1000000)
            .boxed()
            .collect(Collectors.toList());
        
        long startTime = System.nanoTime();
        long sum = largeList.stream()
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> n * n)
            .sum();
        long endTime = System.nanoTime();
        
        System.out.println("Processing time: " + (endTime - startTime) / 1_000_000 + " ms");
        System.out.println("Sum of squares of even numbers: " + sum);
    }
    
    public static void main(String[] args) throws IOException {
        demonstrateStreamCreation();
        demonstrateIntermediateOperations();
        demonstrateStreamOptimization();
        demonstrateStreamDebugging();
    }
}
```

### Stream Operations and Pipelines

```java
// Stream operations and pipeline patterns
public class StreamOperationsAndPipelines {
    
    // Terminal operations
    public static void demonstrateTerminalOperations() {
        System.out.println("=== Stream Terminal Operations ===");
        
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        List<String> words = Arrays.asList("apple", "banana", "cherry", "date");
        
        System.out.println("\n--- Reduction operations ---");
        
        // sum(), average(), max(), min on numeric streams
        int sum = numbers.stream().mapToInt(Integer::intValue).sum();
        System.out.println("Sum: " + sum);
        
        OptionalDouble average = numbers.stream().mapToDouble(Integer::doubleValue).average();
        System.out.println("Average: " + average.orElse(0.0));
        
        OptionalInt max = numbers.stream().mapToInt(Integer::intValue).max();
        System.out.println("Max: " + max.orElse(0));
        
        OptionalInt min = numbers.stream().mapToInt(Integer::intValue).min();
        System.out.println("Min: " + min.orElse(0));
        
        System.out.println("\n--- Matching operations ---");
        
        boolean allEven = numbers.stream().allMatch(n -> n % 2 == 0);
        System.out.println("All numbers even: " + allEven);
        
        boolean anyEven = numbers.stream().anyMatch(n -> n % 2 == 0);
        System.out.println("Any number even: " + anyEven);
        
        boolean noneNegative = numbers.stream().noneMatch(n -> n < 0);
        System.out.println("No negative numbers: " + noneNegative);
        
        System.out.println("\n--- Finding operations ---");
        
        Optional<Integer> firstEven = numbers.stream()
            .filter(n -> n % 2 == 0)
            .findFirst();
        System.out.println("First even number: " + firstEven.orElse(-1));
        
        Optional<Integer> anyEvenNumber = numbers.stream()
            .filter(n -> n % 2 == 0)
            .findAny();
        System.out.println("Any even number: " + anyEvenNumber.orElse(-1));
        
        System.out.println("\n--- Collection operations ---");
        
        List<Integer> evens = numbers.stream()
            .filter(n -> n % 2 == 0)
            .collect(Collectors.toList());
        System.out.println("Even numbers: " + evens);
        
        Set<Integer> uniqueEvens = numbers.stream()
            .filter(n -> n % 2 == 0)
            .collect(Collectors.toSet());
        System.out.println("Unique evens (not applicable here): " + uniqueEvens);
        
        Map<String, Integer> wordLengths = words.stream()
            .collect(Collectors.toMap(
                Function.identity(),  // key mapper
                String::length       // value mapper
            ));
        System.out.println("Word lengths: " + wordLengths);
        
        System.out.println("\n--- Reduction with identity ---");
        
        // reduce with identity value
        int product = numbers.stream()
            .reduce(1, (a, b) -> a * b);
        System.out.println("Product of all numbers: " + product);
        
        // reduce with combiner (for parallel streams)
        Optional<Integer> optionalSum = numbers.stream()
            .reduce(Integer::sum);
        System.out.println("Sum using reduce: " + optionalSum.orElse(0));
        
        // custom reduction
        String concatenated = words.stream()
            .reduce("", String::concat);
        System.out.println("Concatenated words: " + concatenated);
        
        String concatenatedWithSpaces = words.stream()
            .reduce((a, b) -> a + " " + b)
            .orElse("");
        System.out.println("Concatenated with spaces: " + concatenatedWithSpaces);
    }
    
    // Complex stream pipelines
    public static void demonstrateComplexPipelines() {
        System.out.println("\n=== Complex Stream Pipelines ===");
        
        // Data model
        class Person {
            String name;
            int age;
            String city;
            double salary;
            
            Person(String name, int age, String city, double salary) {
                this.name = name;
                this.age = age;
                this.city = city;
                this.salary = salary;
            }
            
            @Override
            public String toString() {
                return String.format("Person{name='%s', age=%d, city='%s', salary=%.2f}", 
                                   name, age, city, salary);
            }
            
            // Getters
            public String getName() { return name; }
            public int getAge() { return age; }
            public String getCity() { return city; }
            public double getSalary() { return salary; }
        }
        
        List<Person> people = Arrays.asList(
            new Person("Alice", 30, "New York", 75000.0),
            new Person("Bob", 25, "San Francisco", 80000.0),
            new Person("Charlie", 35, "New York", 90000.0),
            new Person("Diana", 28, "Chicago", 70000.0),
            new Person("Eve", 32, "San Francisco", 85000.0),
            new Person("Frank", 29, "New York", 65000.0),
            new Person("Grace", 27, "Chicago", 72000.0)
        );
        
        System.out.println("\n--- Complex data processing pipeline ---");
        
        // Find average salary of people in New York over 30
        OptionalDouble nyAvgSalary = people.stream()
            .filter(p -> p.getCity().equals("New York"))
            .filter(p -> p.getAge() > 30)
            .mapToDouble(Person::getSalary)
            .average();
        
        System.out.println("Average salary in NY for age > 30: " + 
                         nyAvgSalary.map(avg -> String.format("%.2f", avg)).orElse("N/A"));
        
        System.out.println("\n--- Grouping and counting ---");
        
        // Count people by city
        Map<String, Long> countByCity = people.stream()
            .collect(Collectors.groupingBy(
                Person::getCity,
                Collectors.counting()
            ));
        System.out.println("Count by city: " + countByCity);
        
        // Average age by city
        Map<String, Double> avgAgeByCity = people.stream()
            .collect(Collectors.groupingBy(
                Person::getCity,
                Collectors.averagingInt(Person::getAge)
            ));
        System.out.println("Average age by city: " + avgAgeByCity);
        
        System.out.println("\n--- Complex transformation pipeline ---");
        
        // Get all names of people earning above average salary
        OptionalDouble overallAvgSalary = people.stream()
            .mapToDouble(Person::getSalary)
            .average();
        
        if (overallAvgSalary.isPresent()) {
            List<String> highEarners = people.stream()
                .filter(p -> p.getSalary() > overallAvgSalary.getAsDouble())
                .map(Person::getName)
                .sorted()
                .collect(Collectors.toList());
            
            System.out.println("High earners (above average " + 
                             String.format("%.2f", overallAvgSalary.getAsDouble()) + "): " + 
                             highEarners);
        }
        
        System.out.println("\n--- Nested collections processing ---");
        
        // Group by city and get list of names
        Map<String, List<String>> namesByCity = people.stream()
            .collect(Collectors.groupingBy(
                Person::getCity,
                Collectors.mapping(Person::getName, Collectors.toList())
            ));
        System.out.println("Names by city: " + namesByCity);
        
        // More complex: Group by city, then sub-group by age range
        Map<String, Map<String, List<Person>>> peopleByCityAndAgeRange = people.stream()
            .collect(Collectors.groupingBy(
                Person::getCity,
                Collectors.groupingBy(p -> p.getAge() < 30 ? "Young" : "Mature")
            ));
        
        System.out.println("\nPeople by city and age range:");
        peopleByCityAndAgeRange.forEach((city, ageGroups) -> {
            System.out.println(city + ": " + ageGroups);
        });
    }
    
    // Stream reuse and pipeline reuse patterns
    public static void demonstratePipelineReuse() {
        System.out.println("\n=== Stream Pipeline Reuse ===");
        
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15);
        
        System.out.println("\n--- Pipeline reuse patterns ---");
        
        // Store intermediate results to reuse
        List<Integer> evenNumbers = numbers.stream()
            .filter(n -> n % 2 == 0)
            .collect(Collectors.toList());
        
        System.out.println("Even numbers: " + evenNumbers);
        System.out.println("Count of even numbers: " + evenNumbers.size());
        System.out.println("Sum of even numbers: " + evenNumbers.stream().mapToInt(Integer::intValue).sum());
        
        // Using Supplier for reusable pipelines
        Supplier<Stream<Integer>> evenStreamSupplier = () -> 
            numbers.stream().filter(n -> n % 2 == 0);
        
        long count = evenStreamSupplier.get().count();
        int sum = evenStreamSupplier.get().mapToInt(Integer::intValue).sum();
        List<Integer> list = evenStreamSupplier.get().collect(Collectors.toList());
        
        System.out.println("Supplier results - Count: " + count + ", Sum: " + sum + ", List size: " + list.size());
        
        // Reusable pipeline with different operations
        class NumberProcessor {
            private final List<Integer> numbers;
            
            public NumberProcessor(List<Integer> numbers) {
                this.numbers = numbers;
            }
            
            public List<Integer> getEvens() {
                return numbers.stream()
                    .filter(n -> n % 2 == 0)
                    .collect(Collectors.toList());
            }
            
            public List<Integer> getOdds() {
                return numbers.stream()
                    .filter(n -> n % 2 != 0)
                    .collect(Collectors.toList());
            }
            
            public List<Integer> getGreaterThan(int threshold) {
                return numbers.stream()
                    .filter(n -> n > threshold)
                    .collect(Collectors.toList());
            }
        }
        
        NumberProcessor processor = new NumberProcessor(numbers);
        System.out.println("Evens: " + processor.getEvens());
        System.out.println("Odds: " + processor.getOdds());
        System.out.println("Greater than 10: " + processor.getGreaterThan(10));
    }
    
    public static void main(String[] args) {
        demonstrateTerminalOperations();
        demonstrateComplexPipelines();
        demonstratePipelineReuse();
    }
}
```

## Advanced Stream Operations

Exploring sophisticated stream operations and specialized use cases.

### Specialized Stream Operations

```java
// Advanced stream operations and specialized techniques
public class AdvancedStreamOperations {
    
    // FlatMap for nested structures
    public static void demonstrateFlatMapOperations() {
        System.out.println("=== FlatMap Operations ===");
        
        // Flattening nested collections
        System.out.println("\n--- Flattening nested collections ---");
        List<List<Integer>> nestedNumbers = Arrays.asList(
            Arrays.asList(1, 2, 3),
            Arrays.asList(4, 5),
            Arrays.asList(6, 7, 8, 9),
            Arrays.asList(10)
        );
        
        List<Integer> flattened = nestedNumbers.stream()
            .flatMap(List::stream)
            .collect(Collectors.toList());
        System.out.println("Flattened: " + flattened);
        
        // Working with strings and their characters
        System.out.println("\n--- Flattening strings to characters ---");
        List<String> words = Arrays.asList("hello", "world", "java");
        
        List<String> allCharacters = words.stream()
            .flatMap(word -> word.chars().mapToObj(c -> String.valueOf((char) c)))
            .collect(Collectors.toList());
        System.out.println("All characters: " + allCharacters);
        
        // Flattening with transformation
        System.out.println("\n--- FlatMap with transformation ---");
        class Department {
            String name;
            List<String> employees;
            
            Department(String name, List<String> employees) {
                this.name = name;
                this.employees = employees;
            }
            
            public String getName() { return name; }
            public List<String> getEmployees() { return employees; }
        }
        
        List<Department> departments = Arrays.asList(
            new Department("Engineering", Arrays.asList("Alice", "Bob", "Charlie")),
            new Department("Marketing", Arrays.asList("Diana", "Eve")),
            new Department("Sales", Arrays.asList("Frank", "Grace", "Henry"))
        );
        
        List<String> allEmployees = departments.stream()
            .flatMap(dept -> dept.getEmployees().stream())
            .collect(Collectors.toList());
        System.out.println("All employees: " + allEmployees);
        
        // FlatMap with Optional
        System.out.println("\n--- FlatMap with Optional ---");
        List<String> strings = Arrays.asList("1", "2", "invalid", "4", "5");
        
        List<Integer> validNumbers = strings.stream()
            .map(s -> {
                try {
                    return Optional.of(Integer.parseInt(s));
                } catch (NumberFormatException e) {
                    return Optional.empty();
                }
            })
            .flatMap(opt -> opt.isPresent() ? Stream.of(opt.get()) : Stream.empty())
            .collect(Collectors.toList());
        System.out.println("Valid numbers: " + validNumbers);
        
        // More elegant approach with Optional.stream() (Java 9+)
        // For Java 8, we can create a utility method
        List<Integer> validNumbersElegant = strings.stream()
            .map(AdvancedStreamOperations::parseIntSafely)
            .filter(Optional::isPresent)
            .map(Optional::get)
            .collect(Collectors.toList());
        System.out.println("Valid numbers (elegant): " + validNumbersElegant);
    }
    
    // Utility method for safe parsing
    private static Optional<Integer> parseIntSafely(String s) {
        try {
            return Optional.of(Integer.parseInt(s));
        } catch (NumberFormatException e) {
            return Optional.empty();
        }
    }
    
    // Advanced reduction operations
    public static void demonstrateAdvancedReductions() {
        System.out.println("\n=== Advanced Reduction Operations ===");
        
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        System.out.println("\n--- Custom reduction with StringBuilder ---");
        String concatenated = numbers.stream()
            .map(String::valueOf)
            .reduce(new StringBuilder(), StringBuilder::append, StringBuilder::append)
            .toString();
        System.out.println("Concatenated: " + concatenated);
        
        System.out.println("\n--- Reduction with custom accumulator ---");
        // Calculate statistics in a single pass
        class Statistics {
            int count = 0;
            long sum = 0;
            int min = Integer.MAX_VALUE;
            int max = Integer.MIN_VALUE;
            
            void combine(Statistics other) {
                this.count += other.count;
                this.sum += other.sum;
                this.min = Math.min(this.min, other.min);
                this.max = Math.max(this.max, other.max);
            }
            
            double getAverage() {
                return count > 0 ? (double) sum / count : 0.0;
            }
        }
        
        Statistics stats = numbers.stream()
            .collect(
                Statistics::new,                    // Supplier
                (stat, num) -> {                   // Accumulator
                    stat.count++;
                    stat.sum += num;
                    stat.min = Math.min(stat.min, num);
                    stat.max = Math.max(stat.max, num);
                },
                Statistics::combine                // Combiner
            );
        
        System.out.println("Statistics: count=" + stats.count + 
                         ", sum=" + stats.sum + 
                         ", min=" + stats.min + 
                         ", max=" + stats.max + 
                         ", avg=" + stats.getAverage());
        
        System.out.println("\n--- Reduction with multiple operations ---");
        // Using Teeing collector (Java 12+)
        // For Java 8/11, we'll simulate this behavior
        List<String> words = Arrays.asList("apple", "banana", "cherry", "date", "elderberry");
        
        // Find both shortest and longest strings in one pass
        class MinMaxResult {
            String min;
            String max;
            
            MinMaxResult(String min, String max) {
                this.min = min;
                this.max = max;
            }
            
            @Override
            public String toString() {
                return "MinMaxResult{min='" + min + "', max='" + max + "'}";
            }
        }
        
        MinMaxResult minMax = words.stream()
            .collect(Collectors.reducing(
                null,
                str -> new MinMaxResult(str, str),
                (r1, r2) -> new MinMaxResult(
                    r1.min.length() <= r2.min.length() ? r1.min : r2.min,
                    r1.max.length() >= r2.max.length() ? r1.max : r2.max
                )
            ));
        
        System.out.println("Min and max length strings: " + minMax);
    }
    
    // Stream partitioning and grouping
    public static void demonstratePartitioningGrouping() {
        System.out.println("\n=== Stream Partitioning and Grouping ===");
        
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12);
        
        System.out.println("\n--- Partitioning ---");
        Map<Boolean, List<Integer>> evenOddPartition = numbers.stream()
            .collect(Collectors.partitioningBy(n -> n % 2 == 0));
        System.out.println("Even/Odd partition: " + evenOddPartition);
        
        // Partitioning with downstream collector
        Map<Boolean, Long> evenOddCount = numbers.stream()
            .collect(Collectors.partitioningBy(
                n -> n % 2 == 0,
                Collectors.counting()
            ));
        System.out.println("Even/Odd counts: " + evenOddCount);
        
        // More complex partitioning
        Map<Boolean, Integer> evenSumPartition = numbers.stream()
            .collect(Collectors.partitioningBy(
                n -> n % 2 == 0,
                Collectors.summingInt(Integer::intValue)
            ));
        System.out.println("Even/Odd sums: " + evenSumPartition);
        
        System.out.println("\n--- Advanced Grouping ---");
        class Student {
            String name;
            int grade;
            String department;
            double gpa;
            
            Student(String name, int grade, String department, double gpa) {
                this.name = name;
                this.grade = grade;
                this.department = department;
                this.gpa = gpa;
            }
            
            @Override
            public String toString() {
                return String.format("Student{name='%s', grade=%d, dept='%s', gpa=%.2f}", 
                                   name, grade, department, gpa);
            }
            
            public String getName() { return name; }
            public int getGrade() { return grade; }
            public String getDepartment() { return department; }
            public double getGpa() { return gpa; }
        }
        
        List<Student> students = Arrays.asList(
            new Student("Alice", 10, "Math", 3.8),
            new Student("Bob", 11, "Science", 3.5),
            new Student("Charlie", 10, "Math", 3.9),
            new Student("Diana", 12, "English", 3.7),
            new Student("Eve", 11, "Science", 3.6),
            new Student("Frank", 10, "Math", 3.4),
            new Student("Grace", 12, "English", 3.9)
        );
        
        // Group by grade and then by department
        Map<Integer, Map<String, List<Student>>> studentsByGradeAndDept = students.stream()
            .collect(Collectors.groupingBy(
                Student::getGrade,
                Collectors.groupingBy(Student::getDepartment)
            ));
        
        System.out.println("\nStudents by grade and department:");
        studentsByGradeAndDept.forEach((grade, deptMap) -> {
            System.out.println("Grade " + grade + ":");
            deptMap.forEach((dept, studentList) -> {
                System.out.println("  " + dept + ": " + studentList.size() + " students");
            });
        });
        
        // Grouping with statistics
        Map<String, Double> avgGpaByDepartment = students.stream()
            .collect(Collectors.groupingBy(
                Student::getDepartment,
                Collectors.averagingDouble(Student::getGpa)
            ));
        System.out.println("\nAverage GPA by department: " + avgGpaByDepartment);
        
        // Grouping with multiple statistics
        Map<String, IntSummaryStatistics> gradeStatsByDept = students.stream()
            .collect(Collectors.groupingBy(
                Student::getDepartment,
                Collectors.summarizingInt(Student::getGrade)
            ));
        
        System.out.println("\nGrade statistics by department:");
        gradeStatsByDept.forEach((dept, stats) -> {
            System.out.println(dept + ": count=" + stats.getCount() + 
                             ", avg=" + stats.getAverage() + 
                             ", min=" + stats.getMin() + 
                             ", max=" + stats.getMax());
        });
    }
    
    // Stream manipulation with custom collectors
    public static void demonstrateCustomCollectors() {
        System.out.println("\n=== Custom Collectors ===");
        
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        System.out.println("\n--- Custom collector for unique elements ---");
        // Custom collector that maintains insertion order but only unique elements
        Collector<Integer, List<Integer>, List<Integer>> uniqueCollector = 
            Collector.of(
                ArrayList::new,                           // Supplier
                (list, item) -> { if (!list.contains(item)) list.add(item); }, // Accumulator
                (list1, list2) -> { list1.addAll(list2); return list1; },     // Combiner
                list -> list.stream().distinct().collect(Collectors.toList()) // Finisher
            );
        
        List<Integer> uniqueNumbers = numbers.stream()
            .map(n -> n % 5) // Create duplicates (0,1,2,3,4,0,1,2,3,4)
            .collect(uniqueCollector);
        System.out.println("Unique modulo results: " + uniqueNumbers);
        
        System.out.println("\n--- Custom collector for frequency map ---");
        String text = "hello world java programming";
        Map<Character, Long> charFrequency = text.chars()
            .filter(c -> c != ' ') // Skip spaces
            .mapToObj(c -> (char) c)
            .collect(Collectors.groupingBy(
                Function.identity(),
                Collectors.counting()
            ));
        
        System.out.println("Character frequencies: " + charFrequency);
        
        // Sort by frequency descending
        List<Map.Entry<Character, Long>> sortedByFrequency = charFrequency.entrySet()
            .stream()
            .sorted(Map.Entry.<Character, Long>comparingByValue().reversed())
            .collect(Collectors.toList());
        
        System.out.println("Characters by frequency: " + sortedByFrequency);
        
        System.out.println("\n--- Custom collector for min/max tracker ---");
        class MinMaxTracker {
            Integer min = null;
            Integer max = null;
            
            void accept(Integer value) {
                if (min == null || value < min) min = value;
                if (max == null || value > max) max = value;
            }
            
            MinMaxTracker combine(MinMaxTracker other) {
                if (other.min != null) {
                    if (min == null || other.min < min) min = other.min;
                }
                if (other.max != null) {
                    if (max == null || other.max > max) max = other.max;
                }
                return this;
            }
        }
        
        MinMaxTracker result = numbers.stream()
            .collect(
                MinMaxTracker::new,
                MinMaxTracker::accept,
                MinMaxTracker::combine
            );
        
        System.out.println("Min: " + result.min + ", Max: " + result.max);
    }
    
    public static void main(String[] args) {
        demonstrateFlatMapOperations();
        demonstrateAdvancedReductions();
        demonstratePartitioningGrouping();
        demonstrateCustomCollectors();
    }
}
```

### Parallel Streams and Performance

Understanding and leveraging parallel processing capabilities of streams.

```java
// Parallel streams and performance optimization
public class ParallelStreamsAndPerformance {
    
    // Parallel vs sequential performance comparison
    public static void demonstrateParallelVsSequential() {
        System.out.println("=== Parallel vs Sequential Streams ===");
        
        // Create a large dataset for comparison
        List<Integer> largeList = IntStream.range(0, 1_000_000)
            .boxed()
            .collect(Collectors.toList());
        
        System.out.println("\n--- Performance comparison ---");
        
        // Sequential processing
        long startTime = System.currentTimeMillis();
        long sequentialSum = largeList.stream()
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> (long) n * n)
            .sum();
        long sequentialTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Sequential time: " + sequentialTime + "ms, Sum: " + sequentialSum);
        
        // Parallel processing
        startTime = System.currentTimeMillis();
        long parallelSum = largeList.parallelStream()
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> (long) n * n)
            .sum();
        long parallelTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Parallel time: " + parallelTime + "ms, Sum: " + parallelSum);
        System.out.println("Speedup ratio: " + (double) sequentialTime / parallelTime + "x");
        
        System.out.println("\n--- When parallel streams work well ---");
        List<Integer> numbers = IntStream.range(0, 100_000)
            .boxed()
            .collect(Collectors.toList());
        
        // Computationally intensive tasks benefit from parallelization
        startTime = System.currentTimeMillis();
        List<Double> sequentialResults = numbers.stream()
            .map(n -> Math.sqrt(n) * Math.sin(n) + Math.cos(n))
            .collect(Collectors.toList());
        long computeSequentialTime = System.currentTimeMillis() - startTime;
        
        startTime = System.currentTimeMillis();
        List<Double> parallelResults = numbers.parallelStream()
            .map(n -> Math.sqrt(n) * Math.sin(n) + Math.cos(n))
            .collect(Collectors.toList());
        long computeParallelTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Computation - Sequential: " + computeSequentialTime + "ms");
        System.out.println("Computation - Parallel: " + computeParallelTime + "ms");
        System.out.println("Computation speedup: " + (double) computeSequentialTime / computeParallelTime + "x");
        
        System.out.println("\n--- When parallel streams don't help ---");
        List<String> smallList = Arrays.asList("apple", "banana", "cherry");
        
        startTime = System.currentTimeMillis();
        String sequentialJoin = smallList.stream()
            .collect(Collectors.joining(", "));
        long smallSequentialTime = System.currentTimeMillis() - startTime;
        
        startTime = System.currentTimeMillis();
        String parallelJoin = smallList.parallelStream()
            .collect(Collectors.joining(", "));
        long smallParallelTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Small dataset - Sequential: " + smallSequentialTime + "ms, Result: " + sequentialJoin);
        System.out.println("Small dataset - Parallel: " + smallParallelTime + "ms, Result: " + parallelJoin);
        System.out.println("Small dataset overhead: " + (double) smallParallelTime / smallSequentialTime + "x slower");
    }
    
    // Parallel processing with stateful operations
    public static void demonstrateParallelStatefulOperations() {
        System.out.println("\n=== Parallel Processing with Stateful Operations ===");
        
        List<Integer> numbers = IntStream.range(0, 1000)
            .boxed()
            .collect(Collectors.toList());
        
        System.out.println("\n--- Parallel processing with side effects (avoid this!) ---");
        List<Integer> results = new ArrayList<>();
        
        // This is problematic in parallel streams!
        numbers.parallelStream()
            .filter(n -> n % 2 == 0)
            .forEach(n -> results.add(n * 2)); // Side effect - not thread-safe!
        
        System.out.println("Results size (may be inconsistent): " + results.size());
        
        // Safe approach - collect to thread-safe structure or avoid side effects
        List<Integer> safeResults = numbers.parallelStream()
            .filter(n -> n % 2 == 0)
            .map(n -> n * 2)
            .collect(Collectors.toList());
        
        System.out.println("Safe parallel results size: " + safeResults.size());
        
        System.out.println("\n--- Parallel processing with ordered vs unordered ---");
        List<Integer> shuffledNumbers = IntStream.range(0, 100)
            .boxed()
            .collect(Collectors.collectingAndThen(Collectors.toList(), list -> {
                Collections.shuffle(list);
                return list;
            }));
        
        // Ordered parallel stream (maintains encounter order)
        startTime = System.currentTimeMillis();
        List<Integer> orderedResults = shuffledNumbers.parallelStream()
            .filter(n -> n % 2 == 0)
            .collect(Collectors.toList());
        long orderedTime = System.currentTimeMillis() - startTime;
        
        // Unordered parallel stream (better performance, no order guarantee)
        startTime = System.currentTimeMillis();
        List<Integer> unorderedResults = shuffledNumbers.parallelStream()
            .unordered() // Remove encounter order guarantee
            .filter(n -> n % 2 == 0)
            .collect(Collectors.toList());
        long unorderedTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Ordered parallel: " + orderedTime + "ms, size: " + orderedResults.size());
        System.out.println("Unordered parallel: " + unorderedTime + "ms, size: " + unorderedResults.size());
        
        // Note: Results may be different due to ordering, but sizes should be same
        Collections.sort(orderedResults);
        Collections.sort(unorderedResults);
        System.out.println("Results are equivalent: " + orderedResults.equals(unorderedResults));
    }
    
    // ForkJoinPool customization for parallel streams
    public static void demonstrateForkJoinPoolCustomization() {
        System.out.println("\n=== ForkJoinPool Customization ===");
        
        // Default parallelism
        System.out.println("Default parallelism: " + ForkJoinPool.getCommonPoolParallelism());
        
        // Custom ForkJoinPool for parallel streams
        ForkJoinPool customPool = new ForkJoinPool(2); // Use only 2 threads
        
        List<Integer> numbers = IntStream.range(0, 100_000)
            .boxed()
            .collect(Collectors.toList());
        
        long startTime = System.currentTimeMillis();
        long customPoolResult = customPool.submit(() -> 
            numbers.parallelStream()
                .filter(n -> n % 2 == 0)
                .mapToLong(n -> n * n)
                .sum()
        ).join();
        
        long customTime = System.currentTimeMillis() - startTime;
        System.out.println("Custom pool (2 threads) time: " + customTime + "ms");
        
        // Compare with default pool
        startTime = System.currentTimeMillis();
        long defaultPoolResult = numbers.parallelStream()
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> n * n)
            .sum();
        long defaultTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Default pool time: " + defaultTime + "ms");
        System.out.println("Results match: " + (customPoolResult == defaultPoolResult));
        
        customPool.shutdown();
    }
    
    // Performance optimization techniques
    public static void demonstratePerformanceOptimizations() {
        System.out.println("\n=== Parallel Stream Performance Optimizations ===");
        
        List<Integer> largeList = IntStream.range(0, 1_000_000)
            .boxed()
            .collect(Collectors.toList());
        
        System.out.println("\n--- Optimization 1: Order of operations ---");
        
        // Less efficient: transform then filter
        startTime = System.currentTimeMillis();
        long inefficient = largeList.parallelStream()
            .map(n -> n * n)  // Transform first
            .filter(n -> n > 100)  // Then filter
            .mapToLong(Long::longValue)
            .sum();
        long inefficientTime = System.currentTimeMillis() - startTime;
        
        // More efficient: filter then transform
        startTime = System.currentTimeMillis();
        long efficient = largeList.parallelStream()
            .filter(n -> n * n > 100)  // Filter first
            .mapToLong(n -> (long) n * n)  // Then transform
            .sum();
        long efficientTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Inefficient ordering: " + inefficientTime + "ms");
        System.out.println("Efficient ordering: " + efficientTime + "ms");
        System.out.println("Improvement: " + (double) inefficientTime / efficientTime + "x");
        
        System.out.println("\n--- Optimization 2: Using primitive streams ---");
        
        // Object stream (less efficient)
        startTime = System.currentTimeMillis();
        long objectStreamResult = largeList.parallelStream()
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> n * n)
            .sum();
        long objectStreamTime = System.currentTimeMillis() - startTime;
        
        // Primitive stream (more efficient)
        startTime = System.currentTimeMillis();
        long primitiveStreamResult = largeList.parallelStream()
            .mapToInt(Integer::intValue)
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> (long) n * n)
            .sum();
        long primitiveStreamTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Object stream: " + objectStreamTime + "ms");
        System.out.println("Primitive stream: " + primitiveStreamTime + "ms");
        System.out.println("Primitive improvement: " + (double) objectStreamTime / primitiveStreamTime + "x");
        
        System.out.println("\n--- Optimization 3: Choosing the right data structure ---");
        
        // ArrayList (good for parallel processing)
        List<Integer> arrayList = IntStream.range(0, 100_000)
            .boxed()
            .collect(Collectors.toCollection(ArrayList::new));
        
        startTime = System.currentTimeMillis();
        long arrayListResult = arrayList.parallelStream()
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> n * n)
            .sum();
        long arrayListTime = System.currentTimeMillis() - startTime;
        
        // LinkedList (not ideal for parallel processing)
        List<Integer> linkedList = IntStream.range(0, 100_000)
            .boxed()
            .collect(Collectors.toCollection(LinkedList::new));
        
        startTime = System.currentTimeMillis();
        long linkedListResult = linkedList.parallelStream()
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> n * n)
            .sum();
        long linkedListTime = System.currentTimeMillis() - startTime;
        
        System.out.println("ArrayList parallel: " + arrayListTime + "ms");
        System.out.println("LinkedList parallel: " + linkedListTime + "ms");
        System.out.println("ArrayList advantage: " + (double) linkedListTime / arrayListTime + "x");
    }
    
    // Parallel processing patterns and anti-patterns
    public static void demonstrateParallelPatterns() {
        System.out.println("\n=== Parallel Processing Patterns and Anti-Patterns ===");
        
        System.out.println("\n--- Pattern 1: Proper use of parallel streams ---");
        List<Double> largeDataSet = DoubleStream.generate(Math::random)
            .limit(1_000_000)
            .boxed()
            .collect(Collectors.toList());
        
        // Good: Computationally intensive operation
        long startTime = System.currentTimeMillis();
        List<Double> processed = largeDataSet.parallelStream()
            .map(x -> Math.pow(Math.sin(x), 2) + Math.pow(Math.cos(x), 2)) // Should be ~1.0
            .filter(x -> x > 0.99) // Keep values close to 1.0
            .collect(Collectors.toList());
        long goodPatternTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Good pattern time: " + goodPatternTime + "ms, Results: " + processed.size());
        
        System.out.println("\n--- Anti-pattern 1: Side effects in parallel streams ---");
        Map<String, Integer> sharedMap = new HashMap<>(); // Shared mutable state
        
        try {
            largeDataSet.parallelStream()
                .limit(10_000) // Limit for demo
                .forEach(n -> {
                    // This is dangerous! Multiple threads modifying sharedMap
                    String key = "key" + ((int)(n * 100) % 10);
                    sharedMap.merge(key, 1, Integer::sum);
                });
            System.out.println("Shared map size (may be incorrect): " + sharedMap.size());
        } catch (Exception e) {
            System.out.println("Error occurred: " + e.getMessage());
        }
        
        // Safe approach: Use thread-safe collections or avoid side effects
        Map<String, Long> safeMap = largeDataSet.parallelStream()
            .limit(10_000)
            .map(n -> "key" + ((int)(n * 100) % 10))
            .collect(Collectors.groupingBy(
                Function.identity(),
                Collectors.counting()
            ));
        System.out.println("Safe map size: " + safeMap.size());
        
        System.out.println("\n--- Pattern 2: Using parallel collectors ---");
        List<String> words = IntStream.range(0, 100_000)
            .mapToObj(i -> "word" + (i % 1000)) // Create many duplicates
            .collect(Collectors.toList());
        
        startTime = System.currentTimeMillis();
        Map<String, Long> wordCount = words.parallelStream()
            .collect(Collectors.groupingByConcurrent(Function.identity(), Collectors.counting()));
        long concurrentCollectTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Concurrent collect time: " + concurrentCollectTime + "ms");
        System.out.println("Unique words: " + wordCount.size());
    }
    
    private static long startTime;
    
    public static void main(String[] args) {
        demonstrateParallelVsSequential();
        demonstrateParallelStatefulOperations();
        demonstrateForkJoinPoolCustomization();
        demonstratePerformanceOptimizations();
        demonstrateParallelPatterns();
    }
}
```

## Collectors and Custom Collectors

Deep dive into the collectors framework and creating custom collectors for specialized operations.

### Built-in Collectors

```java
// Comprehensive collectors demonstration
public class BuiltInCollectors {
    
    // Basic collectors
    public static void demonstrateBasicCollectors() {
        System.out.println("=== Built-in Collectors ===");
        
        List<String> words = Arrays.asList("apple", "banana", "cherry", "date", "elderberry", "fig", "grape");
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        System.out.println("\n--- Basic Collection Collectors ---");
        
        // toList, toSet, toCollection
        List<String> wordList = words.stream().collect(Collectors.toList());
        Set<String> wordSet = words.stream().collect(Collectors.toSet());
        LinkedList<String> wordLinkedList = words.stream()
            .collect(Collectors.toCollection(LinkedList::new));
        
        System.out.println("List: " + wordList);
        System.out.println("Set: " + wordSet);
        System.out.println("LinkedList: " + wordLinkedList);
        
        System.out.println("\n--- Map Collectors ---");
        
        // toMap
        Map<String, Integer> wordLengthMap = words.stream()
            .collect(Collectors.toMap(
                Function.identity(),  // key: the word itself
                String::length        // value: the length of the word
            ));
        System.out.println("Word length map: " + wordLengthMap);
        
        // toMap with merge function (for handling duplicate keys)
        List<String> wordsWithDuplicates = Arrays.asList("apple", "banana", "apple", "cherry", "banana");
        Map<String, Integer> deduplicatedMap = wordsWithDuplicates.stream()
            .collect(Collectors.toMap(
                Function.identity(),           // key
                String::length,               // value
                (existing, replacement) -> existing  // merge function: keep existing
            ));
        System.out.println("Deduplicated map: " + deduplicatedMap);
        
        // toConcurrentMap
        ConcurrentMap<String, Integer> concurrentMap = words.stream()
            .collect(Collectors.toConcurrentMap(
                Function.identity(),
                String::length
            ));
        System.out.println("Concurrent map: " + concurrentMap);
        
        System.out.println("\n--- String Collectors ---");
        
        // joining
        String joined = words.stream().collect(Collectors.joining(", "));
        System.out.println("Joined: " + joined);
        
        String joinedWithPrefixSuffix = words.stream()
            .collect(Collectors.joining(", ", "[", "]"));
        System.out.println("Joined with prefix/suffix: " + joinedWithPrefixSuffix);
        
        System.out.println("\n--- Statistical Collectors ---");
        
        // counting
        long count = words.stream().collect(Collectors.counting());
        System.out.println("Count: " + count);
        
        // summingInt, summingLong, summingDouble
        int totalLength = words.stream()
            .collect(Collectors.summingInt(String::length));
        System.out.println("Total length: " + totalLength);
        
        // averagingInt, averagingLong, averagingDouble
        double averageLength = words.stream()
            .collect(Collectors.averagingInt(String::length));
        System.out.println("Average length: " + averageLength);
        
        // summarizingInt, summarizingLong, summarizingDouble
        IntSummaryStatistics lengthStats = words.stream()
            .collect(Collectors.summarizingInt(String::length));
        System.out.println("Length statistics: " + lengthStats);
        
        System.out.println("\n--- Partitioning and Grouping ---");
        
        // partitioningBy
        Map<Boolean, List<String>> partitioned = words.stream()
            .collect(Collectors.partitioningBy(s -> s.length() > 5));
        System.out.println("Partitioned by length > 5: " + partitioned);
        
        // groupingBy
        Map<Integer, List<String>> groupedByLength = words.stream()
            .collect(Collectors.groupingBy(String::length));
        System.out.println("Grouped by length: " + groupedByLength);
        
        // groupingBy with downstream collector
        Map<Integer, Long> lengthCount = words.stream()
            .collect(Collectors.groupingBy(
                String::length,
                Collectors.counting()
            ));
        System.out.println("Length count: " + lengthCount);
        
        Map<Integer, Set<String>> lengthUniqueWords = words.stream()
            .collect(Collectors.groupingBy(
                String::length,
                Collectors.toSet()
            ));
        System.out.println("Length to unique words: " + lengthUniqueWords);
    }
    
    // Advanced collectors with downstream operations
    public static void demonstrateAdvancedCollectors() {
        System.out.println("\n=== Advanced Collectors with Downstream Operations ===");
        
        class Person {
            String name;
            int age;
            String department;
            double salary;
            
            Person(String name, int age, String department, double salary) {
                this.name = name;
                this.age = age;
                this.department = department;
                this.salary = salary;
            }
            
            @Override
            public String toString() {
                return String.format("Person{name='%s', age=%d, dept='%s', salary=%.2f}", 
                                   name, age, department, salary);
            }
            
            public String getName() { return name; }
            public int getAge() { return age; }
            public String getDepartment() { return department; }
            public double getSalary() { return salary; }
        }
        
        List<Person> people = Arrays.asList(
            new Person("Alice", 30, "Engineering", 75000.0),
            new Person("Bob", 25, "Marketing", 65000.0),
            new Person("Charlie", 35, "Engineering", 85000.0),
            new Person("Diana", 28, "Sales", 70000.0),
            new Person("Eve", 32, "Engineering", 80000.0),
            new Person("Frank", 27, "Marketing", 60000.0),
            new Person("Grace", 29, "Sales", 72000.0),
            new Person("Henry", 31, "Engineering", 78000.0)
        );
        
        System.out.println("\n--- Advanced Grouping Operations ---");
        
        // Group by department and get average salary
        Map<String, Double> avgSalaryByDept = people.stream()
            .collect(Collectors.groupingBy(
                Person::getDepartment,
                Collectors.averagingDouble(Person::getSalary)
            ));
        System.out.println("Average salary by department: " + avgSalaryByDept);
        
        // Group by department and get sum of salaries
        Map<String, Double> totalSalaryByDept = people.stream()
            .collect(Collectors.groupingBy(
                Person::getDepartment,
                Collectors.summingDouble(Person::getSalary)
            ));
        System.out.println("Total salary by department: " + totalSalaryByDept);
        
        // Group by department and get statistics
        Map<String, DoubleSummaryStatistics> salaryStatsByDept = people.stream()
            .collect(Collectors.groupingBy(
                Person::getDepartment,
                Collectors.summarizingDouble(Person::getSalary)
            ));
        
        System.out.println("\nSalary statistics by department:");
        salaryStatsByDept.forEach((dept, stats) -> {
            System.out.printf("%s: count=%d, avg=%.2f, min=%.2f, max=%.2f%n",
                dept, stats.getCount(), stats.getAverage(), stats.getMin(), stats.getMax());
        });
        
        System.out.println("\n--- Nested Grouping ---");
        
        // Group by department, then by experience level
        Map<String, Map<String, List<Person>>> nestedGrouping = people.stream()
            .collect(Collectors.groupingBy(
                Person::getDepartment,
                Collectors.groupingBy(person -> 
                    person.getAge() < 30 ? "Junior" : "Senior")
            ));
        
        System.out.println("Nested grouping by department and experience:");
        nestedGrouping.forEach((dept, experienceGroups) -> {
            System.out.println(dept + ":");
            experienceGroups.forEach((level, personList) -> {
                System.out.println("  " + level + ": " + personList.size() + " people");
            });
        });
        
        System.out.println("\n--- Collecting with Filters and Transformations ---");
        
        // Group by department and get only names of people earning > 70000
        Map<String, List<String>> highEarnersByName = people.stream()
            .filter(person -> person.getSalary() > 70000)
            .collect(Collectors.groupingBy(
                Person::getDepartment,
                Collectors.mapping(Person::getName, Collectors.toList())
            ));
        System.out.println("High earners by department: " + highEarnersByName);
        
        // Group and get only the highest paid person in each department
        Map<String, Optional<Person>> highestPaidByDept = people.stream()
            .collect(Collectors.groupingBy(
                Person::getDepartment,
                Collectors.maxBy(Comparator.comparingDouble(Person::getSalary))
            ));
        
        System.out.println("\nHighest paid in each department:");
        highestPaidByDept.forEach((dept, personOpt) -> {
            if (personOpt.isPresent()) {
                System.out.println(dept + ": " + personOpt.get().getName() + 
                                 " ($" + personOpt.get().getSalary() + ")");
            }
        });
        
        System.out.println("\n--- Min/Max with Comparators ---");
        
        Optional<Person> youngest = people.stream()
            .collect(Collectors.minBy(Comparator.comparingInt(Person::getAge)));
        Optional<Person> oldest = people.stream()
            .collect(Collectors.maxBy(Comparator.comparingInt(Person::getAge)));
        
        System.out.println("Youngest: " + youngest.map(Person::getName).orElse("N/A"));
        System.out.println("Oldest: " + oldest.map(Person::getName).orElse("N/A"));
    }
    
    // Specialized collectors
    public static void demonstrateSpecializedCollectors() {
        System.out.println("\n=== Specialized Collectors ===");
        
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 2, 3, 4, 5);
        
        System.out.println("\n--- Filtering Collectors ---");
        
        // Filtering in downstream collector (Java 9+), for Java 8 we'll use a workaround
        Map<Boolean, List<Integer>> filteredGroups = numbers.stream()
            .collect(Collectors.partitioningBy(n -> n % 2 == 0));
        System.out.println("Even/Odd partition: " + filteredGroups);
        
        // Collecting and then filtering
        List<Integer> evenNumbers = numbers.stream()
            .collect(Collectors.filtering(n -> n % 2 == 0, Collectors.toList()));
        System.out.println("Even numbers: " + evenNumbers);
        
        System.out.println("\n--- Mapping Collectors ---");
        
        // Transform each element before collecting
        List<String> numberStrings = numbers.stream()
            .collect(Collectors.mapping(n -> "Number: " + n, Collectors.toList()));
        System.out.println("Mapped to strings: " + numberStrings);
        
        // Complex mapping with grouping
        Map<String, List<String>> groupedAsStrings = numbers.stream()
            .collect(Collectors.groupingBy(
                n -> n % 3 == 0 ? "MultipleOf3" : "NotMultipleOf3",
                Collectors.mapping(n -> "Val" + n, Collectors.toList())
            ));
        System.out.println("Grouped and mapped: " + groupedAsStrings);
        
        System.out.println("\n--- Teeing Collector (Java 12+) Simulation ---");
        
        // Simulate teeing behavior for Java 8/11
        class MinMaxAvg {
            int min;
            int max;
            double avg;
            
            MinMaxAvg(int min, int max, double avg) {
                this.min = min;
                this.max = max;
                this.avg = avg;
            }
            
            @Override
            public String toString() {
                return String.format("MinMaxAvg{min=%d, max=%d, avg=%.2f}", min, max, avg);
            }
        }
        
        Optional<MinMaxAvg> result = numbers.stream()
            .collect(Collectors.reducing(
                null,
                n -> new MinMaxAvg(n, n, n), // Create MinMaxAvg for single value
                (m1, m2) -> new MinMaxAvg(
                    Math.min(m1.min, m2.min),
                    Math.max(m1.max, m2.max),
                    (m1.avg * (m1.max >= m1.min ? 1 : 0) + m2.avg) / 2 // Simplified average
                )
            ));
        
        // Better approach for Java 8: do multiple operations
        IntSummaryStatistics stats = numbers.stream().mapToInt(Integer::intValue).summaryStatistics();
        MinMaxAvg calculated = new MinMaxAvg(stats.getMin(), stats.getMax(), stats.getAverage());
        System.out.println("Min/Max/Avg: " + calculated);
    }
    
    public static void main(String[] args) {
        demonstrateBasicCollectors();
        demonstrateAdvancedCollectors();
        demonstrateSpecializedCollectors();
    }
}
```

### Custom Collectors

```java
// Creating custom collectors for specialized operations
public class CustomCollectors {
    
    // Basic custom collector implementation
    public static void demonstrateBasicCustomCollector() {
        System.out.println("=== Basic Custom Collector ===");
        
        // Custom collector that creates a custom data structure
        class Statistics {
            private int count = 0;
            private long sum = 0;
            private int min = Integer.MAX_VALUE;
            private int max = Integer.MIN_VALUE;
            
            public void add(int value) {
                count++;
                sum += value;
                min = Math.min(min, value);
                max = Math.max(max, value);
            }
            
            public Statistics combine(Statistics other) {
                this.count += other.count;
                this.sum += other.sum;
                this.min = Math.min(this.min, other.min);
                this.max = Math.max(this.max, other.max);
                return this;
            }
            
            public double getAverage() {
                return count > 0 ? (double) sum / count : 0.0;
            }
            
            @Override
            public String toString() {
                return String.format("Statistics{count=%d, sum=%d, min=%d, max=%d, avg=%.2f}", 
                                   count, sum, min, max, getAverage());
            }
        }
        
        Collector<Integer, Statistics, Statistics> statisticsCollector = 
            Collector.of(
                Statistics::new,                    // Supplier: create new Statistics instance
                Statistics::add,                    // Accumulator: add value to statistics
                Statistics::combine,                // Combiner: combine two statistics
                Function.identity()                 // Finisher: return the result as-is
            );
        
        List<Integer> numbers = Arrays.asList(1, 5, 3, 9, 2, 8, 4, 7, 6);
        Statistics stats = numbers.stream().collect(statisticsCollector);
        System.out.println("Custom statistics: " + stats);
    }
    
    // Custom collector with complex state management
    public static void demonstrateComplexCustomCollector() {
        System.out.println("\n=== Complex Custom Collector ===");
        
        class FrequencyAnalyzer {
            private Map<String, Integer> frequencyMap = new HashMap<>();
            private int totalElements = 0;
            
            public void add(String element) {
                frequencyMap.merge(element, 1, Integer::sum);
                totalElements++;
            }
            
            public FrequencyAnalyzer combine(FrequencyAnalyzer other) {
                other.frequencyMap.forEach((key, value) -> 
                    this.frequencyMap.merge(key, value, Integer::sum));
                this.totalElements += other.totalElements;
                return this;
            }
            
            public Map<String, Double> getRelativeFrequencies() {
                Map<String, Double> relativeFreqs = new HashMap<>();
                for (Map.Entry<String, Integer> entry : frequencyMap.entrySet()) {
                    relativeFreqs.put(entry.getKey(), 
                        (double) entry.getValue() / totalElements);
                }
                return relativeFreqs;
            }
            
            public String getMostFrequent() {
                return frequencyMap.entrySet().stream()
                    .max(Map.Entry.comparingByValue())
                    .map(Map.Entry::getKey)
                    .orElse(null);
            }
            
            public int getTotalElements() { return totalElements; }
            public int getUniqueElements() { return frequencyMap.size(); }
        }
        
        Collector<String, FrequencyAnalyzer, FrequencyAnalyzer> frequencyCollector = 
            Collector.of(
                FrequencyAnalyzer::new,
                FrequencyAnalyzer::add,
                FrequencyAnalyzer::combine
            );
        
        List<String> words = Arrays.asList("apple", "banana", "apple", "cherry", "banana", "apple", "date");
        FrequencyAnalyzer analyzer = words.stream().collect(frequencyCollector);
        
        System.out.println("Frequency analysis:");
        System.out.println("  Total elements: " + analyzer.getTotalElements());
        System.out.println("  Unique elements: " + analyzer.getUniqueElements());
        System.out.println("  Most frequent: " + analyzer.getMostFrequent());
        System.out.println("  Relative frequencies: " + analyzer.getRelativeFrequencies());
    }
    
    // Custom collector for sliding window operations
    public static void demonstrateSlidingWindowCollector() {
        System.out.println("\n=== Sliding Window Custom Collector ===");
        
        class SlidingWindow<T> {
            private final int windowSize;
            private final Deque<T> window = new ArrayDeque<>();
            
            public SlidingWindow(int windowSize) {
                this.windowSize = windowSize;
            }
            
            public void add(T element) {
                window.addLast(element);
                if (window.size() > windowSize) {
                    window.removeFirst();
                }
            }
            
            public SlidingWindow<T> combine(SlidingWindow<T> other) {
                // For sliding windows, combination means taking the most recent elements
                // from both windows and keeping the most recent windowSize elements
                List<T> combined = new ArrayList<>();
                combined.addAll(other.window);
                combined.addAll(this.window);
                
                // Keep only the last windowSize elements
                int start = Math.max(0, combined.size() - windowSize);
                this.window.clear();
                for (int i = start; i < combined.size(); i++) {
                    this.window.addLast(combined.get(i));
                }
                
                return this;
            }
            
            public List<T> getElements() {
                return new ArrayList<>(window);
            }
            
            public int size() {
                return window.size();
            }
            
            public boolean isFull() {
                return window.size() == windowSize;
            }
        }
        
        int windowSize = 3;
        Collector<Integer, SlidingWindow<Integer>, List<Integer>> slidingWindowCollector = 
            Collector.of(
                () -> new SlidingWindow<Integer>(windowSize),
                SlidingWindow::add,
                SlidingWindow::combine,
                SlidingWindow::getElements
            );
        
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        // Process the stream and collect the final window
        List<Integer> finalWindow = numbers.stream().collect(slidingWindowCollector);
        System.out.println("Final sliding window: " + finalWindow);
        
        // To see all windows, we'd need a different approach
        System.out.println("Processing sequence to see all windows:");
        SlidingWindow<Integer> window = new SlidingWindow<>(windowSize);
        for (Integer num : numbers) {
            window.add(num);
            if (window.size() >= 1) { // Show all windows as they form
                System.out.println("Window with " + num + ": " + window.getElements());
            }
        }
    }
    
    // Custom collector for running calculations
    public static void demonstrateRunningCalculationsCollector() {
        System.out.println("\n=== Running Calculations Custom Collector ===");
        
        class RunningAverage {
            private double sum = 0.0;
            private int count = 0;
            
            public void add(double value) {
                sum += value;
                count++;
            }
            
            public RunningAverage combine(RunningAverage other) {
                this.sum += other.sum;
                this.count += other.count;
                return this;
            }
            
            public double getAverage() {
                return count > 0 ? sum / count : 0.0;
            }
            
            public int getCount() { return count; }
            public double getSum() { return sum; }
        }
        
        Collector<Double, RunningAverage, RunningAverage> runningAverageCollector = 
            Collector.of(
                RunningAverage::new,
                RunningAverage::add,
                RunningAverage::combine
            );
        
        List<Double> values = Arrays.asList(10.0, 20.0, 30.0, 40.0, 50.0);
        RunningAverage avgResult = values.stream().collect(runningAverageCollector);
        System.out.println("Running average result: " + avgResult.getAverage() + 
                         " (from " + avgResult.getCount() + " values, sum: " + avgResult.getSum() + ")");
        
        // Demonstrate with a more complex running calculation
        class RunningStatistics {
            private double sum = 0.0;
            private double sumOfSquares = 0.0;
            private int count = 0;
            private double min = Double.POSITIVE_INFINITY;
            private double max = Double.NEGATIVE_INFINITY;
            
            public void add(double value) {
                sum += value;
                sumOfSquares += value * value;
                count++;
                min = Math.min(min, value);
                max = Math.max(max, value);
            }
            
            public RunningStatistics combine(RunningStatistics other) {
                this.sum += other.sum;
                this.sumOfSquares += other.sumOfSquares;
                this.count += other.count;
                this.min = Math.min(this.min, other.min);
                this.max = Math.max(this.max, other.max);
                return this;
            }
            
            public double getMean() { return count > 0 ? sum / count : 0.0; }
            public double getVariance() { 
                if (count <= 1) return 0.0;
                double mean = getMean();
                return (sumOfSquares / count) - (mean * mean);
            }
            public double getStandardDeviation() { return Math.sqrt(getVariance()); }
            public double getMin() { return min; }
            public double getMax() { return max; }
            public int getCount() { return count; }
        }
        
        Collector<Double, RunningStatistics, RunningStatistics> statsCollector = 
            Collector.of(
                RunningStatistics::new,
                RunningStatistics::add,
                RunningStatistics::combine
            );
        
        RunningStatistics stats = values.stream().collect(statsCollector);
        System.out.println("Running statistics:");
        System.out.println("  Mean: " + stats.getMean());
        System.out.println("  Standard Deviation: " + stats.getStandardDeviation());
        System.out.println("  Min: " + stats.getMin());
        System.out.println("  Max: " + stats.getMax());
    }
    
    // Performance-optimized custom collector
    public static void demonstratePerformanceOptimizedCollector() {
        System.out.println("\n=== Performance-Optimized Custom Collector ===");
        
        // Custom collector that's optimized for performance
        class OptimizedStringBuilder {
            private final List<String> parts = new ArrayList<>();
            private int totalLength = 0;
            private final String delimiter;
            
            public OptimizedStringBuilder(String delimiter) {
                this.delimiter = delimiter;
            }
            
            public void add(String element) {
                parts.add(element);
                totalLength += element.length();
                if (!parts.isEmpty()) {
                    totalLength += delimiter.length(); // Account for delimiter
                }
            }
            
            public OptimizedStringBuilder combine(OptimizedStringBuilder other) {
                this.parts.addAll(other.parts);
                this.totalLength += other.totalLength;
                return this;
            }
            
            public String build() {
                if (parts.isEmpty()) return "";
                
                // Pre-calculate final string length for efficiency
                StringBuilder sb = new StringBuilder(totalLength);
                for (int i = 0; i < parts.size(); i++) {
                    if (i > 0) sb.append(delimiter);
                    sb.append(parts.get(i));
                }
                return sb.toString();
            }
        }
        
        Collector<String, OptimizedStringBuilder, String> optimizedJoiningCollector = 
            Collector.of(
                () -> new OptimizedStringBuilder(", "),
                OptimizedStringBuilder::add,
                OptimizedStringBuilder::combine,
                OptimizedStringBuilder::build
            );
        
        List<String> words = Arrays.asList("apple", "banana", "cherry", "date", "elderberry");
        
        // Compare with built-in joining
        long startTime = System.nanoTime();
        String builtInResult = words.stream().collect(Collectors.joining(", "));
        long builtInTime = System.nanoTime() - startTime;
        
        startTime = System.nanoTime();
        String customResult = words.stream().collect(optimizedJoiningCollector);
        long customTime = System.nanoTime() - startTime;
        
        System.out.println("Built-in joining: " + builtInResult);
        System.out.println("Custom optimized: " + customResult);
        System.out.println("Built-in time: " + builtInTime / 1000 + " μs");
        System.out.println("Custom time: " + customTime / 1000 + " μs");
        
        // Custom collector for specific use case: accumulating unique elements
        // preserving order but with better performance than distinct()
        class UniqueAccumulator<T> {
            private final Set<T> seen = new HashSet<>();
            private final List<T> uniqueList = new ArrayList<>();
            
            public void add(T element) {
                if (seen.add(element)) { // add returns false if already present
                    uniqueList.add(element);
                }
            }
            
            public UniqueAccumulator<T> combine(UniqueAccumulator<T> other) {
                for (T element : other.uniqueList) {
                    if (seen.add(element)) {
                        uniqueList.add(element);
                    }
                }
                return this;
            }
            
            public List<T> getUniqueList() {
                return uniqueList;
            }
        }
        
        Collector<Integer, UniqueAccumulator<Integer>, List<Integer>> uniqueCollector = 
            Collector.of(
                UniqueAccumulator::new,
                UniqueAccumulator::add,
                UniqueAccumulator::combine,
                UniqueAccumulator::getUniqueList
            );
        
        List<Integer> withDuplicates = Arrays.asList(1, 2, 2, 3, 4, 4, 5, 1, 6);
        List<Integer> uniqueResult = withDuplicates.stream().collect(uniqueCollector);
        System.out.println("Unique elements (preserving order): " + uniqueResult);
    }
    
    public static void main(String[] args) {
        demonstrateBasicCustomCollector();
        demonstrateComplexCustomCollector();
        demonstrateSlidingWindowCollector();
        demonstrateRunningCalculationsCollector();
        demonstratePerformanceOptimizedCollector();
    }
}
```

## Functional Design Patterns

Applying functional programming principles to implement common design patterns.

### Functional Implementation of Design Patterns

```java
// Functional implementations of common design patterns
public class FunctionalDesignPatterns {
    
    // Strategy pattern with functions
    public static void demonstrateFunctionalStrategy() {
        System.out.println("=== Functional Strategy Pattern ===");
        
        // Traditional Strategy pattern uses classes; with functions, it's much simpler
        class ShoppingCart {
            List<Item> items = new ArrayList<>();
            
            void addItem(Item item) {
                items.add(item);
            }
            
            double calculateTotal(Function<List<Item>, Double> pricingStrategy) {
                return pricingStrategy.apply(items);
            }
        }
        
        class Item {
            String name;
            double price;
            boolean taxable;
            
            Item(String name, double price, boolean taxable) {
                this.name = name;
                this.price = price;
                this.taxable = taxable;
            }
            
            double getPrice() { return price; }
            boolean isTaxable() { return taxable; }
        }
        
        // Different pricing strategies as functions
        Function<List<Item>, Double> regularPricing = items -> 
            items.stream().mapToDouble(Item::getPrice).sum();
        
        Function<List<Item>, Double> taxPricing = items -> {
            double subtotal = items.stream()
                .filter(Item::isTaxable)
                .mapToDouble(Item::getPrice)
                .sum();
            return subtotal * 1.08; // 8% tax
        };
        
        Function<List<Item>, Double> bulkDiscountPricing = items -> {
            double total = items.stream().mapToDouble(Item::getPrice).sum();
            return items.size() > 5 ? total * 0.9 : total; // 10% discount for 6+ items
        };
        
        ShoppingCart cart = new ShoppingCart();
        cart.addItem(new Item("Laptop", 1000.0, true));
        cart.addItem(new Item("Mouse", 25.0, true));
        cart.addItem(new Item("Book", 20.0, false));
        
        System.out.println("Regular pricing: $" + String.format("%.2f", cart.calculateTotal(regularPricing)));
        System.out.println("Tax pricing: $" + String.format("%.2f", cart.calculateTotal(taxPricing)));
        System.out.println("Bulk discount pricing: $" + String.format("%.2f", cart.calculateTotal(bulkDiscountPricing)));
        
        // Strategy selection based on conditions
        Function<List<Item>, Double> dynamicPricing = items -> {
            if (items.size() > 10) {
                return bulkDiscountPricing.apply(items);
            } else if (items.stream().anyMatch(Item::isTaxable)) {
                return taxPricing.apply(items);
            } else {
                return regularPricing.apply(items);
            }
        };
        
        System.out.println("Dynamic pricing: $" + String.format("%.2f", cart.calculateTotal(dynamicPricing)));
    }
    
    // Template Method pattern with functions
    public static void demonstrateTemplateMethod() {
        System.out.println("\n=== Functional Template Method Pattern ===");
        
        class DataProcessor {
            public void process(String data, 
                              Function<String, String> validator,
                              Function<String, String> transformer,
                              Consumer<String> resultHandler) {
                
                // Template: validate -> transform -> handle
                String validatedData = validator.apply(data);
                String transformedData = transformer.apply(validatedData);
                resultHandler.accept(transformedData);
            }
        }
        
        // Different validation functions
        Function<String, String> emailValidator = email -> {
            if (!email.contains("@")) throw new IllegalArgumentException("Invalid email: " + email);
            return email.trim();
        };
        
        Function<String, String> lengthValidator = str -> {
            if (str.length() < 3) throw new IllegalArgumentException("Too short: " + str);
            return str;
        };
        
        // Different transformation functions
        Function<String, String> toUpperCase = String::toUpperCase;
        Function<String, String> reverse = s -> new StringBuilder(s).reverse().toString();
        
        // Different handling functions
        Consumer<String> consoleHandler = System.out::println;
        Consumer<String, String> emailHandler = result -> {
            System.out.println("Sending result via email: " + result);
        };
        
        DataProcessor processor = new DataProcessor();
        
        try {
            processor.process("hello@world.com", 
                            emailValidator, toUpperCase, consoleHandler);
            
            processor.process("sample data", 
                            lengthValidator, reverse, consoleHandler);
        } catch (IllegalArgumentException e) {
            System.out.println("Validation error: " + e.getMessage());
        }
    }
    
    // Observer pattern with functions
    public static void demonstrateFunctionalObserver() {
        System.out.println("\n=== Functional Observer Pattern ===");
        
        class EventPublisher<T> {
            private final List<Consumer<T>> observers = new ArrayList<>();
            
            public void addObserver(Consumer<T> observer) {
                observers.add(observer);
            }
            
            public void removeObserver(Consumer<T> observer) {
                observers.remove(observer);
            }
            
            public void notifyObservers(T event) {
                observers.forEach(observer -> {
                    try {
                        observer.accept(event);
                    } catch (Exception e) {
                        System.err.println("Observer threw exception: " + e.getMessage());
                    }
                });
            }
        }
        
        class StockPrice {
            String symbol;
            double price;
            
            StockPrice(String symbol, double price) {
                this.symbol = symbol;
                this.price = price;
            }
            
            @Override
            public String toString() {
                return symbol + ": $" + price;
            }
        }
        
        EventPublisher<StockPrice> pricePublisher = new EventPublisher<>();
        
        // Different observer functions
        Consumer<StockPrice> consoleObserver = price -> 
            System.out.println("Console: Stock price updated - " + price);
        
        Consumer<StockPrice> thresholdObserver = price -> {
            if (price.price > 200) {
                System.out.println("Alert: High value stock! " + price);
            }
        };
        
        Consumer<StockPrice> logObserver = price -> {
            // In real application, this would log to a file
            System.out.println("Log: " + java.time.LocalDateTime.now() + " - " + price);
        };
        
        pricePublisher.addObserver(consoleObserver);
        pricePublisher.addObserver(thresholdObserver);
        pricePublisher.addObserver(logObserver);
        
        // Simulate price updates
        pricePublisher.notifyObservers(new StockPrice("AAPL", 175.50));
        pricePublisher.notifyObservers(new StockPrice("GOOGL", 2800.00));
        pricePublisher.notifyObservers(new StockPrice("TSLA", 250.75));
    }
    
    // Chain of Responsibility pattern with functions
    public static void demonstrateChainOfResponsibility() {
        System.out.println("\n=== Functional Chain of Responsibility Pattern ===");
        
        class ProcessingResult {
            boolean processed;
            String message;
            Object data;
            
            ProcessingResult(boolean processed, String message, Object data) {
                this.processed = processed;
                this.message = message;
                this.data = data;
            }
            
            @Override
            public String toString() {
                return "ProcessingResult{processed=" + processed + ", message='" + message + "'}";
            }
        }
        
        class Request {
            String type;
            Object payload;
            
            Request(String type, Object payload) {
                this.type = type;
                this.payload = payload;
            }
        }
        
        // Function type for processing requests
        interface RequestHandler extends Function<Request, Optional<ProcessingResult>> {}
        
        // Create a chain of handlers
        class HandlerChain {
            private final List<RequestHandler> handlers = new ArrayList<>();
            
            public HandlerChain addHandler(RequestHandler handler) {
                handlers.add(handler);
                return this;
            }
            
            public ProcessingResult process(Request request) {
                for (RequestHandler handler : handlers) {
                    Optional<ProcessingResult> result = handler.apply(request);
                    if (result.isPresent() && result.get().processed) {
                        return result.get();
                    }
                }
                return new ProcessingResult(false, "No handler processed the request", null);
            }
        }
        
        // Different handlers as functions
        RequestHandler authHandler = request -> {
            if ("user".equals(request.type)) {
                System.out.println("Authentication handler processing user request");
                return Optional.of(new ProcessingResult(true, "User authenticated", request.payload));
            }
            return Optional.empty(); // Not applicable
        };
        
        RequestHandler validationHandler = request -> {
            if ("user".equals(request.type) && request.payload.toString().length() > 3) {
                System.out.println("Validation handler: user name is valid");
                return Optional.of(new ProcessingResult(true, "User data is valid", request.payload));
            }
            return Optional.empty(); // Not applicable
        };
        
        RequestHandler businessHandler = request -> {
            if ("user".equals(request.type)) {
                System.out.println("Business handler creating user account");
                return Optional.of(new ProcessingResult(true, "User account created", request.payload));
            }
            return Optional.empty(); // Not applicable
        };
        
        RequestHandler fallbackHandler = request -> {
            System.out.println("Fallback handler processing: " + request.type);
            return Optional.of(new ProcessingResult(true, "Request processed by fallback", request.payload));
        };
        
        HandlerChain chain = new HandlerChain()
            .addHandler(authHandler)
            .addHandler(validationHandler)
            .addHandler(businessHandler)
            .addHandler(fallbackHandler);
        
        // Process different requests
        System.out.println("Processing user request:");
        ProcessingResult result1 = chain.process(new Request("user", "john_doe"));
        System.out.println("Result: " + result1);
        
        System.out.println("\nProcessing unknown request:");
        ProcessingResult result2 = chain.process(new Request("unknown", "data"));
        System.out.println("Result: " + result2);
    }
    
    // State pattern with functions
    public static void demonstrateFunctionalState() {
        System.out.println("\n=== Functional State Pattern ===");
        
        class Order {
            private String state;
            private final Map<String, Consumer<Order>> stateActions = new HashMap<>();
            
            public Order(String initialState) {
                this.state = initialState;
            }
            
            public void setState(String newState) {
                this.state = newState;
            }
            
            public void executeCurrentStateAction() {
                Consumer<Order> action = stateActions.get(state);
                if (action != null) {
                    action.accept(this);
                } else {
                    System.out.println("No action defined for state: " + state);
                }
            }
            
            public void registerStateAction(String state, Consumer<Order> action) {
                stateActions.put(state, action);
            }
            
            public String getState() { return state; }
        }
        
        Order order = new Order("CREATED");
        
        // Define actions for different states
        order.registerStateAction("CREATED", o -> {
            System.out.println("Order " + o + " is ready to be processed");
            o.setState("PROCESSING");
        });
        
        order.registerStateAction("PROCESSING", o -> {
            System.out.println("Order " + o + " is being processed");
            o.setState("SHIPPED");
        });
        
        order.registerStateAction("SHIPPED", o -> {
            System.out.println("Order " + o + " has been shipped");
            o.setState("DELIVERED");
        });
        
        order.registerStateAction("DELIVERED", o -> {
            System.out.println("Order " + o + " is complete");
        });
        
        // Execute the state transitions
        for (int i = 0; i < 5; i++) {
            order.executeCurrentStateAction();
            System.out.println("Current state: " + order.getState());
        }
    }
    
    // Command pattern with functions
    public static void demonstrateCommandPattern() {
        System.out.println("\n=== Functional Command Pattern ===");
        
        class CommandProcessor {
            private final Deque<Runnable> commandHistory = new ArrayDeque<>();
            
            public void executeCommand(Runnable command) {
                command.run();
                commandHistory.push(command);
            }
            
            public void undoLastCommand(Consumer<Runnable> undoHandler) {
                if (!commandHistory.isEmpty()) {
                    Runnable lastCommand = commandHistory.pop();
                    undoHandler.accept(lastCommand);
                }
            }
        }
        
        class BankAccount {
            private double balance = 0.0;
            
            public void deposit(double amount) {
                balance += amount;
                System.out.println("Deposited: $" + amount + ", Balance: $" + balance);
            }
            
            public void withdraw(double amount) {
                if (amount <= balance) {
                    balance -= amount;
                    System.out.println("Withdrew: $" + amount + ", Balance: $" + balance);
                } else {
                    System.out.println("Insufficient funds");
                }
            }
            
            public double getBalance() { return balance; }
        }
        
        BankAccount account = new BankAccount();
        CommandProcessor processor = new CommandProcessor();
        
        // Commands as lambda expressions
        Runnable depositCommand = () -> account.deposit(100.0);
        Runnable withdrawCommand = () -> account.withdraw(50.0);
        Runnable largeDeposit = () -> account.deposit(1000.0);
        
        processor.executeCommand(depositCommand);
        processor.executeCommand(withdrawCommand);
        processor.executeCommand(largeDeposit);
        
        // Undo functionality
        Consumer<Runnable> undoHandler = command -> {
            System.out.println("Undoing last command...");
            // In a real implementation, you'd need command objects that know how to undo
            // For this example, we'll just print a message
            System.out.println("Command undone");
        };
        
        processor.undoLastCommand(undoHandler);
    }
    
    public static void main(String[] args) {
        demonstrateFunctionalStrategy();
        demonstrateTemplateMethod();
        demonstrateFunctionalObserver();
        demonstrateChainOfResponsibility();
        demonstrateFunctionalState();
        demonstrateCommandPattern();
    }
}
```

## Performance Optimization Techniques

Advanced techniques for optimizing functional programming and stream operations.

### Performance Optimization Strategies

```java
// Performance optimization techniques for functional programming
public class PerformanceOptimizationTechniques {
    
    // Stream operation optimization
    public static void demonstrateStreamOptimizations() {
        System.out.println("=== Stream Operation Optimizations ===");
        
        List<Integer> largeList = IntStream.range(0, 1_000_000)
            .boxed()
            .collect(Collectors.toList());
        
        System.out.println("\n--- Optimization 1: Early filtering ---");
        
        // Inefficient: transform first, then filter
        long startTime = System.nanoTime();
        long inefficientCount = largeList.stream()
            .map(n -> n * 2)              // Transform first (work on all elements)
            .filter(n -> n > 1000)        // Then filter
            .count();
        long inefficientTime = System.nanoTime() - startTime;
        
        // Efficient: filter first, then transform
        startTime = System.nanoTime();
        long efficientCount = largeList.stream()
            .filter(n -> n * 2 > 1000)    // Filter first (fewer elements to process)
            .map(n -> n * 2)              // Then transform
            .count();
        long efficientTime = System.nanoTime() - startTime;
        
        System.out.println("Inefficient approach: " + inefficientTime / 1_000_000 + "ms, Count: " + inefficientCount);
        System.out.println("Efficient approach: " + efficientTime / 1_000_000 + "ms, Count: " + efficientCount);
        System.out.println("Improvement: " + (double) inefficientTime / efficientTime + "x");
        
        System.out.println("\n--- Optimization 2: Using primitive streams ---");
        
        // Object stream (boxing/unboxing overhead)
        startTime = System.nanoTime();
        long objectStreamSum = largeList.stream()
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> (long) n * n)
            .sum();
        long objectStreamTime = System.nanoTime() - startTime;
        
        // Primitive stream (more efficient)
        startTime = System.nanoTime();
        long primitiveStreamSum = largeList.stream()
            .mapToInt(Integer::intValue)
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> (long) n * n)
            .sum();
        long primitiveStreamTime = System.nanoTime() - startTime;
        
        System.out.println("Object stream: " + objectStreamTime / 1_000_000 + "ms");
        System.out.println("Primitive stream: " + primitiveStreamTime / 1_000_000 + "ms");
        System.out.println("Primitive improvement: " + (double) objectStreamTime / primitiveStreamTime + "x");
        
        System.out.println("\n--- Optimization 3: Avoiding intermediate collections ---");
        
        List<String> words = Arrays.asList("apple", "banana", "cherry", "date", "elderberry", 
                                         "fig", "grape", "honeydew", "kiwi", "lemon");
        
        // Creating intermediate collections (less efficient)
        startTime = System.nanoTime();
        List<String> inefficientResult = words.stream()
            .filter(s -> s.length() > 4)
            .collect(Collectors.toList())    // Create intermediate list
            .stream()                        // Create another stream
            .map(String::toUpperCase)
            .collect(Collectors.toList());
        long inefficientCollectionTime = System.nanoTime() - startTime;
        
        // Direct pipeline (more efficient)
        startTime = System.nanoTime();
        List<String> efficientResult = words.stream()
            .filter(s -> s.length() > 4)
            .map(String::toUpperCase)
            .collect(Collectors.toList());
        long efficientCollectionTime = System.nanoTime() - startTime;
        
        System.out.println("With intermediate collection: " + inefficientCollectionTime / 1_000 + " μs");
        System.out.println("Direct pipeline: " + efficientCollectionTime / 1_000 + " μs");
        System.out.println("Direct pipeline improvement: " + 
                         (double) inefficientCollectionTime / efficientCollectionTime + "x");
    }
    
    // Memory optimization techniques
    public static void demonstrateMemoryOptimizations() {
        System.out.println("\n=== Memory Optimization Techniques ===");
        
        System.out.println("\n--- Optimization 1: Choosing appropriate data structures ---");
        
        // Testing different approaches for large datasets
        List<Integer> numbers = IntStream.range(0, 100_000)
            .boxed()
            .collect(Collectors.toList());
        
        // ArrayList (good for random access, efficient for streams)
        startTime = System.nanoTime();
        long arrayListResult = new ArrayList<>(numbers).stream()
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> n * n)
            .sum();
        long arrayListTime = System.nanoTime() - startTime;
        
        // LinkedList (not efficient for streams due to lack of random access)
        startTime = System.nanoTime();
        long linkedListResult = new LinkedList<>(numbers).stream()
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> n * n)
            .sum();
        long linkedListTime = System.nanoTime() - startTime;
        
        System.out.println("ArrayList processing: " + arrayListTime / 1_000_000 + "ms");
        System.out.println("LinkedList processing: " + linkedListTime / 1_000_000 + "ms");
        System.out.println("ArrayList advantage: " + (double) linkedListTime / arrayListResult + "x");
        
        System.out.println("\n--- Optimization 2: Using appropriate collectors ---");
        
        startTime = System.nanoTime();
        // Using toList collector
        List<String> listResult = numbers.stream()
            .limit(10_000)
            .map(String::valueOf)
            .collect(Collectors.toList());
        long toListTime = System.nanoTime() - startTime;
        
        startTime = System.nanoTime();
        // Using toCollection with ArrayList
        List<String> arrayListResult = numbers.stream()
            .limit(10_000)
            .map(String::valueOf)
            .collect(Collectors.toCollection(ArrayList::new));
        long toCollectionTime = System.nanoTime() - startTime;
        
        System.out.println("toList collector: " + toListTime / 1_000 + " μs");
        System.out.println("toCollection collector: " + toCollectionTime / 1_000 + " μs");
        
        System.out.println("\n--- Optimization 3: Lazy evaluation and short-circuiting ---");
        
        List<Integer> testNumbers = IntStream.range(0, 1_000_000)
            .boxed()
            .collect(Collectors.toList());
        
        // Non-short-circuiting (evaluates entire stream)
        startTime = System.nanoTime();
        boolean hasLargeNumber1 = testNumbers.stream()
            .map(n -> n * n)              // This processes ALL elements
            .anyMatch(squared -> squared > 1_000_000_000);
        long nonShortCircuitTime = System.nanoTime() - startTime;
        
        // Short-circuiting (stops as soon as condition is met)
        startTime = System.nanoTime();
        boolean hasLargeNumber2 = testNumbers.stream()
            .anyMatch(n -> n * n > 1_000_000_000);  // This can short-circuit
        long shortCircuitTime = System.nanoTime() - startTime;
        
        System.out.println("Non-short-circuiting: " + nonShortCircuitTime / 1_000_000 + "ms");
        System.out.println("Short-circuiting: " + shortCircuitTime / 1_000_000 + "ms");
        System.out.println("Short-circuiting improvement: " + 
                         (double) nonShortCircuitTime / shortCircuitTime + "x");
    }
    
    // Function composition optimization
    public static void demonstrateFunctionOptimization() {
        System.out.println("\n=== Function Composition Optimizations ===");
        
        List<Integer> numbers = IntStream.range(1, 100_000)
            .boxed()
            .collect(Collectors.toList());
        
        System.out.println("\n--- Optimization 1: Minimizing function composition ---");
        
        // Inefficient: multiple separate maps
        Function<Integer, Integer> times2 = n -> n * 2;
        Function<Integer, Integer> plus5 = n -> n + 5;
        Function<Integer, Integer> squared = n -> n * n;
        
        startTime = System.nanoTime();
        List<Integer> inefficient = numbers.stream()
            .map(times2)
            .map(plus5)
            .map(squared)
            .collect(Collectors.toList());
        long multipleMapsTime = System.nanoTime() - startTime;
        
        // Efficient: composed function
        Function<Integer, Integer> composed = times2
            .andThen(plus5)
            .andThen(squared);
        
        startTime = System.nanoTime();
        List<Integer> efficient = numbers.stream()
            .map(composed)
            .collect(Collectors.toList());
        long composedTime = System.nanoTime() - startTime;
        
        System.out.println("Multiple maps: " + multipleMapsTime / 1_000_000 + "ms");
        System.out.println("Composed function: " + composedTime / 1_000_000 + "ms");
        System.out.println("Composition improvement: " + (double) multipleMapsTime / composedTime + "x");
        
        System.out.println("\n--- Optimization 2: Using method references instead of lambdas ---");
        
        // Lambda (creates new function object each time)
        startTime = System.nanoTime();
        List<String> lambdaResult = numbers.stream()
            .filter(n -> n % 2 == 0)
            .map(n -> String.valueOf(n))
            .limit(1000)
            .collect(Collectors.toList());
        long lambdaTime = System.nanoTime() - startTime;
        
        // Method reference (reuses the same method)
        startTime = System.nanoTime();
        List<String> methodRefResult = numbers.stream()
            .filter(n -> n % 2 == 0)
            .map(String::valueOf)
            .limit(1000)
            .collect(Collectors.toList());
        long methodRefTime = System.nanoTime() - startTime;
        
        System.out.println("Lambda approach: " + lambdaTime / 1_000 + " μs");
        System.out.println("Method reference: " + methodRefTime / 1_000 + " μs");
        
        System.out.println("\n--- Optimization 3: Caching expensive function results ---");
        
        // Expensive function that would benefit from caching
        Function<Integer, Double> expensiveFunction = n -> {
            // Simulate expensive computation
            double result = 0;
            for (int i = 0; i < 1000; i++) {
                result += Math.sqrt(n * i);
            }
            return result;
        };
        
        // Without caching
        startTime = System.nanoTime();
        List<Integer> testList = Arrays.asList(1, 2, 1, 3, 2, 4, 1, 5, 2);
        List<Double> withoutCache = testList.stream()
            .map(expensiveFunction)
            .collect(Collectors.toList());
        long withoutCacheTime = System.nanoTime() - startTime;
        
        // With caching using memoization
        Function<Integer, Double> cachedFunction = memoize(expensiveFunction);
        
        startTime = System.nanoTime();
        List<Double> withCache = testList.stream()
            .map(cachedFunction)
            .collect(Collectors.toList());
        long withCacheTime = System.nanoTime() - startTime;
        
        System.out.println("Without cache: " + withoutCacheTime / 1_000_000 + "ms");
        System.out.println("With cache: " + withCacheTime / 1_000_000 + "ms");
        System.out.println("Caching improvement: " + (double) withoutCacheTime / withCacheTime + "x");
    }
    
    // Memoization utility for function caching
    private static <T, R> Function<T, R> memoize(Function<T, R> function) {
        Map<T, R> cache = new ConcurrentHashMap<>();
        return input -> cache.computeIfAbsent(input, function);
    }
    
    // Parallel stream optimization
    public static void demonstrateParallelOptimizations() {
        System.out.println("\n=== Parallel Stream Optimizations ===");
        
        System.out.println("\n--- Optimization 1: Understanding when to use parallel streams ---");
        
        List<Integer> smallList = IntStream.range(0, 100).boxed().collect(Collectors.toList());
        List<Integer> largeList = IntStream.range(0, 1_000_000).boxed().collect(Collectors.toList());
        
        // Small dataset - parallel overhead
        long startTime = System.nanoTime();
        long smallSequential = smallList.stream().mapToLong(n -> n * n).sum();
        long smallSeqTime = System.nanoTime() - startTime;
        
        startTime = System.nanoTime();
        long smallParallel = smallList.parallelStream().mapToLong(n -> n * n).sum();
        long smallParallelTime = System.nanoTime() - startTime;
        
        System.out.println("Small list - Sequential: " + smallSeqTime / 1_000 + " μs");
        System.out.println("Small list - Parallel: " + smallParallelTime / 1_000 + " μs");
        System.out.println("Small list overhead: " + (double) smallParallelTime / smallSeqTime + "x slower");
        
        // Large dataset - parallel benefit
        startTime = System.nanoTime();
        long largeSequential = largeList.stream().mapToLong(n -> n * n).sum();
        long largeSeqTime = System.nanoTime() - startTime;
        
        startTime = System.nanoTime();
        long largeParallel = largeList.parallelStream().mapToLong(n -> n * n).sum();
        long largeParallelTime = System.nanoTime() - startTime;
        
        System.out.println("Large list - Sequential: " + largeSeqTime / 1_000_000 + " ms");
        System.out.println("Large list - Parallel: " + largeParallelTime / 1_000_000 + " ms");
        System.out.println("Large list speedup: " + (double) largeSeqTime / largeParallelTime + "x");
        
        System.out.println("\n--- Optimization 2: Choosing operations that parallelize well ---");
        
        // Computationally intensive operation - good for parallelization
        List<Double> doubles = DoubleStream.generate(Math::random)
            .limit(100_000)
            .boxed()
            .collect(Collectors.toList());
        
        startTime = System.nanoTime();
        double computeIntensiveSeq = doubles.stream()
            .map(d -> Math.pow(Math.sin(d), 2) + Math.pow(Math.cos(d), 2))
            .mapToDouble(Double::doubleValue)
            .sum();
        long computeIntensiveSeqTime = System.nanoTime() - startTime;
        
        startTime = System.nanoTime();
        double computeIntensivePar = doubles.parallelStream()
            .map(d -> Math.pow(Math.sin(d), 2) + Math.pow(Math.cos(d), 2))
            .mapToDouble(Double::doubleValue)
            .sum();
        long computeIntensiveParTime = System.nanoTime() - startTime;
        
        System.out.println("Compute-intensive - Sequential: " + computeIntensiveSeqTime / 1_000_000 + " ms");
        System.out.println("Compute-intensive - Parallel: " + computeIntensiveParTime / 1_000_000 + " ms");
        System.out.println("Compute-intensive speedup: " + 
                         (double) computeIntensiveSeqTime / computeIntensiveParTime + "x");
        
        System.out.println("\n--- Optimization 3: Reducing parallel stream overhead ---");
        
        // Using appropriate data structures for parallel processing
        ArrayList<Integer> arrayList = new ArrayList<>(largeList);
        LinkedList<Integer> linkedList = new LinkedList<>(largeList);
        
        startTime = System.nanoTime();
        long arrayListParallel = arrayList.parallelStream()
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> n * n)
            .sum();
        long arrayListParTime = System.nanoTime() - startTime;
        
        startTime = System.nanoTime();
        long linkedListParallel = linkedList.parallelStream()  // This will be much slower
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> n * n)
            .sum();
        long linkedListParTime = System.nanoTime() - startTime;
        
        System.out.println("ArrayList parallel: " + arrayListParTime / 1_000_000 + " ms");
        System.out.println("LinkedList parallel: " + linkedListParTime / 1_000_000 + " ms");
        System.out.println("ArrayList advantage: " + (double) linkedListParTime / arrayListParTime + "x");
    }
    
    // Micro-optimizations and best practices
    public static void demonstrateMicroOptimizations() {
        System.out.println("\n=== Micro-optimizations and Best Practices ===");
        
        List<String> words = Arrays.asList("apple", "banana", "cherry", "date", "elderberry");
        
        System.out.println("\n--- Best Practice 1: Using Collectors for multiple operations ---");
        
        // Multiple separate operations (less efficient)
        startTime = System.nanoTime();
        List<String> upperCase = words.stream().map(String::toUpperCase).collect(Collectors.toList());
        List<Integer> lengths = words.stream().map(String::length).collect(Collectors.toList());
        Set<String> uniqueByLength = words.stream()
            .collect(Collectors.groupingBy(String::length))
            .entrySet()
            .stream()
            .map(entry -> entry.getValue().get(0))
            .collect(Collectors.toSet());
        long separateOpsTime = System.nanoTime() - startTime;
        
        // Single operation with multiple collectors (Java 12+ teeing concept simulated)
        startTime = System.nanoTime();
        class CombinedResult {
            List<String> upperCase;
            List<Integer> lengths;
            Set<String> uniqueByLength;
        }
        
        List<String> upperCase2 = words.stream().map(String::toUpperCase).collect(Collectors.toList());
        List<Integer> lengths2 = words.stream().map(String::length).collect(Collectors.toList());
        Set<String> uniqueByLength2 = words.stream()
            .collect(Collectors.toMap(String::length, Function.identity(), (a, b) -> a))
            .values()
            .stream()
            .collect(Collectors.toSet());
        long combinedOpsTime = System.nanoTime() - startTime;
        
        System.out.println("Separate operations: " + separateOpsTime / 1_000 + " μs");
        System.out.println("Combined operations: " + combinedOpsTime / 1_000 + " μs");
        
        System.out.println("\n--- Best Practice 2: Proper use of primitive streams ---");
        
        List<Integer> numbers = IntStream.range(1, 100_000).boxed().collect(Collectors.toList());
        
        // Inefficient: using object streams where primitives would work
        startTime = System.nanoTime();
        long objectStreamSum = numbers.stream()
            .reduce(0, (a, b) -> a + b, Integer::sum);
        long objectSumTime = System.nanoTime() - startTime;
        
        // Efficient: using primitive streams
        startTime = System.nanoTime();
        long primitiveStreamSum = numbers.stream().mapToInt(Integer::intValue).sum();
        long primitiveSumTime = System.nanoTime() - startTime;
        
        System.out.println("Object stream sum: " + objectSumTime / 1_000 + " μs");
        System.out.println("Primitive stream sum: " + primitiveSumTime / 1_000 + " μs");
        System.out.println("Primitive stream improvement: " + (double) objectSumTime / primitiveSumTime + "x");
        
        System.out.println("\n--- Best Practice 3: Avoiding boxing/unboxing in performance-critical code ---");
        
        // Demonstrating the difference between different approaches
        List<Integer> testNumbers = IntStream.range(0, 1_000_000).boxed().collect(Collectors.toList());
        
        startTime = System.nanoTime();
        // Using boxed operations
        long boxedResult = testNumbers.stream()
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> n * n)
            .sum();
        long boxedTime = System.nanoTime() - startTime;
        
        startTime = System.nanoTime();
        // Using primitive operations
        long primitiveResult = testNumbers.stream()
            .mapToInt(Integer::intValue)
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> (long) n * n)
            .sum();
        long primitiveTime = System.nanoTime() - startTime;
        
        System.out.println("Boxed operations: " + boxedTime / 1_000_000 + " ms");
        System.out.println("Primitive operations: " + primitiveTime / 1_000_000 + " ms");
        System.out.println("Primitive operations improvement: " + (double) boxedTime / primitiveTime + "x");
    }
    
    private static long startTime;
    
    public static void main(String[] args) {
        demonstrateStreamOptimizations();
        demonstrateMemoryOptimizations();
        demonstrateFunctionOptimization();
        demonstrateParallelOptimizations();
        demonstrateMicroOptimizations();
    }
}
```

## Functional Programming Best Practices

Establishing guidelines and best practices for effective functional programming in Java.

### Best Practices and Guidelines

```java
// Functional programming best practices and guidelines
public class FunctionalProgrammingBestPractices {
    
    // Immutable data structures and pure functions
    public static void demonstrateImmutability() {
        System.out.println("=== Immutability Best Practices ===");
        
        // Good: Immutable class with defensive copying
        class ImmutablePerson {
            private final String name;
            private final int age;
            private final List<String> hobbies;
            
            public ImmutablePerson(String name, int age, List<String> hobbies) {
                this.name = name;
                this.age = age;
                // Defensive copy to prevent external modification
                this.hobbies = Collections.unmodifiableList(new ArrayList<>(hobbies));
            }
            
            // Factory method for creating modified instances
            public ImmutablePerson withName(String newName) {
                return new ImmutablePerson(newName, this.age, this.hobbies);
            }
            
            public ImmutablePerson withAge(int newAge) {
                return new ImmutablePerson(this.name, newAge, this.hobbies);
            }
            
            public ImmutablePerson withHobbies(List<String> newHobbies) {
                return new ImmutablePerson(this.name, this.age, newHobbies);
            }
            
            // Only getter methods, no setters
            public String getName() { return name; }
            public int getAge() { return age; }
            public List<String> getHobbies() { return hobbies; }
            
            @Override
            public String toString() {
                return String.format("ImmutablePerson{name='%s', age=%d, hobbies=%s}", 
                                   name, age, hobbies);
            }
        }
        
        // Usage example
        List<String> hobbies = Arrays.asList("reading", "swimming", "coding");
        ImmutablePerson person = new ImmutablePerson("Alice", 30, hobbies);
        
        System.out.println("Original: " + person);
        
        // Create new instance with modified value
        ImmutablePerson olderPerson = person.withAge(31);
        System.out.println("Modified: " + olderPerson);
        System.out.println("Original unchanged: " + person);
        
        // Attempt to modify original hobbies list (should not affect the person)
        hobbies.add("gaming");
        System.out.println("After modifying original hobbies list - Person still has: " + person.getHobbies());
        
        System.out.println("\n--- Pure vs Impure Functions ---");
        
        class FunctionExamples {
            // Pure function: same input always produces same output, no side effects
            public static int add(int a, int b) {
                return a + b; // No side effects, deterministic
            }
            
            // Pure function with complex logic
            public static List<Integer> getEvenNumbers(List<Integer> numbers) {
                return numbers.stream()
                    .filter(n -> n % 2 == 0)
                    .collect(Collectors.toList());
            }
            
            // Impure function: has side effects
            public static int addWithLogging(int a, int b) {
                System.out.println("Adding " + a + " and " + b); // Side effect
                return a + b;
            }
            
            // Impure function: depends on external state
            private static int globalCounter = 0;
            public static int addWithCounter(int a, int b) {
                globalCounter++; // Side effect
                return a + b + globalCounter; // Non-deterministic
            }
        }
        
        System.out.println("Pure function result: " + FunctionExamples.add(5, 3));
        System.out.println("Pure function result: " + FunctionExamples.add(5, 3)); // Same result
        
        System.out.println("Impure function result: " + FunctionExamples.addWithLogging(5, 3));
        System.out.println("Impure function result: " + FunctionExamples.addWithLogging(5, 3)); // Different behavior
    }
    
    // Proper error handling in functional programming
    public static void demonstrateErrorHandling() {
        System.out.println("\n=== Functional Error Handling ===");
        
        // Using Optional for better null handling
        class UserService {
            private final Map<String, User> users = new HashMap<>();
            
            public UserService() {
                users.put("alice", new User("Alice", 30));
                users.put("bob", new User("Bob", 25));
            }
            
            // Good: Return Optional instead of null
            public Optional<User> findUser(String name) {
                return Optional.ofNullable(users.get(name.toLowerCase()));
            }
            
            // Functional approach to optional handling
            public String getUserInfo(String name) {
                return findUser(name)
                    .map(user -> user.getName() + " is " + user.getAge() + " years old")
                    .orElse("User not found: " + name);
            }
            
            public void processUser(String name, Consumer<User> processor) {
                findUser(name).ifPresent(processor);
            }
        }
        
        class User {
            private final String name;
            private final int age;
            
            public User(String name, int age) {
                this.name = name;
                this.age = age;
            }
            
            public String getName() { return name; }
            public int getAge() { return age; }
        }
        
        UserService service = new UserService();
        
        // Safe optional usage
        System.out.println("User info for Alice: " + service.getUserInfo("alice"));
        System.out.println("User info for Charlie: " + service.getUserInfo("charlie"));
        
        // Processing with Optional
        service.processUser("alice", user -> 
            System.out.println("Processing user: " + user.getName()));
        service.processUser("nonexistent", user -> 
            System.out.println("This won't be called"));
        
        System.out.println("\n--- Either-like pattern for error handling ---");
        
        // Simple Either implementation for success/failure
        class Result<T> {
            private final T value;
            private final String error;
            private final boolean isSuccess;
            
            private Result(T value, String error, boolean isSuccess) {
                this.value = value;
                this.error = error;
                this.isSuccess = isSuccess;
            }
            
            public static <T> Result<T> success(T value) {
                return new Result<>(value, null, true);
            }
            
            public static <T> Result<T> failure(String error) {
                return new Result<>(null, error, false);
            }
            
            public boolean isSuccess() { return isSuccess; }
            public T getValue() { return value; }
            public String getError() { return error; }
            
            public T orElse(T defaultValue) {
                return isSuccess ? value : defaultValue;
            }
            
            public <R> Result<R> map(Function<T, R> mapper) {
                if (isSuccess) {
                    try {
                        return Result.success(mapper.apply(value));
                    } catch (Exception e) {
                        return Result.failure("Mapping error: " + e.getMessage());
                    }
                } else {
                    return Result.failure(error);
                }
            }
            
            public <R> Result<R> flatMap(Function<T, Result<R>> mapper) {
                if (isSuccess) {
                    try {
                        return mapper.apply(value);
                    } catch (Exception e) {
                        return Result.failure("Flat mapping error: " + e.getMessage());
                    }
                } else {
                    return Result.failure(error);
                }
            }
        }
        
        // Example usage of Result
        Function<String, Result<Integer>> parseInt = str -> {
            try {
                return Result.success(Integer.parseInt(str));
            } catch (NumberFormatException e) {
                return Result.failure("Invalid number format: " + str);
            }
        };
        
        Result<Integer> result1 = parseInt.apply("123");
        Result<Integer> result2 = parseInt.apply("invalid");
        
        System.out.println("Parse result 1: " + (result1.isSuccess() ? 
                         "Success: " + result1.getValue() : "Error: " + result1.getError()));
        System.out.println("Parse result 2: " + (result2.isSuccess() ? 
                         "Success: " + result2.getValue() : "Error: " + result2.getError()));
        
        // Chaining operations with error handling
        Result<String> chainedResult = parseInt.apply("42")
            .map(num -> num * 2)
            .map(num -> "Result: " + num);
        
        System.out.println("Chained result: " + 
                         (chainedResult.isSuccess() ? 
                          chainedResult.getValue() : 
                          "Error: " + chainedResult.getError()));
    }
    
    // Performance and readability best practices
    public static void demonstratePerformanceReadability() {
        System.out.println("\n=== Performance and Readability Best Practices ===");
        
        List<Integer> numbers = IntStream.range(1, 100_000)
            .boxed()
            .collect(Collectors.toList());
        
        System.out.println("\n--- Readability: Meaningful variable names and composition ---");
        
        // Less readable
        List<String> unreadable = numbers.stream()
            .filter(x -> x % 2 == 0)
            .map(x -> x * x)
            .filter(x -> x > 100)
            .map(x -> "Number: " + x)
            .collect(Collectors.toList());
        
        // More readable with meaningful names and composition
        Predicate<Integer> isEven = n -> n % 2 == 0;
        Function<Integer, Integer> square = n -> n * n;
        Predicate<Integer> isGreaterThan100 = n -> n > 100;
        Function<Integer, String> format = n -> "Number: " + n;
        
        List<String> readable = numbers.stream()
            .filter(isEven)
            .map(square)
            .filter(isGreaterThan100)
            .map(format)
            .collect(Collectors.toList());
        
        System.out.println("Both lists have same size: " + (unreadable.size() == readable.size()));
        
        System.out.println("\n--- Performance: Efficient collector selection ---");
        
        // For specific use cases, choose appropriate collectors
        List<String> words = Arrays.asList("apple", "banana", "cherry", "date", "elderberry");
        
        // When you know the approximate size, specify initial capacity
        List<String> sizedList = words.stream()
            .map(String::toUpperCase)
            .collect(ArrayList::new, 
                    (list, item) -> list.add(item), 
                    (list1, list2) -> list1.addAll(list2));
        
        // Using toCollection with specific type
        LinkedList<String> linkedList = words.stream()
            .filter(s -> s.length() > 4)
            .collect(Collectors.toCollection(LinkedList::new));
        
        System.out.println("Filtered words in LinkedList: " + linkedList);
        
        System.out.println("\n--- Performance: Avoiding unnecessary intermediate operations ---");
        
        List<Integer> testNumbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        // Inefficient: multiple passes over the data
        long startTime = System.nanoTime();
        List<Integer> temp = testNumbers.stream()
            .filter(n -> n % 2 == 0)
            .collect(Collectors.toList());
        List<Integer> inefficient = temp.stream()
            .map(n -> n * n)
            .filter(n -> n > 10)
            .collect(Collectors.toList());
        long inefficientTime = System.nanoTime() - startTime;
        
        // Efficient: single pass with composed operations
        startTime = System.nanoTime();
        List<Integer> efficient = testNumbers.stream()
            .filter(n -> n % 2 == 0)
            .map(n -> n * n)
            .filter(n -> n > 10)
            .collect(Collectors.toList());
        long efficientTime = System.nanoTime() - startTime;
        
        System.out.println("Inefficient (2 passes): " + inefficientTime / 1_000 + " μs");
        System.out.println("Efficient (1 pass): " + efficientTime / 1_000 + " μs");
        System.out.println("Efficiency improvement: " + (double) inefficientTime / efficientTime + "x");
        
        System.out.println("\n--- Functional composition and reusability ---");
        
        // Creating reusable function components
        Function<String, Boolean> isLong = s -> s.length() > 5;
        Function<String, String> toUpper = String::toUpperCase;
        Function<String, Integer> getLength = String::length;
        
        // Composing functions for different use cases
        Function<String, String> processForDisplay = isLong
            .andThen(b -> b ? toUpper : Function.identity())
            .andThen(Function.identity());
        
        // Manual composition since the above won't work directly
        Function<String, String> processForDisplayCorrect = s -> 
            isLong.apply(s) ? toUpper.apply(s) : s;
        
        List<String> processed = words.stream()
            .map(processForDisplayCorrect)
            .collect(Collectors.toList());
        
        System.out.println("Processed for display: " + processed);
    }
    
    // Advanced functional patterns and techniques
    public static void demonstrateAdvancedPatterns() {
        System.out.println("\n=== Advanced Functional Patterns ===");
        
        System.out.println("\n--- Lazy evaluation and infinite sequences ---");
        
        // Creating an infinite stream of Fibonacci numbers
        Stream<Integer> fibonacciStream = Stream.iterate(
            new int[]{0, 1},
            pair -> new int[]{pair[1], pair[0] + pair[1]}
        ).map(pair -> pair[0]);
        
        // Taking first 10 Fibonacci numbers (lazy evaluation)
        List<Integer> fib10 = fibonacciStream
            .limit(10)
            .collect(Collectors.toList());
        
        System.out.println("First 10 Fibonacci numbers: " + fib10);
        
        // Creating a stream of prime numbers (simplified implementation)
        Stream<Integer> primes = Stream.iterate(2, n -> n + 1)
            .filter(n -> {
                if (n < 2) return false;
                for (int i = 2; i <= Math.sqrt(n); i++) {
                    if (n % i == 0) return false;
                }
                return true;
            });
        
        List<Integer> first10Primes = primes
            .limit(10)
            .collect(Collectors.toList());
        
        System.out.println("First 10 primes: " + first10Primes);
        
        System.out.println("\n--- Higher-order functions and function factories ---");
        
        // Function that creates other functions
        Function<Integer, Function<Integer, Integer>> createMultiplier = factor ->
            operand -> operand * factor;
        
        Function<Integer, Integer> multiplyBy5 = createMultiplier.apply(5);
        Function<Integer, Integer> multiplyBy10 = createMultiplier.apply(10);
        
        System.out.println("12 * 5 = " + multiplyBy5.apply(12));
        System.out.println("12 * 10 = " + multiplyBy10.apply(12));
        
        // Function that creates filtering functions
        Function<Predicate<Integer>, Function<List<Integer>, List<Integer>>> createFilter = 
            predicate -> list -> list.stream().filter(predicate).collect(Collectors.toList());
        
        Function<List<Integer>, List<Integer>> getEvens = createFilter.apply(n -> n % 2 == 0);
        Function<List<Integer>, List<Integer>> getOdds = createFilter.apply(n -> n % 2 != 0);
        
        List<Integer> testList = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        System.out.println("Evens: " + getEvens.apply(testList));
        System.out.println("Odds: " + getOdds.apply(testList));
        
        System.out.println("\n--- Currying and partial application ---");
        
        // Curried function: multiple parameters one at a time
        Function<Integer, Function<Integer, Function<Integer, Integer>>> curriedAdd = 
            a -> b -> c -> a + b + c;
        
        // Partial application
        Function<Integer, Function<Integer, Integer>> add5 = curriedAdd.apply(5);
        Function<Integer, Integer> add5And3 = add5.apply(3);
        
        int result = add5And3.apply(2); // 5 + 3 + 2 = 10
        System.out.println("Curried addition result: " + result);
        
        // Practical example: curried formatting
        Function<String, Function<Object[], String>> createFormatter = pattern ->
            args -> String.format(pattern, args);
        
        Function<Object[], String> jsonFormatter = createFormatter.apply("{\"value\": %s}");
        System.out.println("JSON formatted: " + jsonFormatter.apply(new Object[]{"test"}));
    }
    
    // Testing and debugging functional code
    public static void demonstrateTestingDebugging() {
        System.out.println("\n=== Testing and Debugging Functional Code ===");
        
        System.out.println("\n--- Using peek for debugging streams ---");
        
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        // Debugging each step of a stream pipeline
        List<Integer> result = numbers.stream()
            .peek(n -> System.out.println("Original: " + n))
            .filter(n -> {
                boolean isEven = n % 2 == 0;
                System.out.println("After filter (" + n + " is even: " + isEven + "): " + n);
                return isEven;
            })
            .map(n -> {
                int squared = n * n;
                System.out.println("After map (squared " + n + "): " + squared);
                return squared;
            })
            .peek(n -> System.out.println("Before limit: " + n))
            .limit(3)
            .peek(n -> System.out.println("Final result: " + n))
            .collect(Collectors.toList());
        
        System.out.println("Final collection: " + result);
        
        System.out.println("\n--- Functional approach to validation ---");
        
        class Validation {
            private final String field;
            private final List<String> errors;
            
            public Validation(String field, List<String> errors) {
                this.field = field;
                this.errors = errors;
            }
            
            public boolean isValid() { return errors.isEmpty(); }
            public String getField() { return field; }
            public List<String> getErrors() { return errors; }
        }
        
        // Validation functions
        Function<String, Validation> notEmpty = value -> 
            value != null && !value.trim().isEmpty() ? 
            new Validation(value, Collections.emptyList()) : 
            new Validation(value, Arrays.asList("Value cannot be empty"));
        
        Function<String, Validation> minLength = value -> {
            List<String> errors = value.length() < 3 ? 
                Arrays.asList("Value must be at least 3 characters") : 
                Collections.emptyList();
            return new Validation(value, errors);
        };
        
        Function<String, Validation> noNumbers = value -> {
            List<String> errors = value.chars().anyMatch(Character::isDigit) ? 
                Arrays.asList("Value cannot contain numbers") : 
                Collections.emptyList();
            return new Validation(value, errors);
        };
        
        // Compose validation functions
        List<Function<String, Validation>> validators = Arrays.asList(notEmpty, minLength, noNumbers);
        
        Function<String, Validation> compositeValidator = input -> {
            List<String> allErrors = new ArrayList<>();
            
            for (Function<String, Validation> validator : validators) {
                Validation result = validator.apply(input);
                allErrors.addAll(result.getErrors());
            }
            
            return new Validation(input, allErrors);
        };
        
        // Test validation
        Validation validResult = compositeValidator.apply("hello");
        Validation invalidResult = compositeValidator.apply("h1");
        
        System.out.println("Valid input 'hello': " + (validResult.isValid() ? "Valid" : "Invalid - " + validResult.getErrors()));
        System.out.println("Invalid input 'h1': " + (invalidResult.isValid() ? "Valid" : "Invalid - " + invalidResult.getErrors()));
    }
    
    public static void main(String[] args) {
        demonstrateImmutability();
        demonstrateErrorHandling();
        demonstratePerformanceReadability();
        demonstrateAdvancedPatterns();
        demonstrateTestingDebugging();
    }
}
```

## Conclusion

Functional programming in Java provides powerful abstractions for writing clean, maintainable, and efficient code. The key benefits include:

1. **Immutability**: Reduces bugs and makes code easier to reason about
2. **Higher-Order Functions**: Enables powerful code reuse and composition
3. **Stream API**: Provides a declarative approach to data processing
4. **Parallel Processing**: Simplifies concurrent data processing
5. **Functional Patterns**: Offers elegant solutions to common programming challenges

The Stream API and functional interfaces have revolutionized how developers approach data manipulation in Java, making code more expressive and less error-prone. Understanding when and how to use these features appropriately is crucial for modern Java development.