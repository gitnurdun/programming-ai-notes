# Java Lambda Expressions and Functional Programming: Complete Guide

## Introduction

Java 8 introduced lambda expressions as part of its functional programming capabilities, revolutionizing how developers write Java code. Lambda expressions provide a concise way to express instances of single-method interfaces and enable functional programming paradigms in Java.

This guide will explore lambda expressions in depth, covering their syntax, usage patterns, and integration with Java's functional interfaces and streams API.

## Understanding Lambda Expressions

### What Are Lambda Expressions?

A lambda expression is an anonymous function that can be passed around as a value. It allows you to treat functionality as a method argument or code as data.

### Basic Syntax

Lambda expressions follow this syntax:
```
(parameters) -> expression
(parameters) -> { statements; }
```

### Simple Lambda Examples

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class LambdaBasics {
    public static void main(String[] args) {
        // Lambda expression implementing Runnable
        Runnable runnable = () -> System.out.println("Hello from lambda!");
        runnable.run();
        
        // Lambda with parameters
        BinaryOperator<Integer> add = (a, b) -> a + b;
        System.out.println("5 + 3 = " + add.apply(5, 3));
        
        // Lambda with multiple statements
        Consumer<String> printer = (str) -> {
            String upper = str.toUpperCase();
            System.out.println("Original: " + str);
            System.out.println("Uppercase: " + upper);
        };
        printer.accept("hello world");
        
        // Lambda with type inference
        Function<String, Integer> stringLength = s -> s.length();
        System.out.println("Length of 'Java': " + stringLength.apply("Java"));
    }
}
```

## Functional Interfaces

### Understanding Functional Interfaces

A functional interface is an interface that contains exactly one abstract method. These interfaces are the foundation for lambda expressions.

```java
import java.util.function.*;
import java.time.LocalDate;

// Custom functional interface
@FunctionalInterface
public interface Calculator {
    double calculate(double a, double b);
    
    // Default methods are allowed
    default void printResult(double result) {
        System.out.println("Result: " + result);
    }
    
    // Static methods are allowed
    static Calculator getAdditionCalculator() {
        return (a, b) -> a + b;
    }
}

// More examples of custom functional interfaces
@FunctionalInterface
public interface StringProcessor {
    String process(String input);
    
    default String processAndLog(String input) {
        String result = process(input);
        System.out.println("Processed: " + result);
        return result;
    }
}

@FunctionalInterface
public interface DateValidator {
    boolean isValid(LocalDate date);
    
    default void validateAndLog(LocalDate date) {
        boolean valid = isValid(date);
        System.out.println("Date " + date + " is " + (valid ? "valid" : "invalid"));
    }
}

public class FunctionalInterfaceDemo {
    public static void main(String[] args) {
        // Using custom functional interfaces
        Calculator addition = (a, b) -> a + b;
        Calculator multiplication = (a, b) -> a * b;
        
        double addResult = addition.calculate(10, 5);
        addition.printResult(addResult);
        
        double multResult = multiplication.calculate(10, 5);
        multiplication.printResult(multResult);
        
        // Using predefined functional interfaces
        Predicate<String> isEmpty = str -> str.isEmpty();
        System.out.println("Is empty: " + isEmpty.test(""));
        
        Function<String, Integer> lengthFunc = String::length;
        System.out.println("Length: " + lengthFunc.apply("Hello"));
        
        Consumer<String> printer = System.out::println;
        printer.accept("Hello using method reference");
        
        Supplier<String> randomString = () -> "Random: " + Math.random();
        System.out.println(randomString.get());
    }
}
```

## Predefined Functional Interfaces

### Supplier Interface

```java
import java.util.function.*;
import java.util.*;
import java.time.LocalDateTime;

public class SupplierExamples {
    public static void main(String[] args) {
        // Basic Supplier examples
        Supplier<String> currentTime = () -> LocalDateTime.now().toString();
        Supplier<Integer> randomInt = () -> (int) (Math.random() * 100);
        Supplier<List<String>> emptyList = ArrayList::new;
        
        System.out.println("Current time: " + currentTime.get());
        System.out.println("Random number: " + randomInt.get());
        System.out.println("Empty list: " + emptyList.get());
        
        // Lazy initialization using Supplier
        LazyInitializer lazy = new LazyInitializer();
        System.out.println("Value: " + lazy.getValue());
        System.out.println("Value (cached): " + lazy.getValue());
    }
    
    static class LazyInitializer {
        private String value;
        private Supplier<String> expensiveOperation = () -> {
            System.out.println("Performing expensive operation...");
            return "Expensive Result";
        };
        
        public String getValue() {
            if (value == null) {
                value = expensiveOperation.get();
            }
            return value;
        }
    }
}
```

### Consumer Interface

```java
import java.util.function.*;
import java.util.*;

