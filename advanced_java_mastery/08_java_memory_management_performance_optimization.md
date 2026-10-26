# Java Memory Management: Deep Dive into Garbage Collection and Performance Optimization

## Introduction

Java memory management is a critical aspect of Java application performance. Unlike languages like C or C++, Java provides automatic memory management through its garbage collector, which handles memory allocation and deallocation automatically. However, understanding how Java manages memory is essential for writing efficient, high-performance applications.

This comprehensive guide will explore Java memory management in depth, covering the JVM memory model, garbage collection mechanisms, memory profiling, and optimization techniques.

## JVM Memory Model

### Memory Areas in JVM

The JVM divides memory into several distinct areas, each serving a specific purpose in the execution of Java applications:

```java
import java.util.*;

public class JVMMemoryModel {
    // Example demonstrating different memory areas
    
    // Method area (PermGen/Metaspace) - stores class metadata
    private static final String CONSTANT_STRING = "Constant Pool String";
    private static final int CONSTANT_INT = 42;
    
    // Heap memory - objects and arrays
    private List<String> heapObjects = new ArrayList<>();
    
    // Instance variables stored in heap with the object
    private int instanceVariable = 100;
    
    public void demonstrateMemoryAreas() {
        // Local variables stored in stack
        int localVariable = 200;
        String localString = "Local String";
        
        // Objects created in heap, references stored in stack
        Object heapObject = new Object();
        List<Integer> heapList = new ArrayList<>();
        
        // Adding objects to demonstrate heap usage
        for (int i = 0; i < 1000; i++) {
            heapObjects.add("Object " + i);
            heapList.add(i);
        }
        
        System.out.println("Local variable: " + localVariable);
        System.out.println("Local string: " + localString);
        System.out.println("Heap objects size: " + heapObjects.size());
        System.out.println("Heap list size: " + heapList.size());
    }
    
    // Native method stack example (for native method calls)
    public native void nativeMethod();
    
    public static void main(String[] args) {
        System.out.println("=== JVM Memory Model Demonstration ===");
        
        JVMMemoryModel demo = new JVMMemoryModel();
        demo.demonstrateMemoryAreas();
        
        // Demonstrate stack memory with recursion
        System.out.println("\nStack depth example:");
        try {
            deepRecursion(1);
        } catch (StackOverflowError e) {
            System.out.println("Stack overflow occurred - this demonstrates stack memory limits");
        }
        
        // Display JVM memory information
        displayMemoryInfo();
    }
    
    private static void deepRecursion(int depth) {
        if (depth > 10000) {
            return;
        }
        deepRecursion(depth + 1);
    }
    
    private static void displayMemoryInfo() {
        Runtime runtime = Runtime.getRuntime();
        
        long maxMemory = runtime.maxMemory();
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        
        System.out.println("\n=== JVM Memory Information ===");
        System.out.println("Max memory: " + maxMemory / (1024 * 1024) + " MB");
        System.out.println("Total memory: " + totalMemory / (1024 * 1024) + " MB");
        System.out.println("Free memory: " + freeMemory / (1024 * 1024) + " MB");
        System.out.println("Used memory: " + usedMemory / (1024 * 1024) + " MB");
        
        // Force garbage collection to see changes
        System.gc();
        try {
            Thread.sleep(1000); // Give GC time to run
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        long freeMemoryAfterGC = runtime.freeMemory();
        long usedMemoryAfterGC = runtime.totalMemory() - freeMemoryAfterGC;
        
        System.out.println("\nAfter GC:");
        System.out.println("Free memory: " + freeMemoryAfterGC / (1024 * 1024) + " MB");
        System.out.println("Used memory: " + usedMemoryAfterGC / (1024 * 1024) + " MB");
    }
}
```

### Detailed Memory Area Breakdown

```java
import java.lang.management.*;
import java.util.*;

public class MemoryAreaDetails {
    public static void main(String[] args) {
        System.out.println("=== Detailed JVM Memory Areas ===");
        
        // Get memory management beans
        List<MemoryPoolMXBean> memoryPools = ManagementFactory.getMemoryPoolMXBeans();
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        
        System.out.println("Available memory pools:");
        for (MemoryPoolMXBean pool : memoryPools) {
            System.out.println("Pool: " + pool.getName());
            System.out.println("  Type: " + pool.getType());
            System.out.println("  Usage: " + formatMemoryUsage(pool.getUsage()));
            System.out.println("  Peak usage: " + formatMemoryUsage(pool.getPeakUsage()));
            if (pool.getCollectionUsage() != null) {
                System.out.println("  Collection usage: " + formatMemoryUsage(pool.getCollectionUsage()));
            }
            System.out.println();
        }
        
        // Heap and non-heap memory
        System.out.println("Heap memory:");
        System.out.println("  Current: " + formatMemoryUsage(memoryBean.getHeapMemoryUsage()));
        
        System.out.println("Non-heap memory:");
        System.out.println("  Current: " + formatMemoryUsage(memoryBean.getNonHeapMemoryUsage()));
        
        // Demonstrate object creation and memory usage
        demonstrateObjectMemoryUsage();
    }
    
    private static String formatMemoryUsage(MemoryUsage usage) {
        if (usage == null) return "N/A";
        return String.format("init=%dMB, used=%dMB, committed=%dMB, max=%dMB",
            usage.getInit() / (1024 * 1024),
            usage.getUsed() / (1024 * 1024),
            usage.getCommitted() / (1024 * 1024),
            usage.getMax() / (1024 * 1024));
    }
    
    private static void demonstrateObjectMemoryUsage() {
        System.out.println("\n=== Object Memory Usage Demonstration ===");
        
        Runtime runtime = Runtime.getRuntime();
        long initialMemory = runtime.totalMemory() - runtime.freeMemory();
        
        // Create objects to observe memory usage
        List<Object> objects = new ArrayList<>();
        for (int i = 0; i < 100000; i++) {
            objects.add(new Object());
        }
        
        long afterCreation = runtime.totalMemory() - runtime.freeMemory();
        System.out.println("Memory used by 100,000 objects: " + 
                          (afterCreation - initialMemory) / 1024 + " KB");
        
        // Clear references to allow garbage collection
        objects.clear();
        System.gc(); // Suggest garbage collection
        
        try {
            Thread.sleep(1000); // Give GC time to run
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        long afterGC = runtime.totalMemory() - runtime.freeMemory();
        System.out.println("Memory after GC: " + 
                          (afterGC - initialMemory) / 1024 + " KB");
    }
}
```

## Garbage Collection Fundamentals

### Types of Garbage Collectors

```java
import java.util.*;
import java.util.concurrent.*;

public class GarbageCollectionTypes {
    public static void main(String[] args) {
        System.out.println("=== Garbage Collection Types ===");
        
        // Display current GC settings
        displayGCSettings();
        
        // Demonstrate different GC behaviors
        demonstrateGCBehavior();
    }
    
    private static void displayGCSettings() {
        System.out.println("Current JVM garbage collector settings:");
        
        List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            System.out.println("GC Name: " + gcBean.getName());
            System.out.println("  Collection count: " + gcBean.getCollectionCount());
            System.out.println("  Collection time: " + gcBean.getCollectionTime() + " ms");
            System.out.println("  Memory pools: " + Arrays.toString(gcBean.getMemoryPoolNames()));
            System.out.println();
        }
        
        // JVM arguments that affect GC
        System.out.println("Common JVM GC arguments:");
        System.out.println("  -XX:+UseSerialGC: Serial garbage collector");
        System.out.println("  -XX:+UseParallelGC: Parallel garbage collector");
        System.out.println("  -XX:+UseG1GC: G1 garbage collector");
        System.out.println("  -XX:+UseZGC: Z garbage collector (Java 11+)");
        System.out.println("  -XX:+UseShenandoahGC: Shenandoah garbage collector (Java 12+)");
    }
    
    private static void demonstrateGCBehavior() {
        System.out.println("\n=== GC Behavior Demonstration ===");
        
        // Create memory pressure to trigger GC
        List<MemoryIntensiveObject> objects = new ArrayList<>();
        
        long startTime = System.currentTimeMillis();
        long initialCollections = getTotalCollectionCount();
        
        // Create objects to fill memory
        try {
            for (int i = 0; i < 1000000; i++) {
                objects.add(new MemoryIntensiveObject());
                
                // Periodically check GC stats
                if (i % 100000 == 0) {
                    long currentCollections = getTotalCollectionCount();
                    System.out.println("Objects created: " + i + 
                                     ", GC collections: " + (currentCollections - initialCollections));
                }
            }
        } catch (OutOfMemoryError e) {
            System.out.println("OutOfMemoryError occurred - GC couldn't keep up");
        }
        
        long endTime = System.currentTimeMillis();
        long finalCollections = getTotalCollectionCount();
        
        System.out.println("Total time: " + (endTime - startTime) + " ms");
        System.out.println("Total GC collections: " + (finalCollections - initialCollections));
        
        // Clear references and trigger GC
        objects.clear();
        System.gc();
        
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        System.out.println("After cleanup, final GC collections: " + 
                          (getTotalCollectionCount() - initialCollections));
    }
    
    private static long getTotalCollectionCount() {
        long total = 0;
        for (GarbageCollectorMXBean gcBean : ManagementFactory.getGarbageCollectorMXBeans()) {
            total += gcBean.getCollectionCount();
        }
        return total;
    }
}

// Memory-intensive object for demonstration
class MemoryIntensiveObject {
    private byte[] data = new byte[1024]; // 1KB per object
    private String id = UUID.randomUUID().toString();
    
    public MemoryIntensiveObject() {
        // Fill data with random values
        for (int i = 0; i < data.length; i++) {
            data[i] = (byte) (Math.random() * 256);
        }
    }
    
    public String getId() {
        return id;
    }
    
    public byte[] getData() {
        return data;
    }
}
```

