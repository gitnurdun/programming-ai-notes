# Advanced Java Functional Programming and Streams API

## Overview
Java 8 introduced functional programming concepts that fundamentally changed how developers write Java code. The Streams API, lambda expressions, and functional interfaces provide powerful tools for processing collections and data in a declarative, functional style.

## Functional Interfaces

Functional interfaces contain exactly one abstract method and can be implemented using lambda expressions. Java provides several built-in functional interfaces:

### Core Functional Interfaces

```java
import java.util.function.*;
import java.util.*;

public class FunctionalInterfacesDemo {
    public static void main(String[] args) {
        // Predicate - takes one argument, returns boolean
        Predicate<Integer> isEven = x -> x % 2 == 0;
        System.out.println("Is 4 even? " + isEven.test(4));
        
        // Function - takes one argument, returns result
        Function<String, Integer> stringLength = s -> s.length();
        System.out.println("Length of 'Hello': " + stringLength.apply("Hello"));
        
        // Consumer - takes one argument, returns nothing
        Consumer<String> printer = s -> System.out.println("Printing: " + s);
        printer.accept("Hello World");
        
        // Supplier - takes no argument, returns result
        Supplier<Double> randomSupplier = () -> Math.random();
        System.out.println("Random value: " + randomSupplier.get());
        
        // UnaryOperator - takes one argument, returns same type
        UnaryOperator<Integer> square = x -> x * x;
        System.out.println("Square of 5: " + square.apply(5));
        
        // BinaryOperator - takes two arguments, returns same type
        BinaryOperator<Integer> sum = (a, b) -> a + b;
        System.out.println("Sum of 3 and 4: " + sum.apply(3, 4));
    }
}
```

### Custom Functional Interfaces

```java
@FunctionalInterface
public interface Calculator<T> {
    T calculate(T a, T b);
    
    // Default methods are allowed
    default T multiply(T a, T b) {
        return calculate(a, b); // This won't work without implementation
    }
    
    // Static methods are allowed
    static <T> Calculator<T> getInstance() {
        return (a, b) -> null; // Placeholder implementation
    }
}

// Implementation
class IntegerCalculator implements Calculator<Integer> {
    @Override
    public Integer calculate(Integer a, Integer b) {
        return a + b;
    }
}

// Using with lambda
Calculator<Integer> adder = (a, b) -> a + b;
Calculator<Integer> multiplier = (a, b) -> a * b;
```

## Lambda Expressions Deep Dive

### Syntax and Scoping

```java
public class LambdaScopingDemo {
    private String instanceVar = "Instance";
    private static String staticVar = "Static";
    
    public void demonstrateScoping() {
        String localVar = "Local";
        int effectivelyFinal = 42; // Must be effectively final
        
        // Lambda accessing different scopes
        Runnable r1 = () -> {
            // Can access instance variables
            System.out.println(instanceVar);
            
            // Can access static variables
            System.out.println(staticVar);
            
            // Can access effectively final local variables
            System.out.println(localVar);
            System.out.println(effectivelyFinal);
            
            // Cannot modify local variables (compile error)
            // localVar = "Modified"; // Error!
        };
        
        // Method references
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
        
        // Static method reference
        names.forEach(System.out::println);
        
        // Instance method reference
        names.forEach(this::printName);
        
        // Constructor reference
        Supplier<ArrayList<String>> listSupplier = ArrayList::new;
    }
    
    private void printName(String name) {
        System.out.println("Name: " + name);
    }
}
```

### Method References Types

```java
import java.util.*;
import java.util.function.*;

public class MethodReferencesDemo {
    public static void main(String[] args) {
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
        
        // 1. Static method reference
        names.forEach(System.out::println);
        
        // 2. Instance method reference of a particular object
        StringBuilder sb = new StringBuilder();
        names.forEach(sb::append);
        System.out.println("Concatenated: " + sb.toString());
        
        // 3. Instance method reference of an arbitrary object
        names.sort(String::compareToIgnoreCase);
        
        // 4. Constructor reference
        Supplier<List<String>> listSupplier = ArrayList::new;
        List<String> newList = listSupplier.get();
        
        // Function with constructor reference
        Function<String, Integer> intParser = Integer::valueOf;
        List<Integer> numbers = names.stream()
            .map(String::length)
            .collect(ArrayList::new, ArrayList::add, ArrayList::addAll);
    }
}
```

## Streams API Comprehensive Guide

### Stream Creation and Characteristics

