# Java Collections Framework: Deep Dive into Data Structures and Algorithms

## Introduction

The Java Collections Framework (JCF) is a unified architecture for representing and manipulating collections, providing both implementations and interfaces for common data structures. It's one of the most important components of the Java platform, offering high-performance, reusable data structures and algorithms.

This comprehensive guide explores the Collections Framework in depth, covering all major interfaces, implementations, and advanced usage patterns.

## Core Collection Interfaces

### Collection Interface Hierarchy

The Collections Framework is built around a hierarchy of core interfaces:

```
Iterable
 └── Collection
     ├── List
     ├── Set
     └── Queue
         └── Deque
```

### Collection Interface

The `Collection` interface is the root interface for all collection types, defining basic operations like adding, removing, and checking elements.

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class CollectionInterfaceDemo {
    public static void main(String[] args) {
        // Demonstrating Collection interface methods
        Collection<String> collection = new ArrayList<>();
        
        // add() - adds element to collection
        collection.add("Apple");
        collection.add("Banana");
        collection.add("Cherry");
        
        System.out.println("Collection: " + collection);
        System.out.println("Size: " + collection.size());
        System.out.println("Contains 'Apple': " + collection.contains("Apple"));
        
        // addAll() - adds all elements from another collection
        Collection<String> moreFruits = Arrays.asList("Date", "Elderberry");
        collection.addAll(moreFruits);
        System.out.println("After addAll: " + collection);
        
        // remove() and removeAll()
        collection.remove("Banana");
        System.out.println("After removing Banana: " + collection);
        
        Collection<String> toRemove = Arrays.asList("Cherry", "Date");
        collection.removeAll(toRemove);
        System.out.println("After removing multiple: " + collection);
        
        // retainAll() - keeps only specified elements
        Collection<String> toKeep = Arrays.asList("Apple", "Fig");
        collection.retainAll(toKeep);
        collection.add("Fig"); // Add Fig to make the example work
        System.out.println("After retainAll: " + collection);
        
        // Iteration methods
        System.out.println("Using iterator:");
        Iterator<String> iterator = collection.iterator();
        while (iterator.hasNext()) {
            System.out.println("  " + iterator.next());
        }
        
        System.out.println("Using enhanced for loop:");
        for (String fruit : collection) {
            System.out.println("  " + fruit);
        }
        
        System.out.println("Using forEach with lambda:");
        collection.forEach(System.out::println);
        
        // toArray() methods
        Object[] array = collection.toArray();
        String[] stringArray = collection.toArray(new String[0]);
        System.out.println("Array: " + Arrays.toString(array));
        System.out.println("String array: " + Arrays.toString(stringArray));
        
        // isEmpty() and clear()
        System.out.println("Is empty: " + collection.isEmpty());
        collection.clear();
        System.out.println("After clear, is empty: " + collection.isEmpty());
    }
}
```

### List Interface

The `List` interface extends `Collection` and represents an ordered collection (sequence) that allows duplicate elements.

```java
import java.util.*;

public class ListInterfaceDemo {
    public static void main(String[] args) {
        // ArrayList implementation
        List<String> arrayList = new ArrayList<>();
        arrayList.add("Zero");
        arrayList.add("One");
        arrayList.add("Two");
        arrayList.add(1, "One and a half"); // Insert at specific position
        
        System.out.println("ArrayList: " + arrayList);
        System.out.println("Element at index 1: " + arrayList.get(1));
        
        // Update element at specific index
        arrayList.set(0, "Modified Zero");
        System.out.println("After set: " + arrayList);
        
        // indexOf() and lastIndexOf()
        arrayList.add("Two"); // Add duplicate
        System.out.println("First index of 'Two': " + arrayList.indexOf("Two"));
        System.out.println("Last index of 'Two': " + arrayList.lastIndexOf("Two"));
        
        // Sublist
        List<String> subList = arrayList.subList(1, 3);
        System.out.println("Sublist [1,3): " + subList);
        
        // Range operations
        List<String> range = new ArrayList<>();
        range.addAll(Arrays.asList("A", "B", "C", "D", "E"));
        List<String> rangeSub = range.subList(1, 4);
        System.out.println("Original range: " + range);
        System.out.println("Range sub: " + rangeSub);
        rangeSub.set(0, "Modified B"); // Modifies original list
        System.out.println("After modifying sub: " + range);
        
        // ArrayList vs LinkedList performance comparison
        System.out.println("\n--- Performance Comparison ---");
        
        // Random access performance (ArrayList wins)
        List<Integer> arrayListPerf = new ArrayList<>();
        List<Integer> linkedListPerf = new LinkedList<>();
        
        int size = 100000;
        for (int i = 0; i < size; i++) {
            arrayListPerf.add(i);
            linkedListPerf.add(i);
        }
        
        // Measure random access time
        long start = System.nanoTime();
        for (int i = 0; i < 1000; i++) {
            arrayListPerf.get(size / 2);
        }
        long arrayListTime = System.nanoTime() - start;
        
        start = System.nanoTime();
        for (int i = 0; i < 1000; i++) {
            linkedListPerf.get(size / 2);
        }
        long linkedListTime = System.nanoTime() - start;
        
        System.out.println("ArrayList random access (1000 ops): " + arrayListTime / 1_000_000 + "ms");
        System.out.println("LinkedList random access (1000 ops): " + linkedListTime / 1_000_000 + "ms");
        
        // Insertion at beginning (LinkedList wins)
        List<Integer> arrayListInsert = new ArrayList<>();
        List<Integer> linkedListInsert = new LinkedList<>();
        
        start = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            arrayListInsert.add(0, i);
        }
        arrayListTime = System.nanoTime() - start;
        
        start = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            linkedListInsert.add(0, i);
        }
        linkedListTime = System.nanoTime() - start;
        
        System.out.println("ArrayList insertion at beginning (10000 ops): " + arrayListTime / 1_000_000 + "ms");
        System.out.println("LinkedList insertion at beginning (10000 ops): " + linkedListTime / 1_000_000 + "ms");
    }
}
```

### Set Interface

The `Set` interface extends `Collection` and represents a collection that cannot contain duplicate elements.

```java
import java.util.*;
import java.util.concurrent.*;

