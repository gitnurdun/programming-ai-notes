# Java 8+ Features: Mastering Modern Java Programming

## Introduction

Java 8 marked a revolutionary change in the Java programming language with the introduction of functional programming concepts, lambda expressions, streams, and many other modern features. Since then, Java has continued to evolve with new features in subsequent releases (Java 9 through Java 21+). This comprehensive guide explores all the major modern Java features that every Java developer should master.

This guide will cover lambda expressions, streams API, functional interfaces, new collection APIs, optional class, CompletableFuture, and other significant features introduced in Java 8 and beyond.

## Lambda Expressions and Functional Programming

### Introduction to Lambda Expressions

Lambda expressions are a key feature introduced in Java 8 that enable functional programming in Java. They provide a concise way to represent instances of single-method interfaces (functional interfaces).

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class LambdaExpressions {
    public static void main(String[] args) {
        System.out.println("=== Lambda Expressions ===");
        
        // Demonstrate basic lambda expressions
        demonstrateBasicLambdas();
        
        // Demonstrate functional interfaces
        demonstrateFunctionalInterfaces();
        
        // Demonstrate method references
        demonstrateMethodReferences();
        
        // Demonstrate variable capture
        demonstrateVariableCapture();
    }
    
    private static void demonstrateBasicLambdas() {
        System.out.println("\n=== Basic Lambda Expressions ===");
        
        // Lambda with no parameters
        Runnable runnable = () -> System.out.println("Hello from lambda!");
        runnable.run();
        
        // Lambda with one parameter
        Consumer<String> printer = message -> System.out.println("Message: " + message);
        printer.accept("Hello World");
        
        // Lambda with multiple parameters
        BinaryOperator<Integer> add = (a, b) -> a + b;
        System.out.println("Addition: " + add.apply(5, 3));
        
        // Lambda with multiple statements
        Consumer<String> complexProcessor = s -> {
            String upper = s.toUpperCase();
            String result = "Processed: " + upper;
            System.out.println(result);
        };
        complexProcessor.accept("hello");
        
        // Lambda with type inference
        Function<String, Integer> stringLength = s -> s.length();
        System.out.println("Length: " + stringLength.apply("Java"));
    }
    
    private static void demonstrateFunctionalInterfaces() {
        System.out.println("\n=== Functional Interfaces ===");
        
        // Supplier - supplies values
        Supplier<String> randomString = () -> "Random: " + Math.random();
        System.out.println("Random string: " + randomString.get());
        
        // Consumer - consumes values
        Consumer<String> upperCasePrinter = s -> System.out.println(s.toUpperCase());
        upperCasePrinter.accept("hello world");
        
        // Predicate - tests conditions
        Predicate<String> isLong = s -> s.length() > 10;
        System.out.println("Is 'verylongstring' long? " + isLong.test("verylongstring"));
        
        // Function - transforms values
        Function<String, Integer> stringToInt = Integer::parseInt;
        System.out.println("Parsed integer: " + stringToInt.apply("123"));
        
        // BinaryOperator - operates on two values of same type
        BinaryOperator<Integer> multiply = (a, b) -> a * b;
        System.out.println("Multiplication: " + multiply.apply(4, 5));
        
        // Chaining functional interfaces
        Function<String, String> addPrefix = s -> "Hello " + s;
        Function<String, String> addSuffix = s -> s + "!";
        Function<String, String> processor = addPrefix.andThen(addSuffix);
        System.out.println("Chained: " + processor.apply("World"));
    }
    
    private static void demonstrateMethodReferences() {
        System.out.println("\n=== Method References ===");
        
        // Static method reference
        Function<String, Integer> parseInt = Integer::parseInt;
        System.out.println("Parsed: " + parseInt.apply("42"));
        
        // Instance method reference of a particular object
        String prefix = "Hello ";
        Function<String, String> addPrefix = prefix::concat;
        System.out.println("Prefixed: " + addPrefix.apply("World"));
        
        // Instance method reference of an arbitrary object of a particular type
        Function<String, Integer> length = String::length;
        System.out.println("Length: " + length.apply("Java"));
        
        // Constructor reference
        Supplier<List<String>> listSupplier = ArrayList::new;
        List<String> list = listSupplier.get();
        System.out.println("Empty list created: " + list);
        
        // Constructor reference with parameters
        Function<String, StringBuilder> sbConstructor = StringBuilder::new;
        StringBuilder sb = sbConstructor.apply("Initial");
        System.out.println("StringBuilder: " + sb);
    }
    
    private static void demonstrateVariableCapture() {
        System.out.println("\n=== Variable Capture ===");
        
        int multiplier = 10; // Effectively final
        Function<Integer, Integer> multiplyBy = x -> x * multiplier;
        System.out.println("Multiply by 10: " + multiplyBy.apply(5));
        
        String prefix = "Result: ";
        Function<Integer, String> formatResult = x -> prefix + (x * multiplier);
        System.out.println("Formatted: " + formatResult.apply(7));
        
        // Method parameter capture
        processNumber(42, n -> "Processed: " + (n * multiplier));
        
        // Local variable capture
        int[] counter = {0}; // Array to allow modification
        Runnable incrementer = () -> counter[0]++;
        incrementer.run();
        System.out.println("Counter after increment: " + counter[0]);
    }
    
    private static void processNumber(int number, Function<Integer, String> processor) {
        System.out.println(processor.apply(number));
    }
}
```

### Advanced Lambda Features

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class AdvancedLambdaFeatures {
    public static void main(String[] args) {
        System.out.println("=== Advanced Lambda Features ===");
        
        // Demonstrate complex functional interface usage
        demonstrateComplexUsage();
        
        // Demonstrate lambda with streams
        demonstrateLambdaWithStreams();
        
        // Demonstrate custom functional interfaces
        demonstrateCustomFunctionalInterfaces();
        
        // Demonstrate functional composition
        demonstrateFunctionalComposition();
    }
    
    private static void demonstrateComplexUsage() {
        System.out.println("\n=== Complex Lambda Usage ===");
        
        // Complex predicate chaining
        List<String> words = Arrays.asList("hello", "world", "java", "programming", "lambda");
        
        Predicate<String> isLong = s -> s.length() > 4;
        Predicate<String> startsWithJ = s -> s.startsWith("j");
        Predicate<String> hasVowel = s -> s.toLowerCase().matches(".*[aeiou].*");
        
        // Chain predicates
        List<String> filtered = words.stream()
            .filter(isLong.and(startsWithJ).and(hasVowel))
            .collect(Collectors.toList());
        
        System.out.println("Filtered words: " + filtered);
        
        // Complex function composition
        Function<String, String> toUpper = String::toUpperCase;
        Function<String, String> addBrackets = s -> "[" + s + "]";
        Function<String, String> addPrefix = s -> ">> " + s;
        
        Function<String, String> complexTransform = toUpper
            .andThen(addBrackets)
            .andThen(addPrefix);
        
        System.out.println("Complex transform: " + complexTransform.apply("hello"));
        
        // Binary operator with reduction
        BinaryOperator<String> concatenate = (s1, s2) -> s1 + ", " + s2;
        String result = words.stream().reduce(concatenate).orElse("");
        System.out.println("Concatenated: " + result);
    }
    
    private static void demonstrateLambdaWithStreams() {
        System.out.println("\n=== Lambda with Streams ===");
        
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        // Filter, map, and collect with lambdas
        List<Integer> evenSquares = numbers.stream()
            .filter(n -> n % 2 == 0)           // Predicate lambda
            .map(n -> n * n)                   // Function lambda
            .collect(Collectors.toList());     // Collector
        System.out.println("Even squares: " + evenSquares);
        
        // Complex stream operations
        List<String> words = Arrays.asList("apple", "banana", "cherry", "date", "elderberry");
        
        Map<Integer, List<String>> groupedByLength = words.stream()
            .collect(Collectors.groupingBy(String::length)); // Method reference
        System.out.println("Grouped by length: " + groupedByLength);
        
        // Find first element satisfying condition
        Optional<String> longWord = words.stream()
            .filter(s -> s.length() > 5)
            .findFirst();
        System.out.println("First long word: " + longWord.orElse("None found"));
        
        // Calculate statistics
        IntSummaryStatistics stats = numbers.stream()
            .mapToInt(Integer::intValue)
            .summaryStatistics();
        System.out.println("Statistics: " + stats);
        
        // Parallel processing
        List<Integer> largeList = IntStream.range(1, 100000)
            .boxed()
            .collect(Collectors.toList());
        
        long startTime = System.currentTimeMillis();
        long sequentialSum = largeList.stream()
            .mapToLong(Long::valueOf)
            .sum();
        long sequentialTime = System.currentTimeMillis() - startTime;
        
        startTime = System.currentTimeMillis();
        long parallelSum = largeList.parallelStream()
            .mapToLong(Long::valueOf)
            .sum();
        long parallelTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Sequential sum: " + sequentialSum + " in " + sequentialTime + "ms");
        System.out.println("Parallel sum: " + parallelSum + " in " + parallelTime + "ms");
    }
    
    private static void demonstrateCustomFunctionalInterfaces() {
        System.out.println("\n=== Custom Functional Interfaces ===");
        
        // Custom functional interface
        @FunctionalInterface
        interface TriFunction<T, U, V, R> {
            R apply(T t, U u, V v);
            
            default <S> TriFunction<T, U, V, S> andThen(Function<R, S> after) {
                Objects.requireNonNull(after);
                return (T t, U u, V v) -> after.apply(apply(t, u, v));
            }
        }
        
        TriFunction<Integer, Integer, Integer, Integer> calculate = (a, b, c) -> a * b + c;
        System.out.println("TriFunction result: " + calculate.apply(2, 3, 4));
        
        // Use default method
        TriFunction<Integer, Integer, Integer, String> calculateString = calculate
            .andThen(result -> "Result: " + result);
        System.out.println("With string transformation: " + calculateString.apply(2, 3, 4));
        
        // Another custom functional interface
        @FunctionalInterface
        interface ThrowingFunction<T, R> {
            R apply(T t) throws Exception;
            
            static <T, R> Function<T, R> unchecked(ThrowingFunction<T, R> f) {
                return t -> {
                    try {
                        return f.apply(t);
                    } catch (Exception e) {
                        throw new RuntimeException(e);
                    }
                };
            }
        }
        
        // Use with exception handling
        Function<String, Integer> safeParse = ThrowingFunction.unchecked(Integer::parseInt);
        System.out.println("Safe parse: " + safeParse.apply("123"));
    }
    
    private static void demonstrateFunctionalComposition() {
        System.out.println("\n=== Functional Composition ===");
        
        // Function composition: f(g(x)) vs g(f(x))
        Function<Integer, Integer> add1 = x -> x + 1;
        Function<Integer, Integer> multiply2 = x -> x * 2;
        Function<Integer, Integer> square = x -> x * x;
        
        // andThen: apply this function first, then the argument
        Function<Integer, Integer> add1ThenMultiply2 = add1.andThen(multiply2);
        System.out.println("(x + 1) * 2 for x=5: " + add1ThenMultiply2.apply(5));
        
        // compose: apply the argument first, then this function
        Function<Integer, Integer> multiply2ThenAdd1 = add1.compose(multiply2);
        System.out.println("x * 2 + 1 for x=5: " + multiply2ThenAdd1.apply(5));
        
        // Complex composition
        Function<Integer, Integer> complex = square
            .andThen(add1)
            .andThen(multiply2);
        System.out.println("((x^2) + 1) * 2 for x=3: " + complex.apply(3));
        
        // Predicate composition
        Predicate<Integer> isEven = x -> x % 2 == 0;
        Predicate<Integer> isPositive = x -> x > 0;
        Predicate<Integer> isGreaterThan10 = x -> x > 10;
        
        Predicate<Integer> complexPredicate = isPositive
            .and(isEven)
            .and(isGreaterThan10);
        
        System.out.println("Is 12 positive, even, and > 10? " + complexPredicate.test(12));
        System.out.println("Is 8 positive, even, and > 10? " + complexPredicate.test(8));
    }
}
```