### Generational Garbage Collection

```java
import java.util.*;

public class GenerationalGC {
    public static void main(String[] args) {
        System.out.println("=== Generational Garbage Collection ===");
        
        // Demonstrate young generation behavior
        demonstrateYoungGeneration();
        
        // Demonstrate old generation behavior
        demonstrateOldGeneration();
        
        // Demonstrate tenuring threshold
        demonstrateTenuring();
    }
    
    private static void demonstrateYoungGeneration() {
        System.out.println("\n=== Young Generation Demonstration ===");
        
        // Young generation consists of Eden space and two Survivor spaces
        // Objects are first allocated in Eden
        List<ShortLivedObject> shortLivedObjects = new ArrayList<>();
        
        System.out.println("Creating short-lived objects in Eden space...");
        
        // Create many short-lived objects
        long startTime = System.currentTimeMillis();
        for (int i = 0; i < 500000; i++) {
            ShortLivedObject obj = new ShortLivedObject();
            shortLivedObjects.add(obj);
            
            // Remove some to allow GC
            if (i % 1000 == 0 && shortLivedObjects.size() > 10000) {
                // Remove oldest objects
                shortLivedObjects.subList(0, 5000).clear();
            }
        }
        
        long endTime = System.currentTimeMillis();
        System.out.println("Created and managed 500,000 short-lived objects in " + 
                          (endTime - startTime) + " ms");
        
        // Clear all references
        shortLivedObjects.clear();
        System.gc(); // Suggest minor GC
        
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    private static void demonstrateOldGeneration() {
        System.out.println("\n=== Old Generation Demonstration ===");
        
        // Objects that survive multiple young GCs are promoted to old generation
        List<LongLivedObject> longLivedObjects = new ArrayList<>();
        
        System.out.println("Creating long-lived objects that will be promoted...");
        
        // Create objects that will survive and be promoted
        for (int i = 0; i < 10000; i++) {
            LongLivedObject obj = new LongLivedObject(i);
            longLivedObjects.add(obj);
        }
        
        System.out.println("Created " + longLivedObjects.size() + " long-lived objects");
        
        // Trigger multiple minor GCs to promote objects
        for (int i = 0; i < 5; i++) {
            System.gc(); // Suggest minor GC
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            System.out.println("Minor GC #" + (i + 1) + " completed");
        }
        
        // Clear references and trigger full GC
        longLivedObjects.clear();
        System.gc(); // Suggest full GC
        
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    private static void demonstrateTenuring() {
        System.out.println("\n=== Tenuring Threshold Demonstration ===");
        
        // Objects that survive enough young GCs are promoted to old generation
        // This is controlled by the tenuring threshold
        
        List<TenuringObject> objects = new ArrayList<>();
        
        System.out.println("Creating objects to demonstrate tenuring...");
        
        // Create objects that will survive multiple GCs
        for (int generation = 0; generation < 10; generation++) {
            System.out.println("Generation " + generation + ": Creating objects");
            
            // Create new objects
            for (int i = 0; i < 10000; i++) {
                objects.add(new TenuringObject(generation, i));
            }
            
            // Remove some old objects to allow others to survive
            if (objects.size() > 20000) {
                objects.subList(0, 5000).clear();
            }
            
            // Trigger minor GC
            System.gc();
            
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            
            System.out.println("After GC - objects count: " + objects.size());
        }
        
        objects.clear();
        System.gc();
    }
}

class ShortLivedObject {
    private byte[] data = new byte[64];
    private String id = UUID.randomUUID().toString();
    
    public ShortLivedObject() {
        Arrays.fill(data, (byte) (Math.random() * 256));
    }
}

class LongLivedObject {
    private byte[] data = new byte[256];
    private int id;
    private String name;
    
    public LongLivedObject(int id) {
        this.id = id;
        this.name = "LongLivedObject-" + id;
        Arrays.fill(data, (byte) (id % 256));
    }
}

class TenuringObject {
    private byte[] data = new byte[128];
    private int generation;
    private int id;
    
    public TenuringObject(int generation, int id) {
        this.generation = generation;
        this.id = id;
        Arrays.fill(data, (byte) ((generation + id) % 256));
    }
}
```

## Memory Profiling and Monitoring

### Memory Profiling Tools and Techniques