```java
import java.util.*;
import java.util.stream.*;

public class StreamCreationDemo {
    public static void main(String[] args) {
        // 1. From collections
        List<String> list = Arrays.asList("a", "b", "c");
        Stream<String> stream1 = list.stream();
        
        // 2. From arrays
        String[] array = {"a", "b", "c"};
        Stream<String> stream2 = Arrays.stream(array);
        
        // 3. Using Stream builder
        Stream<String> stream3 = Stream.<String>builder()
            .add("a").add("b").add("c").build();
        
        // 4. Using Stream.generate (infinite stream)
        Stream<Double> randomStream = Stream.generate(Math::random);
        
        // 5. Using Stream.iterate (infinite stream)
        Stream<Integer> iterateStream = Stream.iterate(0, n -> n + 2);
        
        // 6. Using of() methods
        Stream<String> stream4 = Stream.of("a", "b", "c");
        Stream<String> stream5 = Stream.ofNullable(null); // Empty stream
        
        // 7. Primitive streams
        IntStream intStream = IntStream.range(1, 10); // 1 to 9
        LongStream longStream = LongStream.rangeClosed(1, 10); // 1 to 10
        DoubleStream doubleStream = DoubleStream.of(1.0, 2.0, 3.0);
    }
}
```

### Intermediate Operations

```java
import java.util.*;
import java.util.stream.*;

public class IntermediateOperationsDemo {
    public static void main(String[] args) {
        List<String> names = Arrays.asList(
            "Alice", "Bob", "Charlie", "David", "Eve", "Frank");
        
        // Filter
        List<String> longNames = names.stream()
            .filter(name -> name.length() > 4)
            .collect(Collectors.toList());
        
        // Map
        List<Integer> nameLengths = names.stream()
            .map(String::length)
            .collect(Collectors.toList());
        
        // FlatMap
        List<List<String>> listOfLists = Arrays.asList(
            Arrays.asList("a", "b"),
            Arrays.asList("c", "d", "e"),
            Arrays.asList("f")
        );
        
        List<String> flattened = listOfLists.stream()
            .flatMap(List::stream)
            .collect(Collectors.toList());
        
        // Distinct
        List<Integer> numbers = Arrays.asList(1, 2, 2, 3, 3, 3, 4);
        List<Integer> distinctNumbers = numbers.stream()
            .distinct()
            .collect(Collectors.toList());
        
        // Sorted
        List<String> sortedNames = names.stream()
            .sorted()
            .collect(Collectors.toList());
        
        // Sorted with comparator
        List<String> sortedByLength = names.stream()
            .sorted(Comparator.comparing(String::length))
            .collect(Collectors.toList());
        
        // Peek (for debugging)
        List<String> peekedNames = names.stream()
            .peek(name -> System.out.println("Processing: " + name))
            .filter(name -> name.length() > 3)
            .peek(name -> System.out.println("Filtered: " + name))
            .collect(Collectors.toList());
        
        // Limit and skip
        List<String> limited = names.stream()
            .limit(3)
            .collect(Collectors.toList());
        
        List<String> skipped = names.stream()
            .skip(2)
            .collect(Collectors.toList());
    }
}
```

### Terminal Operations

```java
import java.util.*;
import java.util.stream.*;
import java.util.function.*;

public class TerminalOperationsDemo {
    public static void main(String[] args) {
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        // forEach
        numbers.stream().forEach(System.out::println);
        
        // forEachOrdered (maintains encounter order)
        numbers.parallelStream().forEachOrdered(System.out::println);
        
        // toArray
        Integer[] array = numbers.stream().toArray(Integer[]::new);
        
        // reduce (terminal operation with accumulation)
        Optional<Integer> sum = numbers.stream().reduce(Integer::sum);
        Integer product = numbers.stream().reduce(1, (a, b) -> a * b);
        
        // With identity, accumulator, and combiner (for parallel streams)
        Integer parallelSum = numbers.parallelStream()
            .reduce(0, Integer::sum, Integer::sum);
        
        // collect (most powerful terminal operation)
        List<Integer> evenNumbers = numbers.stream()
            .filter(n -> n % 2 == 0)
            .collect(Collectors.toList());
        
        Set<Integer> numberSet = numbers.stream()
            .collect(Collectors.toSet());
        
        String joined = numbers.stream()
            .map(String::valueOf)
            .collect(Collectors.joining(", ", "[", "]"));
        
        // Grouping
        Map<Boolean, List<Integer>> partitioned = numbers.stream()
            .collect(Collectors.partitioningBy(n -> n % 2 == 0));
        
        Map<Integer, List<Integer>> groupedByRemainder = numbers.stream()
            .collect(Collectors.groupingBy(n -> n % 3));
        
        // Summarizing
        IntSummaryStatistics stats = numbers.stream()
            .collect(Collectors.summarizingInt(Integer::intValue));
        
        // Matching operations
        boolean anyMatch = numbers.stream().anyMatch(n -> n > 5);
        boolean allMatch = numbers.stream().allMatch(n -> n > 0);
        boolean noneMatch = numbers.stream().noneMatch(n -> n < 0);
        
        // Finding operations
        Optional<Integer> first = numbers.stream().findFirst();
        Optional<Integer> any = numbers.stream().findAny();
        
        // Count
        long count = numbers.stream().count();
    }
}
```

## Advanced Collector Patterns