## Streams API

### Introduction to Streams

The Streams API provides a functional approach to processing sequences of elements. It supports various operations like filtering, mapping, reducing, and collecting.

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.function.*;
import java.util.stream.*;

public class StreamsAPI {
    public static void main(String[] args) {
        System.out.println("=== Streams API ===");
        
        // Demonstrate basic stream operations
        demonstrateBasicOperations();
        
        // Demonstrate intermediate and terminal operations
        demonstrateIntermediateTerminal();
        
        // Demonstrate stream collectors
        demonstrateCollectors();
        
        // Demonstrate parallel streams
        demonstrateParallelStreams();
    }
    
    private static void demonstrateBasicOperations() {
        System.out.println("\n=== Basic Stream Operations ===");
        
        List<String> words = Arrays.asList("hello", "world", "java", "stream", "api");
        
        // Filter operation
        List<String> longWords = words.stream()
            .filter(s -> s.length() > 4)
            .collect(Collectors.toList());
        System.out.println("Long words: " + longWords);
        
        // Map operation
        List<Integer> lengths = words.stream()
            .map(String::length)
            .collect(Collectors.toList());
        System.out.println("Word lengths: " + lengths);
        
        // FlatMap operation
        List<String> sentences = Arrays.asList("hello world", "java stream", "programming api");
        List<String> allWords = sentences.stream()
            .flatMap(sentence -> Arrays.stream(sentence.split(" ")))
            .collect(Collectors.toList());
        System.out.println("All words: " + allWords);
        
        // Distinct operation
        List<Integer> numbers = Arrays.asList(1, 2, 2, 3, 3, 4, 5, 5);
        List<Integer> uniqueNumbers = numbers.stream()
            .distinct()
            .collect(Collectors.toList());
        System.out.println("Unique numbers: " + uniqueNumbers);
        
        // Limit operation
        List<String> limited = words.stream()
            .limit(3)
            .collect(Collectors.toList());
        System.out.println("First 3 words: " + limited);
        
        // Skip operation
        List<String> skipped = words.stream()
            .skip(2)
            .collect(Collectors.toList());
        System.out.println("Words after skipping 2: " + skipped);
    }
    
    private static void demonstrateIntermediateTerminal() {
        System.out.println("\n=== Intermediate and Terminal Operations ===");
        
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        // Intermediate operations (lazy, not executed until terminal)
        Stream<Integer> intermediate = numbers.stream()
            .filter(n -> {
                System.out.println("Filtering: " + n);
                return n % 2 == 0;
            })
            .map(n -> {
                System.out.println("Mapping: " + n);
                return n * n;
            });
        
        System.out.println("Intermediate stream created, no operations executed yet");
        
        // Terminal operation (triggers execution of intermediate operations)
        List<Integer> result = intermediate.collect(Collectors.toList());
        System.out.println("Final result: " + result);
        
        // Another example with different terminal operations
        Optional<Integer> firstEven = numbers.stream()
            .filter(n -> n % 2 == 0)
            .findFirst();
        System.out.println("First even number: " + firstEven.orElse(-1));
        
        long count = numbers.stream()
            .filter(n -> n % 2 == 0)
            .count();
        System.out.println("Count of even numbers: " + count);
        
        boolean anyMatch = numbers.stream()
            .anyMatch(n -> n > 8);
        System.out.println("Any number > 8: " + anyMatch);
        
        boolean allMatch = numbers.stream()
            .allMatch(n -> n > 0);
        System.out.println("All numbers > 0: " + allMatch);
    }
    
    private static void demonstrateCollectors() {
        System.out.println("\n=== Stream Collectors ===");
        
        List<String> words = Arrays.asList("hello", "world", "java", "programming", "stream", "api");
        
        // Grouping collectors
        Map<Integer, List<String>> groupedByLength = words.stream()
            .collect(Collectors.groupingBy(String::length));
        System.out.println("Grouped by length: " + groupedByLength);
        
        // Grouping with downstream collector
        Map<Integer, Long> countByLength = words.stream()
            .collect(Collectors.groupingBy(
                String::length,
                Collectors.counting()
            ));
        System.out.println("Count by length: " + countByLength);
        
        // Partitioning collectors
        Map<Boolean, List<String>> partitioned = words.stream()
            .collect(Collectors.partitioningBy(s -> s.length() > 5));
        System.out.println("Partitioned (>5): " + partitioned);
        
        // Mapping collector
        Set<String> upperCaseSet = words.stream()
            .collect(Collectors.mapping(
                String::toUpperCase,
                Collectors.toSet()
            ));
        System.out.println("Uppercase set: " + upperCaseSet);
        
        // Joining collector
        String joined = words.stream()
            .collect(Collectors.joining(", "));
        System.out.println("Joined: " + joined);
        
        // Statistics collectors
        IntSummaryStatistics stats = words.stream()
            .mapToInt(String::length)
            .summaryStatistics();
        System.out.println("Statistics: " + stats);
        
        // Custom collector example
        List<String> customResult = words.stream()
            .collect(
                ArrayList::new,           // Supplier
                (list, item) -> {        // Accumulator
                    if (item.length() > 4) {
                        list.add(item.toUpperCase());
                    }
                },
                (list1, list2) -> {      // Combiner
                    list1.addAll(list2);
                }
            );
        System.out.println("Custom collection: " + customResult);
    }
    
