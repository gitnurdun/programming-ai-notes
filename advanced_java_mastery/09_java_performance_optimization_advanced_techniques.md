# Java Performance Optimization: Advanced Techniques and Best Practices

## Introduction

Java performance optimization is a critical skill for developing high-performance, scalable applications. While Java's automatic memory management and rich ecosystem provide many benefits, achieving optimal performance requires understanding the JVM internals, profiling techniques, and optimization strategies.

This comprehensive guide explores advanced performance optimization techniques, covering everything from JVM tuning to algorithmic improvements and code-level optimizations.

## JVM Performance Tuning

### Understanding JVM Internals for Optimization

```java
import java.lang.management.*;
import java.util.*;
import java.util.concurrent.*;

public class JVMPerformanceTuning {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== JVM Performance Tuning ===");
        
        // Analyze current JVM settings
        analyzeJVMSettings();
        
        // Demonstrate JIT compilation effects
        demonstrateJITCompilation();
        
        // Compare different garbage collectors
        compareGarbageCollectors();
        
        // Demonstrate JVM flags impact
        demonstrateJVMFlags();
    }
    
    private static void analyzeJVMSettings() {
        System.out.println("\n=== Current JVM Settings Analysis ===");
        
        // Runtime information
        Runtime runtime = Runtime.getRuntime();
        System.out.println("Available processors: " + runtime.availableProcessors());
        System.out.println("Max memory: " + runtime.maxMemory() / (1024 * 1024) + " MB");
        System.out.println("Total memory: " + runtime.totalMemory() / (1024 * 1024) + " MB");
        System.out.println("Free memory: " + runtime.freeMemory() / (1024 * 1024) + " MB");
        
        // JVM system properties
        System.out.println("\nJVM Properties:");
        String[] importantProps = {
            "java.version", "java.vm.name", "java.vm.version", 
            "java.vm.info", "os.name", "os.arch", "os.version"
        };
        
        for (String prop : importantProps) {
            System.out.println("  " + prop + ": " + System.getProperty(prop));
        }
        
        // Memory pool information
        System.out.println("\nMemory Pools:");
        List<MemoryPoolMXBean> memoryPools = ManagementFactory.getMemoryPoolMXBeans();
        for (MemoryPoolMXBean pool : memoryPools) {
            System.out.println("  " + pool.getName() + " (" + pool.getType() + ")");
            MemoryUsage usage = pool.getUsage();
            if (usage != null) {
                System.out.printf("    Usage: %d MB / %d MB (%.1f%%)%n",
                    usage.getUsed() / (1024 * 1024),
                    usage.getMax() / (1024 * 1024),
                    (double) usage.getUsed() / usage.getMax() * 100);
            }
        }
    }
    
    private static void demonstrateJITCompilation() throws InterruptedException {
        System.out.println("\n=== JIT Compilation Demonstration ===");
        
        // Warm up JIT compiler
        System.out.println("Warming up JIT compiler...");
        for (int i = 0; i < 100000; i++) {
            calculateSomething(i);
        }
        
        // Measure performance before and after JIT optimization
        System.out.println("Measuring JIT optimization effects:");
        
        // Cold run (interpreted)
        long startTime = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            calculateSomething(i);
        }
        long coldTime = System.nanoTime() - startTime;
        
        // Warm run (compiled)
        startTime = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            calculateSomething(i);
        }
        long warmTime = System.nanoTime() - startTime;
        
        System.out.printf("Cold execution time: %d ns (%.2f ms)%n", 
            coldTime, coldTime / 1_000_000.0);
        System.out.printf("Warm execution time: %d ns (%.2f ms)%n", 
            warmTime, warmTime / 1_000_000.0);
        System.out.printf("Performance improvement: %.2fx%n", 
            (double) coldTime / warmTime);
    }
    
    private static int calculateSomething(int input) {
        // Simple calculation that JIT can optimize
        int result = 0;
        for (int i = 0; i < 100; i++) {
            result += input * i;
        }
        return result;
    }
    
    private static void compareGarbageCollectors() {
        System.out.println("\n=== Garbage Collector Comparison ===");
        
        List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        
        System.out.println("Available Garbage Collectors:");
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            System.out.println("  " + gcBean.getName());
            System.out.println("    Collections: " + gcBean.getCollectionCount());
            System.out.println("    Total time: " + gcBean.getCollectionTime() + " ms");
            System.out.println("    Memory pools: " + Arrays.toString(gcBean.getMemoryPoolNames()));
        }
        
        System.out.println("\nGC Selection Guidelines:");
        System.out.println("Serial GC: Single-threaded applications, small heaps (< 100MB)");
        System.out.println("Parallel GC: Throughput-focused applications, multi-core systems");
        System.out.println("G1 GC: Low-latency applications, large heaps (100MB+)");
        System.out.println("ZGC: Ultra-low latency applications, very large heaps (multi-GB)");
        System.out.println("Shenandoah: Low-latency applications, scalable pause times");
    }
    
    private static void demonstrateJVMFlags() {
        System.out.println("\n=== Important JVM Flags ===");
        
        System.out.println("Memory-related flags:");
        System.out.println("  -Xms<size>: Initial heap size");
        System.out.println("  -Xmx<size>: Maximum heap size");
        System.out.println("  -XX:NewRatio=<ratio>: Old/Young generation ratio");
        System.out.println("  -XX:NewSize=<size>: Young generation size");
        System.out.println("  -XX:MaxNewSize=<size>: Maximum young generation size");
        
        System.out.println("\nGC-related flags:");
        System.out.println("  -XX:+UseSerialGC: Enable Serial GC");
        System.out.println("  -XX:+UseParallelGC: Enable Parallel GC");
        System.out.println("  -XX:+UseG1GC: Enable G1 GC");
        System.out.println("  -XX:+UseZGC: Enable Z GC (Java 11+)");
        System.out.println("  -XX:MaxGCPauseMillis=<ms>: Target GC pause time");
        
        System.out.println("\nJIT-related flags:");
        System.out.println("  -XX:CompileThreshold=<count>: Method invocation threshold");
        System.out.println("  -XX:+TieredCompilation: Enable tiered compilation");
        System.out.println("  -XX:ReservedCodeCacheSize=<size>: Code cache size");
        
        System.out.println("\nMonitoring flags:");
        System.out.println("  -XX:+PrintGC: Print GC events");
        System.out.println("  -XX:+PrintGCDetails: Print detailed GC information");
        System.out.println("  -XX:+PrintGCTimeStamps: Print GC timestamps");
        System.out.println("  -Xlog:gc*:file=gc.log:time,tags: GC logging (Java 9+)");
    }
}
```

### Advanced JIT Compilation Optimization

