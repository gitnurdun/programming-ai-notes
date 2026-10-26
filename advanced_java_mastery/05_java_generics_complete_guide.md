# Java Generics: Complete Guide to Type Safety and Reusability

## Introduction

Java Generics were introduced in Java 5 to provide compile-time type safety and eliminate the need for explicit casting. Generics allow classes, interfaces, and methods to operate on types specified by the user, making code more reusable and type-safe.

This comprehensive guide will explore all aspects of Java generics, from basic concepts to advanced techniques and best practices.

## Understanding Generics

### What Are Generics?

Generics enable types (classes and interfaces) to be parameters when defining classes, interfaces, and methods. This allows for creating type-safe collections, methods, and classes that work with different data types.

### Basic Generic Syntax

```java
import java.util.*;

// Generic class example
public class GenericBox<T> {
    private T content;
    
    public void set(T content) {
        this.content = content;
    }
    
    public T get() {
        return content;
    }
    
    public boolean isEmpty() {
        return content == null;
    }
}

// Generic method example
public class GenericMethods {
    // Generic method with type parameter
    public static <T> void swap(T[] array, int i, int j) {
        if (i >= 0 && i < array.length && j >= 0 && j < array.length) {
            T temp = array[i];
            array[i] = array[j];
            array[j] = temp;
        }
    }
    
    // Generic method with multiple type parameters
    public static <K, V> Map<K, V> createMap(K key, V value) {
        Map<K, V> map = new HashMap<>();
        map.put(key, value);
        return map;
    }
    
    // Generic method with bounded type parameter
    public static <T extends Number> double sum(T[] numbers) {
        double total = 0.0;
        for (T num : numbers) {
            total += num.doubleValue();
        }
        return total;
    }
    
    public static void main(String[] args) {
        // Using generic class
        GenericBox<String> stringBox = new GenericBox<>();
        stringBox.set("Hello Generics");
        System.out.println("String box: " + stringBox.get());
        
        GenericBox<Integer> intBox = new GenericBox<>();
        intBox.set(42);
        System.out.println("Integer box: " + intBox.get());
        
        // Using generic methods
        String[] words = {"apple", "banana", "cherry"};
        System.out.println("Before swap: " + Arrays.toString(words));
        swap(words, 0, 2);
        System.out.println("After swap: " + Arrays.toString(words));
        
        // Create generic map
        Map<String, Integer> scoreMap = createMap("Alice", 95);
        System.out.println("Score map: " + scoreMap);
        
        // Sum numbers using bounded generic
        Integer[] integers = {1, 2, 3, 4, 5};
        Double[] doubles = {1.1, 2.2, 3.3};
        System.out.println("Sum of integers: " + sum(integers));
        System.out.println("Sum of doubles: " + sum(doubles));
    }
}
```

## Generic Classes

### Simple Generic Class

```java
// Generic class with single type parameter
public class Pair<T, U> {
    private T first;
    private U second;
    
    public Pair(T first, U second) {
        this.first = first;
        this.second = second;
    }
    
    public T getFirst() { return first; }
    public U getSecond() { return second; }
    
    public void setFirst(T first) { this.first = first; }
    public void setSecond(U second) { this.second = second; }
    
    @Override
    public String toString() {
        return "Pair{first=" + first + ", second=" + second + "}";
    }
    
    // Generic method within generic class
    public <R> R transform(BiFunction<T, U, R> transformer) {
        return transformer.apply(first, second);
    }
}

// Usage example
public class GenericClassExample {
    public static void main(String[] args) {
        Pair<String, Integer> nameAge = new Pair<>("Alice", 25);
        System.out.println("Name-Age pair: " + nameAge);
        
        Pair<Double, Boolean> coordinateValid = new Pair<>(3.14, true);
        System.out.println("Coordinate-Valid pair: " + coordinateValid);
        
        // Using generic method in generic class
        String description = nameAge.transform((name, age) -> name + " is " + age + " years old");
        System.out.println("Description: " + description);
    }
}
```

### Generic Class with Multiple Type Parameters

```java
// Generic class with multiple type parameters and bounded types
public class Triple<T extends Comparable<T>, U, V> {
    private T first;
    private U second;
    private V third;
    
    public Triple(T first, U second, V third) {
        this.first = first;
        this.second = second;
        this.third = third;
    }
    
    public T getFirst() { return first; }
    public U getSecond() { return second; }
    public V getThird() { return third; }
    
    // Method that uses the bounded type
    public int compareFirstTo(T other) {
        return this.first.compareTo(other);
    }
    
    // Generic method with local type parameter
    public <W> void printAll(W fourth) {
        System.out.println("First: " + first + 
                          ", Second: " + second + 
                          ", Third: " + third + 
                          ", Fourth: " + fourth);
    }
    
    @Override
    public String toString() {
        return "Triple{first=" + first + ", second=" + second + ", third=" + third + "}";
    }
}

// Example usage
class TripleExample {
    public static void main(String[] args) {
        Triple<Integer, String, Boolean> triple = new Triple<>(42, "Hello", true);
        System.out.println("Triple: " + triple);
        
        System.out.println("Compare 42 with 30: " + triple.compareFirstTo(30));
        
        triple.printAll(3.14);
    }
}
```

### Generic Collections Implementation

```java
import java.util.*;

// Custom generic collection implementation
public class GenericStack<T> {
    private List<T> elements = new ArrayList<>();
    
    public void push(T item) {
        elements.add(item);
    }
    
    public T pop() {
        if (isEmpty()) {
            throw new EmptyStackException();
        }
        return elements.remove(elements.size() - 1);
    }
    
    public T peek() {
        if (isEmpty()) {
            throw new EmptyStackException();
        }
        return elements.get(elements.size() - 1);
    }
    
    public boolean isEmpty() {
        return elements.isEmpty();
    }
    
    public int size() {
        return elements.size();
    }
    
    // Generic method to convert to array
    public <R> R[] toArray(Class<R> type) {
        @SuppressWarnings("unchecked")
        R[] array = (R[]) Array.newInstance(type, elements.size());
        return elements.toArray(array);
    }
    
    @Override
    public String toString() {
        return "Stack" + elements.toString();
    }
    
    // Generic method to filter elements
    public GenericStack<T> filter(Predicate<T> predicate) {
        GenericStack<T> filtered = new GenericStack<>();
        for (T element : elements) {
            if (predicate.test(element)) {
                filtered.push(element);
            }
        }
        return filtered;
    }
}

// Example usage
class GenericStackExample {
    public static void main(String[] args) {
        GenericStack<String> stringStack = new GenericStack<>();
        stringStack.push("First");
        stringStack.push("Second");
        stringStack.push("Third");
        
        System.out.println("Stack: " + stringStack);
        System.out.println("Peek: " + stringStack.peek());
        System.out.println("Pop: " + stringStack.pop());
        System.out.println("After pop: " + stringStack);
        
        // Filter operation
        GenericStack<String> filtered = stringStack.filter(s -> s.length() > 4);
        System.out.println("Filtered: " + filtered);
        
        // Generic stack with different type
        GenericStack<Integer> intStack = new GenericStack<>();
        intStack.push(1);
        intStack.push(2);
        intStack.push(3);
        System.out.println("Integer stack: " + intStack);
    }
}
```

