# Advanced Database Access Patterns

## Overview
Advanced database access patterns go beyond basic CRUD operations to provide sophisticated solutions for complex data access requirements. This tutorial covers connection pooling, caching strategies, database sharding, read/write separation, and other advanced patterns for building scalable, high-performance data access layers.

## Connection Pooling and Management

### Advanced Connection Pool Configuration

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.apache.commons.dbcp2.BasicDataSource;
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

public class ConnectionPoolingAdvanced {
    
    // HikariCP advanced configuration
    public static class HikariCPConfiguration {
        
        public DataSource createOptimizedDataSource() {
            HikariConfig config = new HikariConfig();
            
            // Basic configuration
            config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
            config.setUsername("user");
            config.setPassword("password");
            
            // Connection pool sizing
            config.setMaximumPoolSize(20);           // Maximum connections
            config.setMinimumIdle(5);                // Minimum idle connections
            config.setConnectionTimeout(30000);      // 30 seconds
            config.setIdleTimeout(600000);           // 10 minutes
            config.setMaxLifetime(1800000);          // 30 minutes
            config.setLeakDetectionThreshold(60000); // 1 minute
            
            // Performance optimizations
            config.setConnectionTestQuery("SELECT 1");
            config.setInitializationFailTimeout(60000); // 1 minute
            config.setIsolateInternalQueries(true);
            config.setAllowPoolSuspension(false);
            config.setReadOnly(false);
            config.setRegisterMbeans(true);
            
            // Advanced settings
            config.addDataSourceProperty("cachePrepStmts", "true");
            config.addDataSourceProperty("prepStmtCacheSize", "250");
            config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
            config.addDataSourceProperty("useServerPrepStmts", "true");
            config.addDataSourceProperty("useLocalSessionState", "true");
            config.addDataSourceProperty("rewriteBatchedStatements", "true");
            config.addDataSourceProperty("cacheResultSetMetadata", "true");
            config.addDataSourceProperty("cacheServerConfiguration", "true");
            config.addDataSourceProperty("elideSetAutoCommits", "true");
            config.addDataSourceProperty("maintainTimeStats", "false");
            
            return new HikariDataSource(config);
        }
        
        // Multi-tenant connection pool
        public DataSource createMultiTenantDataSource() {
            HikariConfig config = new HikariConfig();
            config.setJdbcUrl("jdbc:postgresql://localhost:5432/");
            config.setUsername("user");
            config.setPassword("password");
            
            // Tenant-specific configuration
            config.setDataSourceClassName("com.zaxxer.hikari.mocks.StubDataSource");
            config.setMaximumPoolSize(10);
            config.setMinimumIdle(2);
            
            // Custom connection provider for multi-tenancy
            config.setDataSource(new MultiTenantDataSource());
            
            return new HikariDataSource(config);
        }
        
        // Connection pool monitoring
        public void monitorConnectionPool(HikariDataSource dataSource) {
            System.out.println("=== Connection Pool Monitoring ===");
            System.out.println("Active connections: " + dataSource.getHikariPoolMXBean().getActiveConnections());
            System.out.println("Idle connections: " + dataSource.getHikariPoolMXBean().getIdleConnections());
            System.out.println("Total connections: " + dataSource.getHikariPoolMXBean().getTotalConnections());
            System.out.println("Threads awaiting connection: " + dataSource.getHikariPoolMXBean().getThreadsAwaitingConnection());
            
            // Connection statistics
            System.out.println("Total connections created: " + dataSource.getHikariPoolMXBean().getTotalConnections());
            System.out.println("Connections timed out: " + dataSource.getHikariPoolMXBean().getConnectionTimeoutCount());
        }
    }
    
    // Apache DBCP2 advanced configuration
    public static class DBCP2Configuration {
        
        public DataSource createDBCPDataSource() {
            BasicDataSource dataSource = new BasicDataSource();
            
            // Basic configuration
            dataSource.setUrl("jdbc:postgresql://localhost:5432/mydb");
            dataSource.setUsername("user");
            dataSource.setPassword("password");
            
            // Pool configuration
            dataSource.setInitialSize(5);
            dataSource.setMaxTotal(20);
            dataSource.setMaxIdle(10);
            dataSource.setMinIdle(2);
            
            // Timeout configuration
            dataSource.setMaxWaitMillis(30000); // 30 seconds
            dataSource.setMinEvictableIdleTimeMillis(600000); // 10 minutes
            dataSource.setTimeBetweenEvictionRunsMillis(30000); // 30 seconds
            
            // Validation configuration
            dataSource.setTestOnBorrow(true);
            dataSource.setTestOnReturn(false);
            dataSource.setTestWhileIdle(true);
            dataSource.setValidationQuery("SELECT 1");
            dataSource.setValidationQueryTimeout(5);
            
            // Advanced configuration
            dataSource.setRemoveAbandonedOnBorrow(true);
            dataSource.setRemoveAbandonedOnMaintenance(true);
            dataSource.setRemoveAbandonedTimeout(60); // 1 minute
            dataSource.setLogAbandoned(true);
            
            dataSource.setPoolPreparedStatements(true);
            dataSource.setMaxOpenPreparedStatements(250);
            
            return dataSource;
        }
        
        // Custom connection factory
        public DataSource createCustomDataSource() {
            BasicDataSource dataSource = new BasicDataSource();
            
            // Custom connection factory
            dataSource.setDriverClassName("org.postgresql.Driver");
            dataSource.setUrl("jdbc:postgresql://localhost:5432/mydb");
            dataSource.setUsername("user");
            dataSource.setPassword("password");
            
            // Custom factory settings
            dataSource.setConnectionProperties("ApplicationName=MyApp;socketTimeout=30");
            
            return dataSource;
        }
    }
    
    // Custom multi-tenant data source
    public static class MultiTenantDataSource implements javax.sql.DataSource {
        private final ThreadLocal<String> tenantId = new ThreadLocal<>();
        private final AtomicInteger connectionCounter = new AtomicInteger(0);
        
        @Override
        public Connection getConnection() throws SQLException {
            String currentTenant = tenantId.get();
            if (currentTenant == null) {
                throw new SQLException("No tenant ID set for current thread");
            }
            
            // Return connection for specific tenant
            Connection connection = createTenantConnection(currentTenant);
            connectionCounter.incrementAndGet();
            return connection;
        }
        
        @Override
        public Connection getConnection(String username, String password) throws SQLException {
            // Implementation for username/password authentication
            return getConnection();
        }
        
        private Connection createTenantConnection(String tenantId) throws SQLException {
            // Create connection specific to tenant
            // This is a simplified example
            return new MockConnection(tenantId);
        }
        
        public void setTenantId(String tenantId) {
            this.tenantId.set(tenantId);
        }
        
        public void clearTenantId() {
            this.tenantId.remove();
        }
        
        public int getConnectionCount() {
            return connectionCounter.get();
        }
        