```java
import java.util.*;
import java.util.stream.*;
import java.util.function.*;

public class AdvancedCollectorsDemo {
    static class Person {
        private String name;
        private int age;
        private String department;
        
        public Person(String name, int age, String department) {
            this.name = name;
            this.age = age;
            this.department = department;
        }
        
        // Getters
        public String getName() { return name; }
        public int getAge() { return age; }
        public String getDepartment() { return department; }
        
        @Override
        public String toString() {
            return name + "(" + age + ", " + department + ")";
        }
    }
    
    public static void main(String[] args) {
        List<Person> people = Arrays.asList(
            new Person("Alice", 30, "Engineering"),
            new Person("Bob", 25, "Marketing"),
            new Person("Charlie", 35, "Engineering"),
            new Person("David", 28, "Sales"),
            new Person("Eve", 32, "Engineering")
        );
        
        // Grouping by department
        Map<String, List<Person>> byDepartment = people.stream()
            .collect(Collectors.groupingBy(Person::getDepartment));
        
        // Grouping by department with counting
        Map<String, Long> countByDepartment = people.stream()
            .collect(Collectors.groupingBy(
                Person::getDepartment,
                Collectors.counting()
            ));
        
        // Grouping by department with average age
        Map<String, Double> avgAgeByDepartment = people.stream()
            .collect(Collectors.groupingBy(
                Person::getDepartment,
                Collectors.averagingInt(Person::getAge)
            ));
        
        // Grouping by department with multiple collectors
        Map<String, Map<String, Object>> statsByDepartment = people.stream()
            .collect(Collectors.groupingBy(
                Person::getDepartment,
                Collectors.collectingAndThen(
                    Collectors.toList(),
                    list -> {
                        Map<String, Object> stats = new HashMap<>();
                        stats.put("count", list.size());
                        stats.put("avgAge", list.stream()
                            .mapToInt(Person::getAge)
                            .average()
                            .orElse(0.0));
                        stats.put("names", list.stream()
                            .map(Person::getName)
                            .collect(Collectors.joining(", ")));
                        return stats;
                    }
                )
            ));
        
        // Partitioning by age
        Map<Boolean, List<Person>> partitionedByAge = people.stream()
            .collect(Collectors.partitioningBy(p -> p.getAge() >= 30));
        
        // Teeing collector (Java 12+)
        // Combines two collectors and applies a function to their results
        /* 
        Map.Entry<List<Person>, Double> result = people.stream()
            .collect(Collectors.teeing(
                Collectors.filtering(p -> p.getDepartment().equals("Engineering"), 
                                   Collectors.toList()),
                Collectors.averagingInt(Person::getAge),
                AbstractMap.SimpleEntry::new
            ));
        */
    }
}
```

## Parallel Streams

```java
import java.util.*;
import java.util.stream.*;

public class ParallelStreamsDemo {
    public static void main(String[] args) {
        List<Integer> numbers = IntStream.range(1, 1000000)
            .boxed()
            .collect(Collectors.toList());
        
        // Sequential stream
        long startTime = System.currentTimeMillis();
        long sequentialSum = numbers.stream()
            .reduce(0L, Long::sum);
        long sequentialTime = System.currentTimeMillis() - startTime;
        
        // Parallel stream
        startTime = System.currentTimeMillis();
        long parallelSum = numbers.parallelStream()
            .reduce(0L, Long::sum);
        long parallelTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Sequential sum: " + sequentialSum + 
                          " in " + sequentialTime + "ms");
        System.out.println("Parallel sum: " + parallelSum + 
                          " in " + parallelTime + "ms");
        
        // When parallel streams help:
        // 1. Large datasets
        // 2. Computationally intensive operations
        // 3. No shared mutable state
        
        // When parallel streams hurt:
        // 1. Small datasets (overhead)
        // 2. Simple operations (overhead)
        // 3. Operations with state dependencies
        // 4. Operations that modify shared resources
        
        // Example of problematic parallel operation
        // List<Integer> result = new ArrayList<>();
        // numbers.parallelStream().forEach(result::add); // Race condition!
        
        // Correct approach for collecting in parallel
        List<Integer> result = numbers.parallelStream()
            .collect(ArrayList::new, 
                    ArrayList::add, 
                    (list1, list2) -> list1.addAll(list2));
    }
}
```

## Performance Considerations

1. **Short-circuiting operations**: `findFirst()`, `findAny()`, `anyMatch()`, etc.
2. **Lazy evaluation**: Streams are not processed until a terminal operation is called
3. **Fusion**: JVM optimizes chained operations
4. **Boxing/Unboxing**: Use primitive streams when possible

## Best Practices

1. **Prefer method references over lambdas** when possible
2. **Use primitive streams** for numerical operations
3. **Be careful with parallel streams** and shared mutable state
4. **Avoid stateful lambda expressions**
5. **Use collectors effectively** for complex aggregation

## Conclusion

Functional programming in Java provides:
- More expressive and concise code
- Better support for parallel processing
- Improved composability and reusability
- Reduced boilerplate code

Mastering these concepts is essential for modern Java development, especially in enterprise applications where data processing and transformation are common requirements.