```java
import java.util.*;
import java.util.concurrent.*;

public class JITOptimization {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== JIT Compilation Optimization ===");
        
        // Demonstrate JIT optimization techniques
        demonstrateJITOptimizations();
        
        // Compare optimized vs non-optimized code
        compareOptimizations();
        
        // Demonstrate escape analysis
        demonstrateEscapeAnalysis();
    }
    
    private static void demonstrateJITOptimizations() throws InterruptedException {
        System.out.println("\n=== JIT Optimization Techniques ===");
        
        // Warm up for JIT compilation
        System.out.println("Warming up JIT compiler...");
        for (int i = 0; i < 150000; i++) {
            optimizedMethod(i);
            nonOptimizedMethod(i);
        }
        
        // Demonstrate loop unrolling
        System.out.println("\nLoop unrolling optimization:");
        testLoopUnrolling();
        
        // Demonstrate inlining
        System.out.println("\nMethod inlining optimization:");
        testMethodInlining();
        
        // Demonstrate dead code elimination
        System.out.println("\nDead code elimination:");
        testDeadCodeElimination();
    }
    
    private static void testLoopUnrolling() {
        final int iterations = 1000000;
        
        // Simple loop
        long startTime = System.nanoTime();
        int sum1 = 0;
        for (int i = 0; i < iterations; i++) {
            sum1 += i;
        }
        long simpleLoopTime = System.nanoTime() - startTime;
        
        // Loop with manual unrolling
        startTime = System.nanoTime();
        int sum2 = 0;
        for (int i = 0; i < iterations; i += 4) {
            sum2 += i;
            sum2 += i + 1;
            sum2 += i + 2;
            sum2 += i + 3;
        }
        long unrolledLoopTime = System.nanoTime() - startTime;
        
        System.out.printf("Simple loop time: %d ns%n", simpleLoopTime);
        System.out.printf("Unrolled loop time: %d ns%n", unrolledLoopTime);
        System.out.printf("Improvement: %.2fx%n", (double) simpleLoopTime / unrolledLoopTime);
    }
    
    private static void testMethodInlining() {
        final int iterations = 1000000;
        
        // Method calls that may be inlined
        long startTime = System.nanoTime();
        int sum = 0;
        for (int i = 0; i < iterations; i++) {
            sum += simpleCalculation(i);
        }
        long inlinedTime = System.nanoTime() - startTime;
        
        // Direct calculation (already optimized)
        startTime = System.nanoTime();
        int directSum = 0;
        for (int i = 0; i < iterations; i++) {
            directSum += i * 2 + 1;
        }
        long directTime = System.nanoTime() - startTime;
        
        System.out.printf("Method call time: %d ns%n", inlinedTime);
        System.out.printf("Direct calculation time: %d ns%n", directTime);
        System.out.printf("Inlining effectiveness: %.2fx%n", (double) inlinedTime / directTime);
    }
    
    private static void testDeadCodeElimination() {
        final int iterations = 1000000;
        
        // Code with dead code
        long startTime = System.nanoTime();
        int sum1 = 0;
        for (int i = 0; i < iterations; i++) {
            sum1 += i;
            // This code is dead and should be eliminated
            if (false) {
                sum1 += 1000000; // Never executed
            }
        }
        long deadCodeTime = System.nanoTime() - startTime;
        
        // Same code without dead code
        startTime = System.nanoTime();
        int sum2 = 0;
        for (int i = 0; i < iterations; i++) {
            sum2 += i;
        }
        long cleanCodeTime = System.nanoTime() - startTime;
        
        System.out.printf("With dead code: %d ns%n", deadCodeTime);
        System.out.printf("Without dead code: %d ns%n", cleanCodeTime);
    }
    
    // Simple method for inlining test
    private static int simpleCalculation(int input) {
        return input * 2 + 1;
    }
    
    // Method that JIT will optimize
    private static int optimizedMethod(int input) {
        int result = 0;
        for (int i = 0; i < 100; i++) {
            result += input + i;
        }
        return result;
    }
    
    // Method that's harder to optimize
    private static int nonOptimizedMethod(int input) {
        Integer result = 0;
        for (int i = 0; i < 100; i++) {
            result = Integer.valueOf(result.intValue() + input + i);
        }
        return result;
    }
    
    private static void compareOptimizations() {
        System.out.println("\n=== Optimized vs Non-Optimized Code ===");
        
        final int iterations = 100000;
        
        // Optimized version
        long startTime = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            optimizedCalculation(i);
        }
        long optimizedTime = System.nanoTime() - startTime;
        
        // Non-optimized version
        startTime = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            nonOptimizedCalculation(i);
        }
        long nonOptimizedTime = System.nanoTime() - startTime;
        
        System.out.printf("Optimized time: %d ns (%.2f ms)%n", 
            optimizedTime, optimizedTime / 1_000_000.0);
        System.out.printf("Non-optimized time: %d ns (%.2f ms)%n", 
            nonOptimizedTime, nonOptimizedTime / 1_000_000.0);
        System.out.printf("Performance gain: %.2fx%n", 
            (double) nonOptimizedTime / optimizedTime);
    }
    
    private static int optimizedCalculation(int input) {
        // Uses primitive types, simple arithmetic
        int result = input * input;
        result += input << 2; // Bit shift instead of multiplication
        return result;
    }
    
    private static Integer nonOptimizedCalculation(int input) {
        // Uses wrapper types, unnecessary object creation
        Integer result = new Integer(input * input);
        result = Integer.valueOf(result.intValue() + input * 4);
        return result;
    }
    
    private static void demonstrateEscapeAnalysis() {
        System.out.println("\n=== Escape Analysis Demonstration ===");
        
        // Test object allocation elimination
        System.out.println("Testing object allocation elimination:");
        
        final int iterations = 1000000;
        
        // Object that doesn't escape - should be optimized
        long startTime = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            Point p = new Point(i, i * 2);
            int distance = p.calculateDistance();
            // p doesn't escape this scope
        }
        long optimizedTime = System.nanoTime() - startTime;
        
        // Object that escapes - cannot be optimized
        startTime = System.nanoTime();
        List<Point> points = new ArrayList<>();
        for (int i = 0; i < iterations; i++) {
            Point p = new Point(i, i * 2);
            points.add(p); // p escapes by being added to list
            int distance = p.calculateDistance();
        }
        long escapedTime = System.nanoTime() - startTime;
        
        System.out.printf("Non-escaping object time: %d ns%n", optimizedTime);
        System.out.printf("Escaping object time: %d ns%n", escapedTime);
        System.out.printf("Escape analysis benefit: %.2fx%n", (double) escapedTime / optimizedTime);
        
        points.clear(); // Clean up
    }
}

class Point {
    private int x, y;
    
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
    
    public int calculateDistance() {
        return (int) Math.sqrt(x * x + y * y);
    }
    
    public int getX() { return x; }
    public int getY() { return y; }
}
```

## Algorithm and Data Structure Optimization

### Efficient Algorithms and Data Structures

