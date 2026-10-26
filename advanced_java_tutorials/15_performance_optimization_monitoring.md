# Advanced Performance Optimization and Monitoring

## Overview
Performance optimization and monitoring are critical for building high-performance, scalable Java applications. This tutorial covers JVM optimization, application-level performance techniques, monitoring strategies, and profiling tools.

## JVM Performance Tuning

### Memory Management and Garbage Collection

```java
import java.lang.management.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;
import java.util.stream.IntStream;

public class JvmPerformanceTuning {
    
    // Memory monitoring utility
    public static class MemoryMonitor {
        private final MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        private final AtomicLong totalCollections = new AtomicLong(0);
        private final AtomicLong totalCollectionTime = new AtomicLong(0);
        
        public void printMemoryUsage() {
            MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
            MemoryUsage nonHeapUsage = memoryBean.getNonHeapMemoryUsage();
            
            System.out.println("=== Memory Usage ===");
            System.out.println("Heap: " + formatBytes(heapUsage.getUsed()) + 
                             " / " + formatBytes(heapUsage.getMax()));
            System.out.println("Non-Heap: " + formatBytes(nonHeapUsage.getUsed()) + 
                             " / " + formatBytes(nonHeapUsage.getMax()));
            
            // GC statistics
            for (GarbageCollectorMXBean gcBean : ManagementFactory.getGarbageCollectorMXBeans()) {
                long collections = gcBean.getCollectionCount();
                long time = gcBean.getCollectionTime();
                
                System.out.println("GC [" + gcBean.getName() + "]: " + 
                                 collections + " collections, " + time + "ms total");
                
                totalCollections.addAndGet(collections);
                totalCollectionTime.addAndGet(time);
            }
        }
        
        public double getAverageGCTime() {
            long collections = totalCollections.get();
            long time = totalCollectionTime.get();
            return collections > 0 ? (double) time / collections : 0.0;
        }
        
        private String formatBytes(long bytes) {
            if (bytes < 1024) return bytes + " B";
            if (bytes < 1024 * 1024) return String.format("%.2f KB", bytes / 1024.0);
            if (bytes < 1024 * 1024 * 1024) return String.format("%.2f MB", bytes / (1024.0 * 1024));
            return String.format("%.2f GB", bytes / (1024.0 * 1024 * 1024));
        }
    }
    
    // GC tuning example
    public static class GcTuningExample {
        private final MemoryMonitor monitor = new MemoryMonitor();
        private final ExecutorService executor = Executors.newFixedThreadPool(10);
        
        public void demonstrateGcBehavior() {
            System.out.println("Starting GC behavior demonstration...");
            
            // Create memory pressure
            for (int i = 0; i < 1000; i++) {
                if (i % 100 == 0) {
                    monitor.printMemoryUsage();
                }
                
                // Create temporary objects that will be garbage collected
                createTemporaryObjects();
                
                // Submit tasks to create more objects
                executor.submit(() -> {
                    String largeString = createLargeString();
                    processString(largeString);
                    // String will be eligible for GC after method completion
                });
            }
            
            executor.shutdown();
            try {
                executor.awaitTermination(10, TimeUnit.SECONDS);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            
            System.gc(); // Suggest GC (not guaranteed to run immediately)
            monitor.printMemoryUsage();
        }
        
        private void createTemporaryObjects() {
            // Create objects that will be eligible for GC
            for (int i = 0; i < 1000; i++) {
                new Object(); // No reference kept, eligible for GC
            }
        }
        
        private String createLargeString() {
            StringBuilder sb = new StringBuilder(10000);
            for (int i = 0; i < 10000; i++) {
                sb.append((char) ('A' + (i % 26)));
            }
            return sb.toString();
        }
        
        private void processString(String str) {
            // Simple processing that doesn't keep string reference
            int length = str.length();
            int hash = str.hashCode();
            // Both length and hash are primitives, no string reference maintained
        }
    }
    
    // Object pooling example
    public static class ObjectPool<T> {
        private final Queue<T> pool = new ConcurrentLinkedQueue<>();
        private final Supplier<T> factory;
        private final Consumer<T> resetter;
        private final int maxSize;
        
        public ObjectPool(Supplier<T> factory, Consumer<T> resetter, int maxSize) {
            this.factory = factory;
            this.resetter = resetter;
            this.maxSize = maxSize;
        }
        
        public T acquire() {
            T object = pool.poll();
            if (object == null) {
                object = factory.get();
            }
            return object;
        }
        
        public void release(T object) {
            if (object != null && pool.size() < maxSize) {
                resetter.accept(object);
                pool.offer(object);
            }
        }
        
        public int getCurrentSize() {
            return pool.size();
        }
    }
    
    // Example pooled object
    public static class PooledBuffer {
        private byte[] buffer;
        private int position;
        
        public PooledBuffer(int size) {
            this.buffer = new byte[size];
        }
        
        public void reset() {
            position = 0;
            // Don't recreate buffer, just reset position
        }
        
        public byte[] getBuffer() {
            return buffer;
        }
        
        public int getPosition() {
            return position;
        }
        
        public void setPosition(int position) {
            this.position = position;
        }
    }
}
```

### JVM Flags and Tuning Parameters