    private static void demonstrateParallelStreams() {
        System.out.println("\n=== Parallel Streams ===");
        
        // Create a large list for demonstration
        List<Integer> largeList = IntStream.range(0, 1000000)
            .boxed()
            .collect(Collectors.toList());
        
        // Sequential vs parallel processing
        long startTime = System.currentTimeMillis();
        long sequentialSum = largeList.stream()
            .mapToLong(Long::valueOf)
            .sum();
        long sequentialTime = System.currentTimeMillis() - startTime;
        
        startTime = System.currentTimeMillis();
        long parallelSum = largeList.parallelStream()
            .mapToLong(Long::valueOf)
            .sum();
        long parallelTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Sequential sum: " + sequentialSum + " in " + sequentialTime + "ms");
        System.out.println("Parallel sum: " + parallelSum + " in " + parallelTime + "ms");
        System.out.println("Speedup: " + (double) sequentialTime / parallelTime + "x");
        
        // Parallel processing with side effects
        List<String> results = Collections.synchronizedList(new ArrayList<>());
        
        largeList.parallelStream()
            .filter(n -> n % 100000 == 0)  // Only process every 100,000th number
            .forEach(n -> results.add("Processed: " + n));
        
        System.out.println("Parallel results: " + results);
        
        // Parallel reduction
        Optional<Integer> parallelMax = largeList.parallelStream()
            .max(Comparator.naturalOrder());
        System.out.println("Parallel max: " + parallelMax.orElse(-1));
        
        // Custom parallel processing with collectors
        Map<String, List<Integer>> parallelGrouped = largeList.parallelStream()
            .filter(n -> n % 100000 == 0)
            .collect(Collectors.groupingByConcurrent(n -> "Group: " + (n / 100000)));
        System.out.println("Parallel grouped: " + parallelGrouped);
    }
}
```

### Advanced Stream Operations

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.function.*;
import java.util.stream.*;

public class AdvancedStreamOperations {
    public static void main(String[] args) {
        System.out.println("=== Advanced Stream Operations ===");
        
        // Demonstrate reduce operations
        demonstrateReduceOperations();
        
        // Demonstrate flatMap operations
        demonstrateFlatMapOperations();
        
        // Demonstrate primitive streams
        demonstratePrimitiveStreams();
        
        // Demonstrate stream builders
        demonstrateStreamBuilders();
    }
    
    private static void demonstrateReduceOperations() {
        System.out.println("\n=== Reduce Operations ===");
        
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        // Simple reduce with binary operator
        Optional<Integer> sum = numbers.stream().reduce(Integer::sum);
        System.out.println("Sum: " + sum.orElse(0));
        
        // Reduce with identity value
        int sumWithIdentity = numbers.stream().reduce(0, Integer::sum);
        System.out.println("Sum with identity: " + sumWithIdentity);
        
        // Reduce with custom combiner
        String concatenated = numbers.stream()
            .map(String::valueOf)
            .reduce("", String::concat);
        System.out.println("Concatenated: " + concatenated);
        
        // Reduce with combiner for parallel processing
        List<String> words = Arrays.asList("hello", "world", "java", "stream");
        String joined = words.stream()
            .reduce("", String::concat, String::concat);
        System.out.println("Joined: " + joined);
        
        // Complex reduce operation
        List<Person> people = Arrays.asList(
            new Person("Alice", 25, "Engineering"),
            new Person("Bob", 30, "Marketing"),
            new Person("Charlie", 35, "Engineering"),
            new Person("Diana", 28, "Marketing")
        );
        
        // Find person with maximum age
        Optional<Person> oldest = people.stream()
            .reduce(BinaryOperator.maxBy(Comparator.comparing(Person::getAge)));
        System.out.println("Oldest person: " + oldest.map(Person::getName).orElse("None"));
        
        // Custom reduce to calculate average
        double averageAge = people.stream()
            .collect(Collectors.averagingInt(Person::getAge));
        System.out.println("Average age: " + averageAge);
        
        // Reduce to find max by custom criteria
        Optional<Person> highestPaid = people.stream()
            .reduce((p1, p2) -> p1.getAge() > p2.getAge() ? p1 : p2);
        System.out.println("Highest paid (by age): " + 
                          highestPaid.map(Person::getName).orElse("None"));
    }
    
    private static void demonstrateFlatMapOperations() {
        System.out.println("\n=== FlatMap Operations ===");
        
        // Flatten nested collections
        List<List<String>> nestedLists = Arrays.asList(
            Arrays.asList("a", "b", "c"),
            Arrays.asList("d", "e"),
            Arrays.asList("f", "g", "h", "i")
        );
        
        List<String> flattened = nestedLists.stream()
            .flatMap(List::stream)
            .collect(Collectors.toList());
        System.out.println("Flattened: " + flattened);
        
        // Map and flatten in one operation
        List<String> words = Arrays.asList("hello world", "java stream", "programming api");
        List<String> allWords = words.stream()
            .flatMap(s -> Arrays.stream(s.split(" ")))
            .collect(Collectors.toList());
        System.out.println("All words: " + allWords);
        
        // FlatMap with objects
        List<Department> departments = Arrays.asList(
            new Department("Engineering", Arrays.asList(
                new Employee("Alice", 25),
                new Employee("Bob", 30)
            )),
            new Department("Marketing", Arrays.asList(
                new Employee("Charlie", 35),
                new Employee("Diana", 28)
            ))
        );
        
        List<String> allEmployeeNames = departments.stream()
            .flatMap(dept -> dept.getEmployees().stream())
            .map(Employee::getName)
            .collect(Collectors.toList());
        System.out.println("All employee names: " + allEmployeeNames);
        
        // Complex flatMap with filtering
        List<String> sentences = Arrays.asList(
            "Hello world Java",
            "Stream API functional",
            "Programming concepts"
        );
        
        List<String> longWords = sentences.stream()
            .flatMap(sentence -> Arrays.stream(sentence.split(" ")))
            .filter(word -> word.length() > 4)
            .collect(Collectors.toList());
        System.out.println("Long words: " + longWords);
        
        // FlatMap with optional values
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "Diana");
        List<String> processedNames = names.stream()
            .flatMap(name -> {
                if (name.length() > 4) {
                    return Stream.of(name.toUpperCase());
                } else {
                    return Stream.empty(); // Return empty stream for short names
                }
            })
            .collect(Collectors.toList());
        System.out.println("Processed names: " + processedNames);
    }
    
    private static void demonstratePrimitiveStreams() {
        System.out.println("\n=== Primitive Streams ===");
        
        // IntStream
        IntStream intStream = IntStream.range(1, 11);
        int sum = intStream.sum();
        System.out.println("IntStream sum: " + sum);
        
        // IntStream from array
        int[] intArray = {1, 2, 3, 4, 5};
        int arraySum = Arrays.stream(intArray).sum();
        System.out.println("Array sum: " + arraySum);
        
        // DoubleStream
        DoubleStream doubleStream = DoubleStream.of(1.1, 2.2, 3.3, 4.4, 5.5);
        double average = doubleStream.average().orElse(0.0);
        System.out.println("DoubleStream average: " + average);
        
        // LongStream
        LongStream longStream = LongStream.rangeClosed(1, 100);
        long count = longStream.count();
        System.out.println("LongStream count: " + count);
        
        // Generate infinite streams
        IntStream randomInts = new Random().ints(10, 1, 100);
        List<Integer> randomList = randomInts.boxed().collect(Collectors.toList());
        System.out.println("Random integers: " + randomList);
        
        // Create streams from ranges
        List<Integer> evenNumbers = IntStream.range(1, 20)
            .filter(n -> n % 2 == 0)
            .boxed()
            .collect(Collectors.toList());
        System.out.println("Even numbers: " + evenNumbers);
        
        // Primitive stream operations
        int[] numbers = {5, 2, 8, 1, 9, 3};
        int max = Arrays.stream(numbers).max().orElse(-1);
        int min = Arrays.stream(numbers).min().orElse(-1);
        double averageInt = Arrays.stream(numbers).average().orElse(0.0);
        
        System.out.println("Max: " + max + ", Min: " + min + ", Average: " + averageInt);
        
        // Summary statistics
        IntSummaryStatistics stats = Arrays.stream(numbers).summaryStatistics();
        System.out.println("Summary: " + stats);
        
        // Boxed streams for object operations
        List<Integer> boxed = IntStream.range(1, 6)
            .boxed()
            .collect(Collectors.toList());
        System.out.println("Boxed stream: " + boxed);
    }
    
    private static void demonstrateStreamBuilders() {
        System.out.println("\n=== Stream Builders ===");
        
        // Using Stream.Builder
        Stream.Builder<String> builder = Stream.builder();
        builder.add("First");
        builder.add("Second");
        builder.add("Third");
        Stream<String> builtStream = builder.build();
        
        List<String> result = builtStream.collect(Collectors.toList());
        System.out.println("Built stream: " + result);
        
        // Using iterate for infinite streams
        List<Integer> firstTenSquares = Stream.iterate(1, n -> n + 1)
            .map(n -> n * n)
            .limit(10)
            .collect(Collectors.toList());
        System.out.println("First 10 squares: " + firstTenSquares);
        
        // Using generate for infinite streams
        List<String> generated = Stream.generate(() -> "Item-" + System.currentTimeMillis())
            .limit(5)
            .collect(Collectors.toList());
        System.out.println("Generated items: " + generated);
        
        // Using of with multiple elements
        Stream<String> ofStream = Stream.of("One", "Two", "Three", "Four", "Five");
        List<String> ofResult = ofStream.collect(Collectors.toList());
        System.out.println("Stream.of result: " + ofResult);
        
        // Using concat to combine streams
        Stream<String> stream1 = Stream.of("A", "B", "C");
        Stream<String> stream2 = Stream.of("D", "E", "F");
        List<String> concatenated = Stream.concat(stream1, stream2)
            .collect(Collectors.toList());
        System.out.println("Concatenated: " + concatenated);
        
        // Using iterate with condition
        List<Integer> evenUntil20 = Stream.iterate(2, n -> n <= 20, n -> n + 2)
            .collect(Collectors.toList());
        System.out.println("Even numbers until 20: " + evenUntil20);
    }
}

// Supporting classes for examples
class Person {
    private String name;
    private int age;
    private String department;
    
    public Person(String name, int age, String department) {
        this.name = name;
        this.age = age;
        this.department = department;
    }
    
    public String getName() { return name; }
    public int getAge() { return age; }
    public String getDepartment() { return department; }
    
    @Override
    public String toString() {
        return name + "(" + age + ", " + department + ")";
    }
}

class Employee {
    private String name;
    private int age;
    
    public Employee(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() { return name; }
    public int getAge() { return age; }
    
    @Override
    public String toString() {
        return name + "(" + age + ")";
    }
}

class Department {
    private String name;
    private List<Employee> employees;
    
    public Department(String name, List<Employee> employees) {
        this.name = name;
        this.employees = employees;
    }
    
    public String getName() { return name; }
    public List<Employee> getEmployees() { return employees; }
    
    @Override
    public String toString() {
        return name + ": " + employees;
    }
}
```