```java
import java.lang.management.*;
import java.util.*;
import java.util.concurrent.*;

public class MemoryProfiling {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Memory Profiling and Monitoring ===");
        
        // Monitor memory usage over time
        monitorMemoryUsage();
        
        // Detect memory leaks
        demonstrateMemoryLeakDetection();
        
        // Profile object allocation
        profileObjectAllocation();
    }
    
    private static void monitorMemoryUsage() throws InterruptedException {
        System.out.println("\n=== Memory Usage Monitoring ===");
        
        ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);
        
        // Schedule memory monitoring task
        executor.scheduleAtFixedRate(() -> {
            Runtime runtime = Runtime.getRuntime();
            long totalMemory = runtime.totalMemory();
            long freeMemory = runtime.freeMemory();
            long usedMemory = totalMemory - freeMemory;
            
            System.out.printf("Memory - Total: %d MB, Used: %d MB, Free: %d MB%n",
                totalMemory / (1024 * 1024),
                usedMemory / (1024 * 1024),
                freeMemory / (1024 * 1024));
        }, 0, 2, TimeUnit.SECONDS);
        
        // Create memory pressure
        List<byte[]> memoryHog = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            memoryHog.add(new byte[1024 * 1024]); // 1MB each
            Thread.sleep(100);
        }
        
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);
        
        memoryHog.clear();
        System.gc();
    }
    
    private static void demonstrateMemoryLeakDetection() throws InterruptedException {
        System.out.println("\n=== Memory Leak Detection ===");
        
        // Simulate a memory leak with a static collection
        MemoryLeakSimulator simulator = new MemoryLeakSimulator();
        
        System.out.println("Starting memory leak simulation...");
        
        // Create objects that will accumulate in static collection
        for (int i = 0; i < 100000; i++) {
            simulator.createLeakingObject();
            
            if (i % 10000 == 0) {
                System.out.println("Created " + i + " objects");
                Thread.sleep(100);
            }
        }
        
        System.out.println("Memory leak simulation completed");
        System.out.println("Static collection size: " + MemoryLeakSimulator.getLeakSize());
        
        // Clear the leak
        MemoryLeakSimulator.clearLeak();
        System.gc();
        Thread.sleep(1000);
        
        System.out.println("After clearing leak: " + MemoryLeakSimulator.getLeakSize());
    }
    
    private static void profileObjectAllocation() throws InterruptedException {
        System.out.println("\n=== Object Allocation Profiling ===");
        
        // Profile different types of object allocations
        ObjectAllocationProfiler profiler = new ObjectAllocationProfiler();
        
        System.out.println("Profiling object allocations...");
        
        // Profile primitive arrays
        profiler.profilePrimitiveArrays();
        
        // Profile object arrays
        profiler.profileObjectArrays();
        
        // Profile collections
        profiler.profileCollections();
        
        // Profile strings
        profiler.profileStrings();
        
        profiler.printResults();
    }
}

class MemoryLeakSimulator {
    private static List<Object> staticLeakCollection = new ArrayList<>();
    
    public void createLeakingObject() {
        // Objects that should be temporary but are kept in static collection
        LeakingObject obj = new LeakingObject();
        staticLeakCollection.add(obj);
    }
    
    public static int getLeakSize() {
        return staticLeakCollection.size();
    }
    
    public static void clearLeak() {
        staticLeakCollection.clear();
    }
}

class LeakingObject {
    private byte[] data = new byte[1024];
    private String id = UUID.randomUUID().toString();
    
    public LeakingObject() {
        Arrays.fill(data, (byte) (Math.random() * 256));
    }
}

class ObjectAllocationProfiler {
    private Map<String, Long> allocationStats = new HashMap<>();
    private Runtime runtime = Runtime.getRuntime();
    
    public void profilePrimitiveArrays() {
        long startMemory = getUsedMemory();
        
        // Create primitive arrays
        for (int i = 0; i < 10000; i++) {
            int[] array = new int[100];
            Arrays.fill(array, i);
        }
        
        long endMemory = getUsedMemory();
        allocationStats.put("Primitive Arrays", endMemory - startMemory);
    }
    
    public void profileObjectArrays() {
        long startMemory = getUsedMemory();
        
        // Create object arrays
        for (int i = 0; i < 10000; i++) {
            String[] array = new String[10];
            for (int j = 0; j < array.length; j++) {
                array[j] = "String-" + i + "-" + j;
            }
        }
        
        long endMemory = getUsedMemory();
        allocationStats.put("Object Arrays", endMemory - startMemory);
    }
    
    public void profileCollections() {
        long startMemory = getUsedMemory();
        
        // Create collections
        for (int i = 0; i < 10000; i++) {
            List<String> list = new ArrayList<>();
            for (int j = 0; j < 10; j++) {
                list.add("Item-" + i + "-" + j);
            }
        }
        
        long endMemory = getUsedMemory();
        allocationStats.put("Collections", endMemory - startMemory);
    }
    
    public void profileStrings() {
        long startMemory = getUsedMemory();
        
        // Create strings
        for (int i = 0; i < 100000; i++) {
            String str = "String-" + i + "-with-some-additional-text-for-memory-consumption";
        }
        
        long endMemory = getUsedMemory();
        allocationStats.put("Strings", endMemory - startMemory);
    }
    
    public void printResults() {
        System.out.println("\nAllocation Profiling Results:");
        for (Map.Entry<String, Long> entry : allocationStats.entrySet()) {
            System.out.printf("%s: %d KB%n", 
                entry.getKey(), entry.getValue() / 1024);
        }
    }
    
    private long getUsedMemory() {
        System.gc(); // Try to get accurate measurement
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return runtime.totalMemory() - runtime.freeMemory();
    }
}
```

### Advanced Memory Monitoring

```java
import java.lang.management.*;
import java.util.*;
import java.util.concurrent.*;

public class AdvancedMemoryMonitoring {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Advanced Memory Monitoring ===");
        
        // Monitor GC events
        monitorGCEvents();
        
        // Monitor heap usage
        monitorHeapUsage();
        
        // Monitor direct memory usage
        monitorDirectMemory();
    }
    
    private static void monitorGCEvents() throws InterruptedException {
        System.out.println("\n=== GC Event Monitoring ===");
        
        // Get GC beans for monitoring
        List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        
        // Print initial GC stats
        System.out.println("Initial GC Statistics:");
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            System.out.printf("  %s - Collections: %d, Time: %d ms%n",
                gcBean.getName(), gcBean.getCollectionCount(), gcBean.getCollectionTime());
        }
        
        // Create memory pressure to trigger GC events
        List<GCEventObject> objects = new ArrayList<>();
        
        ScheduledExecutorService monitor = Executors.newScheduledThreadPool(1);
        monitor.scheduleAtFixedRate(() -> {
            for (GarbageCollectorMXBean gcBean : gcBeans) {
                long count = gcBean.getCollectionCount();
                long time = gcBean.getCollectionTime();
                if (count > 0) {
                    System.out.printf("GC %s - Collections: %d, Time: %d ms%n",
                        gcBean.getName(), count, time);
                }
            }
        }, 1, 1, TimeUnit.SECONDS);
        
        // Create objects to trigger GC
        for (int i = 0; i < 200000; i++) {
            objects.add(new GCEventObject(i));
            
            if (i % 50000 == 0) {
                System.out.println("Created " + i + " objects");
                objects.clear(); // Clear to allow GC
                System.gc(); // Suggest GC
            }
        }
        
        monitor.shutdown();
        monitor.awaitTermination(5, TimeUnit.SECONDS);
    }
    
    private static void monitorHeapUsage() throws InterruptedException {
        System.out.println("\n=== Heap Usage Monitoring ===");
        
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        List<MemoryPoolMXBean> memoryPools = ManagementFactory.getMemoryPoolMXBeans();
        
        // Monitor heap usage over time
        ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);
        executor.scheduleAtFixedRate(() -> {
            MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
            System.out.printf("Heap Usage - Used: %d MB, Max: %d MB, %d%%%n",
                heapUsage.getUsed() / (1024 * 1024),
                heapUsage.getMax() / (1024 * 1024),
                (heapUsage.getUsed() * 100) / heapUsage.getMax());
            
            // Show detailed pool usage
            for (MemoryPoolMXBean pool : memoryPools) {
                if (pool.getType() == MemoryType.HEAP) {
                    MemoryUsage usage = pool.getUsage();
                    if (usage != null && usage.getMax() > 0) {
                        System.out.printf("  %s: %d MB (%d%%)%n",
                            pool.getName(),
                            usage.getUsed() / (1024 * 1024),
                            (usage.getUsed() * 100) / usage.getMax());
                    }
                }
            }
            System.out.println();
        }, 0, 2, TimeUnit.SECONDS);
        
        // Create varying memory usage
        List<VaryingMemoryObject> objects = new ArrayList<>();
        Random random = new Random();
        
        for (int i = 0; i < 60; i++) {
            // Create random number of objects
            int objectCount = random.nextInt(50000) + 10000;
            for (int j = 0; j < objectCount; j++) {
                objects.add(new VaryingMemoryObject());
            }
            
            System.out.println("Iteration " + i + " - Created " + objectCount + " objects");
            
            // Clear some objects to vary memory usage
            if (objects.size() > 100000) {
                objects.subList(0, 50000).clear();
            }
            
            Thread.sleep(2000);
        }
        
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);
        
        objects.clear();
        System.gc();
    }
    
    private static void monitorDirectMemory() {
        System.out.println("\n=== Direct Memory Monitoring ===");
        
        // Note: Direct memory monitoring requires JVM-specific beans
        // This is a simplified example
        
        try {
            // Try to access direct memory beans (Java 7+)
            Class<?> bufferPoolClass = Class.forName("java.lang.management.BufferPoolMXBean");
            @SuppressWarnings("unchecked")
            List<BufferPoolMXBean> bufferPools = 
                ManagementFactory.getPlatformMXBeans((Class<BufferPoolMXBean>) bufferPoolClass);
            
            System.out.println("Direct Buffer Pools:");
            for (BufferPoolMXBean pool : bufferPools) {
                System.out.printf("  %s - Count: %d, Used: %d MB, Capacity: %d MB%n",
                    pool.getName(),
                    pool.getCount(),
                    pool.getMemoryUsed() / (1024 * 1024),
                    pool.getTotalCapacity() / (1024 * 1024));
            }
        } catch (ClassNotFoundException e) {
            System.out.println("Direct buffer pool monitoring not available");
        }
        
        // Demonstrate direct memory usage
        demonstrateDirectMemoryUsage();
    }
    
    private static void demonstrateDirectMemoryUsage() {
        System.out.println("\nDemonstrating direct memory allocation:");
        
        List<java.nio.ByteBuffer> directBuffers = new ArrayList<>();
        
        try {
            // Allocate direct buffers
            for (int i = 0; i < 10; i++) {
                java.nio.ByteBuffer buffer = java.nio.ByteBuffer.allocateDirect(1024 * 1024); // 1MB
                directBuffers.add(buffer);
                System.out.println("Allocated direct buffer #" + (i + 1) + " (1MB)");
            }
        } catch (OutOfMemoryError e) {
            System.out.println("Direct memory allocation failed: " + e.getMessage());
        }
        
        directBuffers.clear();
        System.gc();
    }
}

class GCEventObject {
    private byte[] data = new byte[128];
    private int id;
    
    public GCEventObject(int id) {
        this.id = id;
        Arrays.fill(data, (byte) (id % 256));
    }
}

class VaryingMemoryObject {
    private byte[] data = new byte[64 + (int)(Math.random() * 192)]; // 64-256 bytes
    private String id = UUID.randomUUID().toString();
    
    public VaryingMemoryObject() {
        Arrays.fill(data, (byte) (Math.random() * 256));
    }
}
```