```java
// JVM optimization demonstration
public class JvmOptimizationFlags {
    
    // Memory-related flags demonstration
    public static class MemoryOptimization {
        
        // Use -Xms and -Xmx to set initial and maximum heap size
        // Example: java -Xms2g -Xmx4g YourApplication
        
        // Young generation tuning
        // -XX:NewRatio=2 (old:young = 2:1)
        // -XX:SurvivorRatio=8 (eden:survivor = 8:1)
        // -XX:MaxTenuringThreshold=15
        
        public void demonstrateMemoryAllocation() {
            // Monitor allocation patterns
            MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
            
            long initialUsed = memoryBean.getHeapMemoryUsage().getUsed();
            System.out.println("Initial heap used: " + initialUsed);
            
            // Create objects in different generations
            createShortLivedObjects();
            createLongLivedObjects();
            
            long afterAllocation = memoryBean.getHeapMemoryUsage().getUsed();
            System.out.println("After allocation: " + afterAllocation);
            System.out.println("Difference: " + (afterAllocation - initialUsed));
        }
        
        private void createShortLivedObjects() {
            // These objects should die in young generation
            for (int i = 0; i < 100000; i++) {
                new String("Short-lived object " + i); // Will be GC'd quickly
            }
        }
        
        private void createLongLivedObjects() {
            // These objects will be promoted to old generation
            List<String> longLivedObjects = new ArrayList<>();
            for (int i = 0; i < 10000; i++) {
                longLivedObjects.add("Long-lived object " + i);
            }
            // Keep reference to promote to old generation
        }
    }
    
    // Garbage Collector selection and tuning
    public static class GcTuning {
        
        // G1GC tuning parameters
        // -XX:+UseG1GC
        // -XX:MaxGCPauseMillis=200
        // -XX:G1HeapRegionSize=16m
        // -XX:G1NewSizePercent=15
        // -XX:G1MaxNewSizePercent=30
        
        // Parallel GC (throughput)
        // -XX:+UseParallelGC
        // -XX:ParallelGCThreads=8
        
        // ZGC (low latency, large heaps)
        // -XX:+UseZGC
        // -Xmx32g
        
        public void demonstrateGcImpact() {
            // Create different allocation patterns to see GC behavior
            ExecutorService executor = Executors.newFixedThreadPool(20);
            
            List<Future<?>> futures = new ArrayList<>();
            
            // High allocation rate task
            futures.add(executor.submit(() -> {
                for (int i = 0; i < 1000000; i++) {
                    byte[] data = new byte[1024]; // 1KB allocation
                    data[0] = (byte) (i % 256); // Prevent dead code elimination
                }
            }));
            
            // Long-lived object creation
            List<byte[]> longLived = Collections.synchronizedList(new ArrayList<>());
            futures.add(executor.submit(() -> {
                for (int i = 0; i < 100000; i++) {
                    byte[] data = new byte[10240]; // 10KB allocation
                    data[0] = (byte) (i % 256);
                    longLived.add(data);
                    if (longLived.size() > 1000) {
                        longLived.remove(0); // Prevent OOM
                    }
                }
            }));
            
            // Wait for completion
            futures.forEach(future -> {
                try {
                    future.get();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });
            
            executor.shutdown();
        }
    }
    
    // JIT compiler optimization
    public static class JitOptimization {
        
        // JIT compilation flags
        // -XX:CompileThreshold=10000 (default hotness threshold)
        // -XX:+TieredCompilation (enable tiered compilation)
        // -XX:TieredStopAtLevel=1 (disable C2 compiler for testing)
        
        private int hotMethodCounter = 0;
        
        public void demonstrateJitOptimization() {
            System.out.println("Demonstrating JIT optimization...");
            
            // This method will be called frequently to trigger JIT compilation
            long startTime = System.nanoTime();
            
            // Warm-up phase - JIT compiler will optimize after several calls
            for (int i = 0; i < 100000; i++) {
                hotMethod();
            }
            
            long warmupTime = System.nanoTime();
            
            // Measuring optimized performance
            for (int i = 0; i < 100000; i++) {
                hotMethod();
            }
            
            long optimizedTime = System.nanoTime();
            
            System.out.println("Warm-up time: " + (warmupTime - startTime) / 1_000_000 + "ms");
            System.out.println("Optimized time: " + (optimizedTime - warmupTime) / 1_000_000 + "ms");
        }
        
        // This method will be optimized by JIT after many calls
        public int hotMethod() {
            hotMethodCounter++;
            // Simple computation that JIT can optimize
            int result = hotMethodCounter * 2;
            result = result + 5;
            result = result / 2;
            return result;
        }
        
        // Loop optimization example
        public long optimizedLoop() {
            long sum = 0;
            // JIT will likely optimize this loop
            for (int i = 0; i < 1000000; i++) {
                sum += i * 2;
            }
            return sum;
        }
    }
}
```

## Application-Level Performance Optimization

### Efficient Data Structures and Algorithms

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.stream.*;

public class DataStructureOptimization {
    
    // Efficient collection usage patterns
    public static class CollectionOptimization {
        
        // Choose the right data structure for the job
        public void demonstrateCollectionSelection() {
            // Use ArrayList for frequent indexed access
            List<Integer> arrayList = new ArrayList<>();
            for (int i = 0; i < 1000; i++) {
                arrayList.add(i);
            }
            
            long startTime = System.nanoTime();
            for (int i = 0; i < 1000; i++) {
                arrayList.get(i); // O(1) access
            }
            long arrayListTime = System.nanoTime() - startTime;
            
            // LinkedList for frequent insertions/deletions in middle
            List<Integer> linkedList = new LinkedList<>();
            for (int i = 0; i < 1000; i++) {
                linkedList.add(i);
            }
            
            startTime = System.nanoTime();
            for (int i = 0; i < 100; i++) {
                linkedList.add(500, i + 1000); // Insert in middle
            }
            long linkedListTime = System.nanoTime() - startTime;
            
            System.out.println("ArrayList indexed access time: " + arrayListTime + "ns");
            System.out.println("LinkedList middle insertion time: " + linkedListTime + "ns");
        }
        