        // Other DataSource methods would be implemented similarly
        @Override public <T> T unwrap(Class<T> iface) throws SQLException { return null; }
        @Override public boolean isWrapperFor(Class<?> iface) throws SQLException { return false; }
        @Override public java.io.PrintWriter getLogWriter() throws SQLException { return null; }
        @Override public void setLogWriter(java.io.PrintWriter out) throws SQLException {}
        @Override public void setLoginTimeout(int seconds) throws SQLException {}
        @Override public int getLoginTimeout() throws SQLException { return 0; }
        @Override public java.util.logging.Logger getParentLogger() throws java.sql.SQLFeatureNotSupportedException { return null; }
    }
    
    // Mock connection for demonstration
    private static class MockConnection implements Connection {
        private final String tenantId;
        private boolean closed = false;
        
        public MockConnection(String tenantId) {
            this.tenantId = tenantId;
        }
        
        public String getTenantId() {
            return tenantId;
        }
        
        @Override
        public void close() throws SQLException {
            closed = true;
        }
        
        @Override
        public boolean isClosed() throws SQLException {
            return closed;
        }
        
        // Other Connection methods would be implemented as needed
        @Override public java.sql.Statement createStatement() throws SQLException { return null; }
        @Override public java.sql.PreparedStatement prepareStatement(String sql) throws SQLException { return null; }
        @Override public java.sql.CallableStatement prepareCall(String sql) throws SQLException { return null; }
        @Override public String nativeSQL(String sql) throws SQLException { return null; }
        @Override public void setAutoCommit(boolean autoCommit) throws SQLException {}
        @Override public boolean getAutoCommit() throws SQLException { return false; }
        @Override public void commit() throws SQLException {}
        @Override public void rollback() throws SQLException {}
        @Override public java.sql.DatabaseMetaData getMetaData() throws SQLException { return null; }
        @Override public void setReadOnly(boolean readOnly) throws SQLException {}
        @Override public boolean isReadOnly() throws SQLException { return false; }
        @Override public void setCatalog(String catalog) throws SQLException {}
        @Override public String getCatalog() throws SQLException { return null; }
        @Override public void setTransactionIsolation(int level) throws SQLException {}
        @Override public int getTransactionIsolation() throws SQLException { return 0; }
        @Override public java.sql.SQLWarning getWarnings() throws SQLException { return null; }
        @Override public void clearWarnings() throws SQLException {}
        @Override public java.sql.Statement createStatement(int resultSetType, int resultSetConcurrency) throws SQLException { return null; }
        @Override public java.sql.PreparedStatement prepareStatement(String sql, int resultSetType, int resultSetConcurrency) throws SQLException { return null; }
        @Override public java.sql.CallableStatement prepareCall(String sql, int resultSetType, int resultSetConcurrency) throws SQLException { return null; }
        @Override public java.util.Map<String,Class<?>> getTypeMap() throws SQLException { return null; }
        @Override public void setTypeMap(java.util.Map<String,Class<?>> map) throws SQLException {}
        @Override public void setHoldability(int holdability) throws SQLException {}
        @Override public int getHoldability() throws SQLException { return 0; }
        @Override public java.sql.Savepoint setSavepoint() throws SQLException { return null; }
        @Override public java.sql.Savepoint setSavepoint(String name) throws SQLException { return null; }
        @Override public void rollback(java.sql.Savepoint savepoint) throws SQLException {}
        @Override public void releaseSavepoint(java.sql.Savepoint savepoint) throws SQLException {}
        @Override public java.sql.Statement createStatement(int resultSetType, int resultSetConcurrency, int resultSetHoldability) throws SQLException { return null; }
        @Override public java.sql.PreparedStatement prepareStatement(String sql, int resultSetType, int resultSetConcurrency, int resultSetHoldability) throws SQLException { return null; }
        @Override public java.sql.CallableStatement prepareCall(String sql, int resultSetType, int resultSetConcurrency, int resultSetHoldability) throws SQLException { return null; }
        @Override public java.sql.PreparedStatement prepareStatement(String sql, int autoGeneratedKeys) throws SQLException { return null; }
        @Override public java.sql.PreparedStatement prepareStatement(String sql, int[] columnIndexes) throws SQLException { return null; }
        @Override public java.sql.PreparedStatement prepareStatement(String sql, String[] columnNames) throws SQLException { return null; }
        @Override public java.sql.Clob createClob() throws SQLException { return null; }
        @Override public java.sql.Blob createBlob() throws SQLException { return null; }
        @Override public java.sql.NClob createNClob() throws SQLException { return null; }
        @Override public java.sql.SQLXML createSQLXML() throws SQLException { return null; }
        @Override public boolean isValid(int timeout) throws SQLException { return false; }
        @Override public void setClientInfo(String name, String value) throws java.sql.SQLClientInfoException {}
        @Override public void setClientInfo(java.util.Properties properties) throws java.sql.SQLClientInfoException {}
        @Override public String getClientInfo(String name) throws SQLException { return null; }
        @Override public java.util.Properties getClientInfo() throws SQLException { return null; }
        @Override public java.sql.Array createArrayOf(String typeName, Object[] elements) throws SQLException { return null; }
        @Override public java.sql.Struct createStruct(String typeName, Object[] attributes) throws SQLException { return null; }
        @Override public void setSchema(String schema) throws SQLException {}
        @Override public String getSchema() throws SQLException { return null; }
        @Override public void abort(java.util.concurrent.Executor executor) throws SQLException {}
        @Override public void setNetworkTimeout(java.util.concurrent.Executor executor, int milliseconds) throws SQLException {}
        @Override public int getNetworkTimeout() throws SQLException { return 0; }
        @Override public <T> T unwrap(Class<T> iface) throws SQLException { return null; }
        @Override public boolean isWrapperFor(Class<?> iface) throws SQLException { return false; }
    }
}
```

## Database Sharding and Partitioning

### Advanced Sharding Strategies

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;
import java.util.stream.Collectors;
import javax.sql.DataSource;

public class DatabaseShardingAdvanced {
    
    // Range-based sharding
    public static class RangeBasedSharding {
        private final Map<String, DataSource> shards = new ConcurrentHashMap<>();
        private final Map<Long, String> shardMap = new ConcurrentHashMap<>();
        private final long shardSize;
        
        public RangeBasedSharding(long shardSize) {
            this.shardSize = shardSize;
        }
        
        public void addShard(String shardId, DataSource dataSource) {
            shards.put(shardId, dataSource);
        }
        
        public DataSource getShardForId(Long id) {
            if (id == null) return null;
            
            String shardId = shardMap.get(id);
            if (shardId == null) {
                shardId = calculateShardId(id);
                shardMap.put(id, shardId);
            }
            
            return shards.get(shardId);
        }
        
        private String calculateShardId(Long id) {
            long shardIndex = id / shardSize;
            return "shard_" + shardIndex;
        }
        
        public List<String> getAllShardIds() {
            return new ArrayList<>(shards.keySet());
        }
        
        // Bulk operations across shards
        public <T> List<T> executeOnAllShards(ShardOperation<T> operation) {
            return shards.entrySet().parallelStream()
                .map(entry -> {
                    try {
                        return operation.execute(entry.getKey(), entry.getValue());
                    } catch (Exception e) {
                        throw new RuntimeException("Failed to execute on shard " + entry.getKey(), e);
                    }
                })
                .collect(Collectors.toList());
        }
        
        @FunctionalInterface
        public interface ShardOperation<T> {
            T execute(String shardId, DataSource dataSource) throws Exception;
        }
    }
    
    // Hash-based sharding
    public static class HashBasedSharding {
        private final Map<String, DataSource> shards = new ConcurrentHashMap<>();
        private final int shardCount;
        private final Map<String, String> keyToShardCache = new ConcurrentHashMap<>();
        
        public HashBasedSharding(int shardCount) {
            this.shardCount = shardCount;
        }
        
        public void addShard(String shardId, DataSource dataSource) {
            shards.put(shardId, dataSource);
        }
        
        public DataSource getShardForKey(String key) {
            if (key == null) return null;
            
            String shardId = keyToShardCache.get(key);
            if (shardId == null) {
                shardId = calculateShardId(key);
                keyToShardCache.put(key, shardId);
            }
            
            return shards.get(shardId);
        }
        
        private String calculateShardId(String key) {
            int hash = Math.abs(key.hashCode());
            int shardIndex = hash % shardCount;
            return "shard_" + shardIndex;
        }
        
        // Consistent hashing for better distribution
        public DataSource getShardForKeyConsistent(String key) {
            if (key == null) return null;
            
            String shardId = keyToShardCache.get(key);
            if (shardId == null) {
                shardId = calculateShardIdConsistent(key);
                keyToShardCache.put(key, shardId);
            }
            
            return shards.get(shardId);
        }
        
        private String calculateShardIdConsistent(String key) {
            // Simplified consistent hashing
            long hash = hashKey(key);
            long shardIndex = hash % shardCount;
            return "shard_" + shardIndex;
        }
        
        private long hashKey(String key) {
            // Simple hash function - in production, use a better hash function
            long hash = 0;
            for (char c : key.toCharArray()) {
                hash = hash * 31 + c;
            }
            return Math.abs(hash);
        }
        
        // Rebalancing shards
        public void rebalanceShards(int newShardCount) {
            if (newShardCount <= 0) {
                throw new IllegalArgumentException("Shard count must be positive");
            }
            
            Map<String, String> oldToNewShardMap = new HashMap<>();
            
            // Map old shards to new shards
            keyToShardCache.forEach((key, oldShardId) -> {
                String newShardId = calculateNewShardId(key, newShardCount);
                oldToNewShardMap.put(oldShardId, newShardId);
            });
            
            // Update cache
            keyToShardCache.replaceAll((key, oldShardId) -> 
                calculateNewShardId(key, newShardCount));
            
            this.shardCount = newShardCount;
        }
        
        private String calculateNewShardId(String key, int newShardCount) {
            long hash = hashKey(key);
            long shardIndex = hash % newShardCount;
            return "shard_" + shardIndex;
        }
    }
    
    // Directory-based sharding
    public static class DirectoryBasedSharding {
        private final Map<String, DataSource> shards = new ConcurrentHashMap<>();
        private final Map<String, String> keyToShardMap = new ConcurrentHashMap<>();
        private final AtomicLong shardCounter = new AtomicLong(0);
        
        public void addShard(String shardId, DataSource dataSource) {
            shards.put(shardId, dataSource);
        }
        
        public void assignKeyToShard(String key, String shardId) {
            if (!shards.containsKey(shardId)) {
                throw new IllegalArgumentException("Shard " + shardId + " does not exist");
            }
            keyToShardMap.put(key, shardId);
        }
        
        public DataSource getShardForKey(String key) {
            String shardId = keyToShardMap.get(key);
            return shardId != null ? shards.get(shardId) : null;
        }
        
        public String assignKeyToNewShard(String key) {
            // Assign to next available shard in round-robin fashion
            List<String> shardIds = new ArrayList<>(shards.keySet());
            if (shardIds.isEmpty()) {
                throw new IllegalStateException("No shards available");
            }
            
            int index = (int) (shardCounter.getAndIncrement() % shardIds.size());
            String shardId = shardIds.get(index);
            keyToShardMap.put(key, shardId);
            return shardId;
        }
        
        // Shard migration
        public void migrateKey(String key, String newShardId) {
            if (!shards.containsKey(newShardId)) {
                throw new IllegalArgumentException("Target shard " + newShardId + " does not exist");
            }
            
            keyToShardMap.put(key, newShardId);
        }
        
        public Set<String> getKeysInShard(String shardId) {
            return keyToShardMap.entrySet().stream()
                .filter(entry -> shardId.equals(entry.getValue()))
                .map(Map.Entry::getKey)
                .collect(Collectors.toSet());
        }
    }
    
    // Sharding manager with monitoring
    public static class ShardingManager {
        private final RangeBasedSharding rangeSharding;
        private final HashBasedSharding hashSharding;
        private final DirectoryBasedSharding directorySharding;
        private final Map<String, ShardMetrics> shardMetrics = new ConcurrentHashMap<>();
        
        public ShardingManager(long shardSize, int hashShardCount) {
            this.rangeSharding = new RangeBasedSharding(shardSize);
            this.hashSharding = new HashBasedSharding(hashShardCount);
            this.directorySharding = new DirectoryBasedSharding();
        }
        
        // Add metrics tracking
        public <T> T executeWithMetrics(String operation, String shardId, DataSource dataSource, 
                                       ThrowingSupplier<T> supplier) throws Exception {
            long startTime = System.nanoTime();
            try {
                T result = supplier.get();
                recordSuccess(operation, shardId, System.nanoTime() - startTime);
                return result;
            } catch (Exception e) {
                recordFailure(operation, shardId, System.nanoTime() - startTime);
                throw e;
            }
        }
        
        private void recordSuccess(String operation, String shardId, long duration) {
            shardMetrics.computeIfAbsent(shardId, k -> new ShardMetrics())
                       .recordSuccess(operation, duration);
        }
        
        private void recordFailure(String operation, String shardId, long duration) {
            shardMetrics.computeIfAbsent(shardId, k -> new ShardMetrics())
                       .recordFailure(operation, duration);
        }
        
        public void printShardMetrics() {
            System.out.println("=== Shard Metrics ===");
            shardMetrics.forEach((shardId, metrics) -> {
                System.out.println("Shard: " + shardId);
                metrics.printMetrics();
            });
        }
        
        @FunctionalInterface
        public interface ThrowingSupplier<T> {
            T get() throws Exception;
        }
        
        // Shard metrics tracking
        public static class ShardMetrics {
            private final AtomicLong successCount = new AtomicLong(0);
            private final AtomicLong failureCount = new AtomicLong(0);
            private final AtomicLong totalDuration = new AtomicLong(0);
            private final Map<String, OperationMetrics> operationMetrics = new ConcurrentHashMap<>();
            
            public void recordSuccess(String operation, long duration) {
                successCount.incrementAndGet();
                totalDuration.addAndGet(duration);
                operationMetrics.computeIfAbsent(operation, k -> new OperationMetrics())
                               .recordSuccess(duration);
            }
            
            public void recordFailure(String operation, long duration) {
                failureCount.incrementAndGet();
                totalDuration.addAndGet(duration);
                operationMetrics.computeIfAbsent(operation, k -> new OperationMetrics())
                               .recordFailure(duration);
            }
            
            public void printMetrics() {
                long total = successCount.get() + failureCount.get();
                double successRate = total > 0 ? (double) successCount.get() / total * 100 : 0;
                double avgDuration = total > 0 ? (double) totalDuration.get() / total / 1_000_000 : 0;
                
                System.out.println("  Total operations: " + total);
                System.out.println("  Success rate: " + String.format("%.2f%%", successRate));
                System.out.println("  Average duration: " + String.format("%.2f", avgDuration) + " ms");
                
                operationMetrics.forEach((operation, metrics) -> {
                    System.out.println("  Operation " + operation + ":");
                    metrics.printMetrics("    ");
                });
            }
            
            private static class OperationMetrics {
                private final AtomicLong successCount = new AtomicLong(0);
                private final AtomicLong failureCount = new AtomicLong(0);
                private final AtomicLong totalDuration = new AtomicLong(0);
                
                public void recordSuccess(long duration) {
                    successCount.incrementAndGet();
                    totalDuration.addAndGet(duration);
                }
                
                public void recordFailure(long duration) {
                    failureCount.incrementAndGet();
                    totalDuration.addAndGet(duration);
                }
                
                public void printMetrics(String indent) {
                    long total = successCount.get() + failureCount.get();
                    double successRate = total > 0 ? (double) successCount.get() / total * 100 : 0;
                    double avgDuration = total > 0 ? (double) totalDuration.get() / total / 1_000_000 : 0;
                    
                    System.out.println(indent + "Total: " + total + ", Success: " + 
                                     String.format("%.2f%%", successRate) + ", Avg: " + 
                                     String.format("%.2f", avgDuration) + " ms");
                }
            }
        }
    }
}
```