public class ConsumerExamples {
    public static void main(String[] args) {
        // Basic Consumer examples
        Consumer<String> printer = s -> System.out.println("Print: " + s);
        printer.accept("Hello World");
        
        Consumer<Integer> squarePrinter = i -> System.out.println(i + " squared is " + (i * i));
        squarePrinter.accept(5);
        
        // Chaining Consumers
        Consumer<String> toUpperCase = s -> System.out.println("Upper: " + s.toUpperCase());
        Consumer<String> toLowerCase = s -> System.out.println("Lower: " + s.toLowerCase());
        
        // Using andThen to chain consumers
        Consumer<String> upperAndLower = toUpperCase.andThen(toLowerCase);
        upperAndLower.accept("Hello");
        
        // Using consumers with collections
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "Diana");
        
        // For each with consumer
        names.forEach(name -> System.out.println("Name: " + name));
        
        // Custom consumer for formatting
        Consumer<String> formatter = name -> {
            String formatted = name.substring(0, 1).toUpperCase() + name.substring(1).toLowerCase();
            System.out.println("Formatted: " + formatted);
        };
        names.forEach(formatter);
    }
}
```

### Predicate Interface

```java
import java.util.function.*;
import java.util.*;
import java.util.stream.Collectors;

public class PredicateExamples {
    public static void main(String[] args) {
        // Basic Predicate examples
        Predicate<String> isLong = s -> s.length() > 10;
        Predicate<Integer> isEven = n -> n % 2 == 0;
        Predicate<String> isEmpty = String::isEmpty;
        
        System.out.println("Is 'VeryLongString' long? " + isLong.test("VeryLongString"));
        System.out.println("Is 4 even? " + isEven.test(4));
        System.out.println("Is '' empty? " + isEmpty.test(""));
        
        // Chaining predicates
        Predicate<String> isNotBlank = s -> !s.trim().isEmpty();
        Predicate<String> hasSpaces = s -> s.contains(" ");
        
        // and(), or(), negate() methods
        Predicate<String> validAndHasSpaces = isNotBlank.and(hasSpaces);
        Predicate<String> shortOrEmpty = isLong.negate().or(isEmpty);
        
        System.out.println("Valid and has spaces: " + validAndHasSpaces.test("Hello World"));
        System.out.println("Short or empty: " + shortOrEmpty.test("Short"));
        
        // Using predicates with collections
        List<String> words = Arrays.asList("", "Hello", "World", "Java", "Programming", " ");
        
        // Filter with predicate
        List<String> longWords = words.stream()
            .filter(isLong)
            .collect(Collectors.toList());
        System.out.println("Long words: " + longWords);
        
        // Filter with complex predicate
        List<String> validWords = words.stream()
            .filter(isNotBlank.and(s -> s.length() > 2))
            .collect(Collectors.toList());
        System.out.println("Valid words: " + validWords);
        
        // Custom validation using predicates
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        List<Integer> evenNumbers = filterByPredicate(numbers, isEven);
        System.out.println("Even numbers: " + evenNumbers);
        
        List<Integer> oddNumbers = filterByPredicate(numbers, isEven.negate());
        System.out.println("Odd numbers: " + oddNumbers);
    }
    
    // Generic method to filter collection by predicate
    public static <T> List<T> filterByPredicate(List<T> list, Predicate<T> predicate) {
        return list.stream()
            .filter(predicate)
            .collect(Collectors.toList());
    }
}
```

### Function Interface

```java
import java.util.function.*;
import java.util.*;
import java.util.stream.Collectors;

public class FunctionExamples {
    public static void main(String[] args) {
        // Basic Function examples
        Function<String, Integer> stringToInt = Integer::parseInt;
        Function<Integer, String> intToString = i -> "Number: " + i;
        
        System.out.println("Parsed: " + stringToInt.apply("123"));
        System.out.println("Formatted: " + intToString.apply(456));
        
        // Chaining functions
        Function<String, String> addPrefix = s -> "Hello " + s;
        Function<String, String> addSuffix = s -> s + "!";
        Function<String, String> addBoth = addPrefix.andThen(addSuffix);
        
        System.out.println("Chained: " + addBoth.apply("World"));
        
        // Compose functions (applies the argument to the second function first)
        Function<String, String> composeExample = addSuffix.compose(addPrefix);
        System.out.println("Composed: " + composeExample.apply("World"));
        
        // Using functions with maps
        Map<String, Integer> scores = new HashMap<>();
        scores.put("Alice", 85);
        scores.put("Bob", 92);
        scores.put("Charlie", 78);
        
        // Transform map values using function
        Map<String, String> gradeMap = scores.entrySet().stream()
            .collect(Collectors.toMap(
                Map.Entry::getKey,
                entry -> convertScoreToGrade(entry.getValue())
            ));
        
        System.out.println("Grade map: " + gradeMap);
        
        // More complex function example
        Function<List<String>, List<String>> processList = list -> list.stream()
            .filter(s -> s != null && !s.trim().isEmpty())
            .map(String::trim)
            .map(String::toLowerCase)
            .collect(Collectors.toList());
        
        List<String> inputList = Arrays.asList("  HELLO  ", "", "  World  ", null, "  JAVA  ");
        List<String> processed = processList.apply(inputList);
        System.out.println("Processed list: " + processed);
    }
    