## Generic Methods

### Static Generic Methods

```java
import java.util.*;
import java.util.function.*;

public class StaticGenericMethods {
    // Generic method with single type parameter
    public static <T> T getFirstElement(List<T> list) {
        return list.isEmpty() ? null : list.get(0);
    }
    
    // Generic method with multiple type parameters
    public static <T, R> List<R> map(List<T> list, Function<T, R> mapper) {
        List<R> result = new ArrayList<>();
        for (T item : list) {
            result.add(mapper.apply(item));
        }
        return result;
    }
    
    // Generic method with bounded type parameter
    public static <T extends Comparable<T>> T findMax(List<T> list) {
        if (list.isEmpty()) {
            return null;
        }
        
        T max = list.get(0);
        for (T item : list) {
            if (item.compareTo(max) > 0) {
                max = item;
            }
        }
        return max;
    }
    
    // Generic method with wildcard
    public static void printList(List<?> list) {
        System.out.println("List contents: " + list);
    }
    
    // Generic method with type parameter that extends multiple bounds
    public static <T extends Number & Comparable<T>> double calculateAverage(List<T> numbers) {
        if (numbers.isEmpty()) {
            return 0.0;
        }
        
        double sum = 0.0;
        for (T number : numbers) {
            sum += number.doubleValue();
        }
        return sum / numbers.size();
    }
    
    // Generic method with recursive type bound
    public static <T extends Comparable<T>> void sort(List<T> list) {
        Collections.sort(list);
    }
    
    public static void main(String[] args) {
        // Test getFirstElement
        List<String> words = Arrays.asList("apple", "banana", "cherry");
        System.out.println("First element: " + getFirstElement(words));
        
        // Test map function
        List<String> original = Arrays.asList("hello", "world", "java");
        List<Integer> lengths = map(original, String::length);
        System.out.println("Lengths: " + lengths);
        
        // Test findMax
        List<Integer> numbers = Arrays.asList(3, 1, 4, 1, 5, 9, 2, 6);
        System.out.println("Max: " + findMax(numbers));
        
        // Test printList with wildcard
        List<String> stringList = Arrays.asList("a", "b", "c");
        List<Integer> intList = Arrays.asList(1, 2, 3);
        printList(stringList);
        printList(intList);
        
        // Test calculateAverage
        List<Double> doubles = Arrays.asList(1.1, 2.2, 3.3, 4.4, 5.5);
        System.out.println("Average: " + calculateAverage(doubles));
        
        // Test sort
        List<String> unsorted = new ArrayList<>(Arrays.asList("zebra", "apple", "banana"));
        sort(unsorted);
        System.out.println("Sorted: " + unsorted);
    }
}
```

### Generic Methods in Generic Classes

```java
// Generic class with generic methods
public class GenericProcessor<T> {
    private T value;
    
    public GenericProcessor(T value) {
        this.value = value;
    }
    
    public T getValue() { return value; }
    public void setValue(T value) { this.value = value; }
    
    // Generic method within generic class
    public <R> R process(Function<T, R> function) {
        return function.apply(value);
    }
    
    // Generic method with bounded type parameter
    public <R extends T> boolean isCompatible(R other) {
        return other != null && value != null && value.getClass().isAssignableFrom(other.getClass());
    }
    
    // Generic method with multiple type parameters
    public <U, R> R combine(U other, BiFunction<T, U, R> combiner) {
        return combiner.apply(value, other);
    }
    
    // Generic method that returns GenericProcessor of different type
    public <R> GenericProcessor<R> map(Function<T, R> mapper) {
        return new GenericProcessor<>(mapper.apply(value));
    }
    
    @Override
    public String toString() {
        return "GenericProcessor{value=" + value + "}";
    }
}

// Example usage
class GenericProcessorExample {
    public static void main(String[] args) {
        GenericProcessor<String> stringProcessor = new GenericProcessor<>("Hello World");
        
        // Using generic method to process value
        Integer length = stringProcessor.process(String::length);
        System.out.println("Length: " + length);
        
        // Using generic method to combine with another value
        String combined = stringProcessor.combine(42, (s, n) -> s + " - " + n);
        System.out.println("Combined: " + combined);
        
        // Using map method to create processor of different type
        GenericProcessor<Integer> intProcessor = stringProcessor.map(String::length);
        System.out.println("Integer processor: " + intProcessor);
        
        // Chaining operations
        String result = stringProcessor
            .map(String::toUpperCase)
            .map(s -> "Processed: " + s)
            .process(s -> s + "!");
        
        System.out.println("Chained result: " + result);
    }
}
```

## Wildcards

### Bounded Wildcards