## Read/Write Separation and Replication

### Advanced Replication Patterns

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;

public class ReadWriteSeparationAdvanced {
    
    // Master-slave replication with load balancing
    public static class MasterSlaveReplication {
        private final DataSource master;
        private final List<DataSource> slaves;
        private final AtomicInteger slaveIndex = new AtomicInteger(0);
        private final LoadBalancer loadBalancer;
        
        public MasterSlaveReplication(DataSource master, List<DataSource> slaves) {
            this.master = master;
            this.slaves = new ArrayList<>(slaves);
            this.loadBalancer = new RoundRobinLoadBalancer(slaves);
        }
        
        public Connection getWriteConnection() throws SQLException {
            return master.getConnection();
        }
        
        public Connection getReadConnection() throws SQLException {
            return loadBalancer.getConnection();
        }
        
        // Health checking for slaves
        public void checkSlaveHealth() {
            slaves.parallelStream().forEach(slave -> {
                try (Connection connection = slave.getConnection()) {
                    if (connection.isValid(5)) {
                        // Slave is healthy
                        markSlaveHealthy(slave);
                    } else {
                        // Slave is unhealthy
                        markSlaveUnhealthy(slave);
                    }
                } catch (SQLException e) {
                    markSlaveUnhealthy(slave);
                }
            });
        }
        