    public static String convertScoreToGrade(Integer score) {
        if (score >= 90) return "A";
        else if (score >= 80) return "B";
        else if (score >= 70) return "C";
        else if (score >= 60) return "D";
        else return "F";
    }
}
```

### BinaryOperator and Other Specialized Interfaces

```java
import java.util.function.*;
import java.util.*;

public class BinaryOperatorExamples {
    public static void main(String[] args) {
        // BinaryOperator examples
        BinaryOperator<Integer> add = Integer::sum;
        BinaryOperator<Integer> multiply = (a, b) -> a * b;
        BinaryOperator<String> concatenate = String::concat;
        
        System.out.println("Add: " + add.apply(10, 5));
        System.out.println("Multiply: " + multiply.apply(10, 5));
        System.out.println("Concatenate: " + concatenate.apply("Hello", " World"));
        
        // Min and max operators
        BinaryOperator<Integer> min = BinaryOperator.minBy(Integer::compareTo);
        BinaryOperator<Integer> max = BinaryOperator.maxBy(Integer::compareTo);
        
        System.out.println("Min: " + min.apply(10, 5));
        System.out.println("Max: " + max.apply(10, 5));
        
        // Using BinaryOperator with reduce
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
        
        Optional<Integer> sum = numbers.stream().reduce(add);
        Optional<Integer> product = numbers.stream().reduce(multiply);
        
        System.out.println("Sum: " + sum.orElse(0));
        System.out.println("Product: " + product.orElse(1));
        
        // UnaryOperator (extends Function<T, T>)
        UnaryOperator<String> toUpperCase = String::toUpperCase;
        UnaryOperator<String> addBrackets = s -> "[" + s + "]";
        
        String result = toUpperCase.andThen(addBrackets).apply("hello");
        System.out.println("Unary result: " + result);
        
        // IntBinaryOperator for primitive integers
        IntBinaryOperator intAdd = (a, b) -> a + b;
        IntBinaryOperator intMax = Math::max;
        
        System.out.println("Int add: " + intAdd.applyAsInt(10, 5));
        System.out.println("Int max: " + intMax.applyAsInt(10, 5));
    }
}
```

## Advanced Lambda Features

### Method References

```java
import java.util.function.*;
import java.util.*;
import java.util.stream.Collectors;

public class MethodReferenceExamples {
    public static void main(String[] args) {
        // Static method references
        Function<String, Integer> parseInt = Integer::parseInt;
        BiFunction<String, Integer, Integer> parseIntRadix = Integer::parseInt;
        
        System.out.println("Parsed: " + parseInt.apply("123"));
        System.out.println("Parsed (base 16): " + parseIntRadix.apply("FF", 16));
        
        // Instance method references of a particular object
        String prefix = "Hello ";
        Function<String, String> addPrefix = prefix::concat;
        System.out.println("With prefix: " + addPrefix.apply("World"));
        
        // Instance method references of an arbitrary object of a particular type
        Function<String, String> upperCase = String::toUpperCase;
        Function<String, Integer> stringLength = String::length;
        
        List<String> words = Arrays.asList("hello", "world", "java", "lambda");
        List<String> upperWords = words.stream()
            .map(upperCase)
            .collect(Collectors.toList());
        System.out.println("Uppercase words: " + upperWords);
        
        List<Integer> lengths = words.stream()
            .map(stringLength)
            .collect(Collectors.toList());
        System.out.println("Lengths: " + lengths);
        
        // Constructor references
        Supplier<List<String>> listSupplier = ArrayList::new;
        Function<String, StringBuilder> stringBuilderFunction = StringBuilder::new;
        
        List<String> newList = listSupplier.get();
        StringBuilder sb = stringBuilderFunction.apply("Initial");
        
        System.out.println("New list: " + newList);
        System.out.println("StringBuilder: " + sb);
        
        // Complex example with method references
        List<Person> people = Arrays.asList(
            new Person("Alice", 25),
            new Person("Bob", 30),
            new Person("Charlie", 35)
        );
        
        // Sort by age using method reference
        people.sort(Comparator.comparing(Person::getAge));
        System.out.println("Sorted by age: " + people);
        
        // Create a map using method references
        Map<String, Person> personMap = people.stream()
            .collect(Collectors.toMap(Person::getName, person -> person));
        System.out.println("Person map: " + personMap);
    }
}

class Person {
    private String name;
    private int age;
    
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() { return name; }
    public int getAge() { return age; }
    public void setName(String name) { this.name = name; }
    public void setAge(int age) { this.age = age; }
    
    @Override
    public String toString() {
        return name + "(" + age + ")";
    }
}
```

### Variable Capture and Closures

```java
import java.util.function.*;

public class LambdaClosures {
    private int instanceVar = 10;
    