## Garbage Collection Algorithms

### Serial, Parallel, and CMS Collectors

```java
import java.util.*;
import java.util.concurrent.*;

public class GCTypesDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Garbage Collection Types Comparison ===");
        
        // Demonstrate different GC behaviors
        compareGCTypes();
    }
    
    private static void compareGCTypes() throws InterruptedException {
        System.out.println("\nThis demonstration shows how different GC types handle memory:");
        System.out.println("Note: Actual GC type depends on JVM arguments and version");
        
        // Get current GC information
        List<GarbageCollectorMXBean> gcBeans = 
            java.lang.management.ManagementFactory.getGarbageCollectorMXBeans();
        
        System.out.println("Available garbage collectors:");
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            System.out.println("  " + gcBean.getName() + 
                             " (Collections: " + gcBean.getCollectionCount() + 
                             ", Time: " + gcBean.getCollectionTime() + "ms)");
        }
        
        // Performance comparison simulation
        simulateGCPerformance();
    }
    
    private static void simulateGCPerformance() throws InterruptedException {
        System.out.println("\n=== GC Performance Simulation ===");
        
        // Simulate different workloads for different GC types
        System.out.println("Simulating workloads for different GC scenarios:");
        
        // Serial GC scenario (single-threaded applications)
        simulateSerialGC();
        
        // Parallel GC scenario (multi-threaded CPU-intensive applications)
        simulateParallelGC();
        
        // CMS/G1 scenario (low-latency applications)
        simulateLowLatencyGC();
    }
    
    private static void simulateSerialGC() throws InterruptedException {
        System.out.println("\nSimulating Serial GC (single-threaded):");
        
        long startTime = System.currentTimeMillis();
        long startCollections = getCollectionCount();
        
        // Single-threaded allocation pattern
        List<SerialGCObject> objects = new ArrayList<>();
        for (int i = 0; i < 100000; i++) {
            objects.add(new SerialGCObject(i));
            
            // Periodic cleanup to simulate GC-friendly pattern
            if (i % 10000 == 0) {
                objects.subList(0, 5000).clear();
            }
        }
        
        long endTime = System.currentTimeMillis();
        long endCollections = getCollectionCount();
        
        System.out.printf("Serial GC simulation - Time: %d ms, Collections: %d%n",
            endTime - startTime, endCollections - startCollections);
        
        objects.clear();
        System.gc();
    }
    
    private static void simulateParallelGC() throws InterruptedException {
        System.out.println("\nSimulating Parallel GC (multi-threaded):");
        
        long startTime = System.currentTimeMillis();
        long startCollections = getCollectionCount();
        
        // Multi-threaded allocation pattern
        ExecutorService executor = Executors.newFixedThreadPool(4);
        List<Future<?>> futures = new ArrayList<>();
        
        for (int i = 0; i < 4; i++) {
            final int threadId = i;
            Future<?> future = executor.submit(() -> {
                List<ParallelGCObject> threadObjects = new ArrayList<>();
                for (int j = 0; j < 25000; j++) {
                    threadObjects.add(new ParallelGCObject(threadId, j));
                    
                    if (j % 5000 == 0) {
                        threadObjects.subList(0, 2500).clear();
                    }
                }
                return threadObjects.size();
            });
            futures.add(future);
        }
        
        // Wait for completion
        for (Future<?> future : futures) {
            try {
                future.get();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }
        
        long endTime = System.currentTimeMillis();
        long endCollections = getCollectionCount();
        
        System.out.printf("Parallel GC simulation - Time: %d ms, Collections: %d%n",
            endTime - startTime, endCollections - startCollections);
        
        executor.shutdown();
        System.gc();
    }
    
    private static void simulateLowLatencyGC() throws InterruptedException {
        System.out.println("\nSimulating Low-Latency GC (G1/CMS):");
        
        long startTime = System.currentTimeMillis();
        long startCollections = getCollectionCount();
        
        // Low-pause allocation pattern with mixed object sizes
        List<MixedSizeObject> objects = new ArrayList<>();
        
        for (int i = 0; i < 50000; i++) {
            // Mix of small and large objects
            if (i % 10 == 0) {
                objects.add(new MixedSizeObject(1024 * 10)); // Large object
            } else {
                objects.add(new MixedSizeObject(64)); // Small object
            }
            
            // Frequent small cleanups to keep pause times low
            if (i % 5000 == 0) {
                objects.subList(0, 2500).clear();
                Thread.sleep(10); // Small pause
            }
        }
        
        long endTime = System.currentTimeMillis();
        long endCollections = getCollectionCount();
        
        System.out.printf("Low-Latency GC simulation - Time: %d ms, Collections: %d%n",
            endTime - startTime, endCollections - startCollections);
        
        objects.clear();
        System.gc();
    }
    
    private static long getCollectionCount() {
        long total = 0;
        for (GarbageCollectorMXBean gcBean : 
             java.lang.management.ManagementFactory.getGarbageCollectorMXBeans()) {
            total += gcBean.getCollectionCount();
        }
        return total;
    }
}

class SerialGCObject {
    private byte[] data = new byte[128];
    private int id;
    
    public SerialGCObject(int id) {
        this.id = id;
        Arrays.fill(data, (byte) (id % 256));
    }
}

class ParallelGCObject {
    private byte[] data = new byte[256];
    private int threadId;
    private int id;
    
    public ParallelGCObject(int threadId, int id) {
        this.threadId = threadId;
        this.id = id;
        Arrays.fill(data, (byte) ((threadId + id) % 256));
    }
}

class MixedSizeObject {
    private byte[] data;
    private String id;
    
    public MixedSizeObject(int size) {
        this.data = new byte[size];
        this.id = UUID.randomUUID().toString();
        Arrays.fill(data, (byte) (size % 256));
    }
}
```

### G1 Garbage Collector