```java
import java.util.*;

// Demonstrate different types of wildcards
public class WildcardExamples {
    // Upper bounded wildcard - accepts List of Number or its subclasses
    public static double sumUpperBounded(List<? extends Number> numbers) {
        double sum = 0.0;
        for (Number num : numbers) {
            sum += num.doubleValue();
        }
        return sum;
    }
    
    // Lower bounded wildcard - accepts List of Integer or its superclasses
    public static void addIntegers(List<? super Integer> list) {
        list.add(42);
        list.add(100);
    }
    
    // Unbounded wildcard - accepts any type of List
    public static void printUnbounded(List<?> list) {
        System.out.println("List size: " + list.size());
        System.out.println("First element: " + (list.isEmpty() ? "None" : list.get(0)));
    }
    
    // Generic method with wildcard in parameter
    public static <T> void copy(List<? extends T> src, List<? super T> dest) {
        for (T item : src) {
            dest.add(item);
        }
    }
    
    public static void main(String[] args) {
        // Upper bounded wildcard examples
        List<Integer> integers = Arrays.asList(1, 2, 3, 4, 5);
        List<Double> doubles = Arrays.asList(1.1, 2.2, 3.3);
        List<Number> numbers = Arrays.asList(1, 2.5, 3);
        
        System.out.println("Sum of integers: " + sumUpperBounded(integers));
        System.out.println("Sum of doubles: " + sumUpperBounded(doubles));
        System.out.println("Sum of numbers: " + sumUpperBounded(numbers));
        
        // Lower bounded wildcard examples
        List<Object> objects = new ArrayList<>();
        List<Number> numberList = new ArrayList<>();
        List<Integer> intList = new ArrayList<>();
        
        addIntegers(objects);    // Works: Integer is subclass of Object
        addIntegers(numberList); // Works: Integer is subclass of Number
        // addIntegers(intList);  // Doesn't compile: Integer is not superclass of Integer
        
        System.out.println("Objects after adding integers: " + objects);
        System.out.println("Numbers after adding integers: " + numberList);
        
        // Unbounded wildcard examples
        List<String> stringList = Arrays.asList("a", "b", "c");
        List<Integer> intList2 = Arrays.asList(1, 2, 3);
        
        printUnbounded(stringList);
        printUnbounded(intList2);
        
        // Copy method with wildcards
        List<String> source = Arrays.asList("Hello", "World");
        List<CharSequence> destination = new ArrayList<>();
        
        copy(source, destination);
        System.out.println("Copied to destination: " + destination);
    }
}
```

### PECS (Producer Extends, Consumer Super)

```java
import java.util.*;

// PECS principle: Producer Extends, Consumer Super
public class PECSPrinciple {
    // Producer extends - use <? extends T> when you only read from the collection
    public static double sumOfList(List<? extends Number> list) {
        double sum = 0.0;
        for (Number n : list) {  // Reading from the list
            sum += n.doubleValue();
        }
        return sum;
    }
    
    // Consumer super - use <? super T> when you only write to the collection
    public static void addNumbers(List<? super Integer> list) {
        for (int i = 1; i <= 10; i++) {  // Writing to the list
            list.add(i);
        }
    }
    
    // Example with custom classes
    static class Animal {
        protected String name;
        public Animal(String name) { this.name = name; }
        public String getName() { return name; }
        @Override public String toString() { return name; }
    }
    
    static class Dog extends Animal {
        public Dog(String name) { super(name); }
    }
    
    static class Cat extends Animal {
        public Cat(String name) { super(name); }
    }
    
    // Producer method - returns animals from list
    public static void processAnimals(List<? extends Animal> animals, 
                                     Consumer<? super Animal> processor) {
        for (Animal animal : animals) {
            processor.accept(animal);  // Producer: reading from the list
        }
    }
    
    // Consumer method - adds animals to list
    public static void addAnimals(List<? super Dog> animals, 
                                 Supplier<? extends Dog> dogSupplier, int count) {
        for (int i = 0; i < count; i++) {
            animals.add(dogSupplier.get());  // Consumer: writing to the list
        }
    }
    
    public static void main(String[] args) {
        // PECS examples
        List<Integer> integers = new ArrayList<>();
        List<Number> numbers = new ArrayList<>();
        List<Object> objects = new ArrayList<>();
        
        // Producer extends
        integers.add(1);
        integers.add(2);
        integers.add(3);
        System.out.println("Sum of integers: " + sumOfList(integers));
        
        // Consumer super
        addNumbers(numbers);  // Works: Number is superclass of Integer
        System.out.println("Numbers after adding: " + numbers);
        
        // Complex PECS example
        List<Animal> animalList = new ArrayList<>();
        List<Object> objectList = new ArrayList<>();
        
        // Add dogs to lists that can accept Animal or its superclasses
        addAnimals(animalList, () -> new Dog("Buddy"), 2);
        addAnimals(objectList, () -> new Dog("Max"), 2);
        
        System.out.println("Animal list: " + animalList);
        System.out.println("Object list: " + objectList);
        
        // Process animals
        processAnimals(animalList, System.out::println);
    }
}
```

## Type Bounds

### Upper Bounds

```java
import java.util.*;

// Examples of upper bounds in generics
public class UpperBoundsExamples {
    // Single upper bound
    public static <T extends Number> double calculateSum(List<T> numbers) {
        double sum = 0.0;
        for (T number : numbers) {
            sum += number.doubleValue();
        }
        return sum;
    }
    
    // Multiple bounds (Java 7+)
    static class Calculator implements Comparable<Calculator>, Serializable {
        private double value;
        
        public Calculator(double value) {
            this.value = value;
        }
        
        @Override
        public int compareTo(Calculator other) {
            return Double.compare(this.value, other.value);
        }
        
        public double getValue() { return value; }
        
        @Override
        public String toString() {
            return "Calculator(" + value + ")";
        }
    }
    
    // Method with multiple bounds
    public static <T extends Comparable<T> & Serializable> T findMax(T[] array) {
        if (array.length == 0) return null;
        
        T max = array[0];
        for (T item : array) {
            if (item.compareTo(max) > 0) {
                max = item;
            }
        }
        return max;
    }
    
    // Generic class with bounded type parameter
    public static class BoundedContainer<T extends Comparable<T>> {
        private T value;
        
        public BoundedContainer(T value) {
            this.value = value;
        }
        
        public T getValue() { return value; }
        
        // Method that can use Comparable methods
        public int compareTo(BoundedContainer<T> other) {
            if (other == null) return 1;
            return this.value.compareTo(other.value);
        }
        
        // Generic method within bounded class
        public <U extends T> boolean isGreater(U other) {
            return this.value.compareTo(other) > 0;
        }
    }
    
    // Method with bounded wildcard
    public static void processComparable(List<? extends Comparable<?>> list) {
        for (Comparable<?> item : list) {
            System.out.println("Item: " + item + ", Class: " + item.getClass().getSimpleName());
        }
    }
    
    public static void main(String[] args) {
        // Upper bound examples
        List<Integer> integers = Arrays.asList(1, 2, 3, 4, 5);
        List<Double> doubles = Arrays.asList(1.1, 2.2, 3.3);
        
        System.out.println("Sum of integers: " + calculateSum(integers));
        System.out.println("Sum of doubles: " + calculateSum(doubles));
        
        // Multiple bounds example
        Calculator[] calculators = {
            new Calculator(10.5),
            new Calculator(5.2),
            new Calculator(15.7),
            new Calculator(3.8)
        };
        
        Calculator maxCalculator = findMax(calculators);
        System.out.println("Max calculator: " + maxCalculator);
        
        // Bounded container example
        BoundedContainer<Integer> intContainer = new BoundedContainer<>(42);
        BoundedContainer<String> stringContainer = new BoundedContainer<>("Hello");
        
        System.out.println("Int container: " + intContainer.getValue());
        System.out.println("String container: " + stringContainer.getValue());
        
        // Comparing containers
        System.out.println("Int container > 30: " + intContainer.isGreater(30));
        System.out.println("String container > 'World': " + stringContainer.isGreater("World"));
        
        // Bounded wildcard example
        List<String> strings = Arrays.asList("apple", "banana", "cherry");
        List<Integer> nums = Arrays.asList(1, 2, 3);
        
        System.out.println("Processing string list:");
        processComparable(strings);
        
        System.out.println("\nProcessing integer list:");
        processComparable(nums);
    }
}

// Serializable interface (simplified for this example)
interface Serializable {}
```