## Optional Class

### Handling Null Values with Optional

The Optional class was introduced in Java 8 to provide a better approach to handling null values and avoiding NullPointerExceptions.

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class OptionalClass {
    public static void main(String[] args) {
        System.out.println("=== Optional Class ===");
        
        // Demonstrate Optional creation and basic operations
        demonstrateBasicOptionalOperations();
        
        // Demonstrate Optional chaining
        demonstrateOptionalChaining();
        
        // Demonstrate Optional with streams
        demonstrateOptionalWithStreams();
        
        // Demonstrate real-world Optional usage
        demonstrateRealWorldUsage();
    }
    
    private static void demonstrateBasicOptionalOperations() {
        System.out.println("\n=== Basic Optional Operations ===");
        
        // Creating Optional instances
        Optional<String> optionalWithValue = Optional.of("Hello World");
        Optional<String> optionalEmpty = Optional.empty();
        Optional<String> optionalNullable = Optional.ofNullable(null);
        Optional<String> optionalNotNullable = Optional.ofNullable("Not Null");
        
        System.out.println("Optional with value: " + optionalWithValue);
        System.out.println("Optional empty: " + optionalEmpty);
        System.out.println("Optional nullable (null): " + optionalNullable);
        System.out.println("Optional nullable (not null): " + optionalNotNullable);
        
        // Checking if Optional has value
        System.out.println("Has value (with value): " + optionalWithValue.isPresent());
        System.out.println("Has value (empty): " + optionalEmpty.isPresent());
        
        // Getting values safely
        System.out.println("Value with get(): " + optionalWithValue.get());
        
        // Using orElse for default values
        String defaultValue = optionalEmpty.orElse("Default Value");
        System.out.println("Default value: " + defaultValue);
        
        // Using orElseGet with Supplier (lazy evaluation)
        String lazyValue = optionalEmpty.orElseGet(() -> {
            System.out.println("Computing default value...");
            return "Lazy Default";
        });
        System.out.println("Lazy default value: " + lazyValue);
        
        // Using orElseThrow
        try {
            String thrownValue = optionalEmpty.orElseThrow(() -> new RuntimeException("Value not present"));
        } catch (RuntimeException e) {
            System.out.println("Caught exception: " + e.getMessage());
        }
        
        // Using ifPresent
        optionalWithValue.ifPresent(s -> System.out.println("Value present: " + s));
        optionalEmpty.ifPresent(s -> System.out.println("This won't print"));
    }
    
    private static void demonstrateOptionalChaining() {
        System.out.println("\n=== Optional Chaining ===");
        
        // Using map with Optional
        Optional<String> optionalString = Optional.of("hello world");
        Optional<String> upperCaseOptional = optionalString.map(String::toUpperCase);
        System.out.println("Uppercase: " + upperCaseOptional);
        
        Optional<String> emptyOptional = Optional.empty();
        Optional<String> emptyMapped = emptyOptional.map(String::toUpperCase);
        System.out.println("Empty mapped: " + emptyMapped);
        
        // Using flatMap to avoid nested Optionals
        Optional<String> nameOptional = Optional.of("Alice");
        Optional<Optional<String>> nestedOptional = nameOptional.map(this::processName);
        Optional<String> flattenedOptional = nameOptional.flatMap(this::processName);
        
        System.out.println("Nested Optional: " + nestedOptional);
        System.out.println("Flattened Optional: " + flattenedOptional);
        
        // Complex chaining
        Optional<String> complexChain = Optional.of("programming")
            .filter(s -> s.length() > 5)
            .map(String::toUpperCase)
            .map(s -> "Processed: " + s);
        System.out.println("Complex chain result: " + complexChain);
        
        // Filter with Optional
        Optional<String> filtered = Optional.of("java")
            .filter(s -> s.length() > 3)
            .filter(s -> s.startsWith("j"));
        System.out.println("Filtered: " + filtered);
        
        Optional<String> filteredEmpty = Optional.of("cpp")
            .filter(s -> s.length() > 3);
        System.out.println("Filtered empty: " + filteredEmpty);
        
        // Using Optional with method references
        Optional<String> methodRefExample = Optional.of("hello")
            .map(String::toUpperCase)
            .filter(s -> s.length() > 3);
        System.out.println("Method reference example: " + methodRefExample);
    }
    
    private static void demonstrateOptionalWithStreams() {
        System.out.println("\n=== Optional with Streams ===");
        
        // Filter out empty Optionals from a stream
        List<Optional<String>> optionalList = Arrays.asList(
            Optional.of("Hello"),
            Optional.empty(),
            Optional.of("World"),
            Optional.of("Java"),
            Optional.empty()
        );
        
        List<String> filteredValues = optionalList.stream()
            .filter(Optional::isPresent)
            .map(Optional::get)
            .collect(Collectors.toList());
        System.out.println("Filtered values: " + filteredValues);
        
        // Alternative using flatMap
        List<String> flatMappedValues = optionalList.stream()
            .flatMap(opt -> opt.isPresent() ? Stream.of(opt.get()) : Stream.empty())
            .collect(Collectors.toList());
        System.out.println("Flat mapped values: " + flatMappedValues);
        
        // Find first non-empty Optional
        Optional<String> firstNonEmpty = optionalList.stream()
            .filter(Optional::isPresent)
            .map(Optional::get)
            .findFirst();
        System.out.println("First non-empty: " + firstNonEmpty);
        
        // Using Optional in stream operations
        List<String> words = Arrays.asList("hello", "world", "java", "programming");
        
        Optional<String> longestWord = words.stream()
            .max(Comparator.comparing(String::length));
        System.out.println("Longest word: " + longestWord);
        
        // Optional with reduction
        Optional<String> concatenated = words.stream()
            .reduce((s1, s2) -> s1 + " " + s2);
        System.out.println("Concatenated: " + concatenated);
    }
    
    private static void demonstrateRealWorldUsage() {
        System.out.println("\n=== Real-World Optional Usage ===");
        
        // Simulate data access with potential nulls
        UserService userService = new UserService();
        OrderService orderService = new OrderService();
        
        // Get user and their most recent order
        Optional<User> user = userService.findUserById("user123");
        Optional<Order> recentOrder = user
            .flatMap(u -> orderService.getRecentOrderByUserId(u.getId()));
        
        System.out.println("Recent order: " + recentOrder.map(Order::getId).orElse("No recent order"));
        
        // Process user's favorite product
        Optional<String> favoriteProduct = user
            .flatMap(u -> Optional.ofNullable(u.getFavoriteProduct()))
            .map(String::toUpperCase)
            .filter(product -> product.length() > 3);
        
        System.out.println("Favorite product: " + favoriteProduct.orElse("None or invalid"));
        
        // Calculate average order amount for user
        Optional<Double> averageOrderAmount = user
            .flatMap(u -> orderService.getUserOrders(u.getId()))
            .filter(orders -> !orders.isEmpty())
            .map(orders -> orders.stream()
                .mapToDouble(Order::getAmount)
                .average()
                .orElse(0.0));
        
        System.out.println("Average order amount: " + averageOrderAmount.orElse(0.0));
        
        // Safe navigation through optional chains
        Optional<String> shippingAddress = user
            .map(User::getProfile)
            .map(Profile::getShippingAddress)
            .map(Address::getFullAddress);
        
        System.out.println("Shipping address: " + shippingAddress.orElse("Address not available"));
        
        // Using Optional in method returns
        Optional<String> result = findBestMatch(Arrays.asList("apple", "banana", "cherry"), "app");
        System.out.println("Best match: " + result.orElse("No match found"));
    }
    
    // Helper method for chaining demonstration
    private Optional<String> processName(String name) {
        if (name.length() > 3) {
            return Optional.of(name.toUpperCase());
        }
        return Optional.empty();
    }
    
    // Helper method for real-world usage
    private Optional<String> findBestMatch(List<String> candidates, String prefix) {
        return candidates.stream()
            .filter(s -> s.startsWith(prefix))
            .findFirst();
    }
}