```java
import java.util.*;
import java.util.concurrent.*;

public class G1GCDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== G1 Garbage Collector Demonstration ===");
        
        // Demonstrate G1 GC features
        demonstrateG1Features();
        
        // Compare G1 with other collectors
        compareWithOtherCollectors();
    }
    
    private static void demonstrateG1Features() throws InterruptedException {
        System.out.println("\n=== G1 GC Features Demonstration ===");
        
        System.out.println("G1 GC Key Features:");
        System.out.println("1. Heap divided into regions");
        System.out.println("2. Concurrent marking");
        System.out.println("3. Evacuation pause handling");
        System.out.println("4. Compaction without full GC");
        
        // Simulate G1-like behavior with region-based allocation
        simulateRegionBasedAllocation();
    }
    
    private static void simulateRegionBasedAllocation() throws InterruptedException {
        System.out.println("\nSimulating region-based allocation:");
        
        // Create multiple regions of different sizes
        List<RegionObject> regionObjects = new ArrayList<>();
        
        // Simulate Eden regions (frequent allocation)
        System.out.println("Allocating in Eden-like regions...");
        for (int i = 0; i < 100000; i++) {
            regionObjects.add(new RegionObject(RegionType.EDEN, i));
            
            // Periodic evacuation (like young GC)
            if (i % 20000 == 0) {
                int survivors = regionObjects.size() / 4; // Keep 25% as survivors
                regionObjects.subList(0, regionObjects.size() - survivors).clear();
                System.out.println("Evacuation pause - kept " + survivors + " survivors");
            }
        }
        
        // Simulate old region promotion
        System.out.println("Promoting long-lived objects to Old regions...");
        List<RegionObject> oldRegionObjects = new ArrayList<>();
        for (RegionObject obj : regionObjects) {
            if (obj.getAge() > 5) { // Objects with age > 5 get promoted
                oldRegionObjects.add(new RegionObject(RegionType.OLD, obj.getId()));
            }
        }
        
        System.out.println("Promoted " + oldRegionObjects.size() + " objects to Old regions");
        
        // Simulate mixed GC (both young and old regions)
        System.out.println("Performing mixed GC...");
        for (int i = 0; i < 10000; i++) {
            if (i < oldRegionObjects.size()) {
                oldRegionObjects.set(i, null); // Mark for collection
            }
        }
        
        // Remove null entries (compaction-like behavior)
        oldRegionObjects.removeIf(Objects::isNull);
        System.out.println("After mixed GC - Old region objects: " + oldRegionObjects.size());
        
        regionObjects.clear();
        oldRegionObjects.clear();
        System.gc();
    }
    
    private static void compareWithOtherCollectors() {
        System.out.println("\n=== G1 vs Other Collectors ===");
        
        System.out.println("Serial GC:");
        System.out.println("  - Single-threaded");
        System.out.println("  - Stop-the-world pauses");
        System.out.println("  - Good for small heaps (< 100MB)");
        
        System.out.println("\nParallel GC:");
        System.out.println("  - Multi-threaded");
        System.out.println("  - Stop-the-world pauses");
        System.out.println("  - Good for throughput-focused applications");
        
        System.out.println("\nCMS GC (Deprecated in Java 14):");
        System.out.println("  - Mostly concurrent");
        System.out.println("  - Low pause times");
        System.out.println("  - No compaction (fragmentation issues)");
        
        System.out.println("\nG1 GC:");
        System.out.println("  - Region-based heap");
        System.out.println("  - Mostly concurrent marking");
        System.out.println("  - Predictable pause times");
        System.out.println("  - Compaction included");
        
        System.out.println("\nZGC (Java 11+):");
        System.out.println("  - Ultra-low pause times (< 10ms)");
        System.out.println("  - Scalable to large heaps");
        System.out.println("  - Concurrent operations");
        
        System.out.println("\nShenandoah (Java 12+):");
        System.out.println("  - Ultra-low pause times");
        System.out.println("  - Concurrent compaction");
        System.out.println("  - Heap size independent pause times");
    }
}

enum RegionType {
    EDEN, SURVIVOR, OLD, HUMONGOUS
}

class RegionObject {
    private byte[] data;
    private RegionType regionType;
    private int id;
    private int age;
    
    public RegionObject(RegionType regionType, int id) {
        this.regionType = regionType;
        this.id = id;
        this.age = 0;
        
        // Different sizes based on region type
        int size;
        switch (regionType) {
            case EDEN:
                size = 128 + (id % 128); // 128-255 bytes
                break;
            case OLD:
                size = 512 + (id % 512); // 512-1023 bytes
                break;
            case HUMONGOUS:
                size = 2048 + (id % 2048); // 2048-4095 bytes
                break;
            default:
                size = 64;
        }
        
        this.data = new byte[size];
        Arrays.fill(data, (byte) (id % 256));
    }
    
    public int getAge() {
        return age;
    }
    
    public int getId() {
        return id;
    }
    
    public RegionType getRegionType() {
        return regionType;
    }
    
    public void incrementAge() {
        this.age++;
    }
}
```

## Memory Optimization Techniques

### Object Pooling and Memory Efficiency