```java
import java.util.*;
import java.util.concurrent.*;

public class AlgorithmOptimization {
    public static void main(String[] args) {
        System.out.println("=== Algorithm and Data Structure Optimization ===");
        
        // Compare sorting algorithms
        compareSortingAlgorithms();
        
        // Compare search algorithms
        compareSearchAlgorithms();
        
        // Demonstrate data structure choices
        demonstrateDataStructureChoices();
        
        // Show caching and memoization techniques
        demonstrateCaching();
    }
    
    private static void compareSortingAlgorithms() {
        System.out.println("\n=== Sorting Algorithm Comparison ===");
        
        int[] sizes = {1000, 10000, 100000};
        
        for (int size : sizes) {
            System.out.println("\nArray size: " + size);
            
            // Generate test data
            int[] data1 = generateRandomArray(size);
            int[] data2 = data1.clone();
            int[] data3 = data1.clone();
            
            // Built-in sort (optimized)
            long startTime = System.nanoTime();
            Arrays.sort(data1);
            long javaSortTime = System.nanoTime() - startTime;
            
            // Quick sort implementation
            startTime = System.nanoTime();
            quickSort(data2, 0, data2.length - 1);
            long quickSortTime = System.nanoTime() - startTime;
            
            // Merge sort implementation
            startTime = System.nanoTime();
            mergeSort(data3, 0, data3.length - 1);
            long mergeSortTime = System.nanoTime() - startTime;
            
            System.out.printf("Java Arrays.sort: %d ns (%.2f ms)%n", 
                javaSortTime, javaSortTime / 1_000_000.0);
            System.out.printf("Quick sort: %d ns (%.2f ms)%n", 
                quickSortTime, quickSortTime / 1_000_000.0);
            System.out.printf("Merge sort: %d ns (%.2f ms)%n", 
                mergeSortTime, mergeSortTime / 1_000_000.0);
        }
    }
    
    private static void compareSearchAlgorithms() {
        System.out.println("\n=== Search Algorithm Comparison ===");
        
        int[] sizes = {10000, 100000, 1000000};
        
        for (int size : sizes) {
            System.out.println("\nArray size: " + size);
            
            // Generate sorted test data
            int[] data = generateSortedArray(size);
            int target = data[size / 2]; // Target in middle
            
            // Linear search
            long startTime = System.nanoTime();
            int linearResult = linearSearch(data, target);
            long linearTime = System.nanoTime() - startTime;
            
            // Binary search
            startTime = System.nanoTime();
            int binaryResult = binarySearch(data, target);
            long binaryTime = System.nanoTime() - startTime;
            
            // Hash-based search (using HashSet)
            Set<Integer> hashSet = new HashSet<>();
            for (int value : data) {
                hashSet.add(value);
            }
            
            startTime = System.nanoTime();
            boolean hashResult = hashSet.contains(target);
            long hashTime = System.nanoTime() - startTime;
            
            System.out.printf("Linear search: %d ns, result: %d%n", linearTime, linearResult);
            System.out.printf("Binary search: %d ns, result: %d%n", binaryTime, binaryResult);
            System.out.printf("Hash search: %d ns, result: %b%n", hashTime, hashResult);
            System.out.printf("Binary vs Linear improvement: %.2fx%n", (double) linearTime / binaryTime);
        }
    }
    
    private static void demonstrateDataStructureChoices() {
        System.out.println("\n=== Data Structure Choice Impact ===");
        
        int operations = 100000;
        
        // ArrayList vs LinkedList for different operations
        System.out.println("\nArrayList vs LinkedList:");
        
        // Random access - ArrayList wins
        List<Integer> arrayList = new ArrayList<>();
        List<Integer> linkedList = new LinkedList<>();
        
        for (int i = 0; i < 10000; i++) {
            arrayList.add(i);
            linkedList.add(i);
        }
        
        long startTime = System.nanoTime();
        for (int i = 0; i < 1000; i++) {
            int index = (int) (Math.random() * 10000);
            arrayList.get(index);
        }
        long arrayListTime = System.nanoTime() - startTime;
        
        startTime = System.nanoTime();
        for (int i = 0; i < 1000; i++) {
            int index = (int) (Math.random() * 10000);
            linkedList.get(index);
        }
        long linkedListTime = System.nanoTime() - startTime;
        
        System.out.printf("Random access - ArrayList: %d ns, LinkedList: %d ns%n", 
            arrayListTime, linkedListTime);
        
        // Insertion at beginning - LinkedList wins
        startTime = System.nanoTime();
        for (int i = 0; i < 1000; i++) {
            arrayList.add(0, i);
        }
        long arrayListInsertTime = System.nanoTime() - startTime;
        
        startTime = System.nanoTime();
        for (int i = 0; i < 1000; i++) {
            linkedList.add(0, i);
        }
        long linkedListInsertTime = System.nanoTime() - startTime;
        
        System.out.printf("Insertion at beginning - ArrayList: %d ns, LinkedList: %d ns%n", 
            arrayListInsertTime, linkedListInsertTime);
        
        // HashSet vs TreeSet for lookups
        System.out.println("\nHashSet vs TreeSet:");
        
        Set<Integer> hashSet = new HashSet<>();
        Set<Integer> treeSet = new TreeSet<>();
        
        for (int i = 0; i < 10000; i++) {
            hashSet.add(i);
            treeSet.add(i);
        }
        
        startTime = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            int value = (int) (Math.random() * 10000);
            hashSet.contains(value);
        }
        long hashSetTime = System.nanoTime() - startTime;
        
        startTime = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            int value = (int) (Math.random() * 10000);
            treeSet.contains(value);
        }
        long treeSetTime = System.nanoTime() - startTime;
        
        System.out.printf("Lookup - HashSet: %d ns, TreeSet: %d ns%n", 
            hashSetTime, treeSetTime);
    }
    
    private static void demonstrateCaching() {
        System.out.println("\n=== Caching and Memoization ===");
        
        // Fibonacci without caching
        long startTime = System.nanoTime();
        long fib1 = fibonacciWithoutCache(40);
        long withoutCacheTime = System.nanoTime() - startTime;
        
        // Fibonacci with caching
        startTime = System.nanoTime();
        long fib2 = fibonacciWithCache(40);
        long withCacheTime = System.nanoTime() - startTime;
        
        System.out.printf("Fibonacci(40) without cache: %d ns, result: %d%n", 
            withoutCacheTime, fib1);
        System.out.printf("Fibonacci(40) with cache: %d ns, result: %d%n", 
            withCacheTime, fib2);
        System.out.printf("Caching improvement: %.2fx%n", (double) withoutCacheTime / withCacheTime);
        
        // Demonstrate LRU cache
        System.out.println("\nLRU Cache demonstration:");
        LRUCache<Integer, String> lruCache = new LRUCache<>(3);
        
        lruCache.put(1, "One");
        lruCache.put(2, "Two");
        lruCache.put(3, "Three");
        System.out.println("Cache after 3 inserts: " + lruCache);
        
        lruCache.get(1); // Access first item
        lruCache.put(4, "Four"); // Should evict item 2
        System.out.println("Cache after eviction: " + lruCache);
    }
    
    // Helper methods
    private static int[] generateRandomArray(int size) {
        Random random = new Random();
        int[] array = new int[size];
        for (int i = 0; i < size; i++) {
            array[i] = random.nextInt(size);
        }
        return array;
    }
    
    private static int[] generateSortedArray(int size) {
        int[] array = new int[size];
        for (int i = 0; i < size; i++) {
            array[i] = i;
        }
        return array;
    }
    
    // Linear search implementation
    private static int linearSearch(int[] array, int target) {
        for (int i = 0; i < array.length; i++) {
            if (array[i] == target) {
                return i;
            }
        }
        return -1;
    }
    
    // Binary search implementation
    private static int binarySearch(int[] array, int target) {
        int left = 0;
        int right = array.length - 1;
        
        while (left <= right) {
            int mid = left + (right - left) / 2;
            if (array[mid] == target) {
                return mid;
            } else if (array[mid] < target) {
                left = mid + 1;
            } else {
                right = mid - 1;
            }
        }
        return -1;
    }
    
    // Quick sort implementation
    private static void quickSort(int[] array, int low, int high) {
        if (low < high) {
            int pi = partition(array, low, high);
            quickSort(array, low, pi - 1);
            quickSort(array, pi + 1, high);
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
    
    // Merge sort implementation
    private static void mergeSort(int[] array, int left, int right) {
        if (left < right) {
            int mid = (left + right) / 2;
            mergeSort(array, left, mid);
            mergeSort(array, mid + 1, right);
            merge(array, left, mid, right);
        }
    }
    
    private static void merge(int[] array, int left, int mid, int right) {
        int n1 = mid - left + 1;
        int n2 = right - mid;
        
        int[] leftArray = new int[n1];
        int[] rightArray = new int[n2];
        
        System.arraycopy(array, left, leftArray, 0, n1);
        System.arraycopy(array, mid + 1, rightArray, 0, n2);
        
        int i = 0, j = 0, k = left;
        while (i < n1 && j < n2) {
            if (leftArray[i] <= rightArray[j]) {
                array[k] = leftArray[i];
                i++;
            } else {
                array[k] = rightArray[j];
                j++;
            }
            k++;
        }
        
        while (i < n1) {
            array[k] = leftArray[i];
            i++;
            k++;
        }
        
        while (j < n2) {
            array[k] = rightArray[j];
            j++;
            k++;
        }
    }
    
    // Fibonacci without caching
    private static long fibonacciWithoutCache(int n) {
        if (n <= 1) return n;
        return fibonacciWithoutCache(n - 1) + fibonacciWithoutCache(n - 2);
    }
    
    // Fibonacci with caching
    private static Map<Integer, Long> fibCache = new HashMap<>();
    
    private static long fibonacciWithCache(int n) {
        if (n <= 1) return n;
        
        if (fibCache.containsKey(n)) {
            return fibCache.get(n);
        }
        
        long result = fibonacciWithCache(n - 1) + fibonacciWithCache(n - 2);
        fibCache.put(n, result);
        return result;
    }
}

// Simple LRU Cache implementation
class LRUCache<K, V> {
    private final int capacity;
    private final Map<K, V> cache;
    private final Deque<K> queue;
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new HashMap<>();
        this.queue = new LinkedList<>();
    }
    
    public V get(K key) {
        if (cache.containsKey(key)) {
            // Move to front (most recently used)
            queue.remove(key);
            queue.addFirst(key);
            return cache.get(key);
        }
        return null;
    }
    
    public void put(K key, V value) {
        if (cache.containsKey(key)) {
            // Update existing entry
            queue.remove(key);
        } else if (cache.size() >= capacity) {
            // Remove least recently used
            K lruKey = queue.removeLast();
            cache.remove(lruKey);
        }
        
        // Add new entry
        cache.put(key, value);
        queue.addFirst(key);
    }
    
    @Override
    public String toString() {
        return "LRUCache{cache=" + cache + ", queue=" + queue + "}";
    }
}
```

