# Advanced Java Memory Management and Garbage Collection Optimization

## Table of Contents
1. [Introduction to JVM Memory Management](#introduction-to-jvm-memory-management)
2. [JVM Memory Architecture](#jvm-memory-architecture)
3. [Garbage Collection Algorithms](#garbage-collection-algorithms)
4. [Memory Pools and Generational Model](#memory-pools-and-generational-model)
5. [Garbage Collection Types and Tuning](#garbage-collection-types-and-tuning)
6. [Memory Profiling and Analysis](#memory-profiling-and-analysis)
7. [Performance Optimization Techniques](#performance-optimization-techniques)
8. [Memory Leaks Detection and Prevention](#memory-leaks-detection-and-prevention)
9. [JVM Monitoring and Metrics](#jvm-monitoring-and-metrics)
10. [Best Practices and Advanced Topics](#best-practices-and-advanced-topics)

## Introduction to JVM Memory Management

Java Virtual Machine (JVM) memory management is a critical aspect of application performance and reliability. Understanding how the JVM manages memory, allocates objects, and performs garbage collection is essential for building high-performance Java applications. Modern JVMs employ sophisticated garbage collection algorithms that balance between throughput, pause times, and memory utilization.

The JVM abstracts memory management from developers, but understanding its internal mechanisms allows for better application design and performance optimization. Memory management directly impacts application responsiveness, throughput, and stability, especially under high load conditions.

### Key Concepts in JVM Memory Management

```java
// Example: Understanding object allocation and memory layout
public class MemoryLayoutExample {
    
    private int primitiveField;           // 4 bytes - stored in object header
    private long primitiveFieldLong;      // 8 bytes - stored in object data
    private String referenceField;        // 8 bytes (64-bit JVM) - reference to heap
    private static String staticField;    // 8 bytes - stored in method area
    
    // Object header typically contains: mark word (8 bytes) + class pointer (4 bytes compressed oops)
    
    public static void main(String[] args) {
        // Object allocation in Young Generation (Eden space)
        MemoryLayoutExample obj = new MemoryLayoutExample();
        
        // Accessing object memory layout information (requires JVM tool)
        printObjectLayout(obj);
    }
    
    // Using JOL (Java Object Layout) to analyze memory layout
    private static void printObjectLayout(Object obj) {
        System.out.println(ClassLayout.parseClass(MemoryLayoutExample.class).toPrintable());
    }
    
    // Understanding array memory layout
    private static void analyzeArrayMemory() {
        int[] intArray = new int[1000];  // 1000 * 4 bytes + object header (16 bytes) = ~4016 bytes
        String[] stringArray = new String[1000];  // 1000 * 8 bytes + object header = ~8016 bytes
        
        System.out.println("Int array memory: " + intArray.length * Integer.BYTES);
        System.out.println("String array references: " + stringArray.length * 8);
    }
}
```

## JVM Memory Architecture

The JVM memory architecture is divided into several distinct areas, each serving specific purposes for the execution of Java applications.

### Memory Regions

1. **Method Area (Metaspace in Java 8+)**: Stores class definitions, method code, and static variables
2. **Heap Memory**: Runtime data area where all class instances and arrays are allocated
3. **Stack Memory**: Per-thread memory storing method frames and local variables
4. **Native Method Stack**: Separate stack for native method execution
5. **Program Counter (PC) Register**: Contains the address of the current executing instruction
6. **Direct Memory**: Memory allocated outside the JVM heap for NIO operations

### Heap Memory Structure

```java
// Demonstrating heap memory structure and allocation
public class HeapMemoryStructure {
    
    // Young Generation
    private static List<byte[]> youngGenObjects = new ArrayList<>();
    
    // Old Generation (Promoted objects)
    private static List<byte[]> oldGenObjects = new ArrayList<>();
    
    public static void demonstrateGenerationalStructure() {
        // Objects initially allocated in Young Generation (Eden space)
        for (int i = 0; i < 100; i++) {
            // Small objects typically allocated in Eden
            byte[] smallObject = new byte[1024]; // 1KB object
            youngGenObjects.add(smallObject);
        }
        
        // Objects that survive multiple GC cycles are promoted to Old Generation
        // This happens during Minor GC when Eden space fills up
        performMinorGC();
        
        // Large objects may be allocated directly in Old Generation
        // (depending on GC algorithm and JVM settings)
        for (int i = 0; i < 10; i++) {
            // Large object - might be allocated directly in Old Gen
            byte[] largeObject = new byte[1024 * 1024]; // 1MB object
            oldGenObjects.add(largeObject);
        }
        
        // Force GC to see the effects
        System.gc();
    }
    
    private static void performMinorGC() {
        // In practice, Minor GC happens automatically when Eden fills up
        // This is a demonstration of what happens during Minor GC
        System.out.println("Minor GC would occur when Eden space is full");
        System.out.println("Surviving objects are moved to Survivor spaces");
        System.out.println("Objects surviving multiple Minor GCs are promoted to Old Generation");
    }
    
    public static void main(String[] args) {
        demonstrateGenerationalStructure();
        
        // Analyze memory usage programmatically
        Runtime runtime = Runtime.getRuntime();
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        
        System.out.println("Total Memory: " + totalMemory / (1024 * 1024) + " MB");
        System.out.println("Used Memory: " + usedMemory / (1024 * 1024) + " MB");
        System.out.println("Free Memory: " + freeMemory / (1024 * 1024) + " MB");
    }
}
```

### Memory Allocation Strategies

```java
// Advanced memory allocation techniques
public class MemoryAllocationStrategies {
    
    // Object allocation with different strategies
    private static final int SMALL_OBJECT_SIZE = 1024;        // 1KB
    private static final int MEDIUM_OBJECT_SIZE = 1024 * 64;  // 64KB
    private static final int LARGE_OBJECT_SIZE = 1024 * 1024; // 1MB
    
    // TLAB (Thread Local Allocation Buffer) demonstration
    private static void demonstrateTLAB() {
        // Each thread gets its own TLAB for faster allocation
        // No synchronization needed for object allocation within TLAB
        List<byte[]> objects = new ArrayList<>();
        
        for (int i = 0; i < 1000; i++) {
            // Small objects allocated in TLAB
            objects.add(new byte[SMALL_OBJECT_SIZE]);
        }
        
        System.out.println("Objects allocated in TLAB: " + objects.size());
    }
    
    // Large object allocation behavior
    private static void demonstrateLargeObjectAllocation() {
        List<byte[]> largeObjects = new ArrayList<>();
        
        for (int i = 0; i < 10; i++) {
            // Large objects may be allocated directly in Old Generation
            // depending on JVM settings and GC algorithm
            largeObjects.add(new byte[LARGE_OBJECT_SIZE]);
        }
        
        System.out.println("Large objects allocated: " + largeObjects.size());
    }
    
    // Memory alignment and padding
    static class AlignedObject {
        private byte field1;    // 1 byte
        private byte field2;    // 1 byte
        // 6 bytes padding to align to 8-byte boundary
        private long field3;    // 8 bytes
    }
    
    // Understanding memory alignment
    public static void demonstrateMemoryAlignment() {
        AlignedObject obj = new AlignedObject();
        
        // Using JOL to see actual memory layout
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        
        // The object will be padded to align with JVM word size (typically 8 bytes)
        // This improves memory access performance
    }
    
    public static void main(String[] args) {
        demonstrateTLAB();
        demonstrateLargeObjectAllocation();
        demonstrateMemoryAlignment();
    }
}
```

## Garbage Collection Algorithms

Java's garbage collection algorithms have evolved significantly to address different application requirements such as low latency, high throughput, and large heap sizes.

### Serial Garbage Collector

The Serial GC is the simplest collector, using a single thread for both minor and major collections. It's suitable for single-threaded environments or applications with small heaps.

```java
// JVM startup parameters for Serial GC:
// -XX:+UseSerialGC

public class SerialGCDemo {
    
    // Serial GC characteristics:
    // - Single-threaded
    // - Stop-the-world during GC
    // - Mark-Sweep-Compact algorithm
    // - Suitable for small heaps (< 100MB)
    
    private static final int OBJECT_COUNT = 10000;
    private static final int GC_TRIGGER_SIZE = 1024 * 1024; // 1MB
    
    public static void demonstrateSerialGC() {
        List<byte[]> objects = new ArrayList<>();
        
        for (int i = 0; i < OBJECT_COUNT; i++) {
            // Create objects to fill young generation
            byte[] obj = new byte[1024]; // 1KB each
            objects.add(obj);
            
            // Force GC periodically to see Serial GC behavior
            if (i % 1000 == 0) {
                System.gc(); // Triggers Serial GC
                System.out.println("GC performed at iteration: " + i);
            }
        }
        
        objects.clear(); // Make objects eligible for GC
        System.gc(); // Final cleanup
        
        System.out.println("Serial GC demonstration completed");
    }
}
```

### Parallel Garbage Collector (Throughput Collector)

The Parallel GC uses multiple threads for garbage collection, optimizing for throughput rather than pause times.

```java
// JVM startup parameters for Parallel GC:
// -XX:+UseParallelGC (default in JDK 8-10)
// -XX:ParallelGCThreads=<n> to set number of GC threads

public class ParallelGCDemo {
    
    // Parallel GC characteristics:
    // - Multiple threads for young generation collection
    // - Multiple threads for old generation collection
    // - Stop-the-world during GC
    // - Optimized for throughput
    
    private static final int LARGE_OBJECT_COUNT = 50000;
    private static final int OBJECT_SIZE = 1024 * 16; // 16KB each
    
    public static void demonstrateParallelGC() {
        List<byte[]> objects = new ArrayList<>();
        
        for (int i = 0; i < LARGE_OBJECT_COUNT; i++) {
            // Create larger objects to trigger more frequent GC
            byte[] obj = new byte[OBJECT_SIZE];
            objects.add(obj);
            
            // Monitor GC frequency and pauses
            if (i % 5000 == 0) {
                System.out.println("Objects created: " + i);
                printMemoryStats();
            }
        }
        
        // Calculate memory pressure
        Runtime runtime = Runtime.getRuntime();
        long usedMemory = runtime.totalMemory() - runtime.freeMemory();
        System.out.println("Memory used before GC: " + usedMemory / (1024 * 1024) + " MB");
        
        // Clean up and trigger GC
        objects.clear();
        System.gc();
        
        long usedMemoryAfter = runtime.totalMemory() - runtime.freeMemory();
        System.out.println("Memory used after GC: " + usedMemoryAfter / (1024 * 1024) + " MB");
    }
    
    private static void printMemoryStats() {
        Runtime runtime = Runtime.getRuntime();
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        
        System.out.println("Total: " + totalMemory / (1024 * 1024) + " MB | " +
                          "Used: " + usedMemory / (1024 * 1024) + " MB | " +
                          "Free: " + freeMemory / (1024 * 1024) + " MB");
    }
}
```

### CMS (Concurrent Mark Sweep) Garbage Collector

CMS focuses on minimizing pause times at the expense of throughput.

```java
// JVM startup parameters for CMS:
// -XX:+UseConcMarkSweepGC
// -XX:+UseParNewGC (for young generation)

public class CMSGCDemo {
    
    // CMS GC characteristics:
    // - Low pause time collector
    // - Concurrent marking phase
    // - Stop-the-world during initial mark and remark phases
    // - Less suitable for large heaps due to fragmentation
    
    private static final int OBJECT_COUNT = 100000;
    private static final int OBJECT_SIZE = 1024; // 1KB each
    private static List<byte[]> longLivedObjects = new ArrayList<>();
    
    public static void demonstrateCMS() {
        List<byte[]> shortLivedObjects = new ArrayList<>();
        
        for (int i = 0; i < OBJECT_COUNT; i++) {
            // Create many short-lived objects
            byte[] shortObj = new byte[OBJECT_SIZE];
            shortLivedObjects.add(shortObj);
            
            // Create some long-lived objects
            if (i % 100 == 0) {
                byte[] longObj = new byte[OBJECT_SIZE * 10]; // 10KB long-lived object
                longLivedObjects.add(longObj);
            }
            
            // Monitor GC behavior with many short-lived objects
            if (i % 10000 == 0) {
                System.out.println("Processed " + i + " objects");
            }
        }
        
        // CMS will handle the mix of short and long-lived objects
        shortLivedObjects.clear(); // Clear short-lived objects
        
        // Force a full GC to see CMS behavior
        System.gc();
    }
}
```

### G1 (Garbage First) Garbage Collector

G1 is designed for large heaps with predictable pause times.

```java
// JVM startup parameters for G1:
// -XX:+UseG1GC (default in JDK 9+)
// -XX:MaxGCPauseMillis=<n> target pause time
// -XX:G1HeapRegionSize=<n> region size

public class G1GCDemo {
    
    // G1 GC characteristics:
    // - Designed for large heaps (4GB+)
    // - Predictable pause times
    // - Region-based memory management
    // - Concurrent marking and evacuation
    
    private static final int HEAP_SIZE = 4 * 1024 * 1024 * 1024L; // 4GB
    private static final int REGION_SIZE = 16 * 1024 * 1024; // 16MB regions
    
    public static void demonstrateG1() {
        // G1 divides heap into regions
        // Each region can be young or old generation
        // This allows for targeted collection
        
        System.out.println("G1 GC demonstration");
        System.out.println("Heap will be divided into regions of " + REGION_SIZE / (1024 * 1024) + "MB");
        
        // Create objects with different lifetimes to see G1 behavior
        List<byte[]> objects = new ArrayList<>();
        
        // Create objects with different allocation patterns
        for (int i = 0; i < 100000; i++) {
            byte[] obj;
            
            if (i % 3 == 0) {
                // Small objects
                obj = new byte[1024];
            } else if (i % 3 == 1) {
                // Medium objects
                obj = new byte[1024 * 16];
            } else {
                // Large objects
                obj = new byte[1024 * 128]; // 128KB
            }
            
            objects.add(obj);
            
            if (i % 10000 == 0) {
                System.out.println("Created " + i + " objects");
                // G1 will perform mixed collections periodically
            }
        }
        
        // Clear some objects to create garbage
        for (int i = objects.size() - 1; i >= objects.size() / 2; i--) {
            objects.remove(i);
        }
        
        // G1 will collect garbage regions
        System.gc();
    }
}
```

### ZGC (Z Garbage Collector)

ZGC is designed for very large heaps with ultra-low pause times.

```java
// JVM startup parameters for ZGC (JDK 11+):
// -XX:+UnlockExperimentalVMOptions -XX:+UseZGC (JDK 11-14)
// -XX:+UseZGC (JDK 15+)

public class ZGCDemo {
    
    // ZGC characteristics:
    // - Ultra-low pause times (< 10ms)
    // - Supports very large heaps (TBs)
    // - Concurrent collection of all phases
    // - Designed for applications requiring sub-millisecond pause times
    
    public static void demonstrateZGC() {
        // ZGC can handle very large heaps with minimal pause times
        // It uses colored pointers and load barriers
        
        System.out.println("ZGC demonstration - requires JDK 11+ with ZGC enabled");
        System.out.println("Run with: -XX:+UseZGC");
        
        // ZGC handles memory management differently:
        // - All phases are concurrent
        // - No stop-the-world compacting phase
        // - Uses multiple memory views
        
        // Example of memory-intensive operation that ZGC handles well
        List<byte[]> largeObjects = new ArrayList<>();
        
        for (int i = 0; i < 10000; i++) {
            // Create large objects that would normally cause long pauses
            byte[] largeObj = new byte[1024 * 1024]; // 1MB each
            largeObjects.add(largeObj);
            
            if (i % 1000 == 0) {
                System.out.println("Created " + i + " large objects");
            }
        }
        
        // ZGC can handle this large allocation with minimal pause times
        largeObjects.clear();
        System.gc();
    }
}
```

## Memory Pools and Generational Model

The generational hypothesis suggests that most objects die young, which forms the basis for generational garbage collection.

### Young Generation Structure

```java
// Understanding Young Generation structure
public class YoungGenerationStructure {
    
    // Young Generation is divided into:
    // - Eden Space: Where new objects are allocated
    // - Survivor Space 0: Temporary storage for surviving objects
    // - Survivor Space 1: Another temporary storage, acts as active survivor space
    
    private static final int EDEN_SIZE = 64 * 1024 * 1024; // 64MB
    private static final int SURVIVOR_SIZE = 8 * 1024 * 1024; // 8MB each
    private static final int MAX_TENURING_THRESHOLD = 15;
    
    // Simulate object allocation and promotion in Young Generation
    public static void simulateYoungGenBehavior() {
        List<List<byte[]>> survivorGenerations = new ArrayList<>();
        
        for (int generation = 0; generation < MAX_TENURING_THRESHOLD; generation++) {
            List<byte[]> survivingObjects = new ArrayList<>();
            
            // Create objects that survive this generation
            for (int i = 0; i < 100; i++) {
                byte[] obj = new byte[1024]; // Small object
                survivingObjects.add(obj);
            }
            
            survivorGenerations.add(survivingObjects);
            
            // Simulate objects dying at each generation
            List<byte[]> dyingObjects = new ArrayList<>();
            for (int i = 0; i < 1000; i++) {
                byte[] obj = new byte[512]; // Another small object that dies
                dyingObjects.add(obj);
            }
            // These objects go out of scope and become eligible for GC
            
            System.out.println("Generation " + generation + ": " + 
                             survivingObjects.size() + " objects survived");
        }
        
        // Objects that survive all young generation cycles are promoted to Old Generation
        System.out.println("Objects that survived " + MAX_TENURING_THRESHOLD + 
                          " young GC cycles will be promoted to Old Generation");
    }
    
    // Understanding TLAB (Thread Local Allocation Buffer)
    public static void demonstrateTLAB() {
        // Each thread has its own TLAB for faster allocation
        // Reduces contention during object allocation
        
        ExecutorService executor = Executors.newFixedThreadPool(10);
        CountDownLatch latch = new CountDownLatch(10);
        
        for (int i = 0; i < 10; i++) {
            executor.submit(() -> {
                try {
                    // Each thread allocates in its own TLAB
                    List<byte[]> threadObjects = new ArrayList<>();
                    for (int j = 0; j < 1000; j++) {
                        threadObjects.add(new byte[1024]); // 1KB objects
                    }
                    System.out.println("Thread " + Thread.currentThread().getName() + 
                                     " allocated " + threadObjects.size() + " objects");
                } finally {
                    latch.countDown();
                }
            });
        }
        
        try {
            latch.await();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        executor.shutdown();
    }
    
    public static void main(String[] args) {
        simulateYoungGenBehavior();
        demonstrateTLAB();
    }
}
```

### Old Generation and Tenuring

```java
// Understanding Old Generation and object tenuring
public class OldGenerationManagement {
    
    // Objects are promoted to Old Generation based on:
    // - Age (number of GC cycles survived)
    // - Size (large objects may go directly to Old Gen)
    // - Dynamic aging (JVM optimization)
    
    private static List<byte[]> oldGenObjects = new ArrayList<>();
    
    public static void demonstrateObjectPromotion() {
        // Create objects that will be promoted to Old Generation
        for (int i = 0; i < 100; i++) {
            // These objects will survive multiple Young GCs
            byte[] longLivedObj = new byte[2048]; // 2KB object
            oldGenObjects.add(longLivedObj);
            
            // Force minor GCs to promote these objects
            if (i % 10 == 0) {
                // Create temporary objects to fill Eden and trigger GC
                for (int j = 0; j < 1000; j++) {
                    new byte[1024]; // Temporary objects
                }
                // These temporary objects die young
                // The longLivedObj objects survive and get promoted
            }
        }
        
        System.out.println("Created " + oldGenObjects.size() + " long-lived objects");
        System.out.println("These objects will be promoted to Old Generation after surviving several GC cycles");
    }
    
    // Large object allocation behavior
    public static void demonstrateLargeObjectAllocation() {
        // Large objects may be allocated directly in Old Generation
        // depending on GC algorithm and settings
        
        List<byte[]> largeObjects = new ArrayList<>();
        
        for (int i = 0; i < 50; i++) {
            // Large objects (> TLAB size or > certain threshold)
            byte[] largeObj = new byte[1024 * 128]; // 128KB object
            largeObjects.add(largeObj);
            
            System.out.println("Allocated large object: " + 
                             largeObj.length / 1024 + " KB");
        }
        
        System.out.println("Large objects may be allocated directly in Old Generation");
    }
    
    // Understanding humongous objects in G1
    public static void demonstrateHumongousObjects() {
        // In G1, objects > 50% of region size are humongous
        // They are allocated directly in Old Generation
        
        System.out.println("In G1 GC, objects > 50% of region size are humongous");
        System.out.println("They are allocated directly in Old Generation");
        
        // Assuming default region size of 1MB, objects > 512KB are humongous
        byte[] humongousObj = new byte[1024 * 1024]; // 1MB object = humongous in default G1
        System.out.println("Humongous object allocated: " + humongousObj.length / (1024 * 1024) + " MB");
    }
    
    public static void main(String[] args) {
        demonstrateObjectPromotion();
        demonstrateLargeObjectAllocation();
        demonstrateHumongousObjects();
    }
}
```

## Garbage Collection Types and Tuning

Proper GC tuning is crucial for application performance optimization.

### GC Tuning Parameters

```java
// GC tuning demonstration
public class GCTuningParameters {
    
    // Heap size tuning
    // -Xms<size> Initial heap size
    // -Xmx<size> Maximum heap size
    // -XX:NewRatio=<n> Ratio of old/young generation
    // -XX:SurvivorRatio=<n> Ratio of Eden/Survivor space
    
    public static void demonstrateHeapSizing() {
        Runtime runtime = Runtime.getRuntime();
        
        long maxMemory = runtime.maxMemory();
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        
        System.out.println("Max Memory (Xmx): " + maxMemory / (1024 * 1024) + " MB");
        System.out.println("Total Memory: " + totalMemory / (1024 * 1024) + " MB");
        System.out.println("Used Memory: " + usedMemory / (1024 * 1024) + " MB");
        System.out.println("Free Memory: " + freeMemory / (1024 * 1024) + " MB");
        
        // Calculate heap utilization
        double utilization = (double) usedMemory / maxMemory * 100;
        System.out.println("Heap Utilization: " + String.format("%.2f", utilization) + "%");
    }
    
    // Young generation sizing
    public static void demonstrateYoungGenSizing() {
        // Using JMX to get detailed GC information
        List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            System.out.println("GC Name: " + gcBean.getName());
            System.out.println("Collection Count: " + gcBean.getCollectionCount());
            System.out.println("Collection Time: " + gcBean.getCollectionTime() + " ms");
            
            // Get memory pools associated with this GC
            String[] memoryPoolNames = gcBean.getMemoryPoolNames();
            for (String poolName : memoryPoolNames) {
                System.out.println("  Memory Pool: " + poolName);
            }
        }
    }
    
    // GC-specific tuning parameters
    public static void demonstrateGCTuning() {
        System.out.println("Common GC tuning parameters:");
        System.out.println("-XX:MaxGCPauseMillis=<n> - Target max GC pause time");
        System.out.println("-XX:GCTimeRatio=<n> - GC time ratio (1/(1+n))");
        System.out.println("-XX:+UseAdaptiveSizePolicy - Let JVM adjust sizes automatically");
        System.out.println("-XX:CMSInitiatingOccupancyFraction=<n> - When to start CMS");
        System.out.println("-XX:+UseCMSInitiatingOccupancyOnly - Use fraction consistently");
        System.out.println("-XX:G1HeapRegionSize=<n> - G1 region size");
        System.out.println("-XX:MaxGCPauseMillis=<n> - G1 target pause time");
        System.out.println("-XX:G1NewSizePercent=<n> - Min young gen size percentage");
        System.out.println("-XX:G1MaxNewSizePercent=<n> - Max young gen size percentage");
    }
    
    public static void main(String[] args) {
        demonstrateHeapSizing();
        demonstrateYoungGenSizing();
        demonstrateGCTuning();
    }
}
```

### Monitoring GC Performance

```java
// GC performance monitoring
public class GCPerformanceMonitoring {
    
    private static final ScheduledExecutorService scheduler = 
        Executors.newScheduledThreadPool(2);
    
    public static void monitorGCPerformance() {
        // Schedule GC statistics collection
        scheduler.scheduleAtFixedRate(() -> {
            collectGCStatistics();
        }, 0, 5, TimeUnit.SECONDS);
        
        // Schedule memory usage monitoring
        scheduler.scheduleAtFixedRate(() -> {
            collectMemoryStatistics();
        }, 0, 1, TimeUnit.SECONDS);
    }
    
    private static void collectGCStatistics() {
        List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            long collections = gcBean.getCollectionCount();
            long time = gcBean.getCollectionTime();
            
            System.out.println("[" + new Date() + "] " + 
                             gcBean.getName() + 
                             " - Collections: " + collections + 
                             ", Time: " + time + "ms");
        }
    }
    
    private static void collectMemoryStatistics() {
        Runtime runtime = Runtime.getRuntime();
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        MemoryUsage nonHeapUsage = memoryBean.getNonHeapMemoryUsage();
        
        System.out.println("[" + new Date() + "] Heap: " + 
                         "Used: " + heapUsage.getUsed() / (1024 * 1024) + "MB, " +
                         "Committed: " + heapUsage.getCommitted() / (1024 * 1024) + "MB, " +
                         "Max: " + heapUsage.getMax() / (1024 * 1024) + "MB");
        
        System.out.println("[" + new Date() + "] Non-Heap: " + 
                         "Used: " + nonHeapUsage.getUsed() / (1024 * 1024) + "MB, " +
                         "Committed: " + nonHeapUsage.getCommitted() / (1024 * 1024) + "MB");
    }
    
    // Detailed memory pool monitoring
    public static void monitorMemoryPools() {
        List<MemoryPoolMXBean> memoryPools = ManagementFactory.getMemoryPoolMXBeans();
        
        for (MemoryPoolMXBean pool : memoryPools) {
            String poolName = pool.getName();
            MemoryUsage usage = pool.getUsage();
            
            System.out.println("Pool: " + poolName);
            System.out.println("  Usage: " + usage.getUsed() / (1024 * 1024) + "MB / " + 
                             usage.getMax() / (1024 * 1024) + "MB");
            
            // Check if this is a garbage-collected memory pool
            if (pool.getPeakUsage().getUsed() > 0) {
                System.out.println("  Peak Usage: " + pool.getPeakUsage().getUsed() / (1024 * 1024) + "MB");
            }
        }
    }
    
    public static void main(String[] args) {
        System.out.println("Starting GC Performance Monitoring...");
        
        // Start monitoring
        monitorGCPerformance();
        
        // Create memory pressure to see GC in action
        List<byte[]> objects = new ArrayList<>();
        try {
            // This will create memory pressure and trigger GC
            for (int i = 0; i < 100000; i++) {
                objects.add(new byte[1024 * 10]); // 10KB objects
                if (i % 10000 == 0) {
                    System.out.println("Created " + i + " objects");
                }
            }
        } catch (OutOfMemoryError e) {
            System.out.println("Out of memory reached - this is expected in this demo");
        }
        
        // Monitor memory pools
        monitorMemoryPools();
        
        // Clean up
        objects.clear();
        System.gc();
        
        // Give monitoring time to collect data
        try {
            Thread.sleep(30000); // 30 seconds to see GC patterns
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        scheduler.shutdown();
    }
}
```

## Memory Profiling and Analysis

Advanced memory profiling techniques help identify performance bottlenecks and memory issues.

### Using JFR (Java Flight Recorder)

```java
// JFR profiling configuration
public class JFRMemoryProfiling {
    
    // JFR command-line usage:
    // java -XX:+FlightRecorder -XX:StartFlightRecording=duration=60s,filename=app-profile.jfr MyApp
    // Or programmatic configuration:
    
    public static void configureFlightRecorder() {
        try {
            JFR jfr = new JFR();
            Recording recording = new Recording();
            
            // Enable memory allocation profiling
            recording.enable("jdk.ObjectAllocationInNewTLAB");
            recording.enable("jdk.ObjectAllocationOutsideTLAB");
            recording.enable("jdk.GCHeapSummary");
            recording.enable("jdk.GCPhasePause");
            recording.enable("jdk.GCPhaseConcurrent");
            
            recording.setName("MemoryProfiling");
            recording.setDuration(Duration.ofMinutes(5));
            
            // Start recording
            recording.start();
            System.out.println("JFR recording started");
            
            // Your application code here
            performMemoryIntensiveOperation();
            
            // Stop and save recording
            recording.stop();
            recording.dump(Paths.get("memory-profile.jfr"));
            System.out.println("JFR recording saved to memory-profile.jfr");
            
        } catch (Exception e) {
            System.err.println("JFR configuration failed: " + e.getMessage());
        }
    }
    
    private static void performMemoryIntensiveOperation() {
        // Simulate memory-intensive operation
        List<byte[]> largeObjects = new ArrayList<>();
        
        for (int i = 0; i < 10000; i++) {
            // Create objects of varying sizes
            int size = 1024 + (i % 100) * 1024; // 1KB to 100KB
            largeObjects.add(new byte[size]);
            
            if (i % 1000 == 0) {
                System.out.println("Created " + i + " objects for profiling");
            }
        }
        
        // Simulate object retention and release
        for (int i = largeObjects.size() - 1; i >= largeObjects.size() / 2; i--) {
            largeObjects.remove(i); // Remove half the objects
        }
        
        System.gc(); // Force GC to see memory patterns
    }
    
    public static void main(String[] args) {
        configureFlightRecorder();
    }
}

// Custom JFR event for application-specific memory tracking
@Name("com.example.HeapUsage")
@Description("Custom heap usage tracking")
@Label("Custom Heap Usage")
public class CustomHeapUsageEvent extends Event {
    
    @Label("Used Memory")
    @Description("Amount of used heap memory in bytes")
    public long usedMemory;
    
    @Label("Max Memory") 
    @Description("Maximum heap memory in bytes")
    public long maxMemory;
    
    @Label("Application State")
    @Description("Current application state")
    public String appState;
    
    public CustomHeapUsageEvent() {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        this.usedMemory = heapUsage.getUsed();
        this.maxMemory = heapUsage.getMax();
    }
}

// Using custom event in application
public class ApplicationWithCustomEvents {
    
    public static void performOperationWithEvents() {
        // Start of critical operation
        CustomHeapUsageEvent startEvent = new CustomHeapUsageEvent();
        startEvent.appState = "START_OPERATION";
        startEvent.commit();
        
        // Perform operation
        List<byte[]> objects = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            objects.add(new byte[1024]);
        }
        
        // Mid-operation event
        CustomHeapUsageEvent midEvent = new CustomHeapUsageEvent();
        midEvent.appState = "MID_OPERATION";
        midEvent.commit();
        
        // Clean up
        objects.clear();
        System.gc();
        
        // End of operation
        CustomHeapUsageEvent endEvent = new CustomHeapUsageEvent();
        endEvent.appState = "END_OPERATION";
        endEvent.commit();
    }
}
```

### Memory Analyzer with OQL (Object Query Language)

```java
// Memory analysis utilities
public class MemoryAnalysisUtils {
    
    // Simulate creating different types of memory usage patterns
    private static Map<String, List<byte[]>> categorizedMemoryUsage = new ConcurrentHashMap<>();
    
    public static void createMemoryPatterns() {
        // Create different memory usage patterns for analysis
        
        // Short-lived objects
        List<byte[]> shortLived = new ArrayList<>();
        for (int i = 0; i < 5000; i++) {
            shortLived.add(new byte[512]); // 512B objects
        }
        shortLived.clear(); // These become eligible for GC immediately
        
        // Medium-lived objects
        List<byte[]> mediumLived = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            mediumLived.add(new byte[2048]); // 2KB objects
        }
        categorizedMemoryUsage.put("medium-lived", mediumLived);
        
        // Long-lived objects
        List<byte[]> longLived = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            longLived.add(new byte[1024 * 64]); // 64KB objects
        }
        categorizedMemoryUsage.put("long-lived", longLived);
        
        // Large objects
        List<byte[]> largeObjects = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            largeObjects.add(new byte[1024 * 1024]); // 1MB objects
        }
        categorizedMemoryUsage.put("large-objects", largeObjects);
        
        System.out.println("Created memory patterns:");
        System.out.println("- Short-lived: 5000 objects (cleared immediately)");
        System.out.println("- Medium-lived: 1000 objects (2KB each)");
        System.out.println("- Long-lived: 100 objects (64KB each)");
        System.out.println("- Large objects: 10 objects (1MB each)");
    }
    
    // Memory usage statistics
    public static void printMemoryStatistics() {
        Runtime runtime = Runtime.getRuntime();
        
        long maxMemory = runtime.maxMemory();
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        
        System.out.println("Memory Statistics:");
        System.out.println("  Max Memory: " + maxMemory / (1024 * 1024) + " MB");
        System.out.println("  Total Memory: " + totalMemory / (1024 * 1024) + " MB");
        System.out.println("  Used Memory: " + usedMemory / (1024 * 1024) + " MB");
        System.out.println("  Free Memory: " + freeMemory / (1024 * 1024) + " MB");
        System.out.println("  Utilization: " + (usedMemory * 100 / maxMemory) + "%");
        
        // Calculate categorized memory usage
        long categorizedUsage = 0;
        for (Map.Entry<String, List<byte[]>> entry : categorizedMemoryUsage.entrySet()) {
            long categorySize = entry.getValue().stream()
                .mapToLong(arr -> arr.length)
                .sum();
            categorizedUsage += categorySize;
            System.out.println("  " + entry.getKey() + ": " + 
                             categorySize / (1024 * 1024) + " MB");
        }
        
        System.out.println("  Categorized Total: " + categorizedUsage / (1024 * 1024) + " MB");
    }
    
    // Simulate memory leak for analysis
    public static class MemoryLeakSimulator {
        private static final List<byte[]> LEAKED_OBJECTS = new ArrayList<>();
        
        public static void createMemoryLeak() {
            // This creates a memory leak by holding references to objects
            // that should be garbage collected
            for (int i = 0; i < 100; i++) {
                LEAKED_OBJECTS.add(new byte[1024 * 1024]); // 1MB objects
            }
            System.out.println("Created potential memory leak with " + 
                             LEAKED_OBJECTS.size() + " objects");
        }
        
        public static void clearMemoryLeak() {
            LEAKED_OBJECTS.clear();
            System.out.println("Cleared memory leak");
        }
    }
    
    public static void main(String[] args) {
        createMemoryPatterns();
        printMemoryStatistics();
        
        // Simulate memory leak condition
        MemoryLeakSimulator.createMemoryLeak();
        printMemoryStatistics();
        
        // Clear the leak and see the difference
        MemoryLeakSimulator.clearMemoryLeak();
        printMemoryStatistics();
        
        // Force GC to see the effect
        System.gc();
        System.out.println("After System.gc():");
        printMemoryStatistics();
    }
}
```

## Performance Optimization Techniques

Advanced techniques for optimizing JVM memory performance.

### Object Pooling

```java
// Object pooling for performance optimization
public class ObjectPoolingDemo {
    
    // Pool for reusable objects
    public static class ByteBufferPool {
        private static final int POOL_SIZE = 100;
        private static final int BUFFER_SIZE = 1024 * 8; // 8KB
        private final Queue<byte[]> pool = new ConcurrentLinkedQueue<>();
        
        public ByteBufferPool() {
            // Pre-allocate pool objects
            for (int i = 0; i < POOL_SIZE; i++) {
                pool.offer(new byte[BUFFER_SIZE]);
            }
        }
        
        public byte[] acquire() {
            byte[] buffer = pool.poll();
            if (buffer == null) {
                // Pool exhausted, create new buffer
                // This should be rare with proper sizing
                buffer = new byte[BUFFER_SIZE];
                System.out.println("Pool exhausted, creating new buffer");
            }
            return buffer;
        }
        
        public void release(byte[] buffer) {
            if (buffer.length == BUFFER_SIZE && pool.size() < POOL_SIZE) {
                // Clear buffer if needed
                Arrays.fill(buffer, (byte) 0);
                pool.offer(buffer);
            }
            // If pool is full or wrong size, let it be garbage collected
        }
        
        public int getPoolSize() {
            return pool.size();
        }
    }
    
    // Thread-local pooling for even better performance
    public static class ThreadLocalBufferPool {
        private static final int BUFFER_SIZE = 1024 * 16; // 16KB
        private static final ThreadLocal<byte[]> threadLocalBuffer = 
            ThreadLocal.withInitial(() -> new byte[BUFFER_SIZE]);
        
        public static byte[] getBuffer() {
            return threadLocalBuffer.get();
        }
        
        // No explicit release needed - tied to thread lifecycle
    }
    
    // Demonstrate the performance difference
    public static void comparePoolingApproaches() {
        ByteBufferPool pool = new ByteBufferPool();
        
        int iterations = 100000;
        
        // Without pooling - creates new objects
        long startTime = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            byte[] buffer = new byte[8192]; // 8KB buffer
            // Use buffer...
            // Buffer becomes eligible for GC immediately
        }
        long withoutPoolingTime = System.nanoTime() - startTime;
        
        // With pooling
        startTime = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            byte[] buffer = pool.acquire();
            // Use buffer...
            // Return buffer to pool instead of letting it become garbage
            pool.release(buffer);
        }
        long withPoolingTime = System.nanoTime() - startTime;
        
        System.out.println("Without pooling: " + withoutPoolingTime / 1_000_000 + " ms");
        System.out.println("With pooling: " + withPoolingTime / 1_000_000 + " ms");
        System.out.println("Performance improvement: " + 
                         (double) withoutPoolingTime / withPoolingTime + "x");
        System.out.println("Pool utilization: " + 
                         (POOL_SIZE - pool.getPoolSize()) + "/" + POOL_SIZE);
    }
    
    public static void main(String[] args) {
        comparePoolingApproaches();
        
        // Demonstrate thread-local approach
        ExecutorService executor = Executors.newFixedThreadPool(10);
        
        for (int i = 0; i < 10; i++) {
            executor.submit(() -> {
                byte[] buffer = ThreadLocalBufferPool.getBuffer();
                // Use buffer in thread-local context
                System.out.println(Thread.currentThread().getName() + 
                                 " using thread-local buffer of size: " + buffer.length);
            });
        }
        
        executor.shutdown();
    }
}
```

### String Interning and Memory Optimization

```java
// String optimization techniques
public class StringOptimizationDemo {
    
    // String interning for memory efficiency
    private static final Map<String, String> internedStrings = new ConcurrentHashMap<>();
    
    public static String customIntern(String str) {
        return internedStrings.computeIfAbsent(str, s -> s);
    }
    
    // String builder optimization
    public static class OptimizedStringBuilder {
        private static final ThreadLocal<StringBuilder> threadLocalBuilder = 
            ThreadLocal.withInitial(() -> new StringBuilder(256));
        
        public static String buildString(String... parts) {
            StringBuilder builder = threadLocalBuilder.get();
            builder.setLength(0); // Clear without creating new object
            
            for (String part : parts) {
                builder.append(part);
            }
            
            return builder.toString();
        }
    }
    
    // Memory-efficient string concatenation
    public static String efficientConcatenation(List<String> strings) {
        if (strings.isEmpty()) return "";
        
        // Use StringJoiner for efficiency
        StringJoiner joiner = new StringJoiner(",");
        for (String str : strings) {
            joiner.add(str);
        }
        return joiner.toString();
    }
    
    // Demonstrate string memory optimization
    public static void demonstrateStringOptimization() {
        int iterations = 100000;
        
        // Without optimization - creates many String objects
        long startTime = System.nanoTime();
        List<String> stringList1 = new ArrayList<>();
        for (int i = 0; i < iterations; i++) {
            String key = "key" + i; // Creates new String each time
            stringList1.add(key);
        }
        long withoutOptimization = System.nanoTime() - startTime;
        
        // With interning - reuses String objects
        startTime = System.nanoTime();
        List<String> stringList2 = new ArrayList<>();
        for (int i = 0; i < iterations; i++) {
            String key = customIntern("key" + i); // Reuses interned String
            stringList2.add(key);
        }
        long withInterning = System.nanoTime() - startTime;
        
        System.out.println("Without optimization: " + withoutOptimization / 1_000_000 + " ms");
        System.out.println("With interning: " + withInterning / 1_000_000 + " ms");
        
        // Demonstrate StringBuilder optimization
        List<String> parts = Arrays.asList("Hello", " ", "World", "!");
        String result = OptimizedStringBuilder.buildString(parts.toArray(new String[0]));
        System.out.println("Built string: " + result);
        
        // Memory-efficient concatenation
        List<String> sampleStrings = Arrays.asList("apple", "banana", "cherry", "date");
        String concatenated = efficientConcatenation(sampleStrings);
        System.out.println("Concatenated: " + concatenated);
    }
    
    // String deduplication utility
    public static class StringDeduplicator {
        private final Map<String, String> deduplicated = new ConcurrentHashMap<>();
        
        public String deduplicate(String input) {
            // For frequently repeated strings, return the same reference
            return deduplicated.computeIfAbsent(input, s -> s);
        }
        
        public void clearCache() {
            deduplicated.clear();
        }
        
        public int getCacheSize() {
            return deduplicated.size();
        }
    }
    
    public static void main(String[] args) {
        demonstrateStringOptimization();
        
        StringDeduplicator deduplicator = new StringDeduplicator();
        
        // Test deduplication
        String str1 = "Hello World";
        String str2 = "Hello World";
        String dedup1 = deduplicator.deduplicate(str1);
        String dedup2 = deduplicator.deduplicate(str2);
        
        System.out.println("Original strings equal: " + (str1 == str2)); // May be true due to string interning
        System.out.println("Deduplicated strings equal: " + (dedup1 == dedup2)); // Should be true
        System.out.println("Deduplication cache size: " + deduplicator.getCacheSize());
    }
}
```

## Memory Leaks Detection and Prevention

Memory leaks can severely impact application performance and stability.

### Common Memory Leak Patterns

```java
// Memory leak detection and prevention
public class MemoryLeakDetection {
    
    // Example 1: Static collection holding object references
    public static class StaticCollectionLeak {
        // This static list holds references to objects, preventing GC
        private static final List<Object> LEAKED_REFERENCES = new ArrayList<>();
        
        public static void addToLeak(Object obj) {
            LEAKED_REFERENCES.add(obj);
        }
        
        public static void clearLeak() {
            LEAKED_REFERENCES.clear();
        }
        
        public static int getLeakSize() {
            return LEAKED_REFERENCES.size();
        }
    }
    
    // Example 2: Inner class holding outer class reference
    public static class OuterClass {
        private final byte[] largeData = new byte[1024 * 1024]; // 1MB data
        
        public class NonStaticInnerClass {
            // This inner class holds a reference to OuterClass
            // If the inner class object outlives the outer class,
            // it prevents the outer class from being GC'd
            public void doSomething() {
                // Access outer class members
                System.out.println("Inner class accessing outer data: " + largeData.length);
            }
        }
        
        // Solution: Use static inner class or WeakReference
        public static class StaticInnerClass {
            private final WeakReference<OuterClass> outerRef;
            
            public StaticInnerClass(OuterClass outer) {
                this.outerRef = new WeakReference<>(outer);
            }
            
            public void doSomething() {
                OuterClass outer = outerRef.get();
                if (outer != null) {
                    System.out.println("Static inner accessing outer data: " + outer.largeData.length);
                } else {
                    System.out.println("Outer class has been garbage collected");
                }
            }
        }
    }
    
    // Example 3: Listener/Observer leaks
    public static class EventPublisher {
        // This set can hold references to listeners indefinitely
        private final Set<EventListener> listeners = ConcurrentHashMap.newKeySet();
        
        public void addListener(EventListener listener) {
            listeners.add(listener);
        }
        
        public void removeListener(EventListener listener) {
            listeners.remove(listener);
        }
        
        public void publishEvent(Object event) {
            for (EventListener listener : listeners) {
                listener.onEvent(event);
            }
        }
        
        // Proper cleanup method
        public void shutdown() {
            listeners.clear();
        }
    }
    
    public interface EventListener {
        void onEvent(Object event);
    }
    
    // Example 4: ThreadLocal leaks
    public static class ThreadLocalLeakDemo {
        // This ThreadLocal can cause memory leaks if not properly cleaned up
        private static final ThreadLocal<byte[]> THREAD_LOCAL_DATA = new ThreadLocal<byte[]>() {
            @Override
            protected byte[] initialValue() {
                return new byte[1024 * 64]; // 64KB per thread
            }
        };
        
        // Proper usage with cleanup
        public static void safeThreadLocalUsage() {
            try {
                byte[] data = THREAD_LOCAL_DATA.get();
                // Use the data...
                System.out.println("Using thread-local data: " + data.length + " bytes");
            } finally {
                // Always clean up ThreadLocal to prevent leaks in thread pools
                THREAD_LOCAL_DATA.remove();
            }
        }
        
        // Unsafe usage that can cause leaks
        public static void unsafeThreadLocalUsage() {
            byte[] data = THREAD_LOCAL_DATA.get();
            // If this method doesn't call remove(), 
            // the thread will hold onto the data indefinitely
            System.out.println("Using thread-local data (unsafe): " + data.length + " bytes");
            // Missing THREAD_LOCAL_DATA.remove() - potential leak!
        }
    }
    
    // Memory leak detection utility
    public static class MemoryLeakDetector {
        private final Map<String, Object> watchedObjects = new HashMap<>();
        private final Map<String, Long> allocationTimes = new HashMap<>();
        
        public void watchObject(String key, Object obj) {
            watchedObjects.put(key, obj);
            allocationTimes.put(key, System.currentTimeMillis());
        }
        
        public void unwatchObject(String key) {
            watchedObjects.remove(key);
            allocationTimes.remove(key);
        }
        
        public void checkForLeaks(long maxAgeMillis) {
            long now = System.currentTimeMillis();
            
            for (Map.Entry<String, Object> entry : watchedObjects.entrySet()) {
                String key = entry.getKey();
                Long allocTime = allocationTimes.get(key);
                
                if (allocTime != null && (now - allocTime) > maxAgeMillis) {
                    System.out.println("Potential memory leak detected: " + key + 
                                     " has been alive for " + (now - allocTime) + "ms");
                }
            }
        }
    }
    
    public static void main(String[] args) {
        System.out.println("Demonstrating memory leak patterns...");
        
        // 1. Static collection leak
        for (int i = 0; i < 100; i++) {
            StaticCollectionLeak.addToLeak(new byte[1024 * 10]); // 10KB objects
        }
        System.out.println("Static collection leak size: " + StaticCollectionLeak.getLeakSize());
        StaticCollectionLeak.clearLeak();
        
        // 2. Inner class example
        OuterClass outer = new OuterClass();
        OuterClass.StaticInnerClass inner = outer.new StaticInnerClass(outer);
        inner.doSomething();
        
        // 3. ThreadLocal usage
        ThreadLocalLeakDemo.safeThreadLocalUsage();
        ThreadLocalLeakDemo.unsafeThreadLocalUsage();
        
        // 4. Leak detection
        MemoryLeakDetector detector = new MemoryLeakDetector();
        detector.watchObject("test-object", new Object());
        
        // Simulate some processing time
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        detector.checkForLeaks(500); // Check for objects older than 500ms
        detector.unwatchObject("test-object");
        
        System.out.println("Memory leak detection completed");
    }
}
```

### Advanced Memory Leak Prevention

```java
// Advanced memory leak prevention techniques
public class AdvancedMemoryLeakPrevention {
    
    // PhantomReference for cleanup notification
    public static class ResourceCleanupManager {
        private static final ReferenceQueue<Object> referenceQueue = new ReferenceQueue<>();
        private static final Set<PhantomReference<Object>> phantomRefs = ConcurrentHashMap.newKeySet();
        
        static {
            // Start cleanup thread
            Thread cleanupThread = new Thread(() -> {
                while (!Thread.currentThread().isInterrupted()) {
                    try {
                        PhantomReference<?> ref = (PhantomReference<?>) referenceQueue.remove(1000);
                        if (ref != null) {
                            System.out.println("Object has been garbage collected, performing cleanup");
                            phantomRefs.remove(ref);
                            // Perform cleanup actions here
                        }
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            }, "ResourceCleanupThread");
            
            cleanupThread.setDaemon(true);
            cleanupThread.start();
        }
        
        public static void registerCleanup(Object obj, Runnable cleanupAction) {
            PhantomReference<Object> phantomRef = new PhantomReference<>(obj, referenceQueue) {
                @Override
                protected void finalize() {
                    cleanupAction.run();
                }
            };
            phantomRefs.add(phantomRef);
        }
    }
    
    // WeakHashMap for cache without strong references
    public static class WeakCache<K, V> {
        private final Map<K, V> cache = new WeakHashMap<>();
        
        public V get(K key) {
            return cache.get(key);
        }
        
        public void put(K key, V value) {
            cache.put(key, value);
        }
        
        public void clear() {
            cache.clear();
        }
        
        public int size() {
            return cache.size();
        }
    }
    
    // SoftReference for memory-sensitive caching
    public static class SoftCache<K, V> {
        private final Map<K, SoftReference<V>> cache = new ConcurrentHashMap<>();
        
        public V get(K key) {
            SoftReference<V> ref = cache.get(key);
            if (ref != null) {
                V value = ref.get();
                if (value == null) {
                    // Value was garbage collected due to memory pressure
                    cache.remove(key);
                }
                return value;
            }
            return null;
        }
        
        public void put(K key, V value) {
            cache.put(key, new SoftReference<>(value));
        }
        
        public void clear() {
            cache.clear();
        }
    }
    
    // Example of proper resource management
    public static class ResourceManager implements AutoCloseable {
        private final List<AutoCloseable> resources = new ArrayList<>();
        private final String resourceId;
        
        public ResourceManager(String id) {
            this.resourceId = id;
        }
        
        public <T extends AutoCloseable> T manage(T resource) {
            resources.add(resource);
            return resource;
        }
        
        @Override
        public void close() {
            // Close resources in reverse order
            for (int i = resources.size() - 1; i >= 0; i--) {
                try {
                    resources.get(i).close();
                } catch (Exception e) {
                    System.err.println("Error closing resource: " + e.getMessage());
                }
            }
            resources.clear();
            System.out.println("Resource manager " + resourceId + " closed");
        }
        
        // Finalizer as backup (not recommended, just for demonstration)
        @Override
        protected void finalize() throws Throwable {
            if (!resources.isEmpty()) {
                System.err.println("Resource manager " + resourceId + " not properly closed!");
                close();
            }
            super.finalize();
        }
    }
    
    // Demonstration of resource management
    public static void demonstrateResourceManagement() {
        // Using try-with-resources for automatic cleanup
        try (ResourceManager rm = new ResourceManager("demo")) {
            // Manage various resources
            FileInputStream fileStream = rm.manage(new FileInputStream("example.txt"));
            Connection dbConnection = rm.manage(DriverManager.getConnection("jdbc:h2:mem:test"));
            Thread workerThread = rm.manage(new Thread(() -> {
                System.out.println("Worker thread running");
            }));
            
            // All resources will be automatically closed when exiting try block
            workerThread.start();
            workerThread.join();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
    public static void main(String[] args) {
        System.out.println("Advanced memory leak prevention techniques");
        
        // Demonstrate WeakCache
        WeakCache<String, byte[]> weakCache = new WeakCache<>();
        for (int i = 0; i < 100; i++) {
            weakCache.put("key" + i, new byte[1024]); // 1KB objects
        }
        System.out.println("Weak cache size: " + weakCache.size());
        
        // Force GC and check size
        System.gc();
        System.out.println("After GC, weak cache size: " + weakCache.size());
        
        // Demonstrate SoftCache
        SoftCache<String, byte[]> softCache = new SoftCache<>();
        for (int i = 0; i < 50; i++) {
            softCache.put("soft-key" + i, new byte[1024 * 10]); // 10KB objects
        }
        System.out.println("Soft cache size: " + softCache.size());
        
        // Demonstrate resource management
        demonstrateResourceManagement();
        
        System.out.println("Advanced memory leak prevention completed");
    }
}
```

## JVM Monitoring and Metrics

Comprehensive JVM monitoring is essential for performance optimization.

### JVM Metrics Collection

```java
// Comprehensive JVM metrics collection
public class JVMMetricsCollector {
    
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
    private final List<JVMStatSnapshot> historicalStats = new ArrayList<>();
    
    public static class JVMStatSnapshot {
        private final long timestamp;
        private final long heapUsed;
        private final long heapMax;
        private final long nonHeapUsed;
        private final long nonHeapMax;
        private final int gcCount;
        private final long gcTime;
        private final double cpuUsage;
        
        public JVMStatSnapshot() {
            this.timestamp = System.currentTimeMillis();
            
            MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
            MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
            MemoryUsage nonHeapUsage = memoryBean.getNonHeapMemoryUsage();
            
            this.heapUsed = heapUsage.getUsed();
            this.heapMax = heapUsage.getMax();
            this.nonHeapUsed = nonHeapUsage.getUsed();
            this.nonHeapMax = nonHeapUsage.getMax();
            
            List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
            this.gcCount = (int) gcBeans.stream().mapToLong(GarbageCollectorMXBean::getCollectionCount).sum();
            this.gcTime = gcBeans.stream().mapToLong(GarbageCollectorMXBean::getCollectionTime).sum();
            
            // CPU usage calculation (requires additional metrics)
            this.cpuUsage = calculateCPUUsage();
        }
        
        private double calculateCPUUsage() {
            // Simplified CPU usage calculation
            // In practice, you'd need to track process CPU time over intervals
            return 0.0; // Placeholder
        }
        
        // Getters
        public long getTimestamp() { return timestamp; }
        public long getHeapUsed() { return heapUsed; }
        public long getHeapMax() { return heapMax; }
        public long getNonHeapUsed() { return nonHeapUsed; }
        public long getNonHeapMax() { return nonHeapMax; }
        public int getGcCount() { return gcCount; }
        public long getGcTime() { return gcTime; }
        public double getCpuUsage() { return cpuUsage; }
        
        public double getHeapUtilization() {
            return heapMax > 0 ? (double) heapUsed / heapMax * 100 : 0;
        }
    }
    
    public void startMonitoring() {
        // Collect basic metrics every second
        scheduler.scheduleAtFixedRate(this::collectBasicMetrics, 0, 1, TimeUnit.SECONDS);
        
        // Collect detailed metrics every 10 seconds
        scheduler.scheduleAtFixedRate(this::collectDetailedMetrics, 0, 10, TimeUnit.SECONDS);
    }
    
    private void collectBasicMetrics() {
        JVMStatSnapshot snapshot = new JVMStatSnapshot();
        historicalStats.add(snapshot);
        
        // Keep only last 1000 snapshots
        if (historicalStats.size() > 1000) {
            historicalStats.subList(0, historicalStats.size() - 1000).clear();
        }
        
        // Log basic metrics
        System.out.printf("[%tc] Heap: %.2f%% (%d/%d MB), GC: %d collections%n",
            new Date(snapshot.getTimestamp()),
            snapshot.getHeapUtilization(),
            snapshot.getHeapUsed() / (1024 * 1024),
            snapshot.getHeapMax() / (1024 * 1024),
            snapshot.getGcCount());
    }
    
    private void collectDetailedMetrics() {
        // Collect detailed memory pool information
        List<MemoryPoolMXBean> memoryPools = ManagementFactory.getMemoryPoolMXBeans();
        
        System.out.println("\n--- Detailed Memory Pool Information ---");
        for (MemoryPoolMXBean pool : memoryPools) {
            MemoryUsage usage = pool.getUsage();
            System.out.printf("Pool: %-30s Used: %6d MB, Max: %6d MB%n",
                pool.getName(),
                usage.getUsed() / (1024 * 1024),
                usage.getMax() / (1024 * 1024));
        }
        
        // Collect GC details
        List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        System.out.println("\n--- GC Details ---");
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            System.out.printf("GC: %-25s Collections: %8d, Time: %8d ms%n",
                gcBean.getName(),
                gcBean.getCollectionCount(),
                gcBean.getCollectionTime());
        }
        
        // Thread information
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        System.out.printf("\n--- Thread Information ---%n");
        System.out.printf("Total threads: %d%n", threadBean.getThreadCount());
        System.out.printf("Peak threads: %d%n", threadBean.getPeakThreadCount());
        System.out.printf("Daemon threads: %d%n", threadBean.getDaemonThreadCount());
    }
    
    public List<JVMStatSnapshot> getHistoricalStats() {
        return new ArrayList<>(historicalStats);
    }
    
    public void stopMonitoring() {
        scheduler.shutdown();
        try {
            if (!scheduler.awaitTermination(5, TimeUnit.SECONDS)) {
                scheduler.shutdownNow();
            }
        } catch (InterruptedException e) {
            scheduler.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
    
    // GC pressure simulation and monitoring
    public void simulateGCMonitoring() {
        System.out.println("\n--- Simulating GC Pressure ---");
        
        List<byte[]> objects = new ArrayList<>();
        
        for (int i = 0; i < 1000; i++) {
            // Create objects to trigger GC
            for (int j = 0; j < 100; j++) {
                objects.add(new byte[1024 * 10]); // 10KB objects
            }
            
            // Remove half to create garbage
            int half = objects.size() / 2;
            for (int j = 0; j < half; j++) {
                objects.remove(objects.size() - 1);
            }
            
            if (i % 100 == 0) {
                System.out.println("Iteration " + i + ", Objects in list: " + objects.size());
                // Force GC to see the pattern
                System.gc();
            }
        }
        
        objects.clear();
        System.gc();
        System.out.println("GC pressure simulation completed");
    }
    
    public static void main(String[] args) {
        JVMMetricsCollector collector = new JVMMetricsCollector();
        
        collector.startMonitoring();
        
        // Run for 30 seconds
        try {
            Thread.sleep(30000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        // Run GC simulation
        collector.simulateGCMonitoring();
        
        // Stop monitoring
        collector.stopMonitoring();
        
        // Analyze collected data
        List<JVMStatSnapshot> stats = collector.getHistoricalStats();
        System.out.println("\n--- Final Analysis ---");
        System.out.println("Total samples collected: " + stats.size());
        
        if (!stats.isEmpty()) {
            JVMStatSnapshot first = stats.get(0);
            JVMStatSnapshot last = stats.get(stats.size() - 1);
            
            System.out.printf("Initial heap usage: %.2f MB%n", 
                            first.getHeapUsed() / (1024.0 * 1024.0));
            System.out.printf("Final heap usage: %.2f MB%n", 
                            last.getHeapUsed() / (1024.0 * 1024.0));
            System.out.printf("GC collections during monitoring: %d%n", 
                            last.getGcCount() - first.getGcCount());
        }
    }
}
```

### Custom JVM Metrics with Micrometer

```java
// Using Micrometer for JVM metrics
public class MicrometerJVMMetrics {
    
    private final MeterRegistry registry;
    private final Timer gcTimer;
    private final Counter allocationCounter;
    private final DistributionSummary heapUsageSummary;
    
    public MicrometerJVMMetrics() {
        // Using a simple in-memory registry for demonstration
        this.registry = new SimpleMeterRegistry();
        
        // Initialize custom metrics
        this.gcTimer = Timer.builder("jvm.gc.pause")
            .description("Time spent in garbage collection")
            .register(registry);
            
        this.allocationCounter = Counter.builder("jvm.memory.allocations")
            .description("Count of memory allocations")
            .register(registry);
            
        this.heapUsageSummary = DistributionSummary.builder("jvm.memory.heap.used")
            .baseUnit("bytes")
            .description("Distribution of heap memory usage")
            .register(registry);
        
        // Register built-in JVM metrics
        JvmMemoryMetrics.register(registry);
        JvmGcMetrics.register(registry);
        ProcessorMetrics.register(registry);
        JvmThreadMetrics.register(registry);
    }
    
    // Simulate memory allocation tracking
    public byte[] allocateMemory(int size) {
        allocationCounter.increment(size);
        byte[] data = new byte[size];
        return data;
    }
    
    // Track GC events (simplified)
    public void recordGCPause(long pauseTimeMs) {
        gcTimer.record(pauseTimeMs, TimeUnit.MILLISECONDS);
    }
    
    // Record heap usage
    public void recordHeapUsage() {
        Runtime runtime = Runtime.getRuntime();
        long used = runtime.totalMemory() - runtime.freeMemory();
        heapUsageSummary.record(used);
    }
    
    // Custom memory pool metrics
    public void registerMemoryPoolMetrics() {
        List<MemoryPoolMXBean> pools = ManagementFactory.getMemoryPoolMXBeans();
        
        for (MemoryPoolMXBean pool : pools) {
            Gauge.builder("jvm.memory.pool.used")
                .description("Used memory in pool")
                .tags("pool", pool.getName())
                .register(registry, pool, 
                    p -> p.getUsage().getUsed());
                    
            Gauge.builder("jvm.memory.pool.max")
                .description("Max memory in pool")
                .tags("pool", pool.getName())
                .register(registry, pool,
                    p -> p.getUsage().getMax() > 0 ? p.getUsage().getMax() : 0);
        }
    }
    
    // Memory pressure simulation with metrics
    public void simulateMemoryPressure() {
        System.out.println("Starting memory pressure simulation with metrics...");
        
        for (int i = 0; i < 100; i++) {
            // Allocate objects
            List<byte[]> objects = new ArrayList<>();
            for (int j = 0; j < 100; j++) {
                byte[] obj = allocateMemory(1024 * 10); // 10KB
                objects.add(obj);
            }
            
            // Record current heap usage
            recordHeapUsage();
            
            // Partial cleanup to create memory pressure
            objects.subList(0, 50).clear();
            
            if (i % 20 == 0) {
                System.out.println("Step " + i + " - Recorded metrics");
            }
        }
        
        System.gc(); // Force final cleanup
    }
    
    // Export metrics in various formats
    public void exportMetrics() {
        System.out.println("\n--- Exported Metrics ---");
        
        // Print all registered metrics
        registry.forEachMeter(meter -> {
            System.out.println(meter.getId().getName() + ":");
            for (Measurement measurement : meter.measure()) {
                System.out.printf("  %s = %f%n", 
                    measurement.getStatistic().toString(), 
                    measurement.getValue());
            }
        });
    }
    
    public static void main(String[] args) {
        MicrometerJVMMetrics metrics = new MicrometerJVMMetrics();
        
        // Register memory pool metrics
        metrics.registerMemoryPoolMetrics();
        
        // Run memory pressure simulation
        metrics.simulateMemoryPressure();
        
        // Export collected metrics
        metrics.exportMetrics();
        
        // Show some specific metrics
        Gauge heapUsed = registry.find("jvm.memory.used").gauge();
        if (heapUsed != null) {
            System.out.printf("Current heap used: %.2f MB%n", 
                            heapUsed.value() / (1024.0 * 1024.0));
        }
        
        Counter allocations = registry.find("jvm.memory.allocations").counter();
        if (allocations != null) {
            System.out.printf("Total allocation events recorded: %.0f%n", 
                            allocations.count());
        }
    }
}
```

## Best Practices and Advanced Topics

### JVM Memory Best Practices

```java
// JVM memory best practices implementation
public class JVMMemoryBestPractices {
    
    // 1. Proper object pooling for frequently created objects
    public static class BestPracticeObjectPool<T> {
        private final Queue<T> pool;
        private final Supplier<T> factory;
        private final Consumer<T> resetter;
        private final int maxSize;
        
        public BestPracticeObjectPool(Supplier<T> factory, Consumer<T> resetter, int maxSize) {
            this.pool = new ConcurrentLinkedQueue<>();
            this.factory = factory;
            this.resetter = resetter;
            this.maxSize = maxSize;
        }
        
        public T acquire() {
            T object = pool.poll();
            return object != null ? object : factory.get();
        }
        
        public void release(T object) {
            if (object != null && pool.size() < maxSize) {
                resetter.accept(object);
                pool.offer(object);
            }
        }
    }
    
    // 2. Efficient collection sizing
    public static class EfficientCollectionUsage {
        
        // Pre-size collections when possible
        public List<String> createOptimizedList(int expectedSize) {
            return new ArrayList<>(calculateOptimalCapacity(expectedSize));
        }
        
        // Calculate optimal capacity to minimize resizing
        private int calculateOptimalCapacity(int expectedSize) {
            // ArrayList resizes by 50% when full, so we can account for that
            if (expectedSize < 10) return expectedSize;
            return (int) (expectedSize * 1.1); // 10% buffer
        }
        
        // Use appropriate collection types
        public void demonstrateCollectionChoices() {
            // Use ArrayList for frequent indexed access
            List<String> arrayList = new ArrayList<>();
            
            // Use LinkedList for frequent insertions/deletions in middle
            List<String> linkedList = new LinkedList<>();
            
            // Use HashMap for fast lookups
            Map<String, Object> hashMap = new HashMap<>();
            
            // Use ConcurrentHashMap for thread-safe operations
            Map<String, Object> concurrentMap = new ConcurrentHashMap<>();
            
            // Use WeakHashMap for caches that don't prevent GC
            Map<String, Object> weakMap = new WeakHashMap<>();
        }
    }
    
    // 3. Memory-efficient data structures
    public static class MemoryEfficientDataStructures {
        
        // Use primitive collections when possible
        public void demonstratePrimitiveCollections() {
            // Instead of List<Integer>, consider using specialized libraries
            // or primitive arrays for better memory efficiency
            
            // For example, if using Eclipse Collections or Apache Commons:
            // IntList intList = new IntArrayList();
            
            // Or use primitive arrays directly
            int[] primitiveArray = new int[1000000]; // ~4MB
            Integer[] objectArray = new Integer[1000000]; // Much more memory due to object overhead
            
            System.out.println("Primitive array memory: " + primitiveArray.length * 4 + " bytes");
            System.out.println("Object array reference memory: " + objectArray.length * 8 + " bytes");
        }
        
        // Use StringBuilder for string concatenation
        public String efficientStringConcatenation(List<String> parts) {
            if (parts.isEmpty()) return "";
            
            StringBuilder sb = new StringBuilder(parts.stream()
                .mapToInt(String::length)
                .sum()); // Pre-size to avoid internal array resizing
                
            for (String part : parts) {
                sb.append(part);
            }
            return sb.toString();
        }
    }
    
    // 4. Proper resource management
    public static class ProperResourceManagement {
        
        // Use try-with-resources for automatic cleanup
        public void demonstrateTryWithResources() {
            try (FileInputStream fis = new FileInputStream("input.txt");
                 BufferedInputStream bis = new BufferedInputStream(fis);
                 DataInputStream dis = new DataInputStream(bis)) {
                
                // Process the stream
                int data = dis.readInt();
                System.out.println("Read data: " + data);
                
            } catch (IOException e) {
                System.err.println("Error processing file: " + e.getMessage());
            }
        }
        
        // Custom resource manager for non-AutoCloseable resources
        public static class CustomResourceManager implements Closeable {
            private final List<Runnable> cleanupTasks = new ArrayList<>();
            
            public <T> T manage(T resource, Runnable cleanupTask) {
                if (resource != null) {
                    cleanupTasks.add(cleanupTask);
                }
                return resource;
            }
            
            @Override
            public void close() {
                // Execute cleanup tasks in reverse order
                for (int i = cleanupTasks.size() - 1; i >= 0; i--) {
                    try {
                        cleanupTasks.get(i).run();
                    } catch (Exception e) {
                        System.err.println("Error during cleanup: " + e.getMessage());
                    }
                }
                cleanupTasks.clear();
            }
        }
    }
    
    // 5. Memory monitoring and alerting
    public static class MemoryMonitoring {
        private static final long MEMORY_THRESHOLD = Runtime.getRuntime().maxMemory() * 80 / 100; // 80% threshold
        private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
        
        public void startMemoryMonitoring() {
            scheduler.scheduleAtFixedRate(this::checkMemoryUsage, 0, 5, TimeUnit.SECONDS);
        }
        
        private void checkMemoryUsage() {
            Runtime runtime = Runtime.getRuntime();
            long usedMemory = runtime.totalMemory() - runtime.freeMemory();
            
            if (usedMemory > MEMORY_THRESHOLD) {
                System.out.println("WARNING: Memory usage high - " + 
                                 (usedMemory * 100 / runtime.maxMemory()) + "%");
                
                // Trigger additional monitoring or alerting
                triggerMemoryAlert(usedMemory);
            }
        }
        
        private void triggerMemoryAlert(long usedMemory) {
            // In a real application, this might:
            // - Send an alert to monitoring system
            // - Log detailed memory information
            // - Trigger diagnostic procedures
            System.out.println("Memory alert triggered at " + 
                             new Date() + ", used: " + usedMemory / (1024 * 1024) + "MB");
        }
        
        public void stopMonitoring() {
            scheduler.shutdown();
        }
    }
    
    // 6. JVM startup optimization
    public static class JVMStartupOptimization {
        
        // Example optimal JVM parameters (as comments)
        /*
         * For throughput applications:
         * -Xms4g -Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=200
         * -XX:G1HeapRegionSize=16m -XX:+UseG1GC
         *
         * For low-latency applications:
         * -Xms2g -Xmx2g -XX:+UseZGC (JDK 15+)
         * Or for older JDKs: -XX:+UseG1GC -XX:MaxGCPauseMillis=10
         *
         * For memory-constrained environments:
         * -Xms1g -Xmx1g -XX:+UseSerialGC
         */
        
        public void demonstrateOptimalConfigurations() {
            System.out.println("JVM Optimization Parameters:");
            System.out.println("Throughput: -XX:+UseParallelGC");
            System.out.println("Latency: -XX:+UseZGC or -XX:MaxGCPauseMillis=10 with G1GC");
            System.out.println("Memory Efficiency: -XX:+UseSerialGC for small heaps");
            System.out.println("Large Heaps: -XX:+UseG1GC with proper sizing");
        }
    }
    
    // 7. Memory profiling and analysis
    public static class MemoryProfiling {
        
        public void runMemoryProfile() {
            // In practice, use tools like:
            // - JProfiler
            // - YourKit
            // - VisualVM
            // - JFR + JMC
            // - Eclipse MAT
            
            System.out.println("Memory profiling should use external tools:");
            System.out.println("- jmap -histo <pid> for object histogram");
            System.out.println("- jstat -gc <pid> for GC statistics");
            System.out.println("- jcmd <pid> GC.run for immediate GC");
            System.out.println("- JFR for detailed profiling");
        }
    }
    
    public static void main(String[] args) {
        System.out.println("JVM Memory Best Practices Demonstration");
        
        // Demonstrate object pooling
        BestPracticeObjectPool<StringBuilder> pool = 
            new BestPracticeObjectPool<>(
                StringBuilder::new,
                sb -> sb.setLength(0), // Reset by clearing content
                10 // Max pool size
            );
        
        // Use pooled objects
        StringBuilder sb = pool.acquire();
        sb.append("Hello, ").append("World!");
        System.out.println("Pooled StringBuilder result: " + sb.toString());
        pool.release(sb); // Return to pool
        
        // Demonstrate efficient collections
        EfficientCollectionUsage efficient = new EfficientCollectionUsage();
        List<String> optimizedList = efficient.createOptimizedList(1000);
        efficient.demonstrateCollectionChoices();
        
        // Demonstrate memory-efficient structures
        MemoryEfficientDataStructures memEfficient = new MemoryEfficientDataStructures();
        memEfficient.demonstratePrimitiveCollections();
        
        // Demonstrate proper resource management
        ProperResourceManagement pm = new ProperResourceManagement();
        pm.demonstrateTryWithResources();
        
        // Start memory monitoring
        MemoryMonitoring memMonitor = new MemoryMonitoring();
        memMonitor.startMemoryMonitoring();
        
        // Show JVM optimization
        JVMStartupOptimization optimization = new JVMStartupOptimization();
        optimization.demonstrateOptimalConfigurations();
        
        // Show profiling info
        MemoryProfiling profiling = new MemoryProfiling();
        profiling.runMemoryProfile();
        
        // Give monitoring time to run
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        memMonitor.stopMonitoring();
        
        System.out.println("JVM Memory Best Practices demonstration completed");
    }
}
```

## Conclusion

Advanced JVM memory management and garbage collection optimization are critical for building high-performance Java applications. Key takeaways include:

1. **Memory Architecture Understanding**: Know the JVM memory regions and their purposes
2. **GC Algorithm Selection**: Choose the right GC algorithm based on application requirements
3. **Proper Configuration**: Tune JVM parameters for optimal performance
4. **Memory Profiling**: Use appropriate tools to identify bottlenecks
5. **Leak Prevention**: Implement proper resource management and cleanup
6. **Performance Optimization**: Apply techniques like object pooling and efficient data structures
7. **Monitoring**: Implement comprehensive monitoring for production environments
8. **Best Practices**: Follow established patterns and avoid common anti-patterns

Understanding these concepts allows developers to build Java applications that are both performant and stable under varying load conditions.