    public void demonstrateClosures() {
        int localVar = 20; // Effectively final
        final int finalLocalVar = 30;
        
        // Lambda capturing instance variable
        IntSupplier instanceCapture = () -> this.instanceVar;
        
        // Lambda capturing local variable
        IntSupplier localCapture = () -> localVar;
        
        // Lambda capturing final local variable
        IntSupplier finalLocalCapture = () -> finalLocalVar;
        
        System.out.println("Instance var: " + instanceCapture.getAsInt());
        System.out.println("Local var: " + localCapture.getAsInt());
        System.out.println("Final local var: " + finalLocalCapture.getAsInt());
        
        // Method parameter capture
        processNumber(42, n -> n * 2);
        
        // Nested lambda with variable capture
        Function<Integer, IntSupplier> createSupplier = x -> () -> x * 2;
        IntSupplier supplier = createSupplier.apply(100);
        System.out.println("Nested capture: " + supplier.getAsInt());
    }
    
    public void processNumber(int number, IntFunction<Integer> processor) {
        System.out.println("Processed: " + processor.apply(number));
    }
    
    public void demonstrateScope() {
        int x = 1;
        
        IntFunction<IntFunction<Integer>> nested = a -> b -> a + b + x;
        
        IntFunction<Integer> inner = nested.apply(10);
        System.out.println("Nested scope: " + inner.apply(5)); // 10 + 5 + 1 = 16
    }
    
    public static void main(String[] args) {
        LambdaClosures demo = new LambdaClosures();
        demo.demonstrateClosures();
        demo.demonstrateScope();
    }
}
```

## Streams API Integration

### Stream Operations with Lambda

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class StreamLambdaIntegration {
    public static void main(String[] args) {
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        // Filter, map, and collect operations
        List<Integer> evenSquares = numbers.stream()
            .filter(n -> n % 2 == 0)           // Predicate
            .map(n -> n * n)                   // Function
            .collect(Collectors.toList());
        System.out.println("Even squares: " + evenSquares);
        
        // Reduce operation
        Optional<Integer> sum = numbers.stream()
            .reduce(Integer::sum);             // BinaryOperator
        System.out.println("Sum: " + sum.orElse(0));
        
        // Complex stream operations
        List<String> words = Arrays.asList("hello", "world", "java", "stream", "api");
        
        // Count words with specific conditions
        long longWords = words.stream()
            .filter(word -> word.length() > 4)
            .count();
        System.out.println("Long words count: " + longWords);
        
        // Find first word with specific condition
        Optional<String> firstLong = words.stream()
            .filter(word -> word.length() > 4)
            .findFirst();
        System.out.println("First long word: " + firstLong.orElse("None"));
        
        // Transform and collect to different collection
        Set<String> upperCaseWords = words.stream()
            .map(String::toUpperCase)
            .collect(Collectors.toSet());
        System.out.println("Uppercase set: " + upperCaseWords);
        
        // Group by length
        Map<Integer, List<String>> groupedByLength = words.stream()
            .collect(Collectors.groupingBy(String::length));
        System.out.println("Grouped by length: " + groupedByLength);
        
        // Complex example: Student processing
        List<Student> students = Arrays.asList(
            new Student("Alice", 85, "Math"),
            new Student("Bob", 92, "Science"),
            new Student("Charlie", 78, "Math"),
            new Student("Diana", 95, "Science"),
            new Student("Eve", 88, "Math")
        );
        
        // Average grade by subject
        Map<String, Double> avgGradeBySubject = students.stream()
            .collect(Collectors.groupingBy(
                Student::getSubject,
                Collectors.averagingInt(Student::getGrade)
            ));
        System.out.println("Average grades by subject: " + avgGradeBySubject);
        
        // Top student in each subject
        Map<String, Optional<Student>> topStudentBySubject = students.stream()
            .collect(Collectors.groupingBy(
                Student::getSubject,
                Collectors.maxBy(Comparator.comparingInt(Student::getGrade))
            ));
        System.out.println("Top students by subject: " + topStudentBySubject);
        
        // Students with grade above 90
        List<String> excellentStudents = students.stream()
            .filter(student -> student.getGrade() > 90)
            .map(Student::getName)
            .collect(Collectors.toList());
        System.out.println("Excellent students: " + excellentStudents);
    }
}

class Student {
    private String name;
    private int grade;
    private String subject;
    
    public Student(String name, int grade, String subject) {
        this.name = name;
        this.grade = grade;
        this.subject = subject;
    }
    
    public String getName() { return name; }
    public int getGrade() { return grade; }
    public String getSubject() { return subject; }
    
    @Override
    public String toString() {
        return name + "(" + grade + ", " + subject + ")";
    }
}
```

### Parallel Streams and Performance

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.*;