```java
import java.util.*;
import java.util.concurrent.*;

public class MemoryOptimization {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Memory Optimization Techniques ===");
        
        // Demonstrate object pooling
        demonstrateObjectPooling();
        
        // Demonstrate memory-efficient data structures
        demonstrateMemoryEfficientStructures();
        
        // Demonstrate string optimization
        demonstrateStringOptimization();
    }
    
    private static void demonstrateObjectPooling() throws InterruptedException {
        System.out.println("\n=== Object Pooling Demonstration ===");
        
        // Without pooling
        System.out.println("Without object pooling:");
        long startTime = System.currentTimeMillis();
        long startMemory = getUsedMemory();
        
        for (int i = 0; i < 100000; i++) {
            ExpensiveObject obj = new ExpensiveObject("Object-" + i);
            // Use object
            obj.process();
        }
        
        long endTime = System.currentTimeMillis();
        long endMemory = getUsedMemory();
        
        System.out.printf("Time: %d ms, Memory used: %d KB%n",
            endTime - startTime, (endMemory - startMemory) / 1024);
        
        System.gc();
        Thread.sleep(1000);
        
        // With pooling
        System.out.println("\nWith object pooling:");
        ObjectPool<ExpensiveObject> pool = new ObjectPool<>(() -> 
            new ExpensiveObject("Pooled"));
        
        startTime = System.currentTimeMillis();
        startMemory = getUsedMemory();
        
        for (int i = 0; i < 100000; i++) {
            ExpensiveObject obj = pool.acquire();
            obj.setId("Pooled-Object-" + i);
            obj.process();
            pool.release(obj);
        }
        
        endTime = System.currentTimeMillis();
        endMemory = getUsedMemory();
        
        System.out.printf("Time: %d ms, Memory used: %d KB%n",
            endTime - startTime, (endMemory - startMemory) / 1024);
        
        System.out.println("Pool statistics - Created: " + pool.getCreatedCount() + 
                          ", Reused: " + pool.getReusedCount());
    }
    
    private static void demonstrateMemoryEfficientStructures() {
        System.out.println("\n=== Memory-Efficient Data Structures ===");
        
        // Compare ArrayList vs ArrayDeque for queue operations
        compareQueueStructures();
        
        // Compare different Map implementations
        compareMapStructures();
        
        // Demonstrate primitive collections (conceptual)
        demonstratePrimitiveCollections();
    }
    
    private static void compareQueueStructures() {
        System.out.println("\nQueue structure comparison:");
        
        // ArrayList as queue (inefficient for frequent add/remove from front)
        List<Integer> arrayListQueue = new ArrayList<>();
        long startMemory = getUsedMemory();
        long startTime = System.currentTimeMillis();
        
        for (int i = 0; i < 10000; i++) {
            arrayListQueue.add(i); // Add to end - efficient
        }
        
        for (int i = 0; i < 1000; i++) {
            arrayListQueue.remove(0); // Remove from front - inefficient O(n)
        }
        
        long endTime = System.currentTimeMillis();
        long endMemory = getUsedMemory();
        
        System.out.printf("ArrayList queue - Time: %d ms, Memory: %d KB%n",
            endTime - startTime, (endMemory - startMemory) / 1024);
        
        // ArrayDeque as queue (efficient)
        Deque<Integer> arrayDeque = new ArrayDeque<>();
        startMemory = getUsedMemory();
        startTime = System.currentTimeMillis();
        
        for (int i = 0; i < 10000; i++) {
            arrayDeque.addLast(i); // Add to end - efficient O(1)
        }
        
        for (int i = 0; i < 1000; i++) {
            arrayDeque.removeFirst(); // Remove from front - efficient O(1)
        }
        
        endTime = System.currentTimeMillis();
        endMemory = getUsedMemory();
        
        System.out.printf("ArrayDeque queue - Time: %d ms, Memory: %d KB%n",
            endTime - startTime, (endMemory - startMemory) / 1024);
    }
    
    private static void compareMapStructures() {
        System.out.println("\nMap structure comparison:");
        
        int size = 10000;
        
        // HashMap
        Map<String, Integer> hashMap = new HashMap<>(size);
        long startMemory = getUsedMemory();
        long startTime = System.currentTimeMillis();
        
        for (int i = 0; i < size; i++) {
            hashMap.put("Key-" + i, i);
        }
        
        long endTime = System.currentTimeMillis();
        long endMemory = getUsedMemory();
        
        System.out.printf("HashMap - Time: %d ms, Memory: %d KB%n",
            endTime - startTime, (endMemory - startMemory) / 1024);
        
        // LinkedHashMap (maintains insertion order)
        Map<String, Integer> linkedHashMap = new LinkedHashMap<>(size);
        startMemory = getUsedMemory();
        startTime = System.currentTimeMillis();
        
        for (int i = 0; i < size; i++) {
            linkedHashMap.put("Key-" + i, i);
        }
        
        endTime = System.currentTimeMillis();
        endMemory = getUsedMemory();
        
        System.out.printf("LinkedHashMap - Time: %d ms, Memory: %d KB%n",
            endTime - startTime, (endMemory - startMemory) / 1024);
        
        // TreeMap (sorted)
        Map<String, Integer> treeMap = new TreeMap<>();
        startMemory = getUsedMemory();
        startTime = System.currentTimeMillis();
        
        for (int i = 0; i < size; i++) {
            treeMap.put("Key-" + i, i);
        }
        
        endTime = System.currentTimeMillis();
        endMemory = getUsedMemory();
        
        System.out.printf("TreeMap - Time: %d ms, Memory: %d KB%n",
            endTime - startTime, (endMemory - startMemory) / 1024);
    }
    
    private static void demonstratePrimitiveCollections() {
        System.out.println("\nPrimitive collections (conceptual):");
        System.out.println("Using primitive arrays instead of object wrappers saves memory:");
        
        // Integer array (objects)
        long startMemory = getUsedMemory();
        Integer[] intObjects = new Integer[10000];
        for (int i = 0; i < intObjects.length; i++) {
            intObjects[i] = i;
        }
        long endMemory = getUsedMemory();
        System.out.printf("Integer objects: %d KB%n", (endMemory - startMemory) / 1024);
        
        // int array (primitives)
        startMemory = getUsedMemory();
        int[] intPrimitives = new int[10000];
        for (int i = 0; i < intPrimitives.length; i++) {
            intPrimitives[i] = i;
        }
        endMemory = getUsedMemory();
        System.out.printf("int primitives: %d KB%n", (endMemory - startMemory) / 1024);
        
        System.out.println("Primitive arrays use significantly less memory!");
    }
    
    private static void demonstrateStringOptimization() {
        System.out.println("\n=== String Optimization ===");
        
        // String concatenation comparison
        compareStringConcatenation();
        
        // String interning
        demonstrateStringInterning();
    }
    
    private static void compareStringConcatenation() {
        System.out.println("\nString concatenation methods:");
        
        int iterations = 10000;
        
        // Using + operator (inefficient for loops)
        long startMemory = getUsedMemory();
        long startTime = System.currentTimeMillis();
        String result1 = "";
        for (int i = 0; i < iterations; i++) {
            result1 += "Item-" + i + ";";
        }
        long endTime = System.currentTimeMillis();
        long endMemory = getUsedMemory();
        
        System.out.printf("+ operator - Time: %d ms, Memory: %d KB%n",
            endTime - startTime, (endMemory - startMemory) / 1024);
        
        // Using StringBuilder (efficient)
        startMemory = getUsedMemory();
        startTime = System.currentTimeMillis();
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < iterations; i++) {
            sb.append("Item-").append(i).append(";");
        }
        String result2 = sb.toString();
        endTime = System.currentTimeMillis();
        endMemory = getUsedMemory();
        
        System.out.printf("StringBuilder - Time: %d ms, Memory: %d KB%n",
            endTime - startTime, (endMemory - startMemory) / 1024);
        
        // Using String.join (for collections)
        List<String> items = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            items.add("Item-" + i);
        }
        
        startMemory = getUsedMemory();
        startTime = System.currentTimeMillis();
        String result3 = String.join(";", items);
        endTime = System.currentTimeMillis();
        endMemory = getUsedMemory();
        
        System.out.printf("String.join - Time: %d ms, Memory: %d KB%n",
            endTime - startTime, (endMemory - startMemory) / 1024);
    }
    
    private static void demonstrateStringInterning() {
        System.out.println("\nString interning demonstration:");
        
        // Without interning
        long startMemory = getUsedMemory();
        List<String> stringsWithoutInterning = new ArrayList<>();
        for (int i = 0; i < 10000; i++) {
            stringsWithoutInterning.add(new String("CommonString"));
        }
        long endMemory = getUsedMemory();
        
        System.out.printf("Without interning: %d KB%n", (endMemory - startMemory) / 1024);
        
        // With interning
        startMemory = getUsedMemory();
        List<String> stringsWithInterning = new ArrayList<>();
        for (int i = 0; i < 10000; i++) {
            stringsWithInterning.add(new String("CommonString").intern());
        }
        endMemory = getUsedMemory();
        
        System.out.printf("With interning: %d KB%n", (endMemory - startMemory) / 1024);
        
        System.out.println("String interning can save significant memory for repeated strings!");
    }
    
    private static long getUsedMemory() {
        System.gc();
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        Runtime runtime = Runtime.getRuntime();
        return runtime.totalMemory() - runtime.freeMemory();
    }
}

// Expensive object that benefits from pooling
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
        // Simulate some processing
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

// Simple object pool implementation
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
```

### Memory Leak Prevention and Detection