        // Efficient Map usage
        public void demonstrateMapOptimization() {
            // Use HashMap for general purpose
            Map<String, Object> hashMap = new HashMap<>();
            
            // Pre-size HashMap to avoid rehashing
            Map<String, Object> preSizedMap = new HashMap<>(1000, 0.75f);
            
            // Use ConcurrentHashMap for thread safety
            Map<String, Object> concurrentMap = new ConcurrentHashMap<>();
            
            // Use LinkedHashMap for access order iteration
            Map<String, Object> accessOrderMap = new LinkedHashMap<>(16, 0.75f, true) {
                @Override
                protected boolean removeEldestEntry(Map.Entry<String, Object> eldest) {
                    return size() > 1000; // LRU cache behavior
                }
            };
            
            // Performance test
            long startTime = System.nanoTime();
            for (int i = 0; i < 100000; i++) {
                hashMap.put("key" + i, "value" + i);
            }
            long putTime = System.nanoTime() - startTime;
            
            startTime = System.nanoTime();
            for (int i = 0; i < 100000; i++) {
                hashMap.get("key" + i);
            }
            long getTime = System.nanoTime() - startTime;
            
            System.out.println("HashMap put time: " + putTime + "ns");
            System.out.println("HashMap get time: " + getTime + "ns");
        }
        
        // Set optimization
        public void demonstrateSetOptimization() {
            // Use HashSet for general purpose
            Set<String> hashSet = new HashSet<>();
            
            // Use LinkedHashSet for insertion order
            Set<String> linkedHashSet = new LinkedHashSet<>();
            
            // Use TreeSet for sorted order
            Set<String> treeSet = new TreeSet<>();
            
            // EnumSet for enum types (very efficient)
            Set<DayOfWeek> daySet = EnumSet.of(DayOfWeek.MONDAY, DayOfWeek.WEDNESDAY);
            
            // Performance comparison
            List<Set<Integer>> sets = Arrays.asList(
                new HashSet<>(),
                new LinkedHashSet<>(),
                new TreeSet<>()
            );
            
            for (Set<Integer> set : sets) {
                long startTime = System.nanoTime();
                for (int i = 0; i < 100000; i++) {
                    set.add(i);
                }
                long addTime = System.nanoTime() - startTime;
                
                startTime = System.nanoTime();
                for (int i = 0; i < 100000; i++) {
                    set.contains(i);
                }
                long containsTime = System.nanoTime() - startTime;
                
                System.out.println(set.getClass().getSimpleName() + 
                                 " add time: " + addTime + "ns, " +
                                 "contains time: " + containsTime + "ns");
            }
        }
    }
    
    // String optimization
    public static class StringOptimization {
        
        public void demonstrateStringOptimization() {
            // String concatenation performance
            int iterations = 100000;
            
            // Inefficient: creates many temporary String objects
            long startTime = System.nanoTime();
            String result1 = "";
            for (int i = 0; i < iterations; i++) {
                result1 += "a"; // Creates new String each time
            }
            long stringConcatTime = System.nanoTime() - startTime;
            
            // Efficient: StringBuilder
            startTime = System.nanoTime();
            StringBuilder sb = new StringBuilder(iterations);
            for (int i = 0; i < iterations; i++) {
                sb.append("a");
            }
            String result2 = sb.toString();
            long stringBuilderTime = System.nanoTime() - startTime;
            
            // Most efficient: String.join for known elements
            startTime = System.nanoTime();
            List<String> elements = new ArrayList<>();
            for (int i = 0; i < iterations; i++) {
                elements.add("a");
            }
            String result3 = String.join("", elements);
            long stringJoinTime = System.nanoTime() - startTime;
            
            System.out.println("String concatenation time: " + stringConcatTime + "ns");
            System.out.println("StringBuilder time: " + stringBuilderTime + "ns");
            System.out.println("String.join time: " + stringJoinTime + "ns");
        }
        
        // String interning optimization
        public void demonstrateStringInterning() {
            // Use string interning for frequently compared strings
            String[] strings = new String[100000];
            
            // Without interning
            long startTime = System.nanoTime();
            for (int i = 0; i < strings.length; i++) {
                strings[i] = new String("test"); // Creates new String object
            }
            long withoutInternTime = System.nanoTime() - startTime;
            
            // With interning
            startTime = System.nanoTime();
            for (int i = 0; i < strings.length; i++) {
                strings[i] = new String("test").intern(); // Interned string
            }
            long withInternTime = System.nanoTime() - startTime;
            
            System.out.println("Without interning: " + withoutInternTime + "ns");
            System.out.println("With interning: " + withInternTime + "ns");
            
            // String comparison performance
            long comparisonTime = measureStringComparison(strings, false);
            long internComparisonTime = measureStringComparison(strings, true);
            
            System.out.println("Reference comparison time: " + comparisonTime + "ns");
            System.out.println("Interned comparison time: " + internComparisonTime + "ns");
        }
        
        private long measureStringComparison(String[] strings, boolean intern) {
            long startTime = System.nanoTime();
            int matches = 0;
            
            for (int i = 0; i < strings.length - 1; i++) {
                String current = intern ? strings[i].intern() : strings[i];
                String next = intern ? strings[i + 1].intern() : strings[i + 1];
                
                if (current == next) { // Reference comparison
                    matches++;
                }
            }
            
            return System.nanoTime() - startTime;
        }
    }
    