        private void markSlaveHealthy(DataSource slave) {
            // Implementation to mark slave as healthy
            System.out.println("Slave " + slave + " is healthy");
        }
        
        private void markSlaveUnhealthy(DataSource slave) {
            // Implementation to mark slave as unhealthy
            System.out.println("Slave " + slave + " is unhealthy");
        }
        
        // Dynamic slave management
        public void addSlave(DataSource slave) {
            slaves.add(slave);
            loadBalancer.addDataSource(slave);
        }
        
        public void removeSlave(DataSource slave) {
            slaves.remove(slave);
            loadBalancer.removeDataSource(slave);
        }
    }
    
    // Load balancer implementations
    public static abstract class LoadBalancer {
        protected final List<DataSource> dataSources;
        protected final Set<DataSource> unhealthySources = ConcurrentHashMap.newKeySet();
        
        public LoadBalancer(List<DataSource> dataSources) {
            this.dataSources = new ArrayList<>(dataSources);
        }
        
        public abstract Connection getConnection() throws SQLException;
        
        public void addDataSource(DataSource dataSource) {
            dataSources.add(dataSource);
        }
        
        public void removeDataSource(DataSource dataSource) {
            dataSources.remove(dataSource);
            unhealthySources.remove(dataSource);
        }
        
        protected boolean isHealthy(DataSource dataSource) {
            return !unhealthySources.contains(dataSource);
        }
        
        protected void markUnhealthy(DataSource dataSource) {
            unhealthySources.add(dataSource);
        }
        
        protected void markHealthy(DataSource dataSource) {
            unhealthySources.remove(dataSource);
        }
    }
    
    // Round-robin load balancer
    public static class RoundRobinLoadBalancer extends LoadBalancer {
        private final AtomicInteger currentIndex = new AtomicInteger(0);
        
        public RoundRobinLoadBalancer(List<DataSource> dataSources) {
            super(dataSources);
        }
        
        @Override
        public Connection getConnection() throws SQLException {
            if (dataSources.isEmpty()) {
                throw new SQLException("No data sources available");
            }
            
            int startIndex = currentIndex.get();
            int index = startIndex;
            
            do {
                DataSource dataSource = dataSources.get(index % dataSources.size());
                if (isHealthy(dataSource)) {
                    try {
                        Connection connection = dataSource.getConnection();
                        if (connection.isValid(5)) {
                            currentIndex.set((index + 1) % dataSources.size());
                            return connection;
                        } else {
                            markUnhealthy(dataSource);
                        }
                    } catch (SQLException e) {
                        markUnhealthy(dataSource);
                    }
                }
                index = (index + 1) % dataSources.size();
            } while (index != startIndex);
            
            throw new SQLException("No healthy data sources available");
        }
    }
    
    // Weighted load balancer
    public static class WeightedLoadBalancer extends LoadBalancer {
        private final Map<DataSource, Integer> weights;
        private final Random random = new Random();
        
        public WeightedLoadBalancer(Map<DataSource, Integer> weights) {
            super(new ArrayList<>(weights.keySet()));
            this.weights = new HashMap<>(weights);
        }
        
        @Override
        public Connection getConnection() throws SQLException {
            if (dataSources.isEmpty()) {
                throw new SQLException("No data sources available");
            }
            
            // Calculate total weight of healthy sources
            int totalWeight = dataSources.stream()
                .filter(this::isHealthy)
                .mapToInt(weights::get)
                .sum();
            
            if (totalWeight == 0) {
                throw new SQLException("No healthy data sources available");
            }
            
            // Select based on weighted random
            int randomWeight = random.nextInt(totalWeight);
            int currentWeight = 0;
            
            for (DataSource dataSource : dataSources) {
                if (isHealthy(dataSource)) {
                    currentWeight += weights.get(dataSource);
                    if (randomWeight < currentWeight) {
                        try {
                            Connection connection = dataSource.getConnection();
                            if (connection.isValid(5)) {
                                return connection;
                            } else {
                                markUnhealthy(dataSource);
                            }
                        } catch (SQLException e) {
                            markUnhealthy(dataSource);
                        }
                    }
                }
            }
            
            throw new SQLException("No healthy data sources available");
        }
    }
    