## Code-Level Optimizations

### Object Creation and Memory Management

```java
import java.util.*;
import java.util.concurrent.*;

public class CodeOptimization {
    public static void main(String[] args) {
        System.out.println("=== Code-Level Performance Optimization ===");
        
        // Demonstrate object pooling
        demonstrateObjectPooling();
        
        // Compare string operations
        compareStringOperations();
        
        // Demonstrate lazy initialization
        demonstrateLazyInitialization();
        
        // Show efficient collection usage
        demonstrateEfficientCollections();
    }
    
    private static void demonstrateObjectPooling() {
        System.out.println("\n=== Object Pooling ===");
        
        final int iterations = 100000;
        
        // Without pooling
        long startTime = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            ExpensiveObject obj = new ExpensiveObject("Object-" + i);
            obj.process();
        }
        long withoutPoolingTime = System.nanoTime() - startTime;
        
        // With pooling
        ObjectPool<ExpensiveObject> pool = new ObjectPool<>(() -> new ExpensiveObject("Pooled"));
        startTime = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            ExpensiveObject obj = pool.acquire();
            obj.setId("Object-" + i);
            obj.process();
            pool.release(obj);
        }
        long withPoolingTime = System.nanoTime() - startTime;
        
        System.out.printf("Without pooling: %d ns (%.2f ms)%n", 
            withoutPoolingTime, withoutPoolingTime / 1_000_000.0);
        System.out.printf("With pooling: %d ns (%.2f ms)%n", 
            withPoolingTime, withPoolingTime / 1_000_000.0);
        System.out.printf("Pooling improvement: %.2fx%n", 
            (double) withoutPoolingTime / withPoolingTime);
        System.out.println("Objects created: " + pool.getCreatedCount());
        System.out.println("Objects reused: " + pool.getReusedCount());
    }
    
    private static void compareStringOperations() {
        System.out.println("\n=== String Operation Optimization ===");
        
        final int iterations = 10000;
        
        // String concatenation with + operator
        long startTime = System.nanoTime();
        String result1 = "";
        for (int i = 0; i < iterations; i++) {
            result1 += "Item-" + i + ";";
        }
        long plusOperatorTime = System.nanoTime() - startTime;
        
        // String concatenation with StringBuilder
        startTime = System.nanoTime();
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < iterations; i++) {
            sb.append("Item-").append(i).append(";");
        }
        String result2 = sb.toString();
        long stringBuilderTime = System.nanoTime() - startTime;
        
        // String concatenation with String.join
        List<String> items = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            items.add("Item-" + i);
        }
        startTime = System.nanoTime();
        String result3 = String.join(";", items);
        long stringJoinTime = System.nanoTime() - startTime;
        
        System.out.printf("+ operator: %d ns (%.2f ms)%n", 
            plusOperatorTime, plusOperatorTime / 1_000_000.0);
        System.out.printf("StringBuilder: %d ns (%.2f ms)%n", 
            stringBuilderTime, stringBuilderTime / 1_000_000.0);
        System.out.printf("String.join: %d ns (%.2f ms)%n", 
            stringJoinTime, stringJoinTime / 1_000_000.0);
        System.out.printf("StringBuilder vs + improvement: %.2fx%n", 
            (double) plusOperatorTime / stringBuilderTime);
    }
    
    private static void demonstrateLazyInitialization() {
        System.out.println("\n=== Lazy Initialization ===");
        
        final int iterations = 1000000;
        
        // Eager initialization
        long startTime = System.nanoTime();
        EagerInitialization eager = new EagerInitialization();
        for (int i = 0; i < iterations; i++) {
            eager.getValue();
        }
        long eagerTime = System.nanoTime() - startTime;
        
        // Lazy initialization
        startTime = System.nanoTime();
        LazyInitialization lazy = new LazyInitialization();
        for (int i = 0; i < iterations; i++) {
            lazy.getValue();
        }
        long lazyTime = System.nanoTime() - startTime;
        
        System.out.printf("Eager initialization: %d ns (%.2f ms)%n", 
            eagerTime, eagerTime / 1_000_000.0);
        System.out.printf("Lazy initialization: %d ns (%.2f ms)%n", 
            lazyTime, lazyTime / 1_000_000.0);
    }
    
    private static void demonstrateEfficientCollections() {
        System.out.println("\n=== Efficient Collection Usage ===");
        
        final int size = 100000;
        
        // ArrayList with default capacity vs pre-sized
        long startTime = System.nanoTime();
        List<Integer> defaultList = new ArrayList<>();
        for (int i = 0; i < size; i++) {
            defaultList.add(i);
        }
        long defaultListTime = System.nanoTime() - startTime;
        
        startTime = System.nanoTime();
        List<Integer> preSizedList = new ArrayList<>(size);
        for (int i = 0; i < size; i++) {
            preSizedList.add(i);
        }
        long preSizedListTime = System.nanoTime() - startTime;
        
        System.out.printf("Default ArrayList: %d ns (%.2f ms)%n", 
            defaultListTime, defaultListTime / 1_000_000.0);
        System.out.printf("Pre-sized ArrayList: %d ns (%.2f ms)%n", 
            preSizedListTime, preSizedListTime / 1_000_000.0);
        System.out.printf("Pre-sizing improvement: %.2fx%n", 
            (double) defaultListTime / preSizedListTime);
        
        // HashMap with default vs specified capacity
        startTime = System.nanoTime();
        Map<Integer, String> defaultMap = new HashMap<>();
        for (int i = 0; i < size; i++) {
            defaultMap.put(i, "Value-" + i);
        }
        long defaultMapTime = System.nanoTime() - startTime;
        
        startTime = System.nanoTime();
        Map<Integer, String> specifiedMap = new HashMap<>(size);
        for (int i = 0; i < size; i++) {
            specifiedMap.put(i, "Value-" + i);
        }
        long specifiedMapTime = System.nanoTime() - startTime;
        
        System.out.printf("Default HashMap: %d ns (%.2f ms)%n", 
            defaultMapTime, defaultMapTime / 1_000_000.0);
        System.out.printf("Specified HashMap: %d ns (%.2f ms)%n", 
            specifiedMapTime, specifiedMapTime / 1_000_000.0);
        System.out.printf("Specified capacity improvement: %.2fx%n", 
            (double) defaultMapTime / specifiedMapTime);
    }
}

// Expensive object for pooling demonstration
class ExpensiveObject {
    private String id;
    private byte[] data;
    private List<String> metadata;
    
    public ExpensiveObject(String id) {
        this.id = id;
        this.data = new byte[1024]; // 1KB data
        this.metadata = new ArrayList<>(10);
        Arrays.fill(data, (byte) (id.hashCode() % 256));
    }
    
    public void process() {
        // Simulate processing
        for (int i = 0; i < 5; i++) {
            metadata.add("Metadata-" + i);
        }
    }
    
    public String getId() {
        return id;
    }
    
    public void setId(String id) {
        this.id = id;
    }
    
    public void reset() {
        this.metadata.clear();
        Arrays.fill(data, (byte) 0);
    }
}

// Simple object pool
class ObjectPool<T> {
    private final Queue<T> pool = new ConcurrentLinkedQueue<>();
    private final Supplier<T> factory;
    private final AtomicInteger createdCount = new AtomicInteger(0);
    private final AtomicInteger reusedCount = new AtomicInteger(0);
    
    public ObjectPool(Supplier<T> factory) {
        this.factory = factory;
    }
    
    public T acquire() {
        T object = pool.poll();
        if (object == null) {
            object = factory.get();
            createdCount.incrementAndGet();
        } else {
            reusedCount.incrementAndGet();
            if (object instanceof ExpensiveObject) {
                ((ExpensiveObject) object).reset();
            }
        }
        return object;
    }
    
    public void release(T object) {
        if (object instanceof ExpensiveObject) {
            ((ExpensiveObject) object).reset();
        }
        pool.offer(object);
    }
    
    public int getCreatedCount() {
        return createdCount.get();
    }
    
    public int getReusedCount() {
        return reusedCount.get();
    }
}

// Eager initialization example
class EagerInitialization {
    private final String value = computeExpensiveValue();
    
    private String computeExpensiveValue() {
        // Simulate expensive computation
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 1000; i++) {
            sb.append("Computed-").append(i);
        }
        return sb.toString();
    }
    
    public String getValue() {
        return value;
    }
}

// Lazy initialization example
class LazyInitialization {
    private String value;
    
    public String getValue() {
        if (value == null) {
            value = computeExpensiveValue();
        }
        return value;
    }
    
    private String computeExpensiveValue() {
        // Simulate expensive computation
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 1000; i++) {
            sb.append("Computed-").append(i);
        }
        return sb.toString();
    }
}
```