public class ParallelStreamExamples {
    public static void main(String[] args) {
        // Create a large list for demonstration
        List<Integer> largeList = new ArrayList<>();
        for (int i = 0; i < 1000000; i++) {
            largeList.add(i);
        }
        
        // Sequential stream timing
        long startSeq = System.currentTimeMillis();
        long evenCountSeq = largeList.stream()
            .filter(n -> n % 2 == 0)
            .count();
        long seqTime = System.currentTimeMillis() - startSeq;
        
        System.out.println("Sequential stream - Even numbers: " + evenCountSeq + ", Time: " + seqTime + "ms");
        
        // Parallel stream timing
        long startPar = System.currentTimeMillis();
        long evenCountPar = largeList.parallelStream()
            .filter(n -> n % 2 == 0)
            .count();
        long parTime = System.currentTimeMillis() - startPar;
        
        System.out.println("Parallel stream - Even numbers: " + evenCountPar + ", Time: " + parTime + "ms");
        
        // Demonstrating parallel processing with side effects (not recommended)
        List<String> results = Collections.synchronizedList(new ArrayList<>());
        
        largeList.parallelStream()
            .filter(n -> n % 100000 == 0)  // Only process every 100,000th number
            .forEach(n -> results.add("Processed: " + n));
        
        System.out.println("Parallel results count: " + results.size());
        
        // Safe parallel processing example
        String sentence = "the quick brown fox jumps over the lazy dog";
        String[] words = sentence.split("\\s+");
        
        Map<String, Long> wordCount = Arrays.stream(words)
            .parallel()
            .collect(Collectors.groupingByConcurrent(
                String::toLowerCase,
                Collectors.counting()
            ));
        
        System.out.println("Word counts: " + wordCount);
        
        // Custom parallel processing with CompletableFuture
        List<String> urls = Arrays.asList(
            "http://example.com/1",
            "http://example.com/2",
            "http://example.com/3",
            "http://example.com/4"
        );
        
        List<CompletableFuture<String>> futures = urls.stream()
            .map(url -> CompletableFuture.supplyAsync(() -> {
                // Simulate web request
                try {
                    Thread.sleep(100); // Simulate network delay
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                return "Response from " + url + " at " + System.currentTimeMillis();
            }))
            .collect(Collectors.toList());
        
        List<String> responses = futures.stream()
            .map(CompletableFuture::join)  // Wait for all completions
            .collect(Collectors.toList());
        
        System.out.println("Async responses: " + responses);
    }
}
```

## Functional Programming Patterns

### Chain of Responsibility Pattern

```java
import java.util.function.*;

public class ChainOfResponsibility {
    @FunctionalInterface
    public interface Handler<T> {
        boolean handle(T request);
        
        default Handler<T> andThen(Handler<T> next) {
            return request -> this.handle(request) || next.handle(request);
        }
    }
    
    public static void main(String[] args) {
        // Create handlers using lambda expressions
        Handler<String> emailHandler = request -> {
            if (request.contains("@") && request.contains(".")) {
                System.out.println("Email handler processed: " + request);
                return true;
            }
            return false;
        };
        
        Handler<String> phoneHandler = request -> {
            if (request.matches("\\d{3}-\\d{3}-\\d{4}")) {
                System.out.println("Phone handler processed: " + request);
                return true;
            }
            return false;
        };
        
        Handler<String> defaultHandler = request -> {
            System.out.println("Default handler processed: " + request);
            return true;
        };
        
        // Chain handlers together
        Handler<String> chain = emailHandler
            .andThen(phoneHandler)
            .andThen(defaultHandler);
        
        // Test the chain
        chain.handle("user@example.com");
        chain.handle("123-456-7890");
        chain.handle("Plain text");
    }
}
```

### Strategy Pattern

```java
import java.util.function.*;

public class StrategyPattern {
    // Strategy interface
    @FunctionalInterface
    public interface SortStrategy {
        void sort(int[] array);
    }
    
    public static void main(String[] args) {
        // Different sorting strategies as lambda expressions
        SortStrategy bubbleSort = array -> {
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
            System.out.println("Bubble sorted: " + java.util.Arrays.toString(array));
        };
        
        SortStrategy quickSort = array -> {
            quickSortHelper(array, 0, array.length - 1);
            System.out.println("Quick sorted: " + java.util.Arrays.toString(array));
        };
        
        SortStrategy insertionSort = array -> {
            for (int i = 1; i < array.length; i++) {
                int key = array[i];
                int j = i - 1;
                while (j >= 0 && array[j] > key) {
                    array[j + 1] = array[j];
                    j--;
                }
                array[j + 1] = key;
            }
            System.out.println("Insertion sorted: " + java.util.Arrays.toString(array));
        };
        
        // Context class to use different strategies
        SortContext context = new SortContext();
        
        int[] array1 = {64, 34, 25, 12, 22, 11, 90};
        int[] array2 = {64, 34, 25, 12, 22, 11, 90};
        int[] array3 = {64, 34, 25, 12, 22, 11, 90};
        
        context.setStrategy(bubbleSort);
        context.executeSort(array1);
        
        context.setStrategy(quickSort);
        context.executeSort(array2);
        
        context.setStrategy(insertionSort);
        context.executeSort(array3);
    }
    