public class SetInterfaceDemo {
    public static void main(String[] args) {
        // HashSet - No guaranteed order, fast operations
        Set<String> hashSet = new HashSet<>();
        hashSet.add("Apple");
        hashSet.add("Banana");
        hashSet.add("Cherry");
        hashSet.add("Apple"); // Duplicate - won't be added
        hashSet.add("Date");
        
        System.out.println("HashSet: " + hashSet);
        System.out.println("Size: " + hashSet.size());
        
        // LinkedHashSet - Maintains insertion order
        Set<String> linkedHashSet = new LinkedHashSet<>();
        linkedHashSet.add("First");
        linkedHashSet.add("Second");
        linkedHashSet.add("Third");
        linkedHashSet.add("First"); // Duplicate - won't be added
        
        System.out.println("LinkedHashSet: " + linkedHashSet);
        
        // TreeSet - Maintains sorted order
        Set<String> treeSet = new TreeSet<>();
        treeSet.add("Cherry");
        treeSet.add("Apple");
        treeSet.add("Banana");
        treeSet.add("Date");
        
        System.out.println("TreeSet: " + treeSet);
        
        // TreeSet with custom comparator
        Set<String> treeSetCustom = new TreeSet<>(String.CASE_INSENSITIVE_ORDER);
        treeSetCustom.add("banana");
        treeSetCustom.add("Apple");
        treeSetCustom.add("cherry");
        treeSetCustom.add("Banana");
        
        System.out.println("TreeSet (case insensitive): " + treeSetCustom);
        
        // Set operations
        Set<Integer> set1 = new HashSet<>(Arrays.asList(1, 2, 3, 4, 5));
        Set<Integer> set2 = new HashSet<>(Arrays.asList(4, 5, 6, 7, 8));
        
        System.out.println("\nSet operations:");
        System.out.println("Set 1: " + set1);
        System.out.println("Set 2: " + set2);
        
        // Union (using addAll)
        Set<Integer> union = new HashSet<>(set1);
        union.addAll(set2);
        System.out.println("Union: " + union);
        
        // Intersection (using retainAll)
        Set<Integer> intersection = new HashSet<>(set1);
        intersection.retainAll(set2);
        System.out.println("Intersection: " + intersection);
        
        // Difference (using removeAll)
        Set<Integer> difference = new HashSet<>(set1);
        difference.removeAll(set2);
        System.out.println("Difference (set1 - set2): " + difference);
        
        // Symmetric difference
        Set<Integer> symDiff = new HashSet<>();
        symDiff.addAll(set1);
        symDiff.addAll(set2);
        Set<Integer> temp = new HashSet<>(set1);
        temp.retainAll(set2);
        symDiff.removeAll(temp);
        System.out.println("Symmetric difference: " + symDiff);
        
        // TreeSet navigation methods
        SortedSet<Integer> sortedSet = new TreeSet<>(Arrays.asList(1, 3, 5, 7, 9, 11));
        System.out.println("\nSortedSet: " + sortedSet);
        System.out.println("First: " + sortedSet.first());
        System.out.println("Last: " + sortedSet.last());
        System.out.println("HeadSet (5): " + sortedSet.headSet(5));
        System.out.println("TailSet (5): " + sortedSet.tailSet(5));
        System.out.println("SubSet [3,9): " + sortedSet.subSet(3, 9));
        
        // NavigableSet methods
        NavigableSet<Integer> navigableSet = (NavigableSet<Integer>) sortedSet;
        System.out.println("\nNavigableSet methods:");
        System.out.println("Lower (5): " + navigableSet.lower(5));
        System.out.println("Higher (5): " + navigableSet.higher(5));
        System.out.println("Floor (5): " + navigableSet.floor(5));
        System.out.println("Ceiling (5): " + navigableSet.ceiling(5));
        System.out.println("PollFirst: " + navigableSet.pollFirst());
        System.out.println("After pollFirst: " + navigableSet);
        System.out.println("PollLast: " + navigableSet.pollLast());
        System.out.println("After pollLast: " + navigableSet);
    }
}
```

### Queue Interface

The `Queue` interface extends `Collection` and represents a collection designed for holding elements prior to processing.

```java
import java.util.*;
import java.util.concurrent.*;

public class QueueInterfaceDemo {
    public static void main(String[] args) throws InterruptedException {
        // LinkedList as Queue
        Queue<String> queue = new LinkedList<>();
        queue.offer("First");  // Same as add() but preferred
        queue.offer("Second");
        queue.offer("Third");
        
        System.out.println("Queue: " + queue);
        System.out.println("Element (peek): " + queue.element()); // Throws exception if empty
        System.out.println("Peek: " + queue.peek()); // Returns null if empty
        
        System.out.println("Poll: " + queue.poll()); // Removes and returns, null if empty
        System.out.println("After poll: " + queue);
        
        System.out.println("Remove: " + queue.remove()); // Removes and returns, throws exception if empty
        System.out.println("After remove: " + queue);
        
        // PriorityQueue - maintains heap order
        Queue<Integer> priorityQueue = new PriorityQueue<>();
        priorityQueue.offer(5);
        priorityQueue.offer(1);
        priorityQueue.offer(3);
        priorityQueue.offer(2);
        priorityQueue.offer(4);
        
        System.out.println("\nPriorityQueue: " + priorityQueue); // May not show sorted order
        System.out.println("Elements in priority order:");
        while (!priorityQueue.isEmpty()) {
            System.out.print(priorityQueue.poll() + " ");
        }
        System.out.println();
        
        // Priority queue with custom comparator
        Queue<String> priorityQueueCustom = new PriorityQueue<>(String.CASE_INSENSITIVE_ORDER);
        priorityQueueCustom.offer("banana");
        priorityQueueCustom.offer("Apple");
        priorityQueueCustom.offer("cherry");
        priorityQueueCustom.offer("Banana");
        
        System.out.println("\nPriorityQueue (case insensitive):");
        while (!priorityQueueCustom.isEmpty()) {
            System.out.print(priorityQueueCustom.poll() + " ");
        }
        System.out.println();
        
        // Custom priority with objects
        Queue<Task> taskQueue = new PriorityQueue<>();
        taskQueue.offer(new Task("Low priority", 3));
        taskQueue.offer(new Task("High priority", 1));
        taskQueue.offer(new Task("Medium priority", 2));
        taskQueue.offer(new Task("Critical priority", 0));
        
        System.out.println("\nTask queue (by priority):");
        while (!taskQueue.isEmpty()) {
            System.out.println(taskQueue.poll());
        }
        
        // Deque (Double-ended queue)
        Deque<String> deque = new ArrayDeque<>();
        deque.offerFirst("First");
        deque.offerLast("Last");
        deque.offer("Middle"); // Same as offerLast
        
        System.out.println("\nDeque: " + deque);
        System.out.println("PollFirst: " + deque.pollFirst());
        System.out.println("PollLast: " + deque.pollLast());
        System.out.println("After polls: " + deque);
        
        // Stack operations with Deque
        Deque<String> stack = new ArrayDeque<>();
        stack.push("Bottom");
        stack.push("Middle");
        stack.push("Top");
        
        System.out.println("\nStack: " + stack);
        System.out.println("Pop: " + stack.pop());
        System.out.println("After pop: " + stack);
        
        // BlockingQueue example
        BlockingQueue<String> blockingQueue = new ArrayBlockingQueue<>(3);
        
        // Producer thread
        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    String item = "Item-" + i;
                    blockingQueue.put(item); // Blocks if queue is full
                    System.out.println("Produced: " + item);
                    Thread.sleep(100);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        // Consumer thread
        Thread consumer = new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    String item = blockingQueue.take(); // Blocks if queue is empty
                    System.out.println("Consumed: " + item);
                    Thread.sleep(150);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        producer.start();
        consumer.start();
        
        producer.join();
        consumer.join();
    }
}