### Lower Bounds

```java
import java.util.*;

// Examples of lower bounds in generics
public class LowerBoundsExamples {
    // Generic class with lower bound
    public static class LowerBoundedStack<T> {
        private List<T> elements = new ArrayList<>();
        
        public void push(T item) {
            elements.add(item);
        }
        
        // Method that accepts lower bound - can accept T or any superclass of T
        public void pushAll(List<? super T> destination) {
            for (T element : elements) {
                destination.add(element);  // Writing to the destination
            }
        }
        
        public T pop() {
            if (elements.isEmpty()) {
                throw new EmptyStackException();
            }
            return elements.remove(elements.size() - 1);
        }
        
        public boolean isEmpty() {
            return elements.isEmpty();
        }
        
        public int size() {
            return elements.size();
        }
    }
    
    // Method using lower bound wildcard
    public static void addNumbers(List<? super Integer> list, int... numbers) {
        for (int number : numbers) {
            list.add(number);  // Can add Integer to any list that accepts Integer or its superclass
        }
    }
    
    // Example with custom hierarchy
    static class Person {
        protected String name;
        public Person(String name) { this.name = name; }
        public String getName() { return name; }
        @Override public String toString() { return "Person(" + name + ")"; }
    }
    
    static class Employee extends Person {
        private double salary;
        public Employee(String name, double salary) {
            super(name);
            this.salary = salary;
        }
        public double getSalary() { return salary; }
        @Override public String toString() { return "Employee(" + name + ", $" + salary + ")"; }
    }
    
    static class Manager extends Employee {
        public Manager(String name, double salary) {
            super(name, salary);
        }
        @Override public String toString() { return "Manager(" + name + ", $" + getSalary() + ")"; }
    }
    
    // Method that accepts lower bound for Person hierarchy
    public static void addEmployees(List<? super Person> list, Person... employees) {
        for (Person emp : employees) {
            list.add(emp);  // Can add Person or any subclass to the list
        }
    }
    
    // Generic method with lower bound type parameter
    public static <T> void fill(List<? super T> dest, int count, T item) {
        for (int i = 0; i < count; i++) {
            dest.add(item);
        }
    }
    
    public static void main(String[] args) {
        // Lower bound examples
        List<Object> objects = new ArrayList<>();
        List<Number> numbers = new ArrayList<>();
        List<Integer> integers = new ArrayList<>();
        
        // This works because Integer is a subclass of Number
        addNumbers(numbers, 1, 2, 3, 4, 5);
        System.out.println("Numbers list: " + numbers);
        
        // This works because Integer is a subclass of Object  
        addNumbers(objects, 10, 20, 30);
        System.out.println("Objects list: " + objects);
        
        // This won't compile because we can't add Integer to List<Integer> using lower bound
        // addNumbers(integers, 100, 200); // Would not compile in this context
        
        // Lower bounded stack example
        LowerBoundedStack<String> stringStack = new LowerBoundedStack<>();
        stringStack.push("Hello");
        stringStack.push("World");
        
        List<CharSequence> charSequences = new ArrayList<>();
        stringStack.pushAll(charSequences);  // Works: String is subclass of CharSequence
        System.out.println("CharSequences after pushAll: " + charSequences);
        
        // Person hierarchy example
        List<Object> objList = new ArrayList<>();
        List<Person> personList = new ArrayList<>();
        
        addEmployees(objList, new Manager("Alice", 80000), new Employee("Bob", 50000));
        addEmployees(personList, new Manager("Charlie", 90000), new Employee("Diana", 60000));
        
        System.out.println("Objects list with employees: " + objList);
        System.out.println("Person list with employees: " + personList);
        
        // Fill method example
        List<Number> fillList = new ArrayList<>();
        fill(fillList, 3, 42);        // T = Integer, so ? super Integer includes Number
        fill(fillList, 2, 3.14);      // T = Double, so ? super Double includes Number
        System.out.println("Filled list: " + fillList);
    }
}
```

## Generic Constraints and Advanced Techniques

### Recursive Type Bounds