    private static void quickSortHelper(int[] array, int low, int high) {
        if (low < high) {
            int pi = partition(array, low, high);
            quickSortHelper(array, low, pi - 1);
            quickSortHelper(array, pi + 1, high);
        }
    }
    
    private static int partition(int[] array, int low, int high) {
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
    
    static class SortContext {
        private SortStrategy strategy;
        
        public void setStrategy(SortStrategy strategy) {
            this.strategy = strategy;
        }
        
        public void executeSort(int[] array) {
            strategy.sort(array);
        }
    }
}
```

### Observer Pattern

```java
import java.util.*;
import java.util.function.*;

public class ObserverPattern {
    public static class EventPublisher<T> {
        private List<Consumer<T>> observers = new ArrayList<>();
        
        public void addObserver(Consumer<T> observer) {
            observers.add(observer);
        }
        
        public void removeObserver(Consumer<T> observer) {
            observers.remove(observer);
        }
        
        public void notifyObservers(T event) {
            observers.forEach(observer -> observer.accept(event));
        }
    }
    
    public static void main(String[] args) {
        EventPublisher<String> publisher = new EventPublisher<>();
        
        // Add different observers using lambda expressions
        publisher.addObserver(event -> System.out.println("Logger: " + event));
        publisher.addObserver(event -> {
            if (event.contains("ERROR")) {
                System.out.println("Alert: Critical error detected - " + event);
            }
        });
        publisher.addObserver(event -> System.out.println("Monitor: Event received at " + System.currentTimeMillis()));
        
        // Publish events
        publisher.notifyObservers("INFO: Application started");
        publisher.notifyObservers("ERROR: Database connection failed");
        publisher.notifyObservers("DEBUG: Processing user request");
    }
}
```

## Advanced Functional Programming Concepts

### Monads and Optional

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class OptionalAndMonads {
    public static void main(String[] args) {
        // Using Optional to handle null values gracefully
        Optional<String> optionalValue = Optional.of("Hello World");
        Optional<String> emptyOptional = Optional.empty();
        
        // map, flatMap, filter with Optional
        String result = optionalValue
            .filter(s -> s.length() > 5)
            .map(String::toUpperCase)
            .orElse("Default Value");
        System.out.println("Optional result: " + result);
        
        // Chaining with flatMap
        Optional<String> chained = Optional.of("data.txt")
            .flatMap(OptionalAndMonads::readFile)
            .flatMap(OptionalAndMonads::parseContent)
            .map(String::toUpperCase);
        
        System.out.println("Chained result: " + chained.orElse("No data"));
        
        // Using Optional in business logic
        List<User> users = Arrays.asList(
            new User("Alice", Optional.of("alice@example.com")),
            new User("Bob", Optional.empty()),
            new User("Charlie", Optional.of("charlie@example.com"))
        );
        
        // Find first user with valid email
        Optional<String> firstValidEmail = users.stream()
            .map(User::getEmail)
            .filter(Optional::isPresent)
            .map(Optional::get)
            .findFirst();
        
        System.out.println("First valid email: " + firstValidEmail.orElse("No valid email found"));
        
        // Complex Optional usage
        Optional<Integer> complexResult = Optional.of("123")
            .filter(s -> s.length() > 2)
            .map(Integer::parseInt)
            .filter(n -> n > 100)
            .map(n -> n * 2);
        
        System.out.println("Complex result: " + complexResult.orElse(0));
    }
    
    // Simulate file reading that might return null
    public static Optional<String> readFile(String filename) {
        // Simulate file reading
        if (filename != null && filename.endsWith(".txt")) {
            return Optional.of("File content for " + filename);
        }
        return Optional.empty();
    }
    
    // Simulate content parsing
    public static Optional<String> parseContent(String content) {
        if (content != null && content.contains("File content")) {
            return Optional.of(content.substring(content.indexOf("File content")));
        }
        return Optional.empty();
    }
}

class User {
    private String name;
    private Optional<String> email;
    
    public User(String name, Optional<String> email) {
        this.name = name;
        this.email = email;
    }
    
    public String getName() { return name; }
    public Optional<String> getEmail() { return email; }
}
```

### Functional Composition and Higher-Order Functions

```java
import java.util.function.*;

public class FunctionalComposition {
    public static void main(String[] args) {
        // Higher-order function: function that returns another function
        Function<Integer, Function<Integer, Integer>> createMultiplier = factor -> 
            operand -> operand * factor;
        
        Function<Integer, Integer> multiplyBy5 = createMultiplier.apply(5);
        System.out.println("5 * 10 = " + multiplyBy5.apply(10));
        
        // Function that takes other functions as parameters
        Function<Function<Integer, Integer>, Function<Integer, Integer>> addLogging = 
            func -> x -> {
                System.out.println("Input: " + x);
                int result = func.apply(x);
                System.out.println("Output: " + result);
                return result;
            };
        
        Function<Integer, Integer> loggedIncrement = addLogging.apply(x -> x + 1);
        loggedIncrement.apply(5);
        
        // Currying example
        Function<Integer, Function<Integer, Function<Integer, Integer>>> curryAdd = 
            a -> b -> c -> a + b + c;
        
        Function<Integer, Function<Integer, Integer>> add10 = curryAdd.apply(10);
        Function<Integer, Integer> add15 = add10.apply(5);
        int result = add15.apply(3);  // 10 + 5 + 3 = 18
        System.out.println("Curried result: " + result);
        
        // Memoization example
        Function<Integer, Integer> memoizedFibonacci = memoize(n -> {
            if (n <= 1) return n;
            return memoizedFibonacci.apply(n - 1) + memoizedFibonacci.apply(n - 2);
        });
        
        System.out.println("Fibonacci(10): " + memoizedFibonacci.apply(10));
        
        // Partial application example
        BiFunction<String, String, String> formatter = (prefix, value) -> prefix + ": " + value;
        Function<String, String> emailFormatter = value -> formatter.apply("Email", value);
        Function<String, String> phoneFormatter = value -> formatter.apply("Phone", value);
        
        System.out.println(emailFormatter.apply("user@example.com"));
        System.out.println(phoneFormatter.apply("123-456-7890"));
    }
    
    // Memoization utility
    public static <T, R> Function<T, R> memoize(Function<T, R> function) {
        Map<T, R> cache = new java.util.concurrent.ConcurrentHashMap<>();
        return input -> cache.computeIfAbsent(input, function);
    }
}
```

## Performance Considerations and Best Practices

### Lambda Performance Analysis

```java
import java.util.function.*;
import java.util.*;

public class LambdaPerformance {
    public static void main(String[] args) {
        List<Integer> numbers = new ArrayList<>();
        for (int i = 0; i < 1000000; i++) {
            numbers.add(i);
        }
        
        // Compare different approaches
        long startTime, endTime;
        
        // Traditional for loop
        startTime = System.nanoTime();
        int sum1 = 0;
        for (int num : numbers) {
            if (num % 2 == 0) {
                sum1 += num * num;
            }
        }
        endTime = System.nanoTime();
        System.out.println("Traditional loop: " + (endTime - startTime) / 1_000_000 + "ms");
        
        // Lambda with streams
        startTime = System.nanoTime();
        int sum2 = numbers.stream()
            .filter(n -> n % 2 == 0)
            .mapToInt(n -> n * n)
            .sum();
        endTime = System.nanoTime();
        System.out.println("Stream lambda: " + (endTime - startTime) / 1_000_000 + "ms");
        
        // Parallel stream
        startTime = System.nanoTime();
        int sum3 = numbers.parallelStream()
            .filter(n -> n % 2 == 0)
            .mapToInt(n -> n * n)
            .sum();
        endTime = System.nanoTime();
        System.out.println("Parallel stream: " + (endTime - startTime) / 1_000_000 + "ms");
        
        System.out.println("Results match: " + (sum1 == sum2 && sum2 == sum3));
        
        // Lambda vs anonymous class overhead
        startTime = System.nanoTime();
        for (int i = 0; i < 100000; i++) {
            Runnable lambda = () -> Math.random();
            lambda.run();
        }
        endTime = System.nanoTime();
        System.out.println("Lambda overhead: " + (endTime - startTime) / 1_000_000 + "ms");
        
        startTime = System.nanoTime();
        for (int i = 0; i < 100000; i++) {
            Runnable anon = new Runnable() {
                @Override
                public void run() {
                    Math.random();
                }
            };
            anon.run();
        }
        endTime = System.nanoTime();
        System.out.println("Anonymous class overhead: " + (endTime - startTime) / 1_000_000 + "ms");
    }
}
```

### Best Practices and Guidelines

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.Collectors;

public class LambdaBestPractices {
    public static void main(String[] args) {
        List<String> words = Arrays.asList("hello", "world", "java", "lambda", "stream");
        
        // 1. Prefer method references over lambda expressions when possible
        List<String> upperCase = words.stream()
            .map(String::toUpperCase)  // Good: method reference
            .collect(Collectors.toList());
        
        List<String> upperCaseLambda = words.stream()
            .map(s -> s.toUpperCase())  // Avoid: lambda when method reference is clearer
            .collect(Collectors.toList());
        
        // 2. Keep lambdas short and simple
        List<String> longWords = words.stream()
            .filter(s -> s.length() > 4)  // Good: simple condition
            .collect(Collectors.toList());
        
        // 3. Avoid mutability in lambdas (for parallel streams)
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
        List<Integer> results = new ArrayList<>(); // Don't do this
        
        // 4. Use proper functional interfaces
        Predicate<String> isLong = s -> s.length() > 10;
        Function<String, Integer> lengthFunc = String::length;
        Consumer<String> printer = System.out::println;
        
        // 5. Chain operations efficiently
        String result = words.stream()
            .filter(Objects::nonNull)  // First filter out nulls
            .filter(s -> s.length() > 3)  // Then filter by length
            .map(String::toUpperCase)  // Then transform
            .findFirst()  // Use findFirst when you only need one
            .orElse("Default");
        
        System.out.println("Result: " + result);
        
        // 6. Use appropriate collectors
        Map<Integer, List<String>> groupedByLength = words.stream()
            .collect(Collectors.groupingBy(String::length));
        
        // 7. Handle exceptions properly
        List<String> potentialNumbers = Arrays.asList("1", "2", "abc", "4", "5");
        
        List<Integer> safeNumbers = potentialNumbers.stream()
            .filter(s -> {
                try {
                    Integer.parseInt(s);
                    return true;
                } catch (NumberFormatException e) {
                    return false;
                }
            })
            .map(Integer::parseInt)
            .collect(Collectors.toList());
        
        System.out.println("Safe numbers: " + safeNumbers);
        
        // 8. Demonstrate when NOT to use streams
        // For simple operations on small collections, traditional loops might be clearer
        List<String> simpleList = Arrays.asList("a", "b", "c");
        
        // Good for simple operations
        for (String s : simpleList) {
            System.out.println(s);
        }
        
        // Good for complex operations
        List<String> complexResult = simpleList.stream()
            .map(String::toUpperCase)
            .filter(s -> s.length() > 0)
            .collect(Collectors.toList());
    }
}
```

## Real-World Applications

### Event Processing System

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.function.*;
import java.util.stream.Collectors;

public class EventProcessingSystem {
    // Event types
    interface Event {
        String getId();
        String getType();
        long getTimestamp();
    }
    
    static class UserEvent implements Event {
        private String id;
        private String type;
        private long timestamp;
        private String userId;
        
        public UserEvent(String id, String type, String userId) {
            this.id = id;
            this.type = type;
            this.userId = userId;
            this.timestamp = System.currentTimeMillis();
        }
        
        @Override
        public String getId() { return id; }
        @Override
        public String getType() { return type; }
        @Override
        public long getTimestamp() { return timestamp; }
        public String getUserId() { return userId; }
    }
    
    // Event processor
    public static class EventProcessor {
        private List<Consumer<Event>> processors = new ArrayList<>();
        private List<Predicate<Event>> filters = new ArrayList<>();
        
        public void addProcessor(Consumer<Event> processor) {
            processors.add(processor);
        }
        
        public void addFilter(Predicate<Event> filter) {
            filters.add(filter);
        }
        
        public void processEvent(Event event) {
            boolean shouldProcess = filters.isEmpty() || 
                filters.stream().anyMatch(filter -> filter.test(event));
            
            if (shouldProcess) {
                processors.forEach(processor -> processor.accept(event));
            }
        }
        
        public void processEvents(List<Event> events) {
            events.parallelStream().forEach(this::processEvent);
        }
    }
    
    public static void main(String[] args) {
        EventProcessor processor = new EventProcessor();
        
        // Add different event processors using lambdas
        processor.addProcessor(event -> {
            System.out.println("Logging event: " + event.getId() + " of type: " + event.getType());
        });
        
        processor.addProcessor(event -> {
            if (event.getType().equals("ERROR")) {
                System.out.println("ALERT: Error event detected: " + event.getId());
            }
        });
        
        processor.addProcessor(event -> {
            System.out.println("Metrics updated for event: " + event.getId());
        });
        
        // Add filters
        processor.addFilter(event -> event.getTimestamp() > System.currentTimeMillis() - 10000); // Last 10 seconds
        processor.addFilter(event -> !event.getType().startsWith("DEBUG")); // Exclude debug events
        
        // Create and process events
        List<Event> events = Arrays.asList(
            new UserEvent("1", "LOGIN", "user1"),
            new UserEvent("2", "ERROR", "user2"),
            new UserEvent("3", "PURCHASE", "user3"),
            new UserEvent("4", "DEBUG", "user4"),
            new UserEvent("5", "LOGOUT", "user5")
        );
        
        processor.processEvents(events);
    }
}
```

## Summary

Java lambda expressions and functional programming features provide powerful tools for writing more concise, readable, and maintainable code. Key takeaways include:

1. **Lambda Expressions**: Provide a concise way to implement single-method interfaces
2. **Functional Interfaces**: The foundation for lambda expressions with exactly one abstract method
3. **Predefined Interfaces**: `Function`, `Predicate`, `Consumer`, `Supplier`, and others for common use cases
4. **Method References**: Even more concise syntax for existing methods
5. **Stream API**: Functional approach to processing collections and sequences
6. **Parallel Processing**: Leverage multiple cores for performance gains
7. **Design Patterns**: Functional implementations of common patterns
8. **Best Practices**: Guidelines for effective and efficient use

Mastering these concepts will make you proficient in modern Java development and enable you to write more expressive, functional-style code that's both powerful and maintainable.