```java
import java.util.*;
import java.lang.ref.*;

public class MemoryLeakPrevention {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Memory Leak Prevention and Detection ===");
        
        // Demonstrate common memory leaks
        demonstrateCommonLeaks();
        
        // Show prevention techniques
        demonstratePreventionTechniques();
        
        // Show detection methods
        demonstrateDetectionMethods();
    }
    
    private static void demonstrateCommonLeaks() throws InterruptedException {
        System.out.println("\n=== Common Memory Leaks ===");
        
        // 1. Static collections
        System.out.println("1. Static Collections Leak:");
        demonstrateStaticCollectionLeak();
        
        // 2. Event listeners
        System.out.println("\n2. Event Listener Leak:");
        demonstrateEventListenerLeak();
        
        // 3. Inner classes holding references
        System.out.println("\n3. Inner Class Reference Leak:");
        demonstrateInnerClassLeak();
        
        // 4. ThreadLocal variables
        System.out.println("\n4. ThreadLocal Leak:");
        demonstrateThreadLocalLeak();
    }
    
    private static void demonstrateStaticCollectionLeak() throws InterruptedException {
        System.out.println("Creating objects that accumulate in static collection...");
        
        for (int i = 0; i < 10000; i++) {
            LeakingClass.addToList(new LeakingObject("Object-" + i));
        }
        
        System.out.println("Static list size: " + LeakingClass.getListSize());
        
        // Clear the leak
        LeakingClass.clearList();
        System.gc();
        Thread.sleep(1000);
        
        System.out.println("After clearing: " + LeakingClass.getListSize());
    }
    
    private static void demonstrateEventListenerLeak() throws InterruptedException {
        System.out.println("Creating event listeners that aren't properly removed...");
        
        EventSource source = new EventSource();
        
        // Add many listeners without removing them
        for (int i = 0; i < 1000; i++) {
            EventListener listener = new EventListener("Listener-" + i);
            source.addListener(listener);
        }
        
        System.out.println("Listeners added: " + source.getListenerCount());
        
        // Properly remove listeners
        source.removeAllListeners();
        System.gc();
        Thread.sleep(1000);
        
        System.out.println("After cleanup: " + source.getListenerCount());
    }
    
    private static void demonstrateInnerClassLeak() throws InterruptedException {
        System.out.println("Creating inner class instances that hold outer class references...");
        
        List<OuterClass.InnerClass> innerObjects = new ArrayList<>();
        
        for (int i = 0; i < 1000; i++) {
            OuterClass outer = new OuterClass("Outer-" + i);
            OuterClass.InnerClass inner = outer.new InnerClass();
            innerObjects.add(inner);
            // outer reference is kept alive through inner class
        }
        
        System.out.println("Inner objects created: " + innerObjects.size());
        
        innerObjects.clear();
        System.gc();
        Thread.sleep(1000);
        
        System.out.println("After clearing inner objects");
    }
    
    private static void demonstrateThreadLocalLeak() throws InterruptedException {
        System.out.println("Creating ThreadLocal variables that may leak...");
        
        ExecutorService executor = Executors.newFixedThreadPool(2);
        
        for (int i = 0; i < 100; i++) {
            final int taskId = i;
            executor.submit(() -> {
                // Set ThreadLocal value
                ThreadLocalLeakExample.threadLocal.set("Value-" + taskId);
                
                // Process data
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                
                // Important: Remove ThreadLocal value to prevent leaks
                ThreadLocalLeakExample.threadLocal.remove();
            });
        }
        
        executor.shutdown();
        executor.awaitTermination(10, TimeUnit.SECONDS);
        
        System.out.println("ThreadLocal tasks completed");
    }
    
    private static void demonstratePreventionTechniques() {
        System.out.println("\n=== Prevention Techniques ===");
        
        // 1. Weak references
        System.out.println("1. Using Weak References:");
        demonstrateWeakReferences();
        
        // 2. Proper resource management
        System.out.println("\n2. Proper Resource Management:");
        demonstrateResourceManagement();
        
        // 3. WeakHashMap for caches
        System.out.println("\n3. WeakHashMap for Caching:");
        demonstrateWeakHashMap();
    }
    
    private static void demonstrateWeakReferences() {
        System.out.println("Using WeakReference to prevent memory leaks:");
        
        List<WeakReference<LargeObject>> weakRefs = new ArrayList<>();
        
        // Create objects with weak references
        for (int i = 0; i < 1000; i++) {
            LargeObject obj = new LargeObject("Object-" + i);
            weakRefs.add(new WeakReference<>(obj));
        }
        
        System.out.println("Created 1000 objects with weak references");
        
        // Clear strong references
        System.gc(); // Suggest garbage collection
        
        // Count how many objects are still referenced
        int liveCount = 0;
        for (WeakReference<LargeObject> ref : weakRefs) {
            if (ref.get() != null) {
                liveCount++;
            }
        }
        
        System.out.println("Objects still alive: " + liveCount);
        System.out.println("Objects garbage collected: " + (1000 - liveCount));
    }
    
    private static void demonstrateResourceManagement() {
        System.out.println("Proper resource management with try-with-resources:");
        
        // Good practice: automatic resource management
        try (Resource resource = new Resource("ManagedResource")) {
            resource.use();
        } catch (Exception e) {
            System.out.println("Exception handled: " + e.getMessage());
        }
        // Resource automatically closed
        
        System.out.println("Resource automatically managed and closed");
    }
    
    private static void demonstrateWeakHashMap() {
        System.out.println("Using WeakHashMap for automatic cleanup:");
        
        WeakHashMap<LargeObject, String> weakMap = new WeakHashMap<>();
        
        // Add entries
        for (int i = 0; i < 100; i++) {
            LargeObject key = new LargeObject("Key-" + i);
            weakMap.put(key, "Value-" + i);
        }
        
        System.out.println("WeakHashMap size: " + weakMap.size());
        
        // Force GC to demonstrate automatic cleanup
        System.gc();
        
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        System.out.println("WeakHashMap size after GC: " + weakMap.size());
    }
    
    private static void demonstrateDetectionMethods() {
        System.out.println("\n=== Detection Methods ===");
        
        System.out.println("Memory leak detection methods:");
        System.out.println("1. Heap dumps and analysis tools (VisualVM, JProfiler, YourKit)");
        System.out.println("2. Memory usage monitoring with JMX");
        System.out.println("3. Profiling with JVM flags (-XX:+PrintGC, -XX:+PrintGCDetails)");
        System.out.println("4. Using WeakReference to detect leaks");
        System.out.println("5. Monitoring with custom memory tracking");
        
        // Simple memory tracking example
        demonstrateSimpleTracking();
    }
    
    private static void demonstrateSimpleTracking() {
        System.out.println("\nSimple memory tracking example:");
        
        MemoryTracker tracker = new MemoryTracker();
        
        // Track object creation
        for (int i = 0; i < 1000; i++) {
            TrackedObject obj = new TrackedObject("Tracked-" + i);
            tracker.track(obj);
        }
        
        System.out.println("Tracked objects: " + tracker.getObjectCount());
        
        // Simulate some objects being garbage collected
        tracker.cleanup();
        System.gc();
        
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        System.out.println("Tracked objects after cleanup: " + tracker.getObjectCount());
    }
}

// Classes for demonstrating memory leaks
class LeakingClass {
    private static List<LeakingObject> staticList = new ArrayList<>();
    
    public static void addToList(LeakingObject obj) {
        staticList.add(obj);
    }
    
    public static int getListSize() {
        return staticList.size();
    }
    
    public static void clearList() {
        staticList.clear();
    }
}

class LeakingObject {
    private byte[] data = new byte[1024];
    private String id;
    
    public LeakingObject(String id) {
        this.id = id;
        Arrays.fill(data, (byte) (id.hashCode() % 256));
    }
}

class EventSource {
    private List<EventListener> listeners = new ArrayList<>();
    
    public void addListener(EventListener listener) {
        listeners.add(listener);
    }
    
    public void removeListener(EventListener listener) {
        listeners.remove(listener);
    }
    
    public void removeAllListeners() {
        listeners.clear();
    }
    
    public int getListenerCount() {
        return listeners.size();
    }
}

class EventListener {
    private String name;
    
    public EventListener(String name) {
        this.name = name;
    }
    
    public String getName() {
        return name;
    }
}

class OuterClass {
    private String name;
    
    public OuterClass(String name) {
        this.name = name;
    }
    
    class InnerClass {
        public String getOuterName() {
            return name; // Holds reference to outer class
        }
    }
}

class ThreadLocalLeakExample {
    public static ThreadLocal<String> threadLocal = new ThreadLocal<>();
}

class LargeObject {
    private byte[] data = new byte[1024];
    private String id;
    
    public LargeObject(String id) {
        this.id = id;
        Arrays.fill(data, (byte) (id.hashCode() % 256));
    }
    
    public String getId() {
        return id;
    }
}

class Resource implements AutoCloseable {
    private String name;
    
    public Resource(String name) {
        this.name = name;
        System.out.println("Resource " + name + " created");
    }
    
    public void use() {
        System.out.println("Using resource " + name);
    }
    
    @Override
    public void close() {
        System.out.println("Resource " + name + " closed");
    }
}

// Classes for memory tracking
class MemoryTracker {
    private Map<WeakReference<TrackedObject>, String> trackedObjects = new HashMap<>();
    
    public void track(TrackedObject obj) {
        trackedObjects.put(new WeakReference<>(obj), obj.getId());
    }
    
    public int getObjectCount() {
        // Remove cleared references
        trackedObjects.entrySet().removeIf(entry -> entry.getKey().get() == null);
        return trackedObjects.size();
    }
    
    public void cleanup() {
        trackedObjects.clear();
    }
}

class TrackedObject {
    private String id;
    
    public TrackedObject(String id) {
        this.id = id;
    }
    
    public String getId() {
        return id;
    }
}
```

## JVM Tuning and Performance Optimization

### JVM Arguments and Tuning Parameters