    // Adaptive load balancer based on response time
    public static class AdaptiveLoadBalancer extends LoadBalancer {
        private final Map<DataSource, Long> responseTimes = new ConcurrentHashMap<>();
        private final Map<DataSource, Long> lastCheck = new ConcurrentHashMap<>();
        private final long checkInterval = TimeUnit.SECONDS.toMillis(30);
        
        public AdaptiveLoadBalancer(List<DataSource> dataSources) {
            super(dataSources);
            // Initialize with default response times
            dataSources.forEach(ds -> responseTimes.put(ds, 100L));
        }
        
        @Override
        public Connection getConnection() throws SQLException {
            if (dataSources.isEmpty()) {
                throw new SQLException("No data sources available");
            }
            
            // Update response times periodically
            updateResponseTimes();
            
            // Select fastest healthy source
            return dataSources.stream()
                .filter(this::isHealthy)
                .min(Comparator.comparing(responseTimes::get))
                .map(this::getConnectionWithTiming)
                .orElseThrow(() -> new SQLException("No healthy data sources available"));
        }
        
        private Connection getConnectionWithTiming(DataSource dataSource) {
            long startTime = System.nanoTime();
            try {
                Connection connection = dataSource.getConnection();
                if (connection.isValid(5)) {
                    long responseTime = (System.nanoTime() - startTime) / 1_000_000; // ms
                    responseTimes.put(dataSource, responseTime);
                    return connection;
                } else {
                    markUnhealthy(dataSource);
                    throw new SQLException("Connection is not valid");
                }
            } catch (SQLException e) {
                markUnhealthy(dataSource);
                long responseTime = (System.nanoTime() - startTime) / 1_000_000; // ms
                responseTimes.put(dataSource, responseTime + 1000); // Penalty for failure
                throw e;
            }
        }
        
        private void updateResponseTimes() {
            long now = System.currentTimeMillis();
            dataSources.parallelStream().forEach(dataSource -> {
                Long last = lastCheck.get(dataSource);
                if (last == null || now - last > checkInterval) {
                    lastCheck.put(dataSource, now);
                    // Background check to update response time
                    CompletableFuture.runAsync(() -> {
                        try (Connection connection = dataSource.getConnection()) {
                            long startTime = System.nanoTime();
                            if (connection.isValid(5)) {
                                long responseTime = (System.nanoTime() - startTime) / 1_000_000;
                                responseTimes.put(dataSource, responseTime);
                                markHealthy(dataSource);
                            } else {
                                markUnhealthy(dataSource);
                            }
                        } catch (SQLException e) {
                            markUnhealthy(dataSource);
                        }
                    });
                }
            });
        }
    }
    
    // Transaction-aware routing
    public static class TransactionAwareRouting {
        private final ThreadLocal<Boolean> inTransaction = new ThreadLocal<>();
        private final MasterSlaveReplication replication;
        
        public TransactionAwareRouting(MasterSlaveReplication replication) {
            this.replication = replication;
        }
        
        public Connection getConnection() throws SQLException {
            if (Boolean.TRUE.equals(inTransaction.get())) {
                // Always use master for transactions
                return replication.getWriteConnection();
            } else {
                // Use slave for reads outside transactions
                return replication.getReadConnection();
            }
        }
        
        public void beginTransaction() {
            inTransaction.set(true);
        }
        
        public void endTransaction() {
            inTransaction.remove();
        }
        
        public boolean isInTransaction() {
            return Boolean.TRUE.equals(inTransaction.get());
        }
    }
    
    // Replication monitoring
    public static class ReplicationMonitor {
        private final MasterSlaveReplication replication;
        private final ScheduledExecutorService scheduler = 
            Executors.newScheduledThreadPool(2);
        private final Map<DataSource, ReplicationMetrics> metrics = 
            new ConcurrentHashMap<>();
        
        public ReplicationMonitor(MasterSlaveReplication replication) {
            this.replication = replication;
            startMonitoring();
        }
        
        private void startMonitoring() {
            // Health check every 30 seconds
            scheduler.scheduleAtFixedRate(this::checkHealth, 0, 30, TimeUnit.SECONDS);
            
            // Metrics collection every minute
            scheduler.scheduleAtFixedRate(this::collectMetrics, 0, 1, TimeUnit.MINUTES);
        }
        
        private void checkHealth() {
            try {
                replication.checkSlaveHealth();
            } catch (Exception e) {
                System.err.println("Health check failed: " + e.getMessage());
            }
        }
        
        private void collectMetrics() {
            // Collect replication lag, throughput, etc.
            System.out.println("Collecting replication metrics...");
        }
        
        public void shutdown() {
            scheduler.shutdown();
            try {
                if (!scheduler.awaitTermination(10, TimeUnit.SECONDS)) {
                    scheduler.shutdownNow();
                }
            } catch (InterruptedException e) {
                scheduler.shutdownNow();
                Thread.currentThread().interrupt();
            }
        }
        
        public void printStatus() {
            System.out.println("=== Replication Status ===");
            // Print current status of master and slaves
        }
        
        private static class ReplicationMetrics {
            private final AtomicLong queries = new AtomicLong(0);
            private final AtomicLong errors = new AtomicLong(0);
            private final AtomicLong latencySum = new AtomicLong(0);
            
            public void recordQuery(long latency) {
                queries.incrementAndGet();
                latencySum.addAndGet(latency);
            }
            
            public void recordError() {
                errors.incrementAndGet();
            }
            
            public double getAverageLatency() {
                long queryCount = queries.get();
                return queryCount > 0 ? (double) latencySum.get() / queryCount : 0;
            }
            