## Concurrency and Parallel Processing

### Parallel Processing Optimization

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.*;

public class ParallelProcessingOptimization {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Parallel Processing Optimization ===");
        
        // Compare sequential vs parallel processing
        compareSequentialParallel();
        
        // Demonstrate ForkJoinPool usage
        demonstrateForkJoinPool();
        
        // Show parallel stream optimization
        demonstrateParallelStreams();
        
        // Compare different concurrent collections
        compareConcurrentCollections();
    }
    
    private static void compareSequentialParallel() {
        System.out.println("\n=== Sequential vs Parallel Processing ===");
        
        List<Integer> numbers = IntStream.range(1, 1000000)
            .boxed()
            .collect(Collectors.toList());
        
        // Sequential processing
        long startTime = System.nanoTime();
        long sequentialSum = numbers.stream()
            .mapToLong(Integer::longValue)
            .map(x -> x * x)
            .filter(x -> x % 2 == 0)
            .sum();
        long sequentialTime = System.nanoTime() - startTime;
        
        // Parallel processing
        startTime = System.nanoTime();
        long parallelSum = numbers.parallelStream()
            .mapToLong(Integer::longValue)
            .map(x -> x * x)
            .filter(x -> x % 2 == 0)
            .sum();
        long parallelTime = System.nanoTime() - startTime;
        
        System.out.printf("Sequential sum: %d in %d ns (%.2f ms)%n", 
            sequentialSum, sequentialTime, sequentialTime / 1_000_000.0);
        System.out.printf("Parallel sum: %d in %d ns (%.2f ms)%n", 
            parallelSum, parallelTime, parallelTime / 1_000_000.0);
        System.out.printf("Parallel improvement: %.2fx%n", 
            (double) sequentialTime / parallelTime);
    }
    
    private static void demonstrateForkJoinPool() {
        System.out.println("\n=== ForkJoinPool Demonstration ===");
        
        int[] array = new int[10000000];
        for (int i = 0; i < array.length; i++) {
            array[i] = i;
        }
        
        // Sequential sum
        long startTime = System.nanoTime();
        long sequentialSum = 0;
        for (int value : array) {
            sequentialSum += value;
        }
        long sequentialTime = System.nanoTime() - startTime;
        
        // ForkJoinPool sum
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        startTime = System.nanoTime();
        long forkJoinSum = forkJoinPool.invoke(new SumTask(array, 0, array.length));
        long forkJoinTime = System.nanoTime() - startTime;
        forkJoinPool.shutdown();
        
        System.out.printf("Sequential sum: %d in %d ns (%.2f ms)%n", 
            sequentialSum, sequentialTime, sequentialTime / 1_000_000.0);
        System.out.printf("ForkJoin sum: %d in %d ns (%.2f ms)%n", 
            forkJoinSum, forkJoinTime, forkJoinTime / 1_000_000.0);
        System.out.printf("ForkJoin improvement: %.2fx%n", 
            (double) sequentialTime / forkJoinTime);
    }
    
    private static void demonstrateParallelStreams() {
        System.out.println("\n=== Parallel Streams Optimization ===");
        
        List<String> words = Arrays.asList(
            "performance", "optimization", "parallel", "processing", 
            "concurrency", "efficiency", "scalability", "throughput"
        );
        
        // Create large dataset
        List<String> largeDataset = new ArrayList<>();
        for (int i = 0; i < 100000; i++) {
            largeDataset.add(words.get(i % words.size()) + "-" + i);
        }
        
        // Sequential processing
        long startTime = System.nanoTime();
        List<String> sequentialResult = largeDataset.stream()
            .filter(s -> s.length() > 10)
            .map(String::toUpperCase)
            .sorted()
            .collect(Collectors.toList());
        long sequentialTime = System.nanoTime() - startTime;
        
        // Parallel processing
        startTime = System.nanoTime();
        List<String> parallelResult = largeDataset.parallelStream()
            .filter(s -> s.length() > 10)
            .map(String::toUpperCase)
            .sorted()
            .collect(Collectors.toList());
        long parallelTime = System.nanoTime() - startTime;
        
        System.out.printf("Sequential processing: %d ns (%.2f ms), result size: %d%n", 
            sequentialTime, sequentialTime / 1_000_000.0, sequentialResult.size());
        System.out.printf("Parallel processing: %d ns (%.2f ms), result size: %d%n", 
            parallelTime, parallelTime / 1_000_000.0, parallelResult.size());
        System.out.printf("Parallel improvement: %.2fx%n", 
            (double) sequentialTime / parallelTime);
    }
    
    private static void compareConcurrentCollections() {
        System.out.println("\n=== Concurrent Collections Comparison ===");
        
        final int operations = 100000;
        
        // ConcurrentHashMap
        ConcurrentHashMap<String, Integer> concurrentMap = new ConcurrentHashMap<>();
        long startTime = System.nanoTime();
        for (int i = 0; i < operations; i++) {
            concurrentMap.put("Key-" + i, i);
            concurrentMap.get("Key-" + (i / 2));
        }
        long concurrentMapTime = System.nanoTime() - startTime;
        
        // Collections.synchronizedMap
        Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
        startTime = System.nanoTime();
        for (int i = 0; i < operations; i++) {
            syncMap.put("Key-" + i, i);
            syncMap.get("Key-" + (i / 2));
        }
        long syncMapTime = System.nanoTime() - startTime;
        
        // Regular HashMap (not thread-safe, for comparison)
        Map<String, Integer> regularMap = new HashMap<>();
        startTime = System.nanoTime();
        for (int i = 0; i < operations; i++) {
            regularMap.put("Key-" + i, i);
            regularMap.get("Key-" + (i / 2));
        }
        long regularMapTime = System.nanoTime() - startTime;
        
        System.out.printf("ConcurrentHashMap: %d ns (%.2f ms)%n", 
            concurrentMapTime, concurrentMapTime / 1_000_000.0);
        System.out.printf("SynchronizedMap: %d ns (%.2f ms)%n", 
            syncMapTime, syncMapTime / 1_000_000.0);
        System.out.printf("HashMap: %d ns (%.2f ms)%n", 
            regularMapTime, regularMapTime / 1_000_000.0);
        System.out.printf("ConcurrentHashMap vs SynchronizedMap: %.2fx%n", 
            (double) syncMapTime / concurrentMapTime);
    }
}

// ForkJoinTask for sum calculation
class SumTask extends RecursiveTask<Long> {
    private final int[] array;
    private final int start;
    private final int end;
    private static final int THRESHOLD = 10000;
    