```java
import java.util.*;

// Recursive type bounds - commonly used with Comparable
public class RecursiveTypeBounds {
    // Self-bounded type - T must be comparable to itself
    public static class SelfBounded<T extends SelfBounded<T>> {
        private T value;
        
        public SelfBounded(T value) {
            this.value = value;
        }
        
        public T getValue() { return value; }
        
        // Method that uses the recursive bound
        public boolean isEqual(T other) {
            if (other == null) return false;
            return this.value.equals(other);
        }
    }
    
    // Example implementing comparable with recursive bound
    public static class Version implements Comparable<Version> {
        private int major;
        private int minor;
        private int patch;
        
        public Version(int major, int minor, int patch) {
            this.major = major;
            this.minor = minor;
            this.patch = patch;
        }
        
        @Override
        public int compareTo(Version other) {
            if (this.major != other.major) {
                return Integer.compare(this.major, other.major);
            }
            if (this.minor != other.minor) {
                return Integer.compare(this.minor, other.minor);
            }
            return Integer.compare(this.patch, other.patch);
        }
        
        @Override
        public String toString() {
            return major + "." + minor + "." + patch;
        }
        
        @Override
        public boolean equals(Object obj) {
            if (this == obj) return true;
            if (obj == null || getClass() != obj.getClass()) return false;
            Version version = (Version) obj;
            return major == version.major && minor == version.minor && patch == version.patch;
        }
        
        @Override
        public int hashCode() {
            return Objects.hash(major, minor, patch);
        }
    }
    
    // Generic method with recursive bound
    public static <T extends Comparable<T>> T findMaxRecursive(List<T> list) {
        if (list.isEmpty()) return null;
        
        T max = list.get(0);
        for (T item : list) {
            if (item.compareTo(max) > 0) {
                max = item;
            }
        }
        return max;
    }
    
    // Generic class with recursive bound for chaining operations
    public static class FluentBuilder<T extends FluentBuilder<T>> {
        protected String name;
        protected int value;
        
        @SuppressWarnings("unchecked")
        public T withName(String name) {
            this.name = name;
            return (T) this;
        }
        
        @SuppressWarnings("unchecked")
        public T withValue(int value) {
            this.value = value;
            return (T) this;
        }
        
        public String getName() { return name; }
        public int getValue() { return value; }
        
        @Override
        public String toString() {
            return getClass().getSimpleName() + "{name='" + name + "', value=" + value + "}";
        }
    }
    
    // Concrete implementation of the fluent builder
    public static class ConcreteBuilder extends FluentBuilder<ConcreteBuilder> {
        private String category;
        
        public ConcreteBuilder withCategory(String category) {
            this.category = category;
            return this;
        }
        
        public String getCategory() { return category; }
    }
    
    public static void main(String[] args) {
        // Recursive type bounds example
        Version v1 = new Version(2, 1, 0);
        Version v2 = new Version(2, 0, 1);
        Version v3 = new Version(1, 9, 5);
        
        List<Version> versions = Arrays.asList(v1, v2, v3);
        Version maxVersion = findMaxRecursive(versions);
        System.out.println("Max version: " + maxVersion);
        
        // Fluent builder with recursive bound
        ConcreteBuilder builder = new ConcreteBuilder()
            .withName("TestBuilder")
            .withValue(42)
            .withCategory("Example");
        
        System.out.println("Builder: " + builder);
        System.out.println("Name: " + builder.getName());
        System.out.println("Value: " + builder.getValue());
        System.out.println("Category: " + builder.getCategory());
        
        // Self-bounded example
        SelfBounded<Version> selfBounded = new SelfBounded<>(new Version(1, 0, 0));
        System.out.println("Self-bounded value: " + selfBounded.getValue());
    }
}
```

### Generic Type Token Pattern

```java
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.*;

// Generic type token pattern for runtime type information
public class GenericTypeToken {
    // Type token class to capture generic type information
    public static abstract class TypeToken<T> {
        protected final Class<T> type;
        
        protected TypeToken() {
            Type superclass = getClass().getGenericSuperclass();
            if (superclass instanceof Class) {
                throw new RuntimeException("Missing type parameter");
            }
            this.type = (Class<T>) ((ParameterizedType) superclass).getActualTypeArguments()[0];
        }
        
        public Class<T> getType() {
            return type;
        }
        
        @Override
        public boolean equals(Object obj) {
            return obj instanceof TypeToken && 
                   ((TypeToken<?>) obj).getType().equals(this.type);
        }
        
        @Override
        public int hashCode() {
            return type.hashCode();
        }
    }
    
    // Generic registry using type tokens
    public static class TypeRegistry {
        private Map<TypeToken<?>, Object> registry = new HashMap<>();
        
        public <T> void register(TypeToken<T> token, T instance) {
            registry.put(token, instance);
        }
        
        @SuppressWarnings("unchecked")
        public <T> T get(TypeToken<T> token) {
            return (T) registry.get(token);
        }
        
        public Set<TypeToken<?>> getRegisteredTypes() {
            return registry.keySet();
        }
    }
    
    // Example classes
    static class ServiceA {
        public String getName() { return "Service A"; }
    }
    
    static class ServiceB {
        public String getName() { return "Service B"; }
    }
    
    static class Repository<T> {
        private Class<T> entityClass;
        
        public Repository(Class<T> entityClass) {
            this.entityClass = entityClass;
        }
        
        public Class<T> getEntityClass() {
            return entityClass;
        }
        
        public T createInstance() {
            try {
                return entityClass.getDeclaredConstructor().newInstance();
            } catch (Exception e) {
                throw new RuntimeException("Cannot create instance of " + entityClass, e);
            }
        }
    }
    
    // Entity classes
    static class User {
        private String name;
        public User() {}
        public User(String name) { this.name = name; }
        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        @Override public String toString() { return "User{name='" + name + "'}"; }
    }
    
    static class Product {
        private String name;
        private double price;
        public Product() {}
        public Product(String name, double price) { this.name = name; this.price = price; }
        public String getName() { return name; }
        public double getPrice() { return price; }
        @Override public String toString() { return "Product{name='" + name + "', price=" + price + "}"; }
    }
    
    public static void main(String[] args) {
        // Type token registry example
        TypeRegistry registry = new TypeRegistry();
        
        registry.register(new TypeToken<ServiceA>() {}, new ServiceA());
        registry.register(new TypeToken<ServiceB>() {}, new ServiceB());
        
        ServiceA serviceA = registry.get(new TypeToken<ServiceA>() {});
        ServiceB serviceB = registry.get(new TypeToken<ServiceB>() {});
        
        System.out.println("Service A: " + serviceA.getName());
        System.out.println("Service B: " + serviceB.getName());
        
        // Repository example with class tokens
        Repository<User> userRepository = new Repository<>(User.class);
        Repository<Product> productRepository = new Repository<>(Product.class);
        
        User user = userRepository.createInstance();
        user.setName("John Doe");
        System.out.println("Created user: " + user);
        
        Product product = productRepository.createInstance();
        System.out.println("Created product: " + product);
        
        System.out.println("User entity class: " + userRepository.getEntityClass().getSimpleName());
        System.out.println("Product entity class: " + productRepository.getEntityClass().getSimpleName());
    }
}
```