// Supporting classes for real-world example
class UserService {
    public Optional<User> findUserById(String id) {
        if ("user123".equals(id)) {
            return Optional.of(new User(id, "Alice", "alice@example.com"));
        }
        return Optional.empty();
    }
}

class OrderService {
    public Optional<Order> getRecentOrderByUserId(String userId) {
        if ("user123".equals(userId)) {
            return Optional.of(new Order("order456", 99.99, "2023-12-01"));
        }
        return Optional.empty();
    }
    
    public Optional<List<Order>> getUserOrders(String userId) {
        if ("user123".equals(userId)) {
            return Optional.of(Arrays.asList(
                new Order("order1", 49.99, "2023-11-01"),
                new Order("order2", 149.99, "2023-11-15"),
                new Order("order3", 29.99, "2023-12-01")
            ));
        }
        return Optional.empty();
    }
}

class User {
    private String id;
    private String name;
    private String email;
    private String favoriteProduct;
    private Profile profile;
    
    public User(String id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.favoriteProduct = "laptop";
        this.profile = new Profile();
    }
    
    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public String getFavoriteProduct() { return favoriteProduct; }
    public Profile getProfile() { return profile; }
}

class Order {
    private String id;
    private double amount;
    private String date;
    
    public Order(String id, double amount, String date) {
        this.id = id;
        this.amount = amount;
        this.date = date;
    }
    
    public String getId() { return id; }
    public double getAmount() { return amount; }
    public String getDate() { return date; }
    
    @Override
    public String toString() {
        return "Order{id='" + id + "', amount=" + amount + ", date='" + date + "'}";
    }
}

class Profile {
    private Address shippingAddress;
    
    public Profile() {
        this.shippingAddress = new Address("123 Main St", "Anytown", "ST", "12345");
    }
    
    public Address getShippingAddress() { return shippingAddress; }
}

class Address {
    private String street;
    private String city;
    private String state;
    private String zipCode;
    
    public Address(String street, String city, String state, String zipCode) {
        this.street = street;
        this.city = city;
        this.state = state;
        this.zipCode = zipCode;
    }
    