    // Stream optimization
    public static class StreamOptimization {
        
        public void demonstrateStreamOptimization() {
            List<Integer> numbers = IntStream.range(0, 1000000)
                .boxed()
                .collect(Collectors.toList());
            
            // Sequential stream
            long startTime = System.nanoTime();
            long sequentialResult = numbers.stream()
                .filter(n -> n % 2 == 0)
                .mapToLong(n -> n * n)
                .sum();
            long sequentialTime = System.nanoTime() - startTime;
            
            // Parallel stream
            startTime = System.nanoTime();
            long parallelResult = numbers.parallelStream()
                .filter(n -> n % 2 == 0)
                .mapToLong(n -> n * n)
                .sum();
            long parallelTime = System.nanoTime() - startTime;
            
            System.out.println("Sequential result: " + sequentialResult + ", time: " + sequentialTime + "ns");
            System.out.println("Parallel result: " + parallelResult + ", time: " + parallelTime + "ns");
            
            // Stream chaining optimization
            startTime = System.nanoTime();
            List<Integer> result = numbers.stream()
                .filter(n -> n > 100)
                .filter(n -> n % 3 == 0) // Can be combined with above
                .map(n -> n * 2)
                .filter(n -> n > 500) // Filter early to reduce data
                .limit(1000) // Limit early if possible
                .collect(Collectors.toList());
            long chainedTime = System.nanoTime() - startTime;
            
            System.out.println("Chained operations time: " + chainedTime + "ns, size: " + result.size());
        }
    }
}
```

### Caching Strategies

```java
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import com.github.benmanes.caffeine.cache.LoadingCache;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

public class CachingStrategies {
    
    // Caffeine cache implementation
    public static class CaffeineCacheExample {
        private final LoadingCache<String, String> cache = Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(Duration.ofMinutes(10))
            .expireAfterAccess(Duration.ofMinutes(5))
            .recordStats()
            .build(key -> loadFromDatabase(key));
        
        private final Cache<String, String> asyncCache = Caffeine.newBuilder()
            .maximumSize(1000)
            .executor(Runnable::run) // Use calling thread for async operations
            .build();
        
        public String getCachedValue(String key) {
            return cache.get(key);
        }
        
        public void putValue(String key, String value) {
            cache.put(key, value);
        }
        
        public void putAsyncValue(String key, String value) {
            asyncCache.put(key, value);
        }
        
        private String loadFromDatabase(String key) {
            // Simulate database call
            try {
                Thread.sleep(100); // Simulate network delay
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "Data for " + key;
        }
        
        public void demonstrateCachePerformance() {
            // Warm up cache
            long startTime = System.nanoTime();
            for (int i = 0; i < 1000; i++) {
                getCachedValue("key" + i);
            }
            long cacheTime = System.nanoTime() - startTime;
            
            // Compare with database calls
            startTime = System.nanoTime();
            for (int i = 0; i < 1000; i++) {
                loadFromDatabase("key" + i);
            }
            long databaseTime = System.nanoTime() - startTime;
            
            System.out.println("Cache access time: " + cacheTime / 1_000_000 + "ms");
            System.out.println("Database access time: " + databaseTime / 1_000_000 + "ms");
            System.out.println("Cache is " + (double) databaseTime / cacheTime + "x faster");
        }
        
        // Cache statistics
        public void printCacheStats() {
            var stats = cache.stats();
            System.out.println("Cache stats:");
            System.out.println("  Hit rate: " + stats.hitRate());
            System.out.println("  Request count: " + stats.requestCount());
            System.out.println("  Hit count: " + stats.hitCount());
            System.out.println("  Miss count: " + stats.missCount());
        }
    }
    
    // Spring Cache annotations example
    @Service
    public static class UserService {
        
        @Cacheable(value = "users", key = "#id", unless = "#result == null")
        public User findById(Long id) {
            // Expensive database operation
            return userRepository.findById(id);
        }
        
        @CachePut(value = "users", key = "#result.id")
        public User save(User user) {
            User saved = userRepository.save(user);
            return saved;
        }
        
        @CacheEvict(value = "users", key = "#id")
        public void delete(Long id) {
            userRepository.deleteById(id);
        }
        
        @Cacheable(value = "users", key = "#email", condition = "#email.length() > 5")
        public User findByEmail(String email) {
            return userRepository.findByEmail(email);
        }
        
        // Complex cache key
        @Cacheable(value = "user-orders", 
                  key = "{#userId, #status, #page, #size}")
        public List<Order> getUserOrders(Long userId, String status, int page, int size) {
            return orderRepository.findByUserIdAndStatus(userId, status, page, size);
        }
        
        // Cache synchronization
        @Cacheable(value = "expensive-computation", sync = true)
        public Result expensiveComputation(Parameters params) {
            // Only one thread will execute this method for the same parameters
            return performExpensiveOperation(params);
        }
    }
    
    // Custom cache implementation
    public static class CustomCache<K, V> {
        private final Map<K, CacheEntry<V>> cache = new ConcurrentHashMap<>();
        private final long ttlMillis;
        private final int maxSize;
        private final EvictionPolicy<K, V> evictionPolicy;
        
        public CustomCache(long ttlMillis, int maxSize, EvictionPolicy<K, V> evictionPolicy) {
            this.ttlMillis = ttlMillis;
            this.maxSize = maxSize;
            this.evictionPolicy = evictionPolicy;
        }
        
        public V get(K key) {
            CacheEntry<V> entry = cache.get(key);
            if (entry == null) {
                return null;
            }
            
            if (isExpired(entry)) {
                cache.remove(key);
                return null;
            }
            
            evictionPolicy.onAccess(key, entry);
            return entry.getValue();
        }
        