## Best Practices and Common Pitfalls

### Generic Best Practices

```java
import java.util.*;
import java.util.function.*;

public class GenericBestPractices {
    // 1. Use meaningful type parameter names
    public static class Container<TYPE> {  // Bad: too verbose
        private TYPE value;
        public TYPE getValue() { return value; }
        public void setValue(TYPE value) { this.value = value; }
    }
    
    public static class Box<T> {  // Good: standard convention
        private T value;
        public T getValue() { return value; }
        public void setValue(T value) { this.value = value; }
    }
    
    // 2. Choose appropriate type parameter names
    public static class CollectionUtilities {
        // For collections: E for Element, K/V for Key/Value, T for Type
        public static <E> void printAll(List<E> list) {
            for (E item : list) {
                System.out.println(item);
            }
        }
        
        public static <K, V> void printMap(Map<K, V> map) {
            for (Map.Entry<K, V> entry : map.entrySet()) {
                System.out.println(entry.getKey() + " = " + entry.getValue());
            }
        }
        
        // For iterators or similar: I for Iterator
        public static <T, I extends Iterator<T>> List<T> iteratorToList(I iterator) {
            List<T> result = new ArrayList<>();
            while (iterator.hasNext()) {
                result.add(iterator.next());
            }
            return result;
        }
    }
    
    // 3. Use bounded wildcards appropriately (PECS)
    public static class SafeOperations {
        // Producer - use extends
        public static <T> double sum(List<? extends Number> numbers) {
            return numbers.stream()
                         .mapToDouble(Number::doubleValue)
                         .sum();
        }
        
        // Consumer - use super
        public static <T> void addAll(List<? super T> dest, List<? extends T> src) {
            for (T item : src) {
                dest.add(item);
            }
        }
        
        // Method that both reads and writes - avoid wildcards
        public static <T> void transform(List<T> list, UnaryOperator<T> transformer) {
            for (int i = 0; i < list.size(); i++) {
                list.set(i, transformer.apply(list.get(i)));
            }
        }
    }
    
    // 4. Avoid raw types
    public static void demonstrateRawTypes() {
        // Don't do this - raw type
        List rawList = new ArrayList();  // Raw type - avoid
        rawList.add("String");
        rawList.add(42);  // No compile-time type checking
        
        // Do this - parameterized type
        List<String> stringList = new ArrayList<>();  // Parameterized type - good
        stringList.add("String");
        // stringList.add(42);  // Compile error - good!
        
        // Handle raw type with warning suppression carefully
        @SuppressWarnings("unchecked")
        List<String> safeList = (List<String>) rawList;  // Only when necessary
    }
    
    // 5. Generic methods for utility functions
    public static class GenericUtils {
        // Generic factory method
        public static <T> Optional<T> createOptional(T value) {
            return Optional.ofNullable(value);
        }
        
        // Generic conversion method
        public static <T, R> List<R> convert(List<T> source, Function<T, R> converter) {
            List<R> result = new ArrayList<>(source.size());
            for (T item : source) {
                result.add(converter.apply(item));
            }
            return result;
        }
        
        // Generic filtering method
        public static <T> List<T> filter(List<T> source, Predicate<T> predicate) {
            List<T> result = new ArrayList<>();
            for (T item : source) {
                if (predicate.test(item)) {
                    result.add(item);
                }
            }
            return result;
        }
        
        // Generic reduction method
        public static <T, R> R reduce(List<T> source, R initial, BiFunction<R, T, R> accumulator) {
            R result = initial;
            for (T item : source) {
                result = accumulator.apply(result, item);
            }
            return result;
        }
    }
    
    // 6. Proper exception handling in generic code
    public static class GenericExceptionHandling {
        public static <T> T safeGet(List<T> list, int index, T defaultValue) {
            try {
                return list.get(index);
            } catch (IndexOutOfBoundsException e) {
                System.out.println("Index " + index + " is out of bounds, returning default");
                return defaultValue;
            }
        }
        
        // Generic exception wrapper
        public static <T> T withExceptionHandling(Supplier<T> supplier, Function<Exception, T> errorHandler) {
            try {
                return supplier.get();
            } catch (Exception e) {
                return errorHandler.apply(e);
            }
        }
    }
    
    public static void main(String[] args) {
        // Demonstrate best practices
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
        List<String> words = Arrays.asList("hello", "world", "java");
        
        System.out.println("Sum of numbers: " + SafeOperations.sum(numbers));
        
        List<Number> numberList = new ArrayList<>();
        SafeOperations.addAll(numberList, numbers);
        System.out.println("Number list after addAll: " + numberList);
        
        // Transform operation
        SafeOperations.transform(words, String::toUpperCase);
        System.out.println("Transformed words: " + words);
        
        // Generic utilities
        List<String> upperWords = GenericUtils.convert(words, String::toLowerCase);
        System.out.println("Lowercase words: " + upperWords);
        
        List<String> longWords = GenericUtils.filter(words, s -> s.length() > 4);
        System.out.println("Long words: " + longWords);
        
        // Safe operations
        String safeValue = GenericExceptionHandling.safeGet(words, 10, "default");
        System.out.println("Safe get result: " + safeValue);
        
        String handledResult = GenericExceptionHandling.withExceptionHandling(
            () -> { throw new RuntimeException("Test exception"); },
            ex -> "Handled: " + ex.getMessage()
        );
        System.out.println("Exception handling result: " + handledResult);
    }
}
```

### Common Generic Pitfalls and Solutions