            public double getErrorRate() {
                long total = queries.get() + errors.get();
                return total > 0 ? (double) errors.get() / total * 100 : 0;
            }
        }
    }
}
```

## Advanced Caching Strategies

### Multi-Level Caching Implementation

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;
import java.util.function.Function;
import java.util.function.Supplier;

public class AdvancedCachingStrategies {
    
    // Multi-level cache with L1 (memory) and L2 (distributed)
    public static class MultiLevelCache<K, V> {
        private final Cache<K, V> l1Cache;  // Local memory cache
        private final Cache<K, V> l2Cache;  // Distributed cache (Redis, etc.)
        private final CacheMetrics metrics = new CacheMetrics();
        
        public MultiLevelCache(Cache<K, V> l1Cache, Cache<K, V> l2Cache) {
            this.l1Cache = l1Cache;
            this.l2Cache = l2Cache;
        }
        
        public V get(K key) {
            // Check L1 cache first
            V value = l1Cache.get(key);
            if (value != null) {
                metrics.recordL1Hit();
                return value;
            }
            
            // Check L2 cache
            value = l2Cache.get(key);
            if (value != null) {
                metrics.recordL2Hit();
                // Promote to L1 cache
                l1Cache.put(key, value);
                return value;
            }
            
            metrics.recordMiss();
            return null;
        }
        
        public void put(K key, V value) {
            // Put in both caches
            l1Cache.put(key, value);
            l2Cache.put(key, value);
            metrics.recordPut();
        }
        
        public void invalidate(K key) {
            // Invalidate in both caches
            l1Cache.invalidate(key);
            l2Cache.invalidate(key);
            metrics.recordInvalidate();
        }
        
        public CacheMetrics getMetrics() {
            return metrics;
        }
        
        public void printMetrics() {
            System.out.println("=== Cache Metrics ===");
            System.out.println("L1 Hits: " + metrics.getL1Hits());
            System.out.println("L2 Hits: " + metrics.getL2Hits());
            System.out.println("Misses: " + metrics.getMisses());
            System.out.println("Puts: " + metrics.getPuts());
            System.out.println("Invalidates: " + metrics.getInvalidates());
            
            long totalRequests = metrics.getL1Hits() + metrics.getL2Hits() + metrics.getMisses();
            if (totalRequests > 0) {
                double hitRate = (double) (metrics.getL1Hits() + metrics.getL2Hits()) / totalRequests * 100;
                System.out.println("Overall Hit Rate: " + String.format("%.2f%%", hitRate));
            }
        }
    }
    
    // Cache interface
    public interface Cache<K, V> {
        V get(K key);
        void put(K key, V value);
        void invalidate(K key);
        void clear();
        long size();
    }
    
    // In-memory LRU cache implementation
    public static class LRUCache<K, V> implements Cache<K, V> {
        private final int maxSize;
        private final Map<K, V> cache;
        private final LinkedHashSet<K> accessOrder;
        
        public LRUCache(int maxSize) {
            this.maxSize = maxSize;
            this.cache = new ConcurrentHashMap<>();
            this.accessOrder = new LinkedHashSet<>();
        }
        
        @Override
        public synchronized V get(K key) {
            V value = cache.get(key);
            if (value != null) {
                // Update access order
                accessOrder.remove(key);
                accessOrder.add(key);
            }
            return value;
        }
        
        @Override
        public synchronized void put(K key, V value) {
            if (cache.size() >= maxSize && !cache.containsKey(key)) {
                // Remove LRU entry
                Iterator<K> iterator = accessOrder.iterator();
                if (iterator.hasNext()) {
                    K lruKey = iterator.next();
                    iterator.remove();
                    cache.remove(lruKey);
                }
            }
            
            cache.put(key, value);
            accessOrder.remove(key); // Remove if exists
            accessOrder.add(key);
        }
        
        @Override
        public synchronized void invalidate(K key) {
            cache.remove(key);
            accessOrder.remove(key);
        }
        
        @Override
        public synchronized void clear() {
            cache.clear();
            accessOrder.clear();
        }
        
        @Override
        public synchronized long size() {
            return cache.size();
        }
    }
    
    // Write-through cache
    public static class WriteThroughCache<K, V> implements Cache<K, V> {
        private final Cache<K, V> cache;
        private final Function<K, V> loader;
        private final Function<K, V> writer;
        
        public WriteThroughCache(Cache<K, V> cache, Function<K, V> loader, Function<K, V> writer) {
            this.cache = cache;
            this.loader = loader;
            this.writer = writer;
        }
        
        @Override
        public V get(K key) {
            V value = cache.get(key);
            if (value == null) {
                // Load from source
                value = loader.apply(key);
                if (value != null) {
                    cache.put(key, value);
                }
            }
            return value;
        }
        
        @Override
        public void put(K key, V value) {
            // Write to source first
            V writtenValue = writer.apply(key);
            if (writtenValue != null) {
                cache.put(key, writtenValue);
            }
        }
        
        @Override
        public void invalidate(K key) {
            cache.invalidate(key);
        }
        
        @Override
        public void clear() {
            cache.clear();
        }
        
        @Override
        public long size() {
            return cache.size();
        }
    }
    
    // Cache with expiration
    public static class ExpiringCache<K, V> implements Cache<K, V> {
        private final Cache<K, CacheEntry<V>> cache;
        private final long ttlMillis;
        
        public ExpiringCache(Cache<K, CacheEntry<V>> cache, long ttlMillis) {
            this.cache = cache;
            this.ttlMillis = ttlMillis;
        }
        
        @Override
        public V get(K key) {
            CacheEntry<V> entry = cache.get(key);
            if (entry != null && !entry.isExpired()) {
                return entry.getValue();
            } else if (entry != null) {
                // Expired, remove it
                cache.invalidate(key);
            }
            return null;
        }
        
        @Override
        public void put(K key, V value) {
            CacheEntry<V> entry = new CacheEntry<>(value, System.currentTimeMillis() + ttlMillis);
            cache.put(key, entry);
        }
        
        @Override
        public void invalidate(K key) {
            cache.invalidate(key);
        }
        
        @Override
        public void clear() {
            cache.clear();
        }
        
        @Override
        public long size() {
            return cache.size();
        }
        
        // Cache entry with expiration
        private static class CacheEntry<V> {
            private final V value;
            private final long expirationTime;
            
            public CacheEntry(V value, long expirationTime) {
                this.value = value;
                this.expirationTime = expirationTime;
            }
            
            public V getValue() {
                return value;
            }
            
            public boolean isExpired() {
                return System.currentTimeMillis() > expirationTime;
            }
        }
    }
    
    // Cache metrics tracking
    public static class CacheMetrics {
        private final AtomicLong l1Hits = new AtomicLong(0);
        private final AtomicLong l2Hits = new AtomicLong(0);
        private final AtomicLong misses = new AtomicLong(0);
        private final AtomicLong puts = new AtomicLong(0);
        private final AtomicLong invalidates = new AtomicLong(0);
        
        public void recordL1Hit() {
            l1Hits.incrementAndGet();
        }
        
        public void recordL2Hit() {
            l2Hits.incrementAndGet();
        }
        
        public void recordMiss() {
            misses.incrementAndGet();
        }
        
        public void recordPut() {
            puts.incrementAndGet();
        }
        
        public void recordInvalidate() {
            invalidates.incrementAndGet();
        }
        
        public long getL1Hits() {
            return l1Hits.get();
        }
        
        public long getL2Hits() {
            return l2Hits.get();
        }
        
        public long getMisses() {
            return misses.get();
        }
        
        public long getPuts() {
            return puts.get();
        }
        
        public long getInvalidates() {
            return invalidates.get();
        }
    }
    
    // Cache manager with monitoring
    public static class CacheManager {
        private final Map<String, Cache<?, ?>> caches = new ConcurrentHashMap<>();
        private final ScheduledExecutorService scheduler = 
            Executors.newScheduledThreadPool(2);
        
        public <K, V> void registerCache(String name, Cache<K, V> cache) {
            caches.put(name, cache);
        }
        
        public <K, V> Cache<K, V> getCache(String name) {
            return (Cache<K, V>) caches.get(name);
        }
        
        public void startMonitoring() {
            scheduler.scheduleAtFixedRate(this::printCacheStats, 0, 1, TimeUnit.MINUTES);
        }
        
        private void printCacheStats() {
            System.out.println("=== Cache Statistics ===");
            caches.forEach((name, cache) -> {
                System.out.println(name + ": " + cache.size() + " entries");
            });
        }
        
        public void shutdown() {
            scheduler.shutdown();
            try {
                if (!scheduler.awaitTermination(10, TimeUnit.SECONDS)) {
                    scheduler.shutdownNow();
                }
            } catch (InterruptedException e) {
                scheduler.shutdownNow();
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

## Database Migration and Schema Management

### Advanced Migration Strategies

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Consumer;
import java.sql.*;

public class DatabaseMigrationAdvanced {
    
    // Versioned migration system
    public static class VersionedMigration {
        private final List<Migration> migrations = new ArrayList<>();
        private final DataSource dataSource;
        private final String migrationTable;
        
        public VersionedMigration(DataSource dataSource, String migrationTable) {
            this.dataSource = dataSource;
            this.migrationTable = migrationTable;
            initializeMigrationTable();
        }
        
        public void addMigration(int version, String description, String upScript, String downScript) {
            migrations.add(new Migration(version, description, upScript, downScript));
            migrations.sort(Comparator.comparingInt(Migration::getVersion));
        }
        
        public void migrate() throws SQLException {
            int currentVersion = getCurrentVersion();
            
            for (Migration migration : migrations) {
                if (migration.getVersion() > currentVersion) {
                    executeMigration(migration);
                    updateVersion(migration.getVersion());
                }
            }
        }
        
        public void rollback(int targetVersion) throws SQLException {
            int currentVersion = getCurrentVersion();
            
            if (targetVersion >= currentVersion) {
                throw new IllegalArgumentException("Target version must be less than current version");
            }
            
            List<Migration> migrationsToRollback = migrations.stream()
                .filter(m -> m.getVersion() > targetVersion && m.getVersion() <= currentVersion)
                .sorted((m1, m2) -> Integer.compare(m2.getVersion(), m1.getVersion()))
                .collect(ArrayList::new, (list, item) -> list.add(0, item), (list1, list2) -> list1.addAll(0, list2));
            
            for (Migration migration : migrationsToRollback) {
                executeRollback(migration);
                updateVersion(migration.getVersion() - 1);
            }
        }
        
        private void initializeMigrationTable() {
            try (Connection connection = dataSource.getConnection();
                 Statement statement = connection.createStatement()) {
                
                statement.execute("CREATE TABLE IF NOT EXISTS " + migrationTable + " (" +
                                "version INT PRIMARY KEY, " +
                                "description VARCHAR(255), " +
                                "applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)");
            } catch (SQLException e) {
                throw new RuntimeException("Failed to initialize migration table", e);
            }
        }
        
        private int getCurrentVersion() throws SQLException {
            try (Connection connection = dataSource.getConnection();
                 PreparedStatement statement = connection.prepareStatement(
                     "SELECT MAX(version) FROM " + migrationTable)) {
                
                ResultSet resultSet = statement.executeQuery();
                if (resultSet.next()) {
                    return resultSet.getInt(1);
                }
                return 0;
            }
        }
        
        private void executeMigration(Migration migration) throws SQLException {
            try (Connection connection = dataSource.getConnection();
                 Statement statement = connection.createStatement()) {
                
                connection.setAutoCommit(false);
                
                try {
                    statement.execute(migration.getUpScript());
                    connection.commit();
                } catch (SQLException e) {
                    connection.rollback();
                    throw e;
                }
            }
        }
        
        private void executeRollback(Migration migration) throws SQLException {
            try (Connection connection = dataSource.getConnection();
                 Statement statement = connection.createStatement()) {
                
                connection.setAutoCommit(false);
                
                try {
                    statement.execute(migration.getDownScript());
                    connection.commit();
                } catch (SQLException e) {
                    connection.rollback();
                    throw e;
                }
            }
        }
        
        private void updateVersion(int version) throws SQLException {
            try (Connection connection = dataSource.getConnection();
                 PreparedStatement statement = connection.prepareStatement(
                     "INSERT INTO " + migrationTable + " (version, description) VALUES (?, ?)")) {
                
                // Find migration description
                String description = migrations.stream()
                    .filter(m -> m.getVersion() == version)
                    .map(Migration::getDescription)
                    .findFirst()
                    .orElse("Unknown migration");
                
                statement.setInt(1, version);
                statement.setString(2, description);
                statement.executeUpdate();
            }
        }
        
        private static class Migration {
            private final int version;
            private final String description;
            private final String upScript;
            private final String downScript;
            
            public Migration(int version, String description, String upScript, String downScript) {
                this.version = version;
                this.description = description;
                this.upScript = upScript;
                this.downScript = downScript;
            }
            
            public int getVersion() { return version; }
            public String getDescription() { return description; }
            public String getUpScript() { return upScript; }
            public String getDownScript() { return downScript; }
        }
    }
    
    // Zero-downtime migration
    public static class ZeroDowntimeMigration {
        private final DataSource dataSource;
        private final ExecutorService executor = Executors.newFixedThreadPool(10);
        private final AtomicBoolean migrationInProgress = new AtomicBoolean(false);
        
        public ZeroDowntimeMigration(DataSource dataSource) {
            this.dataSource = dataSource;
        }
        
        public void performZeroDowntimeMigration(String tableName, 
                                               Map<String, String> newColumns,
                                               Consumer<Connection> dataMigrationLogic) 
                throws SQLException {
            
            if (!migrationInProgress.compareAndSet(false, true)) {
                throw new IllegalStateException("Migration already in progress");
            }
            
            try {
                // Phase 1: Add new columns (backwards compatible)
                addNewColumns(tableName, newColumns);
                
                // Phase 2: Migrate data in background
                migrateDataAsync(tableName, dataMigrationLogic);
                
                // Phase 3: Wait for data migration to complete
                waitForDataMigration();
                
                // Phase 4: Drop old columns (optional, requires new deployment)
                // dropOldColumns(tableName, oldColumns);
                
            } finally {
                migrationInProgress.set(false);
            }
        }
        
        private void addNewColumns(String tableName, Map<String, String> newColumns) 
                throws SQLException {
            try (Connection connection = dataSource.getConnection();
                 Statement statement = connection.createStatement()) {
                
                for (Map.Entry<String, String> column : newColumns.entrySet()) {
                    String sql = "ALTER TABLE " + tableName + 
                               " ADD COLUMN IF NOT EXISTS " + column.getKey() + 
                               " " + column.getValue();
                    statement.execute(sql);
                }
            }
        }
        
        private void migrateDataAsync(String tableName, Consumer<Connection> dataMigrationLogic) {
            executor.submit(() -> {
                try (Connection connection = dataSource.getConnection()) {
                    dataMigrationLogic.accept(connection);
                } catch (SQLException e) {
                    System.err.println("Data migration failed: " + e.getMessage());
                }
            });
        }
        
        private void waitForDataMigration() throws SQLException {
            // In a real implementation, you would have a more sophisticated
            // mechanism to track migration progress
            try {
                Thread.sleep(5000); // Wait for background migration
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new SQLException("Migration interrupted", e);
            }
        }
        
        private void dropOldColumns(String tableName, List<String> oldColumns) 
                throws SQLException {
            try (Connection connection = dataSource.getConnection();
                 Statement statement = connection.createStatement()) {
                
                for (String column : oldColumns) {
                    String sql = "ALTER TABLE " + tableName + " DROP COLUMN " + column;
                    statement.execute(sql);
                }
            }
        }
    }
    
    // Database migration with rollback support
    public static class RollbackableMigration {
        private final DataSource dataSource;
        private final List<MigrationStep> migrationSteps = new ArrayList<>();
        private final Stack<MigrationStep> executedSteps = new Stack<>();
        
        public RollbackableMigration(DataSource dataSource) {
            this.dataSource = dataSource;
        }
        
        public void addMigrationStep(String description, 
                                   String upSql, 
                                   String downSql) {
            migrationSteps.add(new MigrationStep(description, upSql, downSql));
        }
        
        public void executeMigration() throws SQLException {
            for (MigrationStep step : migrationSteps) {
                try {
                    executeStep(step.getUpSql());
                    executedSteps.push(step);
                } catch (SQLException e) {
                    // Rollback all executed steps
                    rollback();
                    throw e;
                }
            }
        }
        
        private void executeStep(String sql) throws SQLException {
            try (Connection connection = dataSource.getConnection();
                 Statement statement = connection.createStatement()) {
                statement.execute(sql);
            }
        }
        
        private void rollback() {
            while (!executedSteps.isEmpty()) {
                MigrationStep step = executedSteps.pop();
                try {
                    executeStep(step.getDownSql());
                } catch (SQLException e) {
                    System.err.println("Rollback failed for step: " + step.getDescription());
                }
            }
        }
        
        private static class MigrationStep {
            private final String description;
            private final String upSql;
            private final String downSql;
            
            public MigrationStep(String description, String upSql, String downSql) {
                this.description = description;
                this.upSql = upSql;
                this.downSql = downSql;
            }
            
            public String getDescription() { return description; }
            public String getUpSql() { return upSql; }
            public String getDownSql() { return downSql; }
        }
    }
    
    // Migration monitoring and reporting
    public static class MigrationMonitor {
        private final DataSource dataSource;
        private final String migrationTable;
        private final List<MigrationListener> listeners = new ArrayList<>();
        
        public MigrationMonitor(DataSource dataSource, String migrationTable) {
            this.dataSource = dataSource;
            this.migrationTable = migrationTable;
        }
        
        public void addListener(MigrationListener listener) {
            listeners.add(listener);
        }
        
        public MigrationReport generateReport() throws SQLException {
            MigrationReport report = new MigrationReport();
            
            try (Connection connection = dataSource.getConnection();
                 Statement statement = connection.createStatement();
                 ResultSet resultSet = statement.executeQuery(
                     "SELECT version, description, applied_at FROM " + migrationTable + 
                     " ORDER BY version")) {
                
                while (resultSet.next()) {
                    report.addAppliedMigration(
                        resultSet.getInt("version"),
                        resultSet.getString("description"),
                        resultSet.getTimestamp("applied_at")
                    );
                }
            }
            
            return report;
        }
        
        public void notifyMigrationStarted(int version, String description) {
            MigrationEvent event = new MigrationEvent(version, description, 
                                                    MigrationEvent.Type.STARTED);
            listeners.forEach(listener -> listener.onMigrationEvent(event));
        }
        
        public void notifyMigrationCompleted(int version, String description) {
            MigrationEvent event = new MigrationEvent(version, description, 
                                                    MigrationEvent.Type.COMPLETED);
            listeners.forEach(listener -> listener.onMigrationEvent(event));
        }
        
        public void notifyMigrationFailed(int version, String description, Exception error) {
            MigrationEvent event = new MigrationEvent(version, description, 
                                                    MigrationEvent.Type.FAILED, error);
            listeners.forEach(listener -> listener.onMigrationEvent(event));
        }
        
        public interface MigrationListener {
            void onMigrationEvent(MigrationEvent event);
        }
        
        public static class MigrationEvent {
            public enum Type { STARTED, COMPLETED, FAILED }
            
            private final int version;
            private final String description;
            private final Type type;
            private final Exception error;
            private final long timestamp;
            
            public MigrationEvent(int version, String description, Type type) {
                this(version, description, type, null);
            }
            
            public MigrationEvent(int version, String description, Type type, Exception error) {
                this.version = version;
                this.description = description;
                this.type = type;
                this.error = error;
                this.timestamp = System.currentTimeMillis();
            }
            
            // Getters
            public int getVersion() { return version; }
            public String getDescription() { return description; }
            public Type getType() { return type; }
            public Exception getError() { return error; }
            public long getTimestamp() { return timestamp; }
        }
        
        public static class MigrationReport {
            private final List<AppliedMigration> appliedMigrations = new ArrayList<>();
            
            public void addAppliedMigration(int version, String description, Timestamp appliedAt) {
                appliedMigrations.add(new AppliedMigration(version, description, appliedAt));
            }
            
            public List<AppliedMigration> getAppliedMigrations() {
                return new ArrayList<>(appliedMigrations);
            }
            
            public String generateSummary() {
                StringBuilder sb = new StringBuilder();
                sb.append("=== Migration Report ===\n");
                sb.append("Total migrations applied: ").append(appliedMigrations.size()).append("\n");
                
                appliedMigrations.forEach(migration -> {
                    sb.append("Version ").append(migration.getVersion())
                      .append(": ").append(migration.getDescription())
                      .append(" (").append(migration.getAppliedAt()).append(")\n");
                });
                
                return sb.toString();
            }
            
            private static class AppliedMigration {
                private final int version;
                private final String description;
                private final Timestamp appliedAt;
                
                public AppliedMigration(int version, String description, Timestamp appliedAt) {
                    this.version = version;
                    this.description = description;
                    this.appliedAt = appliedAt;
                }
                
                // Getters
                public int getVersion() { return version; }
                public String getDescription() { return description; }
                public Timestamp getAppliedAt() { return appliedAt; }
            }
        }
    }
}
```

## Conclusion

Advanced Database Access Patterns provide:
- Sophisticated connection pooling and management strategies
- Database sharding and partitioning techniques for scalability
- Read/write separation and replication for high availability
- Multi-level caching for performance optimization
- Zero-downtime migration strategies for continuous deployment

Mastering these patterns is essential for:
- Building scalable, high-performance database applications
- Managing complex data access requirements
- Ensuring data consistency and availability
- Optimizing database performance and resource utilization
- Implementing robust, production-ready data access layers