        public void put(K key, V value) {
            CacheEntry<V> newEntry = new CacheEntry<>(value, System.currentTimeMillis());
            cache.put(key, newEntry);
            evictionPolicy.onPut(key, newEntry);
            
            if (cache.size() > maxSize) {
                K keyToEvict = evictionPolicy.selectKeyToEvict(cache.keySet());
                if (keyToEvict != null) {
                    cache.remove(keyToEvict);
                }
            }
        }
        
        private boolean isExpired(CacheEntry<V> entry) {
            return System.currentTimeMillis() - entry.getTimestamp() > ttlMillis;
        }
        
        // Eviction policies
        public interface EvictionPolicy<K, V> {
            void onAccess(K key, CacheEntry<V> entry);
            void onPut(K key, CacheEntry<V> entry);
            K selectKeyToEvict(Set<K> keys);
        }
        
        public static class LruEvictionPolicy<K, V> implements EvictionPolicy<K, V> {
            private final Map<K, Long> accessTimes = new ConcurrentHashMap<>();
            
            @Override
            public void onAccess(K key, CacheEntry<V> entry) {
                accessTimes.put(key, System.currentTimeMillis());
            }
            
            @Override
            public void onPut(K key, CacheEntry<V> entry) {
                accessTimes.put(key, System.currentTimeMillis());
            }
            
            @Override
            public K selectKeyToEvict(Set<K> keys) {
                return keys.stream()
                    .min(Comparator.comparing(accessTimes::get))
                    .orElse(null);
            }
        }
        
        // Cache entry with timestamp
        private static class CacheEntry<V> {
            private final V value;
            private final long timestamp;
            
            public CacheEntry(V value, long timestamp) {
                this.value = value;
                this.timestamp = timestamp;
            }
            
            public V getValue() { return value; }
            public long getTimestamp() { return timestamp; }
        }
    }
}
```

## Monitoring and Profiling

### Application Performance Monitoring (APM)

```java
import io.micrometer.core.instrument.*;
import io.micrometer.core.instrument.binder.jvm.*;
import io.micrometer.core.instrument.binder.system.*;
import io.micrometer.prometheus.PrometheusConfig;
import io.micrometer.prometheus.PrometheusMeterRegistry;
import org.springframework.boot.actuator.metrics.MeterRegistryCustomizer;
import org.springframework.context.annotation.Bean;

public class ApplicationMonitoring {
    
    // Micrometer metrics setup
    public static class MetricsConfiguration {
        
        private final MeterRegistry meterRegistry;
        
        public MetricsConfiguration() {
            this.meterRegistry = new PrometheusMeterRegistry(PrometheusConfig.DEFAULT);
            
            // Register common JVM metrics
            new JvmMemoryMetrics().bindTo(meterRegistry);
            new JvmGcMetrics().bindTo(meterRegistry);
            new ProcessorMetrics().bindTo(meterRegistry);
            new UptimeMetrics().bindTo(meterRegistry);
            new FileDescriptorMetrics().bindTo(meterRegistry);
        }
        
        // Custom metrics
        public void recordBusinessMetrics(String operation, long duration, String status) {
            Timer.Sample sample = Timer.start(meterRegistry);
            
            sample.stop(Timer.builder("business.operation")
                .tag("operation", operation)
                .tag("status", status)
                .register(meterRegistry));
            
            // Record execution time
            Timer.builder("business.operation.duration")
                .tag("operation", operation)
                .tag("status", status)
                .register(meterRegistry)
                .record(duration, TimeUnit.MILLISECONDS);
        }
        
        // Counter for business events
        public void incrementBusinessCounter(String eventType) {
            Counter.builder("business.events")
                .tag("type", eventType)
                .register(meterRegistry)
                .increment();
        }
        
        // Gauge for current state
        public void registerGauge(String name, Supplier<Number> valueSupplier) {
            Gauge.builder(name)
                .register(meterRegistry, valueSupplier);
        }
        
        // Distribution summary for value distribution
        public void recordValueDistribution(String name, double value) {
            DistributionSummary.builder(name)
                .register(meterRegistry)
                .record(value);
        }
        
        public MeterRegistry getMeterRegistry() {
            return meterRegistry;
        }
    }
    
    // Service with built-in monitoring
    @Service
    public static class MonitoredUserService {
        
        private final MeterRegistry meterRegistry;
        private final Timer userOperationTimer;
        private final Counter userOperationCounter;
        private final DistributionSummary userResponseSize;
        
        public MonitoredUserService(MeterRegistry meterRegistry) {
            this.meterRegistry = meterRegistry;
            
            this.userOperationTimer = Timer.builder("user.operation.time")
                .description("Time taken for user operations")
                .register(meterRegistry);
                
            this.userOperationCounter = Counter.builder("user.operations")
                .description("Count of user operations")
                .register(meterRegistry);
                
            this.userResponseSize = DistributionSummary.builder("user.response.size")
                .baseUnit("bytes")
                .register(meterRegistry);
        }
        
        @Timed(value = "user.find", description = "Time to find user", percentiles = {0.5, 0.95, 0.99})
        public User findUser(Long userId) {
            return userOperationTimer.recordCallable(() -> {
                userOperationCounter.increment();
                
                User user = userRepository.findById(userId);
                
                if (user != null) {
                    userResponseSize.record(user.toString().length());
                }
                
                return user;
            });
        }
        