```java
import java.util.*;

public class GenericPitfalls {
    // Pitfall 1: Type erasure issues
    public static class TypeErasureExample {
        // This won't compile - cannot overload based on generic type
        /*
        public static void process(List<String> list) { }
        public static void process(List<Integer> list) { }  // Compilation error
        */
        
        // Solution: Use different method names or additional parameters
        public static void processStrings(List<String> list) { }
        public static void processIntegers(List<Integer> list) { }
        
        // Or use a type token parameter
        public static <T> void process(List<T> list, Class<T> type) { }
    }
    
    // Pitfall 2: Cannot create arrays of generic types
    public static class ArrayCreationExample<T> {
        // This won't compile
        // private T[] array = new T[10];  // Compilation error
        
        // Solutions:
        private T[] array;
        
        @SuppressWarnings("unchecked")
        public ArrayCreationExample(Class<T> type, int size) {
            // Solution 1: Use reflection to create array
            this.array = (T[]) Array.newInstance(type, size);
        }
        
        public ArrayCreationExample(int size) {
            // Solution 2: Create array of Object and cast (with warning suppression)
            this.array = (T[]) new Object[size];
        }
        
        public T get(int index) {
            return array[index];
        }
        
        public void set(int index, T value) {
            array[index] = value;
        }
    }
    
    // Pitfall 3: Non-reifiable varargs with generics
    public static class VarargsPitfall {
        // This is potentially unsafe but allowed with @SafeVarargs
        @SafeVarargs
        public static <T> List<T> asList(T... items) {
            List<T> result = new ArrayList<>();
            Collections.addAll(result, items);
            return result;
        }
        
        // When varargs are modified, use explicit collections
        public static <T> void processItems(List<T>... lists) {
            for (List<T> list : lists) {
                System.out.println("Processing list: " + list);
            }
        }
    }
    
    // Pitfall 4: Generic type inference issues
    public static class InferenceIssues {
        public static <T> T getFirstOrNull(List<T> list) {
            return list.isEmpty() ? null : list.get(0);
        }
        
        // Sometimes need explicit type specification
        public static void demonstrateInference() {
            // This might have inference issues in complex scenarios
            List<String> strings = new ArrayList<>();
            String first = getFirstOrNull(strings);  // Usually works fine
            
            // In complex cases, might need explicit specification
            String firstExplicit = GenericPitfalls.<String>getFirstOrNull(strings);
        }
    }
    
    // Pitfall 5: Casts and unchecked warnings
    public static class UncheckedWarnings {
        // Method that might generate unchecked warnings
        @SuppressWarnings("unchecked")
        public static <T> T[] toArray(List<?> list, Class<T> type) {
            T[] array = (T[]) Array.newInstance(type, list.size());
            for (int i = 0; i < list.size(); i++) {
                array[i] = type.cast(list.get(i));  // Safe cast
            }
            return array;
        }
        
        // Safe alternative using generic array creation helper
        public static <T> T[] safeToArray(List<T> list, T[] template) {
            return list.toArray(template);
        }
    }
    
    // Complete example showing solutions to common pitfalls
    public static class GenericContainer<T> {
        private Object[] elements;
        private int size;
        
        public GenericContainer(int initialCapacity) {
            this.elements = new Object[initialCapacity];
            this.size = 0;
        }
        
        public void add(T element) {
            ensureCapacity();
            elements[size++] = element;
        }
        
        @SuppressWarnings("unchecked")
        public T get(int index) {
            if (index < 0 || index >= size) {
                throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + size);
            }
            return (T) elements[index];
        }
        
        public int size() {
            return size;
        }
        
        private void ensureCapacity() {
            if (size >= elements.length) {
                int newCapacity = elements.length == 0 ? 10 : elements.length * 2;
                elements = Arrays.copyOf(elements, newCapacity);
            }
        }
        
        // Safe array creation method
        @SuppressWarnings("unchecked")
        public T[] toArray(Class<T> type) {
            T[] result = (T[]) Array.newInstance(type, size);
            System.arraycopy(elements, 0, result, 0, size);
            return result;
        }
        
        // Type-safe contains method
        public boolean contains(Object element) {
            for (int i = 0; i < size; i++) {
                if (Objects.equals(elements[i], element)) {
                    return true;
                }
            }
            return false;
        }
    }
    
    public static void main(String[] args) {
        // Demonstrate solutions
        GenericContainer<String> container = new GenericContainer<>(5);
        container.add("Hello");
        container.add("World");
        container.add("Java");
        
        System.out.println("Container size: " + container.size());
        System.out.println("First element: " + container.get(0));
        System.out.println("Container contains 'World': " + container.contains("World"));
        
        // Convert to array safely
        String[] array = container.toArray(String.class);
        System.out.println("Array: " + Arrays.toString(array));
        
        // Varargs example
        List<String> list1 = Arrays.asList("a", "b");
        List<String> list2 = Arrays.asList("c", "d");
        VarargsPitfall.processItems(list1, list2);
        
        // Safe varargs usage
        List<String> convertedList = VarargsPitfall.asList("Hello", "Generic", "World");
        System.out.println("Varargs result: " + convertedList);
    }
}
```

## Advanced Generic Applications

### Generic Design Patterns