    public String getFullAddress() {
        return street + ", " + city + ", " + state + " " + zipCode;
    }
}
```

## CompletableFuture and Asynchronous Programming

### Introduction to CompletableFuture

CompletableFuture was introduced in Java 8 to provide a robust framework for asynchronous programming, allowing developers to write non-blocking, asynchronous code more easily.

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.function.*;
import java.util.stream.*;

public class CompletableFutureExample {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        System.out.println("=== CompletableFuture ===");
        
        // Demonstrate basic CompletableFuture creation
        demonstrateBasicCompletableFuture();
        
        // Demonstrate chaining and composition
        demonstrateChainingComposition();
        
        // Demonstrate exception handling
        demonstrateExceptionHandling();
        
        // Demonstrate real-world async patterns
        demonstrateRealWorldPatterns();
    }
    
    private static void demonstrateBasicCompletableFuture() throws ExecutionException, InterruptedException {
        System.out.println("\n=== Basic CompletableFuture ===");
        
        // Create completed future
        CompletableFuture<String> completedFuture = CompletableFuture.completedFuture("Already done!");
        System.out.println("Completed future: " + completedFuture.get());
        
        // Run async task (no return value)
        CompletableFuture<Void> runFuture = CompletableFuture.runAsync(() -> {
            System.out.println("Running asynchronously: " + Thread.currentThread().getName());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            System.out.println("RunAsync completed");
        });
        
        // Supply async task (with return value)
        CompletableFuture<String> supplyFuture = CompletableFuture.supplyAsync(() -> {
            System.out.println("Supplying asynchronously: " + Thread.currentThread().getName());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "SupplyAsync result";
        });
        
        runFuture.get();
        String result = supplyFuture.get();
        System.out.println("SupplyAsync result: " + result);
        
        // Create future with custom executor
        ExecutorService executor = Executors.newFixedThreadPool(2);
        CompletableFuture<String> customExecutorFuture = CompletableFuture.supplyAsync(() -> {
            System.out.println("Running on custom executor: " + Thread.currentThread().getName());
            return "Custom executor result";
        }, executor);
        
        System.out.println("Custom executor result: " + customExecutorFuture.get());
        executor.shutdown();
    }
    
    private static void demonstrateChainingComposition() throws ExecutionException, InterruptedException {
        System.out.println("\n=== Chaining and Composition ===");
        
        // thenApply - transform result
        CompletableFuture<String> chainedFuture = CompletableFuture
            .supplyAsync(() -> {
                System.out.println("Step 1: Getting user ID");
                return "user-123";
            })
            .thenApply(userId -> {
                System.out.println("Step 2: Getting user details for " + userId);
                return "UserDetails{name='John Doe', id=" + userId + "}";
            })
            .thenApply(userDetails -> {
                System.out.println("Step 3: Formatting user details: " + userDetails);
                return userDetails.toUpperCase();
            });
        
        System.out.println("Chained result: " + chainedFuture.get());
        
        // thenCompose - flatMap equivalent
        CompletableFuture<String> composedFuture = CompletableFuture
            .supplyAsync(() -> {
                System.out.println("Getting order ID");
                return "order-456";
            })
            .thenCompose(orderId -> {
                // Returns another CompletableFuture
                return CompletableFuture.supplyAsync(() -> {
                    System.out.println("Getting order details for " + orderId);
                    return "Order{id=" + orderId + ", status=SHIPPED}";
                });
            });
        
        System.out.println("Composed result: " + composedFuture.get());
        
        // thenCombine - combine results from two futures
        CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(1000); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            System.out.println("Future 1 completed with 10");
            return 10;
        });
        
        CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(1500); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            System.out.println("Future 2 completed with 20");
            return 20;
        });
        
        CompletableFuture<Integer> combinedFuture = future1.thenCombine(future2, Integer::sum);
        System.out.println("Combined result (10 + 20): " + combinedFuture.get());
        
        // allOf - wait for all futures to complete
        CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(1000); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            return "Result 1";
        });
        
        CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(2000); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            return "Result 2";
        });
        
        CompletableFuture<String> f3 = CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(500); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            return "Result 3";
        });
        
        CompletableFuture<Void> allOfFuture = CompletableFuture.allOf(f1, f2, f3);
        allOfFuture.join(); // Wait for all to complete
        
        List<String> allResults = Arrays.asList(f1.get(), f2.get(), f3.get());
        System.out.println("All results: " + allResults);
        
        // anyOf - complete when any future completes
        CompletableFuture<Object> anyOfFuture = CompletableFuture.anyOf(f1, f2, f3);
        System.out.println("Any of result: " + anyOfFuture.get());
    }
    
    private static void demonstrateExceptionHandling() throws ExecutionException, InterruptedException {
        System.out.println("\n=== Exception Handling ===");
        
        // exceptionally - handle exceptions
        CompletableFuture<String> exceptionFuture = CompletableFuture
            .supplyAsync(() -> {
                throw new RuntimeException("Something went wrong!");
            })
            .thenApply(result -> "This won't execute")
            .exceptionally(throwable -> {
                System.out.println("Exception caught: " + throwable.getMessage());
                return "Default Value";
            });
        
        System.out.println("Exception handling result: " + exceptionFuture.get());
        
        // handle - more control over exception handling
        CompletableFuture<String> handleFuture = CompletableFuture
            .supplyAsync(() -> {
                throw new RuntimeException("Handle exception!");
            })
            .handle((result, throwable) -> {
                if (throwable != null) {
                    System.out.println("Handled exception: " + throwable.getMessage());
                    return "Handled Result";
                } else {
                    return result;
                }
            });
        
        System.out.println("Handle result: " + handleFuture.get());
        
        // whenComplete - perform action regardless of success/failure
        CompletableFuture<String> completeFuture = CompletableFuture
            .supplyAsync(() -> {
                if (Math.random() > 0.5) {
                    throw new RuntimeException("Random failure!");
                }
                return "Success result";
            })
            .whenComplete((result, throwable) -> {
                if (throwable != null) {
                    System.out.println("Completed with exception: " + throwable.getMessage());
                } else {
                    System.out.println("Completed successfully: " + result);
                }
            })
            .exceptionally(throwable -> "Fallback result");
        
        System.out.println("Complete future result: " + completeFuture.get());
    }
    
    private static void demonstrateRealWorldPatterns() throws ExecutionException, InterruptedException {
        System.out.println("\n=== Real-World Async Patterns ===");
        
        // Simulate an API call chain
        CompletableFuture<UserProfile> profileFuture = fetchUserAsync("user-123")
            .thenCompose(user -> fetchUserOrdersAsync(user.getId())
                .thenApply(orders -> new UserProfile(user, orders)));
        
        UserProfile profile = profileFuture.get();
        System.out.println("User Profile: " + profile);
        
        // Parallel execution of independent operations
        CompletableFuture<String> userInfoFuture = fetchUserInfoAsync("user-123");
        CompletableFuture<String> userPreferencesFuture = fetchUserPreferencesAsync("user-123");
        CompletableFuture<String> userActivityFuture = fetchUserActivityAsync("user-123");
        
        CompletableFuture<Void> allUserTasks = CompletableFuture.allOf(
            userInfoFuture, userPreferencesFuture, userActivityFuture);
        
        allUserTasks.join();
        
        String userInfo = userInfoFuture.get();
        String userPreferences = userPreferencesFuture.get();
        String userActivity = userActivityFuture.get();
        
        System.out.println("User Info: " + userInfo);
        System.out.println("User Preferences: " + userPreferences);
        System.out.println("User Activity: " + userActivity);
        
        // Timeout handling
        CompletableFuture<String> timeoutFuture = CompletableFuture
            .supplyAsync(() -> {
                try {
                    Thread.sleep(5000); // This will timeout
                    return "This won't return";
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return "Interrupted";
                }
            })
            .orTimeout(2, TimeUnit.SECONDS) // Timeout after 2 seconds
            .exceptionally(throwable -> {
                System.out.println("Timed out: " + throwable.getMessage());
                return "Timed Out Result";
            });
        
        System.out.println("Timeout result: " + timeoutFuture.get());
        
        // Retry pattern with CompletableFuture
        CompletableFuture<String> retryFuture = attemptOperationWithRetry(3);
        System.out.println("Retry result: " + retryFuture.get());
    }
    
    // Helper methods for real-world examples
    private static CompletableFuture<User> fetchUserAsync(String userId) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000); // Simulate network delay
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return new User(userId, "John Doe", "john@example.com");
        });
    }
    
    private static CompletableFuture<List<Order>> fetchUserOrdersAsync(String userId) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1500); // Simulate network delay
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return Arrays.asList(
                new Order("order-001", 99.99, "SHIPPED"),
                new Order("order-002", 149.50, "DELIVERED")
            );
        });
    }
    
    private static CompletableFuture<String> fetchUserInfoAsync(String userId) {
        return CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(500); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            return "User Info for " + userId;
        });
    }
    
    private static CompletableFuture<String> fetchUserPreferencesAsync(String userId) {
        return CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(800); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            return "Preferences for " + userId;
        });
    }
    
    private static CompletableFuture<String> fetchUserActivityAsync(String userId) {
        return CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(600); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            return "Activity for " + userId;
        });
    }
    
    private static CompletableFuture<String> attemptOperationWithRetry(int maxRetries) {
        CompletableFuture<String> future = new CompletableFuture<>();
        
        attemptOperationWithRetry(future, maxRetries, 0);
        return future;
    }
    
    private static void attemptOperationWithRetry(CompletableFuture<String> future, int maxRetries, int attempt) {
        if (attempt >= maxRetries) {
            future.completeExceptionally(new RuntimeException("Max retries exceeded"));
            return;
        }
        
        CompletableFuture.supplyAsync(() -> {
            if (Math.random() < 0.7 && attempt < maxRetries - 1) { // Simulate failure
                throw new RuntimeException("Simulated failure on attempt " + attempt);
            }
            return "Success on attempt " + attempt;
        }).handle((result, throwable) -> {
            if (throwable != null) {
                System.out.println("Attempt " + attempt + " failed: " + throwable.getMessage());
                // Retry after delay
                CompletableFuture.delayedExecutor(1, TimeUnit.SECONDS).execute(() -> 
                    attemptOperationWithRetry(future, maxRetries, attempt + 1));
            } else {
                future.complete(result);
            }
            return null;
        });
    }
}

// Supporting classes for CompletableFuture examples
class UserProfile {
    private User user;
    private List<Order> orders;
    
    public UserProfile(User user, List<Order> orders) {
        this.user = user;
        this.orders = orders;
    }
    
    @Override
    public String toString() {
        return "UserProfile{user=" + user + ", orders=" + orders + "}";
    }
}
```

## New Features in Java 9-21

### Java 9: Modules, Reactive Streams, and More

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.function.*;
import java.util.stream.*;

public class Java9PlusFeatures {
    public static void main(String[] args) {
        System.out.println("=== Java 9+ Features ===");
        
        // Demonstrate Java 9+ collection factory methods
        demonstrateCollectionFactoryMethods();
        
        // Demonstrate Java 9+ stream improvements
        demonstrateStreamImprovements();
        
        // Demonstrate Java 9+ Optional improvements
        demonstrateOptionalImprovements();
        
        // Demonstrate Java 10+ var keyword
        demonstrateVarKeyword();
        
        // Demonstrate Java 14+ record classes
        demonstrateRecordClasses();
        
        // Demonstrate Java 17+ sealed classes
        demonstrateSealedClasses();
    }
    