```java
import java.util.*;

public class JVMTuning {
    public static void main(String[] args) {
        System.out.println("=== JVM Tuning and Performance Optimization ===");
        
        // Display current JVM settings
        displayJVMSettings();
        
        // Demonstrate heap sizing
        demonstrateHeapSizing();
        
        // Demonstrate GC tuning
        demonstrateGCTuning();
        
        // Demonstrate performance monitoring
        demonstratePerformanceMonitoring();
    }
    
    private static void displayJVMSettings() {
        System.out.println("\n=== Current JVM Settings ===");
        
        System.out.println("JVM Name: " + System.getProperty("java.vm.name"));
        System.out.println("JVM Version: " + System.getProperty("java.vm.version"));
        System.out.println("Java Version: " + System.getProperty("java.version"));
        
        // Memory settings
        System.out.println("Initial heap size: " + Runtime.getRuntime().freeMemory() / (1024 * 1024) + " MB");
        System.out.println("Maximum heap size: " + Runtime.getRuntime().maxMemory() / (1024 * 1024) + " MB");
        System.out.println("Total heap size: " + Runtime.getRuntime().totalMemory() / (1024 * 1024) + " MB");
        
        // GC-related system properties
        System.out.println("GC settings:");
        String[] gcProperties = {
            "java.vm.info",
            "sun.gc.policy",
            "sun.gc.heap.size"
        };
        
        for (String prop : gcProperties) {
            String value = System.getProperty(prop, "Not set");
            System.out.println("  " + prop + ": " + value);
        }
    }
    
    private static void demonstrateHeapSizing() {
        System.out.println("\n=== Heap Sizing Demonstration ===");
        
        System.out.println("Common heap sizing JVM arguments:");
        System.out.println("  -Xms<size>: Initial heap size");
        System.out.println("  -Xmx<size>: Maximum heap size");
        System.out.println("  -XX:NewRatio=<ratio>: Old/Young generation ratio");
        System.out.println("  -XX:NewSize=<size>: Young generation size");
        System.out.println("  -XX:MaxNewSize=<size>: Maximum young generation size");
        System.out.println("  -XX:SurvivorRatio=<ratio>: Eden/Survivor space ratio");
        
        // Simulate different heap scenarios
        simulateHeapScenarios();
    }
    
    private static void simulateHeapScenarios() {
        System.out.println("\nSimulating heap usage scenarios:");
        
        // Scenario 1: Small heap with frequent allocation
        System.out.println("Scenario 1: Small heap behavior");
        simulateSmallHeap();
        
        // Scenario 2: Large heap with infrequent allocation
        System.out.println("\nScenario 2: Large heap behavior");
        simulateLargeHeap();
    }
    
    private static void simulateSmallHeap() {
        // This would typically be run with -Xmx64m
        System.out.println("Simulating small heap (64MB) allocation pattern...");
        
        List<byte[]> smallObjects = new ArrayList<>();
        try {
            for (int i = 0; i < 1000; i++) {
                smallObjects.add(new byte[32 * 1024]); // 32KB each
                if (i % 100 == 0) {
                    System.out.println("Allocated " + (i * 32) + " KB");
                }
            }
        } catch (OutOfMemoryError e) {
            System.out.println("OutOfMemoryError caught - this is expected with small heap");
        }
        
        smallObjects.clear();
        System.gc();
    }
    
    private static void simulateLargeHeap() {
        // This would typically be run with -Xmx2g
        System.out.println("Simulating large heap (2GB) allocation pattern...");
        
        List<LargeObject> largeObjects = new ArrayList<>();
        try {
            for (int i = 0; i < 10000; i++) {
                largeObjects.add(new LargeObject(1024 * 1024)); // 1MB each
                if (i % 1000 == 0) {
                    System.out.println("Allocated " + (i / 1024) + " MB");
                }
            }
        } catch (OutOfMemoryError e) {
            System.out.println("OutOfMemoryError caught");
        }
        
        largeObjects.clear();
        System.gc();
    }
    
    private static void demonstrateGCTuning() {
        System.out.println("\n=== GC Tuning Demonstration ===");
        
        System.out.println("Common GC tuning JVM arguments:");
        System.out.println("Serial GC:");
        System.out.println("  -XX:+UseSerialGC");
        
        System.out.println("Parallel GC:");
        System.out.println("  -XX:+UseParallelGC");
        System.out.println("  -XX:ParallelGCThreads=<threads>");
        System.out.println("  -XX:+UseParallelOldGC");
        
        System.out.println("G1 GC:");
        System.out.println("  -XX:+UseG1GC");
        System.out.println("  -XX:MaxGCPauseMillis=<millis>");
        System.out.println("  -XX:G1HeapRegionSize=<size>");
        
        System.out.println("ZGC (Java 11+):");
        System.out.println("  -XX:+UseZGC");
        
        System.out.println("CMS GC (deprecated):");
        System.out.println("  -XX:+UseConcMarkSweepGC");
        
        // Demonstrate GC behavior tuning
        demonstrateGCBehaviorTuning();
    }
    
    private static void demonstrateGCBehaviorTuning() {
        System.out.println("\nGC behavior tuning parameters:");
        
        System.out.println("Young Generation Tuning:");
        System.out.println("  -XX:NewRatio=2 (Old:Young = 2:1)");
        System.out.println("  -XX:NewSize=256m (Young generation size)");
        System.out.println("  -XX:MaxNewSize=512m (Max young generation size)");
        System.out.println("  -XX:SurvivorRatio=8 (Eden:Survivor = 8:1)");
        
        System.out.println("\nGC Triggering:");
        System.out.println("  -XX:MaxTenuringThreshold=15 (Object promotion threshold)");
        System.out.println("  -XX:TargetSurvivorRatio=50 (Target survivor space usage)");
        
        System.out.println("\nGC Logging:");
        System.out.println("  -Xlog:gc*:file=gc.log:time,tags (Java 9+)");
        System.out.println("  -XX:+PrintGC (Java 8 and earlier)");
        System.out.println("  -XX:+PrintGCDetails");
        System.out.println("  -XX:+PrintGCTimeStamps");
    }
    
    private static void demonstratePerformanceMonitoring() {
        System.out.println("\n=== Performance Monitoring ===");
        
        System.out.println("JVM monitoring tools and techniques:");
        System.out.println("1. JConsole - Built-in JVM monitoring tool");
        System.out.println("2. VisualVM - Enhanced monitoring and profiling");
        System.out.println("3. JMC (Java Mission Control) - Advanced profiling");
        System.out.println("4. JMH (Java Microbenchmark Harness) - Benchmarking");
        
        // Demonstrate simple performance monitoring
        demonstrateSimpleMonitoring();
    }
    
    private static void demonstrateSimpleMonitoring() {
        System.out.println("\nSimple performance monitoring example:");
        
        // Monitor memory usage
        Runtime runtime = Runtime.getRuntime();
        long initialMemory = runtime.totalMemory() - runtime.freeMemory();
        
        // Perform some work
        List<PerformanceTestObject> objects = new ArrayList<>();
        for (int i = 0; i < 100000; i++) {
            objects.add(new PerformanceTestObject(i));
        }
        
        long afterAllocation = runtime.totalMemory() - runtime.freeMemory();
        
        // Clear references
        objects.clear();
        System.gc();
        
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        long afterGC = runtime.totalMemory() - runtime.freeMemory();
        
        System.out.printf("Memory usage - Initial: %d KB, After allocation: %d KB, After GC: %d KB%n",
            initialMemory / 1024, afterAllocation / 1024, afterGC / 1024);
        
        // Monitor time performance
        long startTime = System.nanoTime();
        performTimedOperation();
        long endTime = System.nanoTime();
        
        System.out.printf("Operation time: %d ns (%.2f ms)%n",
            endTime - startTime, (endTime - startTime) / 1_000_000.0);
    }
    
    private static void performTimedOperation() {
        // Simulate some work
        for (int i = 0; i < 1000000; i++) {
            Math.sqrt(i);
        }
    }
}

class LargeObject {
    private byte[] data;
    private String id;
    
    public LargeObject(int size) {
        this.data = new byte[size];
        this.id = UUID.randomUUID().toString();
        Arrays.fill(data, (byte) (size % 256));
    }
    
    public String getId() {
        return id;
    }
}

class PerformanceTestObject {
    private int id;
    private String data;
    
    public PerformanceTestObject(int id) {
        this.id = id;
        this.data = "PerformanceTestObject-" + id;
    }
    
    public int getId() {
        return id;
    }
}
```

## Summary

Java memory management is a complex but crucial aspect of Java application development. This guide covered:

1. **JVM Memory Model**: Understanding the different memory areas (heap, stack, method area, etc.)
2. **Garbage Collection**: How different GC algorithms work and when to use them
3. **Memory Profiling**: Techniques for monitoring and analyzing memory usage
4. **Optimization Techniques**: Best practices for efficient memory usage
5. **Memory Leak Prevention**: Common causes and prevention strategies
6. **JVM Tuning**: How to optimize JVM settings for better performance

Key takeaways:
- Understand the JVM memory model to write more efficient code
- Choose the right garbage collector for your application's needs
- Monitor memory usage to identify performance bottlenecks
- Prevent memory leaks through proper resource management
- Use appropriate data structures and optimization techniques
- Tune JVM parameters for optimal performance

Mastering Java memory management leads to more robust, efficient, and scalable applications.