class Task implements Comparable<Task> {
    private String name;
    private int priority;
    
    public Task(String name, int priority) {
        this.name = name;
        this.priority = priority;
    }
    
    @Override
    public int compareTo(Task other) {
        return Integer.compare(this.priority, other.priority);
    }
    
    @Override
    public String toString() {
        return name + "(priority: " + priority + ")";
    }
}
```

## Map Interface and Implementations

### Map Interface Hierarchy

Maps are not collections but are part of the Collections Framework:

```
Map
 ├── HashMap
 ├── LinkedHashMap
 └── TreeMap
     └── ConcurrentSkipListMap
```

### Map Interface and Basic Implementations

```java
import java.util.*;
import java.util.concurrent.*;

public class MapInterfaceDemo {
    public static void main(String[] args) {
        // HashMap - Basic usage
        Map<String, Integer> hashMap = new HashMap<>();
        hashMap.put("Apple", 5);
        hashMap.put("Banana", 3);
        hashMap.put("Cherry", 8);
        hashMap.put("Date", 12);
        
        System.out.println("HashMap: " + hashMap);
        System.out.println("Value for 'Apple': " + hashMap.get("Apple"));
        System.out.println("Contains key 'Banana': " + hashMap.containsKey("Banana"));
        System.out.println("Contains value 8: " + hashMap.containsValue(8));
        
        // putIfAbsent - only adds if key doesn't exist
        hashMap.putIfAbsent("Apple", 10); // Won't change value since key exists
        System.out.println("After putIfAbsent 'Apple': " + hashMap.get("Apple"));
        
        hashMap.putIfAbsent("Elderberry", 15); // Will add new entry
        System.out.println("After putIfAbsent 'Elderberry': " + hashMap);
        
        // replace methods
        hashMap.replace("Banana", 3, 6); // Replace only if current value matches
        System.out.println("After conditional replace: " + hashMap);
        
        hashMap.replace("Banana", 7); // Unconditional replace
        System.out.println("After unconditional replace: " + hashMap);
        
        // compute methods (Java 8+)
        hashMap.compute("Apple", (key, value) -> value * 2);
        System.out.println("After compute: " + hashMap);
        
        // computeIfPresent and computeIfAbsent
        hashMap.computeIfPresent("NonExistent", (key, value) -> value + 1); // Won't execute
        hashMap.computeIfAbsent("Fig", key -> 9); // Will add new entry
        System.out.println("After computeIfAbsent: " + hashMap);
        
        // merge method
        hashMap.merge("Grape", 4, Integer::sum); // Add new entry
        hashMap.merge("Apple", 10, Integer::sum); // Add to existing value
        System.out.println("After merge: " + hashMap);
        
        // Iteration methods
        System.out.println("\n--- Iteration methods ---");
        System.out.println("Keys: " + hashMap.keySet());
        System.out.println("Values: " + hashMap.values());
        System.out.println("Entries: " + hashMap.entrySet());
        
        // Iterate over entries
        for (Map.Entry<String, Integer> entry : hashMap.entrySet()) {
            System.out.println(entry.getKey() + " = " + entry.getValue());
        }
        
        // Using forEach with lambda
        hashMap.forEach((key, value) -> 
            System.out.println(key + " has count " + value));
        
        // LinkedHashMap - maintains insertion order
        Map<String, Integer> linkedHashMap = new LinkedHashMap<>();
        linkedHashMap.put("First", 1);
        linkedHashMap.put("Second", 2);
        linkedHashMap.put("Third", 3);
        linkedHashMap.put("First", 10); // Update existing key
        
        System.out.println("\nLinkedHashMap: " + linkedHashMap);
        
        // AccessOrder LinkedHashMap (for LRU cache)
        Map<String, Integer> accessOrderMap = new LinkedHashMap<String, Integer>(16, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<String, Integer> eldest) {
                return size() > 3; // Maintain only 3 entries
            }
        };
        
        accessOrderMap.put("A", 1);
        accessOrderMap.put("B", 2);
        accessOrderMap.put("C", 3);
        System.out.println("Before access: " + accessOrderMap);
        
        accessOrderMap.get("A"); // Access 'A' - it becomes most recently used
        accessOrderMap.put("D", 4); // Add 'D' - 'B' should be removed due to size limit
        System.out.println("After access and add: " + accessOrderMap);
        
        // TreeMap - maintains sorted order
        Map<String, Integer> treeMap = new TreeMap<>();
        treeMap.put("Cherry", 8);
        treeMap.put("Apple", 5);
        treeMap.put("Banana", 3);
        treeMap.put("Date", 12);
        
        System.out.println("\nTreeMap: " + treeMap);
        
        // NavigableMap operations
        NavigableMap<String, Integer> navMap = (NavigableMap<String, Integer>) treeMap;
        System.out.println("First entry: " + navMap.firstEntry());
        System.out.println("Last entry: " + navMap.lastEntry());
        System.out.println("Lower entry ('B'): " + navMap.lowerEntry("B"));
        System.out.println("Higher entry ('B'): " + navMap.higherEntry("B"));
        System.out.println("Floor entry ('C'): " + navMap.floorEntry("C"));
        System.out.println("Ceiling entry ('C'): " + navMap.ceilingEntry("C"));
        