    private static void demonstrateCollectionFactoryMethods() {
        System.out.println("\n=== Java 9+ Collection Factory Methods ===");
        
        // Immutable List
        List<String> immutableList = List.of("Apple", "Banana", "Cherry");
        System.out.println("Immutable List: " + immutableList);
        
        // Immutable Set
        Set<String> immutableSet = Set.of("Red", "Green", "Blue");
        System.out.println("Immutable Set: " + immutableSet);
        
        // Immutable Map
        Map<String, Integer> immutableMap = Map.of("One", 1, "Two", 2, "Three", 3);
        System.out.println("Immutable Map: " + immutableMap);
        
        // Map with entries
        Map<String, String> mapWithEntries = Map.ofEntries(
            Map.entry("Key1", "Value1"),
            Map.entry("Key2", "Value2"),
            Map.entry("Key3", "Value3")
        );
        System.out.println("Map with entries: " + mapWithEntries);
        
        // Try to modify (will throw exception)
        try {
            immutableList.add("Orange"); // This will throw UnsupportedOperationException
        } catch (UnsupportedOperationException e) {
            System.out.println("Cannot modify immutable list: " + e.getMessage());
        }
    }
    
    private static void demonstrateStreamImprovements() {
        System.out.println("\n=== Java 9+ Stream Improvements ===");
        
        // takeWhile - take elements while condition is true
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        List<Integer> takenWhile = numbers.stream()
            .takeWhile(n -> n < 5)
            .collect(Collectors.toList());
        System.out.println("Take while < 5: " + takenWhile);
        
        // dropWhile - drop elements while condition is true
        List<Integer> droppedWhile = numbers.stream()
            .dropWhile(n -> n < 5)
            .collect(Collectors.toList());
        System.out.println("Drop while < 5: " + droppedWhile);
        
        // iterate with condition (Java 9+)
        List<Integer> evenNumbers = Stream.iterate(2, n -> n <= 20, n -> n + 2)
            .collect(Collectors.toList());
        System.out.println("Even numbers using iterate: " + evenNumbers);
        
        // ofNullable - create stream from potentially null value
        String[] array = {"Hello", null, "World"};
        List<String> processed = Arrays.stream(array)
            .flatMap(Stream::ofNullable)
            .collect(Collectors.toList());
        System.out.println("Processed with ofNullable: " + processed);
        
        // Stream.of with null handling
        List<String> safeStream = Stream.ofNullable("Hello")
            .collect(Collectors.toList());
        System.out.println("Safe stream: " + safeStream);
    }
    
    private static void demonstrateOptionalImprovements() {
        System.out.println("\n=== Java 9+ Optional Improvements ===");
        
        Optional<String> optional = Optional.of("Hello");
        
        // ifPresentOrElse
        optional.ifPresentOrElse(
            s -> System.out.println("Value present: " + s),
            () -> System.out.println("Value not present")
        );
        
        Optional<String> emptyOptional = Optional.empty();
        emptyOptional.ifPresentOrElse(
            s -> System.out.println("This won't print"),
            () -> System.out.println("Value not present (else block executed)")
        );
        
        // Optional.filter
        Optional<String> filtered = Optional.of("Hello World")
            .filter(s -> s.length() > 10);
        System.out.println("Filtered optional: " + filtered);
        
        Optional<String> filteredOut = Optional.of("Hi")
            .filter(s -> s.length() > 10);
        System.out.println("Filtered out optional: " + filteredOut);
        
        // Optional.or (for providing alternative Optional)
        Optional<String> result = Optional.empty()
            .or(() -> Optional.of("Fallback value"));
        System.out.println("Optional.or result: " + result);
        
        // Stream from Optional
        List<String> streamFromOptional = Optional.of("Hello")
            .stream()
            .collect(Collectors.toList());
        System.out.println("Stream from optional: " + streamFromOptional);
        
        List<String> emptyStreamFromOptional = Optional.empty()
            .stream()
            .collect(Collectors.toList());
        System.out.println("Empty stream from optional: " + emptyStreamFromOptional);
    }
    
    private static void demonstrateVarKeyword() {
        System.out.println("\n=== Java 10+ var Keyword ===");
        
        // var for local variables
        var message = "Hello from var!";
        var number = 42;
        var list = Arrays.asList("a", "b", "c");
        var map = new HashMap<String, Integer>();
        
        System.out.println("Message: " + message);
        System.out.println("Number: " + number);
        System.out.println("List: " + list);
        System.out.println("Map type: " + map.getClass().getSimpleName());
        
        // var with complex generics
        var complexList = new ArrayList<Map.Entry<String, List<Integer>>>();
        System.out.println("Complex list type: " + complexList.getClass().getSimpleName());
        
        // var in enhanced for loop
        var items = Arrays.asList("item1", "item2", "item3");
        for (var item : items) {
            System.out.println("Item: " + item);
        }
        
        // var in try-with-resources
        var resource = new CustomResource();
        try (resource) {
            System.out.println("Using var with try-with-resources");
        } catch (Exception e) {
            System.out.println("Exception: " + e.getMessage());
        }
        
        // var with lambda (inferred)
        var processor = (Function<String, Integer>) String::length;
        System.out.println("Processor result: " + processor.apply("Java"));
    }
    
    private static void demonstrateRecordClasses() {
        System.out.println("\n=== Java 14+ Record Classes ===");
        
        // Create record instances
        var person = new PersonRecord("Alice", 30);
        var address = new AddressRecord("123 Main St", "Anytown", "ST", "12345");
        var employee = new EmployeeRecord("Bob", 25, "Engineering", 75000.0);
        
        System.out.println("Person: " + person);
        System.out.println("Person name: " + person.name()); // Accessor method
        System.out.println("Person age: " + person.age());   // Accessor method
        
        System.out.println("Address: " + address);
        System.out.println("Employee: " + employee);
        
        // Record pattern matching (Java 19+ preview)
        Object obj = person;
        if (obj instanceof PersonRecord(String name, int age)) {
            System.out.println("Pattern matched - Name: " + name + ", Age: " + age);
        }
        
        // Record with custom constructor
        var customPoint = new PointRecord(5, 10) {
            public double distanceFromOrigin() {
                return Math.sqrt(x() * x() + y() * y());
            }
        };
        
        // Record with custom method
        var namedPoint = new NamedPointRecord(3, 4, "origin");
        System.out.println("Named point: " + namedPoint);
        System.out.println("Distance from origin: " + namedPoint.distanceFromOrigin());
        
        // Record equals and hashCode work automatically
        var person2 = new PersonRecord("Alice", 30);
        System.out.println("Records equal: " + person.equals(person2));
        System.out.println("Hash codes equal: " + (person.hashCode() == person2.hashCode()));
        
        // Record toString works automatically
        System.out.println("Person toString: " + person.toString());
    }
    
    private static void demonstrateSealedClasses() {
        System.out.println("\n=== Java 17+ Sealed Classes ===");
        
        // Create instances of sealed class hierarchy
        Shape circle = new Circle(5.0);
        Shape rectangle = new Rectangle(4.0, 6.0);
        Shape triangle = new Triangle(3.0, 4.0, 5.0);
        
        System.out.println("Circle area: " + circle.calculateArea());
        System.out.println("Rectangle area: " + rectangle.calculateArea());
        System.out.println("Triangle area: " + triangle.calculateArea());
        
        // Process shapes polymorphically
        List<Shape> shapes = Arrays.asList(circle, rectangle, triangle);
        shapes.forEach(shape -> {
            System.out.println("Shape " + shape.getClass().getSimpleName() + 
                             " area: " + shape.calculateArea());
        });
        
        // Switch expressions with sealed classes (exhaustive)
        for (Shape shape : shapes) {
            String description = switch (shape) {
                case Circle c -> "Circle with radius " + c.getRadius();
                case Rectangle r -> "Rectangle " + r.getLength() + "x" + r.getWidth();
                case Triangle t -> "Triangle with sides " + t.getSideA() + ", " + 
                                  t.getSideB() + ", " + t.getSideC();
            };
            System.out.println("Description: " + description);
        }
    }
}

// Record classes example
record PersonRecord(String name, int age) {}
record AddressRecord(String street, String city, String state, String zipCode) {}
record EmployeeRecord(String name, int age, String department, double salary) {}

// Record with custom method
record NamedPointRecord(double x, double y, String name) {
    public double distanceFromOrigin() {
        return Math.sqrt(x * x + y * y);
    }
}

// Sealed class hierarchy example
sealed abstract class Shape permits Circle, Rectangle, Triangle {
    public abstract double calculateArea();
}

final class Circle extends Shape {
    private final double radius;
    
    public Circle(double radius) {
        this.radius = radius;
    }
    
    public double getRadius() { return radius; }
    
    @Override
    public double calculateArea() {
        return Math.PI * radius * radius;
    }
}

final class Rectangle extends Shape {
    private final double length, width;
    
    public Rectangle(double length, double width) {
        this.length = length;
        this.width = width;
    }
    