        @Counted(value = "user.create", description = "User creation attempts")
        public User createUser(User user) {
            return userOperationTimer.recordCallable(() -> {
                User createdUser = userRepository.save(user);
                userResponseSize.record(createdUser.toString().length());
                return createdUser;
            });
        }
        
        // Error rate monitoring
        public User findUserWithErrorHandling(Long userId) {
            try {
                return findUser(userId);
            } catch (Exception e) {
                Counter.builder("user.operation.errors")
                    .tag("operation", "find")
                    .tag("error", e.getClass().getSimpleName())
                    .register(meterRegistry)
                    .increment();
                throw e;
            }
        }
    }
    
    // Health check monitoring
    @Component
    public static class CustomHealthIndicator implements HealthIndicator {
        
        private final MeterRegistry meterRegistry;
        
        public CustomHealthIndicator(MeterRegistry meterRegistry) {
            this.meterRegistry = meterRegistry;
        }
        
        @Override
        public Health health() {
            long startTime = System.currentTimeMillis();
            
            try {
                // Check critical service dependencies
                boolean databaseHealthy = checkDatabaseHealth();
                boolean externalApiHealthy = checkExternalApiHealth();
                
                Health.Builder builder = Health.up()
                    .withDetail("database", databaseHealthy)
                    .withDetail("externalApi", externalApiHealthy);
                
                long responseTime = System.currentTimeMillis() - startTime;
                
                // Record health check metrics
                Timer.builder("health.check.duration")
                    .tag("component", "custom")
                    .register(meterRegistry)
                    .record(responseTime, TimeUnit.MILLISECONDS);
                
                if (databaseHealthy && externalApiHealthy) {
                    return builder.build();
                } else {
                    return Health.outOfService()
                        .withDetail("database", databaseHealthy)
                        .withDetail("externalApi", externalApiHealthy)
                        .build();
                }
            } catch (Exception e) {
                return Health.down()
                    .withException(e)
                    .build();
            }
        }
        
        private boolean checkDatabaseHealth() {
            try {
                // Perform quick database operation
                return userRepository.count() >= 0;
            } catch (Exception e) {
                return false;
            }
        }
        
        private boolean checkExternalApiHealth() {
            try {
                // Check external service health
                return externalApiClient.healthCheck();
            } catch (Exception e) {
                return false;
            }
        }
    }
    
    // Spring Boot Actuator configuration
    @Configuration
    public static class ActuatorConfig {
        
        @Bean
        public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
            return registry -> registry.config().commonTags("application", "user-service");
        }
        
        @Bean
        public TimedAspect timedAspect(MeterRegistry meterRegistry) {
            return new TimedAspect(meterRegistry);
        }
    }
}
```

### Distributed Tracing and Logging

```java
import brave.Tracing;
import brave.sampler.Sampler;
import org.springframework.cloud.sleuth.Tracer;
import org.springframework.cloud.sleuth.Span;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import ch.qos.logback.classic.LoggerContext;
import ch.qos.logback.core.rolling.RollingFileAppender;
import net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder;

public class DistributedTracingLogging {
    
    // Distributed tracing with Sleuth
    @Service
    public static class TracedUserService {
        
        private static final Logger log = LoggerFactory.getLogger(TracedUserService.class);
        
        private final Tracer tracer;
        private final RestTemplate restTemplate;
        
        public TracedUserService(Tracer tracer, RestTemplate restTemplate) {
            this.tracer = tracer;
            this.restTemplate = restTemplate;
        }
        
        public User findUserWithTracing(Long userId) {
            // Create a new span for this operation
            Span span = tracer.nextSpan().name("find-user-operation").start();
            
            try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
                log.info("Starting to find user with ID: {}", userId);
                
                // Add custom tags to the span
                span.tag("user.id", userId.toString());
                span.tag("operation", "find-user");
                
                // Call external service (trace will continue automatically)
                User user = userRepository.findById(userId);
                
                if (user != null) {
                    span.tag("user.found", "true");
                    log.info("Successfully found user: {}", user.getUsername());
                } else {
                    span.tag("user.found", "false");
                    log.warn("User not found with ID: {}", userId);
                }
                
                return user;
            } catch (Exception e) {
                span.tag("error", e.getMessage());
                span.tag("error.type", e.getClass().getSimpleName());
                log.error("Error finding user with ID: {}", userId, e);
                throw e;
            } finally {
                span.end();
            }
        }
        
        public void processUserOrder(Long userId, Order order) {
            // Continue existing trace or create new one
            Span parentSpan = tracer.currentSpan();
            
            Span processOrderSpan = tracer.nextSpan()
                .name("process-user-order")
                .start();
            
            try (Tracer.SpanInScope ws = tracer.withSpanInScope(processOrderSpan)) {
                log.info("Processing order for user: {}", userId);
                
                // Tag with business information
                processOrderSpan.tag("user.id", userId.toString());
                processOrderSpan.tag("order.id", order.getId().toString());
                processOrderSpan.tag("order.amount", order.getAmount().toString());
                
                // Call payment service
                Span paymentSpan = tracer.nextSpan()
                    .name("payment-processing")
                    .start();
                
                try (Tracer.SpanInScope paymentWs = tracer.withSpanInScope(paymentSpan)) {
                    paymentSpan.tag("payment.amount", order.getAmount().toString());
                    processPayment(order);
                } finally {
                    paymentSpan.end();
                }
                
                // Call notification service
                Span notificationSpan = tracer.nextSpan()
                    .name("send-notification")
                    .start();
                
                try (Tracer.SpanInScope notifyWs = tracer.withSpanInScope(notificationSpan)) {
                    sendNotification(userId, "Order processed successfully");
                } finally {
                    notificationSpan.end();
                }
                
            } finally {
                processOrderSpan.end();
            }
        }
        