        // Submap operations
        System.out.println("SubMap [A, C): " + navMap.subMap("A", "C"));
        System.out.println("HeadMap ('C'): " + navMap.headMap("C"));
        System.out.println("TailMap ('B'): " + navMap.tailMap("B"));
    }
}
```

### ConcurrentHashMap for Thread Safety

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class ConcurrentMapDemo {
    public static void main(String[] args) throws InterruptedException {
        // Basic ConcurrentHashMap usage
        ConcurrentHashMap<String, Integer> concurrentMap = new ConcurrentHashMap<>();
        
        // Thread-safe operations
        concurrentMap.put("Apple", 5);
        concurrentMap.put("Banana", 3);
        
        // Atomic operations
        concurrentMap.putIfAbsent("Cherry", 8);
        concurrentMap.replace("Banana", 3, 6); // Atomic conditional replacement
        
        System.out.println("Initial map: " + concurrentMap);
        
        // KeySet view that supports removal
        Set<String> keySet = concurrentMap.keySet();
        System.out.println("KeySet: " + keySet);
        
        // Atomic operations on values
        concurrentMap.compute("Apple", (key, value) -> value != null ? value + 1 : 1);
        concurrentMap.merge("Grape", 4, Integer::sum);
        
        System.out.println("After compute/merge: " + concurrentMap);
        
        // Parallel operations
        AtomicInteger total = new AtomicInteger(0);
        concurrentMap.forEach((key, value) -> {
            total.addAndGet(value);
        });
        System.out.println("Total count: " + total.get());
        
        // Parallel reduction
        Integer sum = concurrentMap.reduceValues(2, Integer::sum);
        System.out.println("Parallel sum: " + sum);
        
        // Concurrent modification example
        ExecutorService executor = Executors.newFixedThreadPool(10);
        CountDownLatch latch = new CountDownLatch(100);
        
        for (int i = 0; i < 100; i++) {
            final int taskId = i;
            executor.submit(() -> {
                try {
                    String key = "Thread-" + taskId % 10;
                    concurrentMap.compute(key, (k, v) -> (v == null) ? 1 : v + 1);
                    Thread.sleep(10); // Simulate work
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    latch.countDown();
                }
            });
        }
        
        latch.await();
        executor.shutdown();
        
        System.out.println("After concurrent modifications: " + concurrentMap);
        
        // Performance comparison
        System.out.println("\n--- Performance Comparison ---");
        
        final int iterations = 100000;
        final int threads = 10;
        
        // HashMap with synchronization
        Map<String, Integer> synchronizedMap = Collections.synchronizedMap(new HashMap<>());
        testMapPerformance(synchronizedMap, "Synchronized HashMap", iterations, threads);
        
        // ConcurrentHashMap
        Map<String, Integer> concurrentHashMap = new ConcurrentHashMap<>();
        testMapPerformance(concurrentHashMap, "ConcurrentHashMap", iterations, threads);
    }
    
    private static void testMapPerformance(Map<String, Integer> map, String name, int iterations, int threads) {
        ExecutorService executor = Executors.newFixedThreadPool(threads);
        CountDownLatch latch = new CountDownLatch(threads);
        
        long startTime = System.currentTimeMillis();
        
        for (int t = 0; t < threads; t++) {
            executor.submit(() -> {
                for (int i = 0; i < iterations / threads; i++) {
                    String key = "key-" + (i % 100);
                    map.compute(key, (k, v) -> (v == null) ? 1 : v + 1);
                }
                latch.countDown();
            });
        }
        
        try {
            latch.await();
            long endTime = System.currentTimeMillis();
            System.out.println(name + " time: " + (endTime - startTime) + "ms");
            System.out.println("Final map size: " + map.size());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            executor.shutdown();
        }
    }
}
```

## Advanced Collection Operations

### Streams Integration

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.function.*;
import java.util.stream.*;