    public double getLength() { return length; }
    public double getWidth() { return width; }
    
    @Override
    public double calculateArea() {
        return length * width;
    }
}

final class Triangle extends Shape {
    private final double sideA, sideB, sideC;
    
    public Triangle(double sideA, double sideB, double sideC) {
        this.sideA = sideA;
        this.sideB = sideB;
        this.sideC = sideC;
    }
    
    public double getSideA() { return sideA; }
    public double getSideB() { return sideB; }
    public double getSideC() { return sideC; }
    
    @Override
    public double calculateArea() {
        // Using Heron's formula
        double s = (sideA + sideB + sideC) / 2;
        return Math.sqrt(s * (s - sideA) * (s - sideB) * (s - sideC));
    }
}

// Custom resource for var demonstration
class CustomResource implements AutoCloseable {
    @Override
    public void close() throws Exception {
        System.out.println("Custom resource closed");
    }
}
```

## Pattern Matching and Switch Expressions

### Modern Pattern Matching Features

Java has introduced several pattern matching features that make code more concise and readable.

```java
import java.util.*;
import java.util.stream.*;

public class PatternMatchingFeatures {
    public static void main(String[] args) {
        System.out.println("=== Pattern Matching Features ===");
        
        // Demonstrate instanceof pattern matching
        demonstrateInstanceofPatternMatching();
        
        // Demonstrate switch expressions
        demonstrateSwitchExpressions();
        
        // Demonstrate record pattern matching
        demonstrateRecordPatternMatching();
        
        // Demonstrate null handling with pattern matching
        demonstrateNullHandling();
    }
    
    private static void demonstrateInstanceofPatternMatching() {
        System.out.println("\n=== Instanceof Pattern Matching ===");
        
        Object[] objects = {
            "Hello World",
            42,
            3.14,
            Arrays.asList(1, 2, 3),
            new PersonRecord("Alice", 30)
        };
        
        for (Object obj : objects) {
            // Traditional instanceof check
            if (obj instanceof String str) {
                System.out.println("String: " + str.toUpperCase() + " (length: " + str.length() + ")");
            } else if (obj instanceof Integer num) {
                System.out.println("Integer: " + num + " (square: " + (num * num) + ")");
            } else if (obj instanceof Double d) {
                System.out.println("Double: " + d + " (rounded: " + Math.round(d) + ")");
            } else if (obj instanceof List<?> list) {
                System.out.println("List: " + list + " (size: " + list.size() + ")");
            } else if (obj instanceof PersonRecord person) {
                System.out.println("Person: " + person.name() + " (age: " + person.age() + ")");
            } else {
                System.out.println("Other: " + obj);
            }
        }
        
        // Pattern matching with complex conditions
        processObject(new EmployeeRecord("Bob", 25, "Engineering", 75000.0));
        processObject("Just a string");
        processObject(100);
    }
    
    private static void processObject(Object obj) {
        if (obj instanceof String str && str.length() > 5) {
            System.out.println("Long string: " + str.substring(0, 5) + "...");
        } else if (obj instanceof Number num && num.doubleValue() > 50) {
            System.out.println("Large number: " + num);
        } else if (obj instanceof List<?> list && list.size() > 2) {
            System.out.println("Large list: " + list.subList(0, 2) + "...");
        } else {
            System.out.println("Other object: " + obj);
        }
    }
    
    private static void demonstrateSwitchExpressions() {
        System.out.println("\n=== Switch Expressions ===");
        
        // Simple switch expression
        Day day = Day.WEDNESDAY;
        String activity = switch (day) {
            case MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY -> "Work day";
            case SATURDAY, SUNDAY -> "Weekend";
        };
        System.out.println(day + " is a " + activity);
        
        // Switch with multiple operations
        int num = 5;
        String result = switch (num) {
            case 1, 2 -> "Small number";
            case 3, 4, 5 -> {
                System.out.println("Processing medium number: " + num);
                yield "Medium number";
            }
            case 6, 7, 8, 9 -> "Large number";
            default -> "Very large or small number";
        };
        System.out.println("Number " + num + " is " + result);
        
        // Switch with guard expressions (when clause)
        String status = "ACTIVE";
        int priority = 2;
        String processingResult = switch (status) {
            case "ACTIVE" when priority > 5 -> "High priority active";
            case "ACTIVE" when priority > 2 -> "Medium priority active";
            case "ACTIVE" -> "Low priority active";
            case "INACTIVE" -> "Inactive item";
            default -> "Unknown status";
        };
        System.out.println("Processing result: " + processingResult);
        
        // Switch with pattern matching
        Object value = "Hello World";
        String processedValue = switch (value) {
            case String str when str.length() > 10 -> "Long string: " + str;
            case String str -> "Short string: " + str;
            case Integer i when i > 100 -> "Large integer: " + i;
            case Integer i -> "Small integer: " + i;
            case null -> "Null value";
            default -> "Other type: " + value.getClass().getSimpleName();
        };
        System.out.println("Processed value: " + processedValue);
    }
    
    private static void demonstrateRecordPatternMatching() {
        System.out.println("\n=== Record Pattern Matching ===");
        
        // Create record instances
        Object[] records = {
            new PersonRecord("Alice", 30),
            new EmployeeRecord("Bob", 25, "Engineering", 75000.0),
            new AddressRecord("123 Main St", "Anytown", "ST", "12345")
        };
        
        for (Object obj : records) {
            String description = switch (obj) {
                case PersonRecord(String name, int age) -> 
                    "Person: " + name + ", Age: " + age;
                case EmployeeRecord(String name, int age, String dept, double salary) -> 
                    "Employee: " + name + ", Dept: " + dept + ", Salary: $" + salary;
                case AddressRecord(String street, String city, String state, String zip) -> 
                    "Address: " + street + ", " + city + ", " + state + " " + zip;
                default -> "Unknown record type";
            };
            System.out.println(description);
        }
        
        // Nested record patterns
        var company = new CompanyRecord("Tech Corp", 
            new EmployeeRecord("Charlie", 35, "Management", 95000.0));
        
        if (company instanceof CompanyRecord(String name, 
               EmployeeRecord(String empName, int empAge, String empDept, double empSalary))) {
            System.out.println("Company: " + name + ", Manager: " + empName + 
                             " in " + empDept + " department");
        }
    }
    
    private static void demonstrateNullHandling() {
        System.out.println("\n=== Null Handling with Pattern Matching ===");
        
        Object[] values = {"Hello", null, 42, "World", null};
        
        // Safe processing with pattern matching
        for (Object obj : values) {
            String result = switch (obj) {
                case null -> "Null value encountered";
                case String str when str.isEmpty() -> "Empty string";
                case String str -> "String: " + str;
                case Integer i -> "Integer: " + i;
                default -> "Other: " + obj;
            };
            System.out.println(result);
        }
        
        // Pattern matching in streams (safe processing)
        List<Object> mixedList = Arrays.asList("hello", null, 42, "world", null, 3.14);
        
        List<String> processed = mixedList.stream()
            .map(obj -> switch (obj) {
                case null -> "NULL";
                case String s -> s.toUpperCase();
                case Number n -> "NUMBER: " + n;
                default -> "OTHER: " + obj.toString();
            })
            .collect(Collectors.toList());
        
        System.out.println("Processed list: " + processed);
    }
    
    enum Day { MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY }
}

// Additional record for pattern matching demonstration
record CompanyRecord(String name, EmployeeRecord manager) {}
```

## Summary

Java 8+ features have revolutionized the Java programming language by introducing functional programming concepts, modern APIs, and more concise syntax. This guide covered:

1. **Lambda Expressions**: Enabling functional programming with concise anonymous functions
2. **Streams API**: Providing functional-style operations on sequences of elements
3. **Optional Class**: Better null handling and avoiding NullPointerException
4. **CompletableFuture**: Powerful asynchronous programming framework
5. **Java 9-21 Features**: Module system, collection factories, var keyword, records, sealed classes
6. **Pattern Matching**: Modern instanceof, switch expressions, and record patterns

Key takeaways:
- Lambda expressions and streams enable more functional, declarative programming styles
- Modern Java features promote code conciseness, readability, and maintainability
- Asynchronous programming with CompletableFuture enables non-blocking, scalable applications
- New language features like records and sealed classes provide better data modeling
- Pattern matching features make type checking and data extraction more elegant

Mastering these modern Java features is essential for writing contemporary, efficient, and maintainable Java applications.