    public SumTask(int[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }
    
    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            // Sequential computation for small segments
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += array[i];
            }
            return sum;
        } else {
            // Split the task into two subtasks
            int mid = start + (end - start) / 2;
            SumTask leftTask = new SumTask(array, start, mid);
            SumTask rightTask = new SumTask(array, mid, end);
            
            // Fork the left task and compute the right task directly
            leftTask.fork();
            long rightResult = rightTask.compute();
            long leftResult = leftTask.join();
            
            return leftResult + rightResult;
        }
    }
}
```

## Profiling and Monitoring

### Performance Profiling Techniques

```java
import java.lang.management.*;
import java.util.*;
import java.util.concurrent.*;

public class PerformanceProfiling {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Performance Profiling and Monitoring ===");
        
        // Demonstrate custom profiling
        demonstrateCustomProfiling();
        
        // Show memory profiling
        demonstrateMemoryProfiling();
        
        // Demonstrate CPU profiling
        demonstrateCPUProfiling();
        
        // Show thread profiling
        demonstrateThreadProfiling();
    }
    
    private static void demonstrateCustomProfiling() {
        System.out.println("\n=== Custom Profiling Implementation ===");
        
        Profiler profiler = new Profiler();
        
        // Profile different operations
        profiler.start("Array Initialization");
        int[] array = new int[1000000];
        for (int i = 0; i < array.length; i++) {
            array[i] = i;
        }
        profiler.stop("Array Initialization");
        
        profiler.start("Array Processing");
        long sum = 0;
        for (int value : array) {
            sum += value * value;
        }
        profiler.stop("Array Processing");
        
        profiler.start("Collection Operations");
        List<Integer> list = new ArrayList<>();
        for (int i = 0; i < 100000; i++) {
            list.add(i);
        }
        profiler.stop("Collection Operations");
        
        profiler.printReport();
    }
    
    private static void demonstrateMemoryProfiling() {
        System.out.println("\n=== Memory Profiling ===");
        
        MemoryProfiler memoryProfiler = new MemoryProfiler();
        
        System.out.println("Initial memory usage:");
        memoryProfiler.printMemoryUsage();
        
        // Create objects to observe memory changes
        memoryProfiler.startMeasurement("Object Creation");
        List<MemoryIntensiveObject> objects = new ArrayList<>();
        for (int i = 0; i < 100000; i++) {
            objects.add(new MemoryIntensiveObject());
        }
        memoryProfiler.stopMeasurement("Object Creation");
        
        System.out.println("After object creation:");
        memoryProfiler.printMemoryUsage();
        
        // Clear objects and force GC
        objects.clear();
        System.gc();
        Thread.sleep(1000);
        
        System.out.println("After cleanup and GC:");
        memoryProfiler.printMemoryUsage();
        
        memoryProfiler.printReport();
    }
    
    private static void demonstrateCPUProfiling() throws InterruptedException {
        System.out.println("\n=== CPU Profiling ===");
        
        CPUMonitor cpuMonitor = new CPUMonitor();
        
        System.out.println("Starting CPU-intensive operations...");
        
        // CPU-intensive task 1
        cpuMonitor.startMonitoring("Mathematical Calculations");
        performMathematicalCalculations();
        cpuMonitor.stopMonitoring("Mathematical Calculations");
        
        // CPU-intensive task 2
        cpuMonitor.startMonitoring("String Processing");
        performStringProcessing();
        cpuMonitor.stopMonitoring("String Processing");
        
        // Multi-threaded task
        cpuMonitor.startMonitoring("Parallel Processing");
        performParallelProcessing();
        cpuMonitor.stopMonitoring("Parallel Processing");
        
        cpuMonitor.printReport();
    }
    
    private static void demonstrateThreadProfiling() throws InterruptedException {
        System.out.println("\n=== Thread Profiling ===");
        
        ThreadProfiler threadProfiler = new ThreadProfiler();
        
        System.out.println("Creating and monitoring threads:");
        
        ExecutorService executor = Executors.newFixedThreadPool(4);
        CountDownLatch latch = new CountDownLatch(4);
        
        for (int i = 0; i < 4; i++) {
            final int threadId = i;
            executor.submit(() -> {
                threadProfiler.startThread("Worker-" + threadId);
                try {
                    performThreadWork(threadId);
                } finally {
                    threadProfiler.endThread("Worker-" + threadId);
                    latch.countDown();
                }
            });
        }
        
        latch.await();
        executor.shutdown();
        
        threadProfiler.printReport();
    }
    
    // Helper methods for profiling demonstrations
    private static void performMathematicalCalculations() {
        double result = 0;
        for (int i = 0; i < 1000000; i++) {
            result += Math.sqrt(i) * Math.log(i + 1);
        }
    }
    
    private static void performStringProcessing() {
        List<String> strings = new ArrayList<>();
        for (int i = 0; i < 100000; i++) {
            strings.add("String-" + i + "-processing");
        }
        
        for (String s : strings) {
            s.toUpperCase().toLowerCase().substring(0, Math.min(10, s.length()));
        }
    }
    
    private static void performParallelProcessing() throws InterruptedException {
        List<Integer> numbers = IntStream.range(1, 100000)
            .boxed()
            .collect(Collectors.toList());
        
        numbers.parallelStream()
            .map(x -> x * x)
            .filter(x -> x % 2 == 0)
            .mapToLong(Long::valueOf)
            .sum();
    }
    
    private static void performThreadWork(int threadId) {
        try {
            for (int i = 0; i < 10000; i++) {
                Math.sqrt(i) + Math.log(i + 1);
                if (i % 1000 == 0) {
                    Thread.sleep(1);
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

// Simple profiler implementation
class Profiler {
    private Map<String, Long> startTimes = new HashMap<>();
    private Map<String, Long> durations = new HashMap<>();
    
    public void start(String operation) {
        startTimes.put(operation, System.nanoTime());
    }
    
    public void stop(String operation) {
        Long startTime = startTimes.get(operation);
        if (startTime != null) {
            long duration = System.nanoTime() - startTime;
            durations.put(operation, duration);
        }
    }
    
    public void printReport() {
        System.out.println("\n=== Profiling Report ===");
        durations.entrySet().stream()
            .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
            .forEach(entry -> {
                System.out.printf("%s: %d ns (%.2f ms)%n", 
                    entry.getKey(), entry.getValue(), entry.getValue() / 1_000_000.0);
            });
    }
}

// Memory profiler implementation
class MemoryProfiler {
    private Map<String, Long> memoryMeasurements = new HashMap<>();
    private Runtime runtime = Runtime.getRuntime();
    
    public void printMemoryUsage() {
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        
        System.out.printf("  Total: %d MB, Used: %d MB, Free: %d MB%n",
            totalMemory / (1024 * 1024),
            usedMemory / (1024 * 1024),
            freeMemory / (1024 * 1024));
    }
    
    public void startMeasurement(String operation) {
        System.gc(); // Try to get consistent measurements
        memoryMeasurements.put(operation + "_start", 
            runtime.totalMemory() - runtime.freeMemory());
    }
    
    public void stopMeasurement(String operation) {
        System.gc();
        long endMemory = runtime.totalMemory() - runtime.freeMemory();
        Long startMemory = memoryMeasurements.get(operation + "_start");
        if (startMemory != null) {
            memoryMeasurements.put(operation, endMemory - startMemory);
        }
    }
    
    public void printReport() {
        System.out.println("\n=== Memory Profiling Report ===");
        memoryMeasurements.entrySet().stream()
            .filter(entry -> !entry.getKey().endsWith("_start"))
            .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
            .forEach(entry -> {
                System.out.printf("%s: %d KB%n", 
                    entry.getKey(), entry.getValue() / 1024);
            });
    }
}

// CPU monitor implementation
class CPUMonitor {
    private Map<String, Long> startTimes = new HashMap<>();
    private Map<String, Long> cpuTimes = new HashMap<>();
    
    public void startMonitoring(String operation) {
        startTimes.put(operation, System.nanoTime());
    }
    
    public void stopMonitoring(String operation) {
        Long startTime = startTimes.get(operation);
        if (startTime != null) {
            long duration = System.nanoTime() - startTime;
            cpuTimes.put(operation, duration);
        }
    }
    
    public void printReport() {
        System.out.println("\n=== CPU Profiling Report ===");
        cpuTimes.entrySet().stream()
            .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
            .forEach(entry -> {
                System.out.printf("%s: %d ns (%.2f ms)%n", 
                    entry.getKey(), entry.getValue(), entry.getValue() / 1_000_000.0);
            });
    }
}

// Thread profiler implementation
class ThreadProfiler {
    private Map<String, Long> threadStartTimes = new HashMap<>();
    private Map<String, Long> threadDurations = new HashMap<>();
    
    public void startThread(String threadName) {
        threadStartTimes.put(threadName, System.nanoTime());
    }
    
    public void endThread(String threadName) {
        Long startTime = threadStartTimes.get(threadName);
        if (startTime != null) {
            long duration = System.nanoTime() - startTime;
            threadDurations.put(threadName, duration);
        }
    }
    
    public void printReport() {
        System.out.println("\n=== Thread Profiling Report ===");
        threadDurations.entrySet().stream()
            .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
            .forEach(entry -> {
                System.out.printf("%s: %d ns (%.2f ms)%n", 
                    entry.getKey(), entry.getValue(), entry.getValue() / 1_000_000.0);
            });
    }
}

// Memory-intensive object for profiling
class MemoryIntensiveObject {
    private byte[] data = new byte[1024]; // 1KB per object
    private String id = UUID.randomUUID().toString();
    
    public MemoryIntensiveObject() {
        Arrays.fill(data, (byte) (Math.random() * 256));
    }
}
```

## JVM Monitoring and Tuning Tools

### Advanced JVM Monitoring

```java
import java.lang.management.*;
import java.util.*;
import java.util.concurrent.*;

public class JVMToolsAndMonitoring {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== JVM Monitoring and Tuning Tools ===");
        
        // Demonstrate JMX monitoring
        demonstrateJMXMonitoring();
        
        // Show garbage collection monitoring
        demonstrateGCMonitoring();
        
        // Demonstrate memory pool monitoring
        demonstrateMemoryPoolMonitoring();
        
        // Show thread monitoring
        demonstrateThreadMonitoring();
    }
    
    private static void demonstrateJMXMonitoring() {
        System.out.println("\n=== JMX Monitoring ===");
        
        // Get runtime MBean
        RuntimeMXBean runtimeBean = ManagementFactory.getRuntimeMXBean();
        System.out.println("JVM Name: " + runtimeBean.getVmName());
        System.out.println("JVM Version: " + runtimeBean.getVmVersion());
        System.out.println("Uptime: " + runtimeBean.getUptime() / 1000 + " seconds");
        System.out.println("Start time: " + new Date(runtimeBean.getStartTime()));
        
        // Get operating system MBean
        OperatingSystemMXBean osBean = ManagementFactory.getOperatingSystemMXBean();
        System.out.println("Available processors: " + osBean.getAvailableProcessors());
        System.out.println("System load average: " + osBean.getSystemLoadAverage());
        
        // Get compilation MBean
        CompilationMXBean compilationBean = ManagementFactory.getCompilationMXBean();
        if (compilationBean != null) {
            System.out.println("Total compilation time: " + 
                compilationBean.getTotalCompilationTime() + " ms");
        }
    }
    
    private static void demonstrateGCMonitoring() throws InterruptedException {
        System.out.println("\n=== Garbage Collection Monitoring ===");
        
        List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        
        System.out.println("Garbage Collectors:");
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            System.out.println("  " + gcBean.getName());
            System.out.println("    Collections: " + gcBean.getCollectionCount());
            System.out.println("    Total time: " + gcBean.getCollectionTime() + " ms");
        }
        
        // Monitor GC during memory allocation
        System.out.println("\nMonitoring GC during memory allocation:");
        
        // Take initial readings
        Map<String, Long> initialCollections = new HashMap<>();
        Map<String, Long> initialTimes = new HashMap<>();
        
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            initialCollections.put(gcBean.getName(), gcBean.getCollectionCount());
            initialTimes.put(gcBean.getName(), gcBean.getCollectionTime());
        }
        
        // Create memory pressure
        List<byte[]> memoryHog = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            memoryHog.add(new byte[1024 * 1024]); // 1MB each
            if (i % 100 == 0) {
                Thread.sleep(100);
            }
        }
        
        // Take final readings
        System.out.println("\nAfter memory allocation:");
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            Long initialCount = initialCollections.get(gcBean.getName());
            Long initialTime = initialTimes.get(gcBean.getName());
            
            if (initialCount != null && initialTime != null) {
                long collections = gcBean.getCollectionCount() - initialCount;
                long time = gcBean.getCollectionTime() - initialTime;
                
                System.out.printf("  %s - New collections: %d, New time: %d ms%n",
                    gcBean.getName(), collections, time);
            }
        }
        
        memoryHog.clear();
        System.gc();
    }
    
    private static void demonstrateMemoryPoolMonitoring() {
        System.out.println("\n=== Memory Pool Monitoring ===");
        
        List<MemoryPoolMXBean> memoryPools = ManagementFactory.getMemoryPoolMXBeans();
        
        System.out.println("Memory Pools:");
        for (MemoryPoolMXBean pool : memoryPools) {
            System.out.println("  " + pool.getName() + " (" + pool.getType() + ")");
            
            MemoryUsage usage = pool.getUsage();
            if (usage != null) {
                System.out.printf("    Current: %d MB / %d MB (%.1f%%)%n",
                    usage.getUsed() / (1024 * 1024),
                    usage.getMax() / (1024 * 1024),
                    (double) usage.getUsed() / usage.getMax() * 100);
            }
            
            MemoryUsage peak = pool.getPeakUsage();
            if (peak != null) {
                System.out.printf("    Peak: %d MB%n", peak.getUsed() / (1024 * 1024));
            }
        }
        
        // Monitor heap usage
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        MemoryUsage nonHeapUsage = memoryBean.getNonHeapMemoryUsage();
        
        System.out.printf("\nHeap Memory: %d MB / %d MB (%.1f%%)%n",
            heapUsage.getUsed() / (1024 * 1024),
            heapUsage.getMax() / (1024 * 1024),
            (double) heapUsage.getUsed() / heapUsage.getMax() * 100);
        
        System.out.printf("Non-Heap Memory: %d MB / %d MB (%.1f%%)%n",
            nonHeapUsage.getUsed() / (1024 * 1024),
            nonHeapUsage.getMax() / (1024 * 1024),
            (double) nonHeapUsage.getUsed() / nonHeapUsage.getMax() * 100);
    }
    
    private static void demonstrateThreadMonitoring() throws InterruptedException {
        System.out.println("\n=== Thread Monitoring ===");
        
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        
        System.out.println("Thread Information:");
        System.out.println("  Current thread count: " + threadBean.getThreadCount());
        System.out.println("  Peak thread count: " + threadBean.getPeakThreadCount());
        System.out.println("  Total started threads: " + threadBean.getTotalStartedThreadCount());
        System.out.println("  Daemon threads: " + threadBean.getDaemonThreadCount());
        
        // Monitor thread creation
        System.out.println("\nMonitoring thread creation:");
        long initialThreadCount = threadBean.getTotalStartedThreadCount();
        
        // Create some threads
        ExecutorService executor = Executors.newFixedThreadPool(5);
        CountDownLatch latch = new CountDownLatch(10);
        
        for (int i = 0; i < 10; i++) {
            executor.submit(() -> {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    latch.countDown();
                }
            });
        }
        
        latch.await();
        executor.shutdown();
        
        long finalThreadCount = threadBean.getTotalStartedThreadCount();
        System.out.println("Threads created: " + (finalThreadCount - initialThreadCount));
        System.out.println("Current thread count: " + threadBean.getThreadCount());
    }
}
```

## Performance Testing and Benchmarking

### JMH (Java Microbenchmark Harness) Examples

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

// Note: This would typically be in separate files for JMH
// This is a simplified demonstration of JMH concepts

public class PerformanceTesting {
    public static void main(String[] args) {
        System.out.println("=== Performance Testing and Benchmarking ===");
        
        // Demonstrate microbenchmark concepts
        demonstrateMicrobenchmarking();
        
        // Show benchmark result analysis
        demonstrateBenchmarkAnalysis();
        
        // Demonstrate proper benchmark setup
        demonstrateProperBenchmarking();
    }
    
    private static void demonstrateMicrobenchmarking() {
        System.out.println("\n=== Microbenchmarking Concepts ===");
        
        // Example of what NOT to do in benchmarks
        System.out.println("Common benchmarking mistakes:");
        System.out.println("1. Not warming up the JIT compiler");
        System.out.println("2. Including setup/teardown in measurements");
        System.out.println("3. Not accounting for dead code elimination");
        System.out.println("4. Testing with unrealistic data sizes");
        System.out.println("5. Not running multiple iterations");
        
        // Proper benchmarking approach
        System.out.println("\nProper benchmarking approach:");
        demonstrateProperApproach();
    }
    
    private static void demonstrateProperApproach() {
        System.out.println("Running proper benchmark simulation:");
        
        // Warm up phase
        System.out.println("Warming up JIT compiler...");
        for (int i = 0; i < 50000; i++) {
            performBenchmarkOperation(i);
        }
        
        // Measurement phase
        final int iterations = 100000;
        
        long startTime = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            performBenchmarkOperation(i);
        }
        long totalTime = System.nanoTime() - startTime;
        
        double avgTime = (double) totalTime / iterations;
        System.out.printf("Average time per operation: %.2f ns%n", avgTime);
        System.out.printf("Total time for %d operations: %.2f ms%n", 
            iterations, totalTime / 1_000_000.0);
    }
    
    private static int performBenchmarkOperation(int input) {
        // Simple operation that can be optimized
        return input * input + input;
    }
    
    private static void demonstrateBenchmarkAnalysis() {
        System.out.println("\n=== Benchmark Result Analysis ===");
        
        // Compare different implementations
        System.out.println("Comparing ArrayList vs LinkedList for random access:");
        
        final int size = 10000;
        final int accesses = 1000;
        
        // ArrayList benchmark
        List<Integer> arrayList = new ArrayList<>();
        for (int i = 0; i < size; i++) {
            arrayList.add(i);
        }
        
        long startTime = System.nanoTime();
        Random random = new Random();
        for (int i = 0; i < accesses; i++) {
            int index = random.nextInt(size);
            arrayList.get(index);
        }
        long arrayListTime = System.nanoTime() - startTime;
        
        // LinkedList benchmark
        List<Integer> linkedList = new LinkedList<>();
        for (int i = 0; i < size; i++) {
            linkedList.add(i);
        }
        
        startTime = System.nanoTime();
        for (int i = 0; i < accesses; i++) {
            int index = random.nextInt(size);
            linkedList.get(index);
        }
        long linkedListTime = System.nanoTime() - startTime;
        
        System.out.printf("ArrayList random access: %d ns (%.2f ns/operation)%n", 
            arrayListTime, (double) arrayListTime / accesses);
        System.out.printf("LinkedList random access: %d ns (%.2f ns/operation)%n", 
            linkedListTime, (double) linkedListTime / accesses);
        System.out.printf("ArrayList is %.2fx faster%n", 
            (double) linkedListTime / arrayListTime);
        
        // Statistical analysis
        System.out.println("\nStatistical analysis:");
        System.out.println("Confidence in results: High (large sample size)");
        System.out.println("Variance: Low (consistent operations)");
        System.out.println("Significance: Very high (large difference)");
    }
    
    private static void demonstrateProperBenchmarking() {
        System.out.println("\n=== Proper Benchmarking Setup ===");
        
        System.out.println("Key principles for proper benchmarking:");
        System.out.println("1. Warm up the JVM (10,000+ iterations)");
        System.out.println("2. Run multiple iterations (1000+)");
        System.out.println("3. Use realistic data sizes");
        System.out.println("4. Account for JVM optimizations");
        System.out.println("5. Measure steady-state performance");
        System.out.println("6. Use proper statistical analysis");
        
        // Demonstrate proper setup
        demonstrateProperSetup();
    }
    
    private static void demonstrateProperSetup() {
        System.out.println("\nDemonstrating proper benchmark setup:");
        
        // Configuration
        final int warmupIterations = 20000;
        final int measurementIterations = 100000;
        
        // Warmup phase
        System.out.println("Warmup phase (" + warmupIterations + " iterations)...");
        for (int i = 0; i < warmupIterations; i++) {
            benchmarkTargetMethod(i);
        }
        
        // Measurement phase
        System.out.println("Measurement phase (" + measurementIterations + " iterations)...");
        long[] times = new long[measurementIterations];
        
        for (int i = 0; i < measurementIterations; i++) {
            long startTime = System.nanoTime();
            benchmarkTargetMethod(i);
            times[i] = System.nanoTime() - startTime;
        }
        
        // Analysis
        long totalTime = Arrays.stream(times).sum();
        double averageTime = (double) totalTime / measurementIterations;
        double minTime = Arrays.stream(times).min().orElse(0);
        double maxTime = Arrays.stream(times).max().orElse(0);
        
        // Calculate standard deviation
        double variance = Arrays.stream(times)
            .mapToDouble(t -> Math.pow(t - averageTime, 2))
            .average()
            .orElse(0);
        double stdDev = Math.sqrt(variance);
        
        System.out.printf("Results:%n");
        System.out.printf("  Average time: %.2f ns%n", averageTime);
        System.out.printf("  Minimum time: %.2f ns%n", minTime);
        System.out.printf("  Maximum time: %.2f ns%n", maxTime);
        System.out.printf("  Standard deviation: %.2f ns%n", stdDev);
        System.out.printf("  Total time: %.2f ms%n", totalTime / 1_000_000.0);
    }
    
    private static int benchmarkTargetMethod(int input) {
        // Method to benchmark
        return Integer.numberOfLeadingZeros(input) + 
               Integer.numberOfTrailingZeros(input);
    }
}

// Example of what a JMH benchmark might look like
/*
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
@Fork(1)
@Warmup(iterations = 3, time = 5, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 10, timeUnit = TimeUnit.SECONDS)
public class ExampleBenchmark {
    
    private List<Integer> arrayList;
    private List<Integer> linkedList;
    private Random random;
    
    @Setup
    public void setup() {
        arrayList = new ArrayList<>();
        linkedList = new LinkedList<>();
        random = new Random();
        
        for (int i = 0; i < 10000; i++) {
            arrayList.add(i);
            linkedList.add(i);
        }
    }
    
    @Benchmark
    public Integer arrayListRandomAccess() {
        return arrayList.get(random.nextInt(arrayList.size()));
    }
    
    @Benchmark
    public Integer linkedListRandomAccess() {
        return linkedList.get(random.nextInt(linkedList.size()));
    }
}
*/
```

## Summary

Java performance optimization is a multifaceted discipline that requires understanding of:

1. **JVM Internals**: How the JVM works, JIT compilation, garbage collection
2. **Algorithm Efficiency**: Choosing the right algorithms and data structures
3. **Code Optimization**: Writing efficient code, minimizing object creation
4. **Concurrency**: Leveraging parallel processing and concurrent collections
5. **Profiling and Monitoring**: Using tools to identify performance bottlenecks
6. **Benchmarking**: Properly measuring and comparing performance

Key optimization strategies include:
- Understanding and leveraging JIT compilation
- Choosing appropriate data structures for specific use cases
- Minimizing object allocation and using object pooling
- Optimizing string operations and collection usage
- Leveraging parallel processing for CPU-intensive tasks
- Using proper profiling and monitoring tools
- Writing meaningful benchmarks for performance testing

Remember that premature optimization is often counterproductive. Always profile first to identify actual bottlenecks, then apply targeted optimizations based on measurements rather than assumptions.