        private void processPayment(Order order) {
            // Payment processing logic
        }
        
        private void sendNotification(Long userId, String message) {
            // Notification sending logic
        }
    }
    
    // Structured logging with MDC
    @Service
    public static class StructuredLoggingService {
        
        private static final Logger log = LoggerFactory.getLogger(StructuredLoggingService.class);
        
        public void processUserRequest(Long userId, String operationType) {
            // Add context to MDC
            MDC.put("userId", userId.toString());
            MDC.put("operation", operationType);
            MDC.put("timestamp", Instant.now().toString());
            
            try {
                log.info("Starting user operation");
                
                // Perform operation
                performOperation(userId);
                
                log.info("Operation completed successfully");
            } catch (Exception e) {
                log.error("Operation failed", e);
            } finally {
                // Clean up MDC
                MDC.clear();
            }
        }
        
        private void performOperation(Long userId) {
            // Operation logic here
        }
    }
    
    // Logback configuration for structured logging
    public static class LogbackConfig {
        
        public void configureStructuredLogging() {
            LoggerContext context = (LoggerContext) LoggerFactory.getILoggerFactory();
            
            // Create JSON encoder for structured logging
            LoggingEventCompositeJsonEncoder jsonEncoder = new LoggingEventCompositeJsonEncoder();
            
            // Configure rolling file appender with JSON
            RollingFileAppender<ILoggingEvent> fileAppender = new RollingFileAppender<>();
            fileAppender.setContext(context);
            fileAppender.setName("JSON_FILE");
            fileAppender.setFile("logs/app.log");
            
            // Set encoder and start appender
            fileAppender.setEncoder(jsonEncoder);
            fileAppender.start();
            
            // Add appender to root logger
            ch.qos.logback.classic.Logger rootLogger = 
                context.getLogger(Logger.ROOT_LOGGER_NAME);
            rootLogger.addAppender(fileAppender);
        }
    }
    
    // Performance monitoring with custom metrics
    @Service
    public static class PerformanceMonitoringService {
        
        private final MeterRegistry meterRegistry;
        private final Timer.Sample.SampleContext sampleContext;
        
        public PerformanceMonitoringService(MeterRegistry meterRegistry) {
            this.meterRegistry = meterRegistry;
        }
        
        public <T> T withPerformanceMonitoring(String operationName, Supplier<T> operation) {
            Timer.Sample sample = Timer.start(meterRegistry);
            
            try {
                T result = operation.get();
                
                sample.stop(Timer.builder("application.operation.duration")
                    .tag("operation", operationName)
                    .tag("status", "success")
                    .register(meterRegistry));
                    
                Counter.builder("application.operation.count")
                    .tag("operation", operationName)
                    .tag("status", "success")
                    .register(meterRegistry)
                    .increment();
                
                return result;
            } catch (Exception e) {
                sample.stop(Timer.builder("application.operation.duration")
                    .tag("operation", operationName)
                    .tag("status", "error")
                    .tag("error", e.getClass().getSimpleName())
                    .register(meterRegistry));
                    
                Counter.builder("application.operation.count")
                    .tag("operation", operationName)
                    .tag("status", "error")
                    .tag("error", e.getClass().getSimpleName())
                    .register(meterRegistry)
                    .increment();
                
                throw e;
            }
        }
        
        // Memory usage monitoring
        public void logMemoryUsage(String context) {
            Runtime runtime = Runtime.getRuntime();
            long totalMemory = runtime.totalMemory();
            long freeMemory = runtime.freeMemory();
            long usedMemory = totalMemory - freeMemory;
            
            Gauge.builder("jvm.memory.used")
                .tag("context", context)
                .register(meterRegistry, runtime, r -> r.totalMemory() - r.freeMemory());
                
            Gauge.builder("jvm.memory.free")
                .tag("context", context)
                .register(meterRegistry, runtime, Runtime::freeMemory);
                
            Gauge.builder("jvm.memory.total")
                .tag("context", context)
                .register(meterRegistry, Runtime::totalMemory);
        }
    }
}
```

## Profiling and Diagnostic Tools

### JVM Profiling Techniques

```java
import jdk.jfr.*;
import jdk.jfr.consumer.*;
import java.nio.file.Path;
import java.util.concurrent.atomic.AtomicLong;

public class JvmProfiling {
    
    // Flight Recorder for detailed profiling
    @Label("User Service Performance")
    @Description("Tracks user service performance metrics")
    @StackTrace
    @Event(name = "UserService.FindUser")
    public static class FindUserEvent extends Event {
        @Label("User ID")
        long userId;
        
        @Label("Response Time")
        @Timespan
        long responseTime;
        
        @Label("Result")
        String result;
        
        @Label("Error")
        String error;
    }
    
    public static class FlightRecorderExample {
        
        private final AtomicLong callCount = new AtomicLong(0);
        
        public void demonstrateFlightRecorder() {
            // Start recording
            Recording recording = new Recording();
            recording.setName("UserService-Performance");
            recording.setDuration(Duration.ofMinutes(5));
            recording.start();
            
            try {
                // Perform operations that will be recorded
                for (int i = 0; i < 1000; i++) {
                    findUserWithRecording(i);
                }
            } finally {
                recording.stop();
                
                // Save recording
                Path recordingFile = Path.of("user-service-recording.jfr");
                recording.dump(recordingFile);
                
                System.out.println("Recording saved to: " + recordingFile);
            }
        }
        