public class StreamsIntegrationDemo {
    public static void main(String[] args) {
        List<String> words = Arrays.asList("hello", "world", "java", "stream", "api", "collections");
        
        // Filter and collect
        List<String> longWords = words.stream()
            .filter(word -> word.length() > 4)
            .collect(Collectors.toList());
        System.out.println("Long words: " + longWords);
        
        // Map and collect
        Set<Integer> lengths = words.stream()
            .map(String::length)
            .collect(Collectors.toSet());
        System.out.println("Word lengths: " + lengths);
        
        // Grouping
        Map<Integer, List<String>> groupedByLength = words.stream()
            .collect(Collectors.groupingBy(String::length));
        System.out.println("Grouped by length: " + groupedByLength);
        
        // Partitioning
        Map<Boolean, List<String>> partitioned = words.stream()
            .collect(Collectors.partitioningBy(word -> word.length() > 4));
        System.out.println("Partitioned (>4): " + partitioned);
        
        // Advanced grouping with downstream collector
        Map<Integer, Long> lengthCount = words.stream()
            .collect(Collectors.groupingBy(
                String::length,
                Collectors.counting()
            ));
        System.out.println("Length count: " + lengthCount);
        
        // Custom collectors
        String concatenated = words.stream()
            .collect(Collectors.joining(", "));
        System.out.println("Joined: " + concatenated);
        
        // Statistics
        IntSummaryStatistics stats = words.stream()
            .mapToInt(String::length)
            .summaryStatistics();
        System.out.println("Length statistics: " + stats);
        
        // Parallel processing
        List<Integer> numbers = IntStream.range(1, 1000000)
            .boxed()
            .collect(Collectors.toList());
        
        long startTime = System.currentTimeMillis();
        OptionalInt sequentialMax = numbers.stream()
            .mapToInt(Integer::intValue)
            .max();
        long sequentialTime = System.currentTimeMillis() - startTime;
        
        startTime = System.currentTimeMillis();
        OptionalInt parallelMax = numbers.parallelStream()
            .mapToInt(Integer::intValue)
            .max();
        long parallelTime = System.currentTimeMillis() - startTime;
        
        System.out.println("\nMax value (sequential): " + sequentialMax.getAsInt() + ", Time: " + sequentialTime + "ms");
        System.out.println("Max value (parallel): " + parallelMax.getAsInt() + ", Time: " + parallelTime + "ms");
        
        // Collecting into custom collections
        Deque<String> deque = words.stream()
            .filter(word -> word.length() > 3)
            .collect(Collectors.toCollection(ArrayDeque::new));
        System.out.println("Deque: " + deque);
        
        // Grouping with custom map factory
        SortedMap<Integer, Set<String>> sortedGrouping = words.stream()
            .collect(Collectors.groupingBy(
                String::length,
                TreeMap::new,
                Collectors.toSet()
            ));
        System.out.println("Sorted grouping: " + sortedGrouping);
        
        // Student example with complex operations
        List<Student> students = Arrays.asList(
            new Student("Alice", 85, "Math"),
            new Student("Bob", 92, "Science"),
            new Student("Charlie", 78, "Math"),
            new Student("Diana", 95, "Science"),
            new Student("Eve", 88, "Math"),
            new Student("Frank", 91, "Science")
        );
        
        // Group by subject and calculate average
        Map<String, Double> avgBySubject = students.stream()
            .collect(Collectors.groupingBy(
                Student::getSubject,
                Collectors.averagingInt(Student::getGrade)
            ));
        System.out.println("\nAverage grades by subject: " + avgBySubject);
        
        // Find top student in each subject
        Map<String, Optional<Student>> topBySubject = students.stream()
            .collect(Collectors.groupingBy(
                Student::getSubject,
                Collectors.maxBy(Comparator.comparingInt(Student::getGrade))
            ));
        System.out.println("Top students: " + topBySubject);
        
        // Count students in each grade range
        Map<String, Long> gradeRanges = students.stream()
            .collect(Collectors.groupingBy(
                student -> {
                    int grade = student.getGrade();
                    if (grade >= 90) return "A (90-100)";
                    else if (grade >= 80) return "B (80-89)";
                    else return "Below B (<80)";
                },
                Collectors.counting()
            ));
        System.out.println("Grade ranges: " + gradeRanges);
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

### Collection Utilities and Algorithms

```java
import java.util.*;
import java.util.concurrent.ThreadLocalRandom;

public class CollectionUtilitiesDemo {
    public static void main(String[] args) {
        // Collections class utilities
        List<Integer> numbers = Arrays.asList(3, 1, 4, 1, 5, 9, 2, 6, 5, 3);
        System.out.println("Original: " + numbers);
        
        // Sorting
        List<Integer> sorted = new ArrayList<>(numbers);
        Collections.sort(sorted);
        System.out.println("Sorted: " + sorted);
        
        // Binary search (requires sorted list)
        int index = Collections.binarySearch(sorted, 5);
        System.out.println("Index of 5: " + index);
        
        // Frequency count
        int freq = Collections.frequency(numbers, 1);
        System.out.println("Frequency of 1: " + freq);
        
        // Disjoint sets
        List<Integer> list1 = Arrays.asList(1, 2, 3);
        List<Integer> list2 = Arrays.asList(4, 5, 6);
        List<Integer> list3 = Arrays.asList(3, 4, 5);
        
        System.out.println("Lists 1 and 2 disjoint: " + Collections.disjoint(list1, list2));
        System.out.println("Lists 1 and 3 disjoint: " + Collections.disjoint(list1, list3));
        
        // Shuffling
        List<Integer> shuffled = new ArrayList<>(numbers);
        Collections.shuffle(shuffled);
        System.out.println("Shuffled: " + shuffled);
        
        // Reversing
        List<Integer> reversed = new ArrayList<>(numbers);
        Collections.reverse(reversed);
        System.out.println("Reversed: " + reversed);
        
        // Filling
        List<String> filled = new ArrayList<>(Arrays.asList("a", "b", "c", "d", "e"));
        Collections.fill(filled, "X");
        System.out.println("Filled: " + filled);
        
        // Collections with specific properties
        List<String> emptyList = Collections.emptyList();
        Set<Integer> emptySet = Collections.emptySet();
        Map<String, Integer> emptyMap = Collections.emptyMap();
        
        System.out.println("Empty list: " + emptyList);
        System.out.println("Empty set: " + emptySet);
        System.out.println("Empty map: " + emptyMap);
        
        // Unmodifiable collections
        List<String> original = Arrays.asList("a", "b", "c");
        List<String> unmodifiable = Collections.unmodifiableList(new ArrayList<>(original));
        
        System.out.println("Unmodifiable: " + unmodifiable);
        try {
            unmodifiable.add("d"); // This will throw UnsupportedOperationException
        } catch (UnsupportedOperationException e) {
            System.out.println("Cannot modify unmodifiable list: " + e.getMessage());
        }
        
        // Synchronized collections
        List<String> synchronizedList = Collections.synchronizedList(new ArrayList<>());
        Set<String> synchronizedSet = Collections.synchronizedSet(new HashSet<>());
        Map<String, String> synchronizedMap = Collections.synchronizedMap(new HashMap<>());
        
        // Min and Max
        int min = Collections.min(numbers);
        int max = Collections.max(numbers);
        System.out.println("Min: " + min + ", Max: " + max);
        
        // Min and Max with custom comparator
        List<String> words = Arrays.asList("apple", "a", "elephant", "hi");
        String shortest = Collections.min(words, Comparator.comparing(String::length));
        String longest = Collections.max(words, Comparator.comparing(String::length));
        System.out.println("Shortest: " + shortest + ", Longest: " + longest);
        
        // Rotating
        List<Integer> rotated = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        Collections.rotate(rotated, 2); // Move last 2 elements to front
        System.out.println("Rotated by 2: " + rotated);
        
        // Swap
        List<String> swapList = new ArrayList<>(Arrays.asList("a", "b", "c", "d"));
        Collections.swap(swapList, 0, 2);
        System.out.println("After swapping 0 and 2: " + swapList);
        
        // Copy
        List<String> source = Arrays.asList("1", "2", "3", "4", "5");
        List<String> destination = Arrays.asList("a", "b", "c", "d", "e");
        List<String> destList = new ArrayList<>(destination);
        Collections.copy(destList, source);
        System.out.println("After copy: " + destList);
        
        // New algorithms in Java 8+
        List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10));
        
        // RemoveIf
        List<Integer> evenNumbers = new ArrayList<>(list);
        evenNumbers.removeIf(n -> n % 2 == 0);
        System.out.println("After removing evens: " + evenNumbers);
        
        // ReplaceAll
        List<String> strings = new ArrayList<>(Arrays.asList("hello", "world", "java"));
        strings.replaceAll(String::toUpperCase);
        System.out.println("After replaceAll: " + strings);
        
        // Sort with Comparator
        List<String> names = Arrays.asList("Charlie", "Alice", "Bob", "David");
        names = new ArrayList<>(names);
        names.sort(Comparator.naturalOrder());
        System.out.println("Sorted names: " + names);
        
        // Collections utility methods for searching
        List<String> sortedNames = Arrays.asList("Alice", "Bob", "Charlie", "David");
        int foundIndex = Collections.binarySearch(sortedNames, "Charlie");
        System.out.println("Index of 'Charlie': " + foundIndex);
        
        // If not found, binarySearch returns insertion point
        int notFoundIndex = Collections.binarySearch(sortedNames, "Betty");
        System.out.println("'Betty' not found, would be inserted at: " + (-notFoundIndex - 1));
    }
}
```

## Custom Collection Implementations

### Custom List Implementation

```java
import java.util.*;
import java.util.function.*;

public class CustomListImplementation implements List<String> {
    private String[] data;
    private int size;
    private static final int DEFAULT_CAPACITY = 10;
    
    public CustomListImplementation() {
        this.data = new String[DEFAULT_CAPACITY];
        this.size = 0;
    }
    
    @Override
    public int size() {
        return size;
    }
    
    @Override
    public boolean isEmpty() {
        return size == 0;
    }
    
    @Override
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }
    
    @Override
    public Iterator<String> iterator() {
        return new CustomIterator();
    }
    
    @Override
    public Object[] toArray() {
        return Arrays.copyOf(data, size);
    }
    
    @Override
    public <T> T[] toArray(T[] a) {
        if (a.length < size) {
            @SuppressWarnings("unchecked")
            T[] result = (T[]) java.lang.reflect.Array
                .newInstance(a.getClass().getComponentType(), size);
            System.arraycopy(data, 0, result, 0, size);
            return result;
        }
        System.arraycopy(data, 0, a, 0, size);
        if (a.length > size) {
            a[size] = null;
        }
        return a;
    }
    
    @Override
    public boolean add(String s) {
        ensureCapacity();
        data[size++] = s;
        return true;
    }
    
    @Override
    public boolean remove(Object o) {
        int index = indexOf(o);
        if (index >= 0) {
            remove(index);
            return true;
        }
        return false;
    }
    
    @Override
    public boolean containsAll(Collection<?> c) {
        for (Object o : c) {
            if (!contains(o)) {
                return false;
            }
        }
        return true;
    }
    
    @Override
    public boolean addAll(Collection<? extends String> c) {
        for (String s : c) {
            add(s);
        }
        return !c.isEmpty();
    }
    
    @Override
    public boolean addAll(int index, Collection<? extends String> c) {
        rangeCheckForAdd(index);
        
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacity(size + numNew);
        
        int numMoved = size - index;
        if (numMoved > 0) {
            System.arraycopy(data, index, data, index + numNew, numMoved);
        }
        
        System.arraycopy(a, 0, data, index, numNew);
        size += numNew;
        return numNew != 0;
    }
    
    @Override
    public boolean removeAll(Collection<?> c) {
        boolean modified = false;
        Iterator<String> it = iterator();
        while (it.hasNext()) {
            if (c.contains(it.next())) {
                it.remove();
                modified = true;
            }
        }
        return modified;
    }
    
    @Override
    public boolean retainAll(Collection<?> c) {
        boolean modified = false;
        Iterator<String> it = iterator();
        while (it.hasNext()) {
            if (!c.contains(it.next())) {
                it.remove();
                modified = true;
            }
        }
        return modified;
    }
    
    @Override
    public String get(int index) {
        rangeCheck(index);
        return data[index];
    }
    
    @Override
    public String set(int index, String element) {
        rangeCheck(index);
        String oldValue = data[index];
        data[index] = element;
        return oldValue;
    }
    
    @Override
    public void add(int index, String element) {
        rangeCheckForAdd(index);
        ensureCapacity();
        
        System.arraycopy(data, index, data, index + 1, size - index);
        data[index] = element;
        size++;
    }
    
    @Override
    public String remove(int index) {
        rangeCheck(index);
        
        String oldValue = data[index];
        int numMoved = size - index - 1;
        if (numMoved > 0) {
            System.arraycopy(data, index + 1, data, index, numMoved);
        }
        data[--size] = null; // Clear to let GC do its work
        return oldValue;
    }
    
    @Override
    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++) {
                if (data[i] == null) {
                    return i;
                }
            }
        } else {
            for (int i = 0; i < size; i++) {
                if (o.equals(data[i])) {
                    return i;
                }
            }
        }
        return -1;
    }
    
    @Override
    public int lastIndexOf(Object o) {
        if (o == null) {
            for (int i = size - 1; i >= 0; i--) {
                if (data[i] == null) {
                    return i;
                }
            }
        } else {
            for (int i = size - 1; i >= 0; i--) {
                if (o.equals(data[i])) {
                    return i;
                }
            }
        }
        return -1;
    }
    
    @Override
    public ListIterator<String> listIterator() {
        return new CustomListIterator(0);
    }
    
    @Override
    public ListIterator<String> listIterator(int index) {
        rangeCheckForAdd(index);
        return new CustomListIterator(index);
    }
    
    @Override
    public List<String> subList(int fromIndex, int toIndex) {
        // Simplified implementation - in real code, you'd return a view
        return new ArrayList<>(Arrays.asList(data).subList(fromIndex, toIndex));
    }
    
    private void rangeCheck(int index) {
        if (index >= size || index < 0) {
            throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + size);
        }
    }
    
    private void rangeCheckForAdd(int index) {
        if (index > size || index < 0) {
            throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + size);
        }
    }
    
    private void ensureCapacity() {
        if (size >= data.length) {
            int newCapacity = data.length * 2;
            data = Arrays.copyOf(data, newCapacity);
        }
    }
    
    private class CustomIterator implements Iterator<String> {
        private int cursor = 0;
        
        @Override
        public boolean hasNext() {
            return cursor != size;
        }
        
        @Override
        public String next() {
            if (cursor >= size) {
                throw new NoSuchElementException();
            }
            return data[cursor++];
        }
        
        @Override
        public void remove() {
            if (cursor < 1) {
                throw new IllegalStateException();
            }
            CustomListImplementation.this.remove(cursor - 1);
            cursor--;
        }
    }
    
    private class CustomListIterator implements ListIterator<String> {
        private int cursor;
        private int lastRet = -1;
        
        CustomListIterator(int index) {
            cursor = index;
        }
        
        @Override
        public boolean hasNext() {
            return cursor != size;
        }
        
        @Override
        public String next() {
            if (!hasNext()) {
                throw new NoSuchElementException();
            }
            lastRet = cursor;
            return data[cursor++];
        }
        
        @Override
        public boolean hasPrevious() {
            return cursor != 0;
        }
        
        @Override
        public String previous() {
            if (!hasPrevious()) {
                throw new NoSuchElementException();
            }
            lastRet = --cursor;
            return data[lastRet];
        }
        
        @Override
        public int nextIndex() {
            return cursor;
        }
        
        @Override
        public int previousIndex() {
            return cursor - 1;
        }
        
        @Override
        public void remove() {
            if (lastRet < 0) {
                throw new IllegalStateException();
            }
            CustomListImplementation.this.remove(lastRet);
            cursor = lastRet;
            lastRet = -1;
        }
        
        @Override
        public void set(String e) {
            if (lastRet < 0) {
                throw new IllegalStateException();
            }
            CustomListImplementation.this.set(lastRet, e);
        }
        
        @Override
        public void add(String e) {
            CustomListImplementation.this.add(cursor++, e);
            lastRet = -1;
        }
    }
    
    public static void main(String[] args) {
        CustomListImplementation list = new CustomListImplementation();
        
        // Test basic operations
        list.add("Hello");
        list.add("World");
        list.add("Java");
        list.add(1, "Beautiful");
        
        System.out.println("List contents: " + list);
        System.out.println("Size: " + list.size());
        System.out.println("Contains 'Java': " + list.contains("Java"));
        System.out.println("Index of 'World': " + list.indexOf("World"));
        
        // Test iteration
        System.out.println("Iterating:");
        for (String s : list) {
            System.out.println("  " + s);
        }
        
        // Test removal
        list.remove("Beautiful");
        System.out.println("After removal: " + list);
        
        // Test list iterator
        ListIterator<String> lit = list.listIterator();
        System.out.println("List iterator (forward):");
        while (lit.hasNext()) {
            System.out.println("  " + lit.next() + " at index " + lit.previousIndex());
        }
        
        System.out.println("List iterator (backward):");
        while (lit.hasPrevious()) {
            System.out.println("  " + lit.previous() + " at index " + lit.nextIndex());
        }
    }
}
```

## Performance Considerations and Best Practices

### Collection Performance Comparison

```java
import java.util.*;
import java.util.concurrent.*;

public class CollectionPerformanceComparison {
    public static void main(String[] args) {
        int size = 100000;
        
        System.out.println("Performance comparison for size: " + size);
        
        // ArrayList vs LinkedList for different operations
        testListPerformance(size);
        
        // HashSet vs TreeSet vs LinkedHashSet
        testSetPerformance(size);
        
        // HashMap vs LinkedHashMap vs TreeMap
        testMapPerformance(size);
        
        // ArrayList vs ArrayDeque for stack operations
        testStackPerformance(size);
    }
    
    private static void testListPerformance(int size) {
        System.out.println("\n--- List Performance ---");
        
        // ArrayList
        List<Integer> arrayList = new ArrayList<>();
        long start = System.nanoTime();
        for (int i = 0; i < size; i++) {
            arrayList.add(i);
        }
        long arrayAddTime = System.nanoTime() - start;
        
        start = System.nanoTime();
        for (int i = 0; i < 1000; i++) {
            arrayList.get(size / 2); // Random access
        }
        long arrayAccessTime = System.nanoTime() - start;
        
        start = System.nanoTime();
        for (int i = 0; i < 1000; i++) {
            arrayList.indexOf(50000); // Search operation
        }
        long arraySearchTime = System.nanoTime() - start;
        
        // LinkedList
        List<Integer> linkedList = new LinkedList<>();
        start = System.nanoTime();
        for (int i = 0; i < size; i++) {
            linkedList.add(i);
        }
        long linkedAddTime = System.nanoTime() - start;
        
        start = System.nanoTime();
        for (int i = 0; i < 1000; i++) {
            linkedList.get(size / 2); // Random access - slow for LinkedList
        }
        long linkedAccessTime = System.nanoTime() - start;
        
        start = System.nanoTime();
        for (int i = 0; i < 1000; i++) {
            linkedList.indexOf(50000); // Search operation
        }
        long linkedSearchTime = System.nanoTime() - start;
        
        System.out.printf("ArrayList - Add: %.2fms, Access: %.2fms, Search: %.2fms%n",
            arrayAddTime / 1_000_000.0, arrayAccessTime / 1_000_000.0, arraySearchTime / 1_000_000.0);
        System.out.printf("LinkedList - Add: %.2fms, Access: %.2fms, Search: %.2fms%n",
            linkedAddTime / 1_000_000.0, linkedAccessTime / 1_000_000.0, linkedSearchTime / 1_000_000.0);
    }
    
    private static void testSetPerformance(int size) {
        System.out.println("\n--- Set Performance ---");
        
        // HashSet
        Set<Integer> hashSet = new HashSet<>();
        long start = System.nanoTime();
        for (int i = 0; i < size; i++) {
            hashSet.add(i);
        }
        long hashAddTime = System.nanoTime() - start;
        
        start = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            hashSet.contains(size / 2 + i % 1000);
        }
        long hashContainsTime = System.nanoTime() - start;
        
        // TreeSet
        Set<Integer> treeSet = new TreeSet<>();
        start = System.nanoTime();
        for (int i = 0; i < size; i++) {
            treeSet.add(i);
        }
        long treeAddTime = System.nanoTime() - start;
        
        start = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            treeSet.contains(size / 2 + i % 1000);
        }
        long treeContainsTime = System.nanoTime() - start;
        
        // LinkedHashSet
        Set<Integer> linkedHashSet = new LinkedHashSet<>();
        start = System.nanoTime();
        for (int i = 0; i < size; i++) {
            linkedHashSet.add(i);
        }
        long linkedHashAddTime = System.nanoTime() - start;
        
        start = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            linkedHashSet.contains(size / 2 + i % 1000);
        }
        long linkedHashContainsTime = System.nanoTime() - start;
        
        System.out.printf("HashSet - Add: %.2fms, Contains: %.2fms%n",
            hashAddTime / 1_000_000.0, hashContainsTime / 1_000_000.0);
        System.out.printf("TreeSet - Add: %.2fms, Contains: %.2fms%n",
            treeAddTime / 1_000_000.0, treeContainsTime / 1_000_000.0);
        System.out.printf("LinkedHashSet - Add: %.2fms, Contains: %.2fms%n",
            linkedHashAddTime / 1_000_000.0, linkedHashContainsTime / 1_000_000.0);
    }
    
    private static void testMapPerformance(int size) {
        System.out.println("\n--- Map Performance ---");
        
        // HashMap
        Map<Integer, String> hashMap = new HashMap<>();
        long start = System.nanoTime();
        for (int i = 0; i < size; i++) {
            hashMap.put(i, "Value" + i);
        }
        long hashPutTime = System.nanoTime() - start;
        
        start = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            hashMap.get(size / 2 + i % 1000);
        }
        long hashGetTime = System.nanoTime() - start;
        
        // TreeMap
        Map<Integer, String> treeMap = new TreeMap<>();
        start = System.nanoTime();
        for (int i = 0; i < size; i++) {
            treeMap.put(i, "Value" + i);
        }
        long treePutTime = System.nanoTime() - start;
        
        start = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            treeMap.get(size / 2 + i % 1000);
        }
        long treeGetTime = System.nanoTime() - start;
        
        // LinkedHashMap
        Map<Integer, String> linkedHashMap = new LinkedHashMap<>();
        start = System.nanoTime();
        for (int i = 0; i < size; i++) {
            linkedHashMap.put(i, "Value" + i);
        }
        long linkedHashPutTime = System.nanoTime() - start;
        
        start = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            linkedHashMap.get(size / 2 + i % 1000);
        }
        long linkedHashGetTime = System.nanoTime() - start;
        
        System.out.printf("HashMap - Put: %.2fms, Get: %.2fms%n",
            hashPutTime / 1_000_000.0, hashGetTime / 1_000_000.0);
        System.out.printf("TreeMap - Put: %.2fms, Get: %.2fms%n",
            treePutTime / 1_000_000.0, treeGetTime / 1_000_000.0);
        System.out.printf("LinkedHashMap - Put: %.2fms, Get: %.2fms%n",
            linkedHashPutTime / 1_000_000.0, linkedHashGetTime / 1_000_000.0);
    }
    
    private static void testStackPerformance(int size) {
        System.out.println("\n--- Stack Performance ---");
        
        // ArrayList as stack (inefficient)
        List<Integer> arrayListStack = new ArrayList<>();
        long start = System.nanoTime();
        for (int i = 0; i < size; i++) {
            arrayListStack.add(i); // Add to end
        }
        long arrayPushTime = System.nanoTime() - start;
        
        start = System.nanoTime();
        for (int i = 0; i < size; i++) {
            arrayListStack.remove(arrayListStack.size() - 1); // Remove from end
        }
        long arrayPopTime = System.nanoTime() - start;
        
        // ArrayDeque as stack (efficient)
        Deque<Integer> dequeStack = new ArrayDeque<>();
        start = System.nanoTime();
        for (int i = 0; i < size; i++) {
            dequeStack.push(i);
        }
        long dequePushTime = System.nanoTime() - start;
        
        start = System.nanoTime();
        for (int i = 0; i < size; i++) {
            dequeStack.pop();
        }
        long dequePopTime = System.nanoTime() - start;
        
        System.out.printf("ArrayList (stack) - Push: %.2fms, Pop: %.2fms%n",
            arrayPushTime / 1_000_000.0, arrayPopTime / 1_000_000.0);
        System.out.printf("ArrayDeque (stack) - Push: %.2fms, Pop: %.2fms%n",
            dequePushTime / 1_000_000.0, dequePopTime / 1_000_000.0);
    }
}
```

### Best Practices and Guidelines

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.function.*;

public class CollectionBestPractices {
    public static void main(String[] args) {
        // 1. Choose the right collection for the job
        
        // For frequent random access - use List with fast random access
        List<String> listForRandomAccess = new ArrayList<>();
        
        // For frequent insertions/deletions in the middle
        List<String> listForInsertions = new LinkedList<>();
        
        // For unique elements without order importance
        Set<String> setForUniqueness = new HashSet<>();
        
        // For unique elements in insertion order
        Set<String> setForOrder = new LinkedHashSet<>();
        
        // For unique elements in sorted order
        Set<String> setForSorting = new TreeSet<>();
        
        // For key-value pairs, fast lookup
        Map<String, Integer> mapForFastLookup = new HashMap<>();
        
        // For key-value pairs in insertion order
        Map<String, Integer> mapForOrder = new LinkedHashMap<>();
        
        // For key-value pairs in sorted key order
        Map<String, Integer> mapForSortedKeys = new TreeMap<>();
        
        // 2. Initialize with appropriate capacity
        List<String> sizedList = new ArrayList<>(1000); // Avoids resizing
        
        // 3. Use appropriate initial capacity for HashMap
        Map<String, Integer> sizedMap = new HashMap<>(16, 0.75f); // Default load factor
        
        // 4. Use contains() instead of manual iteration when possible
        List<String> items = Arrays.asList("a", "b", "c", "d");
        boolean hasB = items.contains("b"); // Good
        // boolean hasB = false; for(String s : items) { if(s.equals("b")) { hasB = true; break; } } // Avoid
        
        // 5. Use streams for complex operations
        List<String> words = Arrays.asList("hello", "world", "java", "collections");
        
        // Instead of manual loops
        List<String> upperCaseWords = new ArrayList<>();
        for (String word : words) {
            if (word.length() > 4) {
                upperCaseWords.add(word.toUpperCase());
            }
        }
        
        // Use streams
        List<String> streamResult = words.stream()
            .filter(word -> word.length() > 4)
            .map(String::toUpperCase)
            .collect(ArrayList::new, ArrayList::add, ArrayList::addAll);
        
        // 6. Use appropriate methods for bulk operations
        List<String> source = Arrays.asList("a", "b", "c");
        List<String> destination = new ArrayList<>();
        
        destination.addAll(source); // More efficient than individual adds
        
        // 7. Use fail-fast iterators carefully in multi-threaded environments
        List<String> concurrentList = Collections.synchronizedList(new ArrayList<>());
        // When iterating, must synchronize manually
        synchronized(concurrentList) {
            for (String s : concurrentList) {
                // Process s
            }
        }
        
        // 8. Use immutable collections when possible
        List<String> immutableList = List.of("a", "b", "c"); // Java 9+
        Set<String> immutableSet = Set.of("x", "y", "z"); // Java 9+
        Map<String, Integer> immutableMap = Map.of("a", 1, "b", 2); // Java 9+
        
        // 9. Use Comparator.comparing() for complex sorting
        List<Person> people = Arrays.asList(
            new Person("Alice", 30),
            new Person("Bob", 25),
            new Person("Charlie", 35)
        );
        
        // Sort by age
        people.sort(Comparator.comparing(Person::getAge));
        
        // Sort by name, then by age
        people.sort(Comparator.comparing(Person::getName)
            .thenComparing(Person::getAge));
        
        // 10. Use Map methods effectively
        Map<String, Integer> wordCount = new HashMap<>();
        String[] text = {"hello", "world", "hello", "java", "world", "hello"};
        
        for (String word : text) {
            // Old way
            wordCount.put(word, wordCount.getOrDefault(word, 0) + 1);
            
            // Or using compute
            // wordCount.compute(word, (k, v) -> (v == null) ? 1 : v + 1);
        }
        
        System.out.println("Word counts: " + wordCount);
        
        // 11. Use specific collection types for specific needs
        Deque<String> deque = new ArrayDeque<>(); // Better than Stack class
        Queue<String> queue = new LinkedList<>(); // For FIFO operations
        PriorityQueue<String> priorityQueue = new PriorityQueue<>(); // For priority-based processing
        
        // 12. Performance considerations for large datasets
        // Use primitive collections if available (Trove, Eclipse Collections, etc.)
        // Or use streams with parallel processing for large datasets
        
        List<Integer> largeList = IntStream.range(0, 1000000)
            .boxed()
            .collect(Collectors.toList());
        
        long startTime = System.currentTimeMillis();
        // Sequential processing
        long sequentialSum = largeList.stream()
            .mapToLong(Integer::longValue)
            .sum();
        long sequentialTime = System.currentTimeMillis() - startTime;
        
        startTime = System.currentTimeMillis();
        // Parallel processing
        long parallelSum = largeList.parallelStream()
            .mapToLong(Integer::longValue)
            .sum();
        long parallelTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Sequential sum: " + sequentialSum + " in " + sequentialTime + "ms");
        System.out.println("Parallel sum: " + parallelSum + " in " + parallelTime + "ms");
        
        // 13. Memory efficiency - use the right data structure
        // For small collections, arrays might be more efficient
        String[] smallArray = {"a", "b", "c"}; // More efficient than List for 3 elements
        
        // For large collections, use appropriate structure based on usage pattern
        List<String> largeListOptimized = new ArrayList<>(10000); // Pre-sized to avoid resizing
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
    
    @Override
    public String toString() {
        return name + "(" + age + ")";
    }
}
```

## Summary

The Java Collections Framework provides a comprehensive set of interfaces and implementations for working with groups of objects. Key takeaways include:

1. **Core Interfaces**: Understand Collection, List, Set, Queue, Deque, and Map hierarchies
2. **Implementation Selection**: Choose the right implementation based on performance needs and functionality
3. **Performance Characteristics**: Know the time complexity of operations for different implementations
4. **Thread Safety**: Use concurrent implementations or Collections.synchronized* methods when needed
5. **Stream Integration**: Leverage streams for functional-style operations on collections
6. **Best Practices**: Initialize with appropriate capacity, use bulk operations, and choose the right data structure for your use case

Mastering the Collections Framework is essential for efficient Java programming, as it provides the foundation for working with groups of objects in a type-safe and efficient manner.