```java
import java.util.*;
import java.util.function.*;

// Advanced generic patterns and applications
public class AdvancedGenericPatterns {
    // 1. Generic Builder Pattern
    public static class PersonBuilder {
        private String firstName;
        private String lastName;
        private int age;
        private String email;
        
        public static <T extends PersonBuilder> T create(Class<T> builderClass) {
            try {
                return builderClass.getDeclaredConstructor().newInstance();
            } catch (Exception e) {
                throw new RuntimeException("Cannot create builder instance", e);
            }
        }
        
        public PersonBuilder firstName(String firstName) {
            this.firstName = firstName;
            return this;
        }
        
        public PersonBuilder lastName(String lastName) {
            this.lastName = lastName;
            return this;
        }
        
        public PersonBuilder age(int age) {
            this.age = age;
            return this;
        }
        
        public PersonBuilder email(String email) {
            this.email = email;
            return this;
        }
        
        public Person build() {
            return new Person(firstName, lastName, age, email);
        }
    }
    
    // 2. Generic Strategy Pattern
    public static interface ValidationStrategy<T> {
        boolean isValid(T value);
        String getErrorMessage();
    }
    
    public static class EmailValidationStrategy implements ValidationStrategy<String> {
        @Override
        public boolean isValid(String email) {
            return email != null && email.contains("@") && email.contains(".");
        }
        
        @Override
        public String getErrorMessage() {
            return "Invalid email format";
        }
    }
    
    public static class AgeValidationStrategy implements ValidationStrategy<Integer> {
        private int minAge;
        private int maxAge;
        
        public AgeValidationStrategy(int minAge, int maxAge) {
            this.minAge = minAge;
            this.maxAge = maxAge;
        }
        
        @Override
        public boolean isValid(Integer age) {
            return age != null && age >= minAge && age <= maxAge;
        }
        
        @Override
        public String getErrorMessage() {
            return "Age must be between " + minAge + " and " + maxAge;
        }
    }
    
    // Generic validator class
    public static class GenericValidator<T> {
        private ValidationStrategy<T> strategy;
        
        public GenericValidator(ValidationStrategy<T> strategy) {
            this.strategy = strategy;
        }
        
        public ValidationResult validate(T value) {
            boolean valid = strategy.isValid(value);
            return new ValidationResult(valid, valid ? null : strategy.getErrorMessage());
        }
    }
    
    public static class ValidationResult {
        private boolean valid;
        private String errorMessage;
        
        public ValidationResult(boolean valid, String errorMessage) {
            this.valid = valid;
            this.errorMessage = errorMessage;
        }
        
        public boolean isValid() { return valid; }
        public String getErrorMessage() { return errorMessage; }
        
        @Override
        public String toString() {
            return "ValidationResult{valid=" + valid + ", errorMessage='" + errorMessage + "'}";
        }
    }
    
    // 3. Generic Repository Pattern
    public static interface GenericRepository<T, ID> {
        T save(T entity);
        Optional<T> findById(ID id);
        List<T> findAll();
        void delete(T entity);
        boolean exists(ID id);
    }
    
    public static class InMemoryRepository<T, ID> implements GenericRepository<T, ID> {
        private Map<ID, T> storage = new HashMap<>();
        private Function<T, ID> idExtractor;
        
        public InMemoryRepository(Function<T, ID> idExtractor) {
            this.idExtractor = idExtractor;
        }
        
        @Override
        public T save(T entity) {
            ID id = idExtractor.apply(entity);
            storage.put(id, entity);
            return entity;
        }
        
        @Override
        public Optional<T> findById(ID id) {
            return Optional.ofNullable(storage.get(id));
        }
        
        @Override
        public List<T> findAll() {
            return new ArrayList<>(storage.values());
        }
        
        @Override
        public void delete(T entity) {
            ID id = idExtractor.apply(entity);
            storage.remove(id);
        }
        
        @Override
        public boolean exists(ID id) {
            return storage.containsKey(id);
        }
    }
    
    // Example entity classes
    static class User {
        private Long id;
        private String name;
        private String email;
        private int age;
        
        public User(Long id, String name, String email, int age) {
            this.id = id;
            this.name = name;
            this.email = email;
            this.age = age;
        }
        
        // Getters and setters
        public Long getId() { return id; }
        public void setId(Long id) { this.id = id; }
        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
        public String getEmail() { return email; }
        public void setEmail(String email) { this.email = email; }
        public int getAge() { return age; }
        public void setAge(int age) { this.age = age; }
        
        @Override
        public String toString() {
            return "User{id=" + id + ", name='" + name + "', email='" + email + "', age=" + age + "}";
        }
    }
    
    static class Product {
        private String sku;
        private String name;
        private double price;
        
        public Product(String sku, String name, double price) {
            this.sku = sku;
            this.name = name;
            this.price = price;
        }
        
        // Getters
        public String getSku() { return sku; }
        public String getName() { return name; }
        public double getPrice() { return price; }
        
        @Override
        public String toString() {
            return "Product{sku='" + sku + "', name='" + name + "', price=" + price + "}";
        }
    }
    
    public static void main(String[] args) {
        // Builder pattern example
        Person person = new PersonBuilder()
            .firstName("John")
            .lastName("Doe")
            .age(30)
            .email("john.doe@example.com")
            .build();
        System.out.println("Built person: " + person);
        
        // Validation pattern example
        GenericValidator<String> emailValidator = new GenericValidator<>(new EmailValidationStrategy());
        GenericValidator<Integer> ageValidator = new GenericValidator<>(new AgeValidationStrategy(18, 100));
        
        ValidationResult emailResult = emailValidator.validate("test@example.com");
        ValidationResult ageResult = ageValidator.validate(25);
        ValidationResult invalidAgeResult = ageValidator.validate(15);
        
        System.out.println("Email validation: " + emailResult);
        System.out.println("Age validation (25): " + ageResult);
        System.out.println("Age validation (15): " + invalidAgeResult);
        
        // Repository pattern example
        GenericRepository<User, Long> userRepository = 
            new InMemoryRepository<>(User::getId);
        
        GenericRepository<Product, String> productRepository = 
            new InMemoryRepository<>(Product::getSku);
        
        // Add users
        User user1 = new User(1L, "Alice", "alice@example.com", 28);
        User user2 = new User(2L, "Bob", "bob@example.com", 35);
        
        userRepository.save(user1);
        userRepository.save(user2);
        
        System.out.println("All users: " + userRepository.findAll());
        System.out.println("User 1: " + userRepository.findById(1L));
        
        // Add products
        Product product1 = new Product("P001", "Laptop", 999.99);
        Product product2 = new Product("P002", "Mouse", 29.99);
        
        productRepository.save(product1);
        productRepository.save(product2);
        
        System.out.println("All products: " + productRepository.findAll());
        System.out.println("Product P001: " + productRepository.findById("P001"));
    }
}

// Person class for the example
class Person {
    private String firstName;
    private String lastName;
    private int age;
    private String email;
    
    public Person(String firstName, String lastName, int age, String email) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.age = age;
        this.email = email;
    }
    
    // Getters
    public String getFirstName() { return firstName; }
    public String getLastName() { return lastName; }
    public int getAge() { return age; }
    public String getEmail() { return email; }
    
    @Override
    public String toString() {
        return "Person{firstName='" + firstName + "', lastName='" + lastName + 
               "', age=" + age + ", email='" + email + "'}";
    }
}
```

## Summary

Java Generics provide powerful type safety and code reusability features. Key takeaways include:

1. **Basic Generics**: Understand type parameters, generic classes, methods, and interfaces
2. **Wildcards**: Master PECS (Producer Extends, Consumer Super) principle for bounded wildcards
3. **Type Bounds**: Use upper bounds (`extends`) and lower bounds (`super`) appropriately
4. **Best Practices**: Follow naming conventions, avoid raw types, handle unchecked warnings properly
5. **Common Pitfalls**: Understand type erasure, array creation issues, and variance problems
6. **Advanced Applications**: Apply generics to design patterns like Builder, Strategy, and Repository

Mastering generics will make your Java code more robust, maintainable, and reusable while providing compile-time type safety. Generics are essential for working with Java collections and modern Java frameworks.