        private User findUserWithRecording(long userId) {
            long startTime = System.nanoTime();
            
            FindUserEvent event = new FindUserEvent();
            event.userId = userId;
            
            try {
                // Simulate user finding operation
                User user = findUser(userId);
                
                event.result = user != null ? "FOUND" : "NOT_FOUND";
                return user;
            } catch (Exception e) {
                event.error = e.getMessage();
                throw e;
            } finally {
                event.responseTime = System.nanoTime() - startTime;
                event.commit();
            }
        }
        
        private User findUser(long userId) {
            // Simulate database call
            try {
                Thread.sleep(10); // Simulate processing time
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return new User("user" + userId, "user" + userId + "@example.com");
        }
        
        // Analyze JFR file
        public void analyzeRecording(Path recordingFile) {
            try (var reader = new RecordingFile(recordingFile)) {
                while (reader.hasMoreEvents()) {
                    Event event = reader.readEvent();
                    if (event instanceof FindUserEvent findEvent) {
                        System.out.printf("User %d: %s, Response Time: %d ns%n",
                            findEvent.userId, findEvent.result, findEvent.responseTime);
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
    
    // CPU profiling example
    public static class CpuProfilingExample {
        
        public void demonstrateCpuProfiling() {
            // Simulate CPU-intensive operations
            ExecutorService executor = Executors.newFixedThreadPool(10);
            
            List<Future<?>> futures = new ArrayList<>();
            
            // Heavy computation task
            futures.add(executor.submit(() -> {
                for (int i = 0; i < 1000000; i++) {
                    Math.sqrt(i * Math.PI); // CPU-intensive operation
                }
            }));
            
            // String processing task
            futures.add(executor.submit(() -> {
                StringBuilder sb = new StringBuilder();
                for (int i = 0; i < 100000; i++) {
                    sb.append("Processing item ").append(i).append("\n");
                    if (i % 10000 == 0) {
                        sb.setLength(0); // Reset to prevent memory buildup
                    }
                }
            }));
            
            // Wait for completion
            futures.forEach(future -> {
                try {
                    future.get();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });
            
            executor.shutdown();
        }
    }
    
    // Memory profiling example
    public static class MemoryProfilingExample {
        
        private List<byte[]> memoryHogList = new ArrayList<>();
        
        public void demonstrateMemoryProfiling() {
            // Create memory pressure
            for (int i = 0; i < 1000; i++) {
                // Create 10KB byte arrays
                byte[] array = new byte[10 * 1024];
                
                // Fill with data to prevent JVM optimization
                for (int j = 0; j < array.length; j++) {
                    array[j] = (byte) (j % 256);
                }
                
                memoryHogList.add(array);
                
                // Occasionally remove some to simulate object lifecycle
                if (i > 100 && i % 50 == 0) {
                    memoryHogList.remove(0);
                }
            }
            
            // Force GC to see memory behavior
            System.gc();
            
            // Print memory statistics
            printMemoryStats("After memory profiling");
        }
        
        private void printMemoryStats(String label) {
            Runtime runtime = Runtime.getRuntime();
            long totalMemory = runtime.totalMemory();
            long freeMemory = runtime.freeMemory();
            long usedMemory = totalMemory - freeMemory;
            
            System.out.printf("%s - Used: %d MB, Free: %d MB, Total: %d MB%n",
                label,
                usedMemory / (1024 * 1024),
                freeMemory / (1024 * 1024),
                totalMemory / (1024 * 1024));
        }
    }
    
    // Thread profiling
    public static class ThreadProfilingExample {
        
        public void demonstrateThreadProfiling() {
            ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
            
            // Print thread information
            System.out.println("Thread count: " + threadBean.getThreadCount());
            System.out.println("Peak thread count: " + threadBean.getPeakThreadCount());
            System.out.println("Total started threads: " + threadBean.getTotalStartedThreadCount());
            
            // Monitor thread contention if available
            if (threadBean.isThreadContentionMonitoringSupported()) {
                threadBean.setThreadContentionMonitoringEnabled(true);
                
                // Create threads that might cause contention
                ExecutorService executor = Executors.newFixedThreadPool(5);
                
                for (int i = 0; i < 5; i++) {
                    executor.submit(() -> {
                        synchronized (this) {
                            try {
                                Thread.sleep(1000); // Hold lock
                            } catch (InterruptedException e) {
                                Thread.currentThread().interrupt();
                            }
                        }
                    });
                }
                
                executor.shutdown();
                
                // Print thread dump for analysis
                long[] threadIds = threadBean.getAllThreadIds();
                ThreadInfo[] threadInfos = threadBean.getThreadInfo(threadIds, true, true);
                
                for (ThreadInfo threadInfo : threadInfos) {
                    if (threadInfo != null) {
                        System.out.println("Thread: " + threadInfo.getThreadName());
                        System.out.println("  State: " + threadInfo.getThreadState());
                        System.out.println("  CPU time: " + threadBean.getThreadCpuTime(threadInfo.getThreadId()));
                        System.out.println("  Block time: " + threadInfo.getBlockedTime());
                        System.out.println("  Wait time: " + threadInfo.getWaitedTime());
                    }
                }
            }
        }
    }
}
```

## Conclusion

Performance Optimization and Monitoring provide:
- JVM tuning and garbage collection optimization
- Application-level performance improvements
- Comprehensive monitoring and metrics collection
- Distributed tracing for microservices
- Profiling tools for performance analysis

Mastering these techniques is essential for:
- Building high-performance Java applications
- Ensuring scalability and reliability
- Implementing effective monitoring strategies
- Diagnosing and resolving performance issues
- Optimizing resource utilization