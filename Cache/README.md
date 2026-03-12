# Phần 1: Caching là gì?

Caching là một kỹ thuật lưu trữ dữ liệu tạm thời ở một nơi truy cập nhanh (thường là RAM) để giảm thời gian truy xuất cho các request tiếp theo. Thay vì phải truy vấn database hoặc tính toán lại mỗi lần, hệ thống có thể lấy dữ liệu trực tiếp từ cache, giúp cải thiện đáng kể hiệu suất.

**Vì sao cần Caching/Redis?**

- **Tăng hiệu suất**: Truy xuất dữ liệu từ RAM (cache) nhanh hơn rất nhiều so với disk (database). Redis có thể xử lý hàng trăm nghìn operations/giây.
- **Giảm tải cho Database**: Các truy vấn phổ biến được cache lại, giảm số lượng queries đến database, tránh overload.
- **Cải thiện trải nghiệm người dùng**: Response time nhanh hơn, giảm latency, người dùng có trải nghiệm mượt mà hơn.
- **Tiết kiệm chi phí**: Giảm tải database có thể giúp sử dụng instance nhỏ hơn, tiết kiệm chi phí infrastructure.
- **Khả năng mở rộng**: Cache giúp hệ thống có thể phục vụ nhiều users hơn với cùng một database.

**Lợi ích của việc sử dụng Caching**

- **Hiệu suất cao**: Latency thấp (sub-millisecond), throughput lớn.
- **Giảm Database Load**: Database chỉ cần xử lý cache miss và write operations.
- **Dễ dàng scale**: Có thể thêm cache nodes để tăng capacity.
- **Flexible TTL**: Kiểm soát thời gian sống của dữ liệu cache.
- **Rich data structures**: Redis hỗ trợ nhiều kiểu dữ liệu (String, List, Set, Sorted Set, Hash...).

**Nhược điểm của việc sử dụng Caching**

- **Stale data (dữ liệu cũ)**: Cache có thể chứa dữ liệu đã lỗi thời nếu không được invalidate kịp thời.
- **Cache consistency**: Khó đảm bảo đồng bộ giữa cache và database trong môi trường distributed.
- **Thêm độ phức tạp**: Cần quản lý cache invalidation, warming, monitoring.
- **Memory cost**: Redis lưu trữ trong RAM, đắt hơn disk storage.
- **Cache failures**: Nếu cache server down, có thể gây "cache stampede" - nhiều requests đổ về database cùng lúc.

Vì vậy, việc sử dụng Caching cần được thiết kế cẩn thận với các strategies phù hợp để tận dụng tối đa lợi ích và giảm thiểu rủi ro.

---

# Phần 2: Các chiến lược Caching phổ biến

## 1. Cache-Aside (Lazy Loading)

**Cách hoạt động:**
```
1. Application kiểm tra cache
2. Nếu có (cache hit) → trả về ngay
3. Nếu không có (cache miss):
   - Query database
   - Lưu vào cache
   - Trả về kết quả
```

**Ưu điểm:**
- Chỉ cache dữ liệu thực sự được sử dụng
- Cache miss không gây lỗi (fallback to DB)
- Dễ implement

**Nhược điểm:**
- Cache miss có latency cao (query DB + set cache)
- Có thể có stale data

**Use case:**
- Read-heavy applications
- Dữ liệu ít thay đổi (user profiles, product catalog)

**Implementation:**
```java
@Cacheable(value = "users", key = "#id")
public Optional<User> getUserById(Long id) {
    // Spring tự động check cache trước
    // Nếu miss, query DB và cache lại
    return userRepository.findById(id);
}
```

---

## 2. Write-Through

**Cách hoạt động:**
```
1. Application write data
2. Update cache trước (hoặc cùng lúc)
3. Write to database
4. Trả về kết quả
```

**Ưu điểm:**
- Cache luôn consistent với database
- Read sau write ngay lập tức có trong cache (no cache miss)

**Nhược điểm:**
- Write latency cao hơn (phải write cả cache và DB)
- Có thể cache nhiều dữ liệu không cần thiết

**Use case:**
- Write-heavy với read ngay sau write
- Cần consistency cao

**Implementation:**
```java
@CachePut(value = "users", key = "#id")
public User updateUser(Long id, User userDetails) {
    // Spring tự động update cache với return value
    User updatedUser = userRepository.save(userDetails);
    return updatedUser;
}
```

---

## 3. Write-Behind (Write-Back)

**Cách hoạt động:**
```
1. Application write to cache
2. Trả về ngay (async)
3. Cache service write to DB sau (batch/periodic)
```

**Ưu điểm:**
- Write latency rất thấp
- Có thể batch multiple writes

**Nhược điểm:**
- Rủi ro mất dữ liệu nếu cache crash trước khi sync DB
- Phức tạp trong implementation

**Use case:**
- High write throughput (logging, analytics)
- Có thể chấp nhận eventual consistency

---

## 4. Refresh-Ahead

**Cách hoạt động:**
```
1. Monitor TTL của cached items
2. Trước khi expire, tự động refresh từ DB
3. User luôn hit fresh cache
```

**Ưu điểm:**
- Giảm cache miss
- Predictable performance

**Nhược điểm:**
- Tốn resources refresh dữ liệu ít dùng
- Phức tạp hơn

**Use case:**
- Critical data cần high availability
- Predictable access patterns

---

## So sánh các chiến lược

| Chiến lược | Read Latency | Write Latency | Consistency | Complexity | Use Case |
|------------|--------------|---------------|-------------|------------|----------|
| Cache-Aside (Lazy) | Medium (miss) / Low (hit) | Low | Eventual | Low | ✅ Recommended phổ biến |
| Write-Through | Low | High | Strong | Medium | High consistency needs |
| Write-Behind | Low | Very Low | Eventual | High | High write throughput |
| Refresh-Ahead | Very Low | Low | Eventual | High | Critical hot data |

**Recommendation cho dự án**: Sử dụng **Cache-Aside** kết hợp với **Write-Through** cho update operations.

---

# Phần 3: Kiểu dữ liệu cơ bản trong Redis

Redis không chỉ là key-value store đơn giản mà hỗ trợ nhiều data structures phức tạp.

## 1. String

**Mô tả:** Kiểu dữ liệu cơ bản nhất, lưu text hoặc binary data (tối đa 512MB).

**Use cases:**
- Cache JSON objects
- Session storage
- Counters, rate limiting

**Commands:**
```redis
SET user:1 "John Doe"               # Lưu string
GET user:1                          # Lấy string
SETEX user:1 3600 "John Doe"       # Set với TTL 3600s
INCR counter                        # Tăng counter lên 1
INCRBY views:post:1 10             # Tăng lên 10
```

**Java/Spring implementation:**
```java
redisTemplate.opsForValue().set("user:1", userObject);
User user = (User) redisTemplate.opsForValue().get("user:1");
```

---

## 2. Hash

**Mô tả:** Lưu trữ field-value pairs, giống như object/map.

**Use cases:**
- User profiles (nhiều fields)
- Product details
- Configuration settings

**Commands:**
```redis
HSET user:1 name "John" age 30      # Set multiple fields
HGET user:1 name                    # Get single field
HGETALL user:1                      # Get all fields
HINCRBY user:1 age 1               # Increment age field
```

**Java implementation:**
```java
redisTemplate.opsForHash().put("user:1", "name", "John");
String name = (String) redisTemplate.opsForHash().get("user:1", "name");
Map<Object, Object> user = redisTemplate.opsForHash().entries("user:1");
```

**Ưu điểm:** Tiết kiệm memory hơn so với nhiều key riêng lẻ.

---

## 3. List

**Mô tả:** Danh sách có thứ tự, có thể add/remove từ cả hai đầu (head/tail).

**Use cases:**
- Activity feeds
- Task queues
- Recent items
- Notifications

**Commands:**
```redis
LPUSH messages "Hello"              # Add to head (left)
RPUSH messages "World"              # Add to tail (right)
LRANGE messages 0 -1                # Get all items
LPOP messages                       # Remove from head
RPOP messages                       # Remove from tail
LLEN messages                       # Get list length
```

**Java implementation:**
```java
redisTemplate.opsForList().leftPush("notifications", notification);
List<Object> recent = redisTemplate.opsForList().range("notifications", 0, 9); // 10 recent
```

---

## 4. Set

**Mô tả:** Tập hợp các giá trị unique, không có thứ tự.

**Use cases:**
- Tags
- Unique visitors
- Relationships (followers, friends)
- Remove duplicates

**Commands:**
```redis
SADD tags:post:1 "java" "redis" "cache"  # Add members
SMEMBERS tags:post:1                      # Get all members
SISMEMBER tags:post:1 "java"             # Check if member exists
SCARD tags:post:1                         # Get set size
SINTER tags:post:1 tags:post:2           # Intersection (common tags)
```

**Java implementation:**
```java
redisTemplate.opsForSet().add("unique:visitors", userId);
Boolean exists = redisTemplate.opsForSet().isMember("unique:visitors", userId);
```

---

## 5. Sorted Set (ZSet)

**Mô tả:** Set với score cho mỗi member, tự động sắp xếp theo score.

**Use cases:**
- **Leaderboards (bảng xếp hạng)**
- Priority queues
- Time-series data
- Rate limiting with sliding window

**Commands:**
```redis
ZADD leaderboard 100 "player1"      # Add with score
ZADD leaderboard 200 "player2"
ZADD leaderboard 150 "player3"
ZREVRANGE leaderboard 0 9           # Top 10 (descending)
ZRANK leaderboard "player1"         # Get rank (ascending)
ZINCRBY leaderboard 50 "player1"    # Increase score
ZCOUNT leaderboard 100 200          # Count in range
```

**Java implementation:**
```java
redisTemplate.opsForZSet().add("leaderboard", userId, score);
Set<Object> top10 = redisTemplate.opsForZSet().reverseRange("leaderboard", 0, 9);
Long rank = redisTemplate.opsForZSet().reverseRank("leaderboard", userId);
```

---

## Bảng so sánh các kiểu dữ liệu

| Data Type | Ordering | Unique | Use Case | Commands |
|-----------|----------|--------|----------|----------|
| String | No | N/A | Simple cache, counters | GET, SET, INCR |
| Hash | No | Fields unique | Objects, profiles | HGET, HSET, HGETALL |
| List | Yes | No | Queues, feeds | LPUSH, RPUSH, LRANGE |
| Set | No | Yes | Tags, unique items | SADD, SMEMBERS |
| Sorted Set | Yes (by score) | Yes | Leaderboards, rankings | ZADD, ZRANGE |

---

# Phần 4: Implement Caching với Spring Boot & Redis

## 1. Cài đặt Redis

### Cài đặt bằng Docker

```bash
docker run -d \
  --name redis \
  -p 6379:6379 \
  redis:7-alpine
```

Hoặc viết trong file `docker-compose.yml`:

```yaml
redis:
  image: redis:7-alpine
  container_name: redis
  ports:
    - "6379:6379"
  command: redis-server --appendonly yes
  volumes:
    - redis-data:/data
  networks:
    - minilab

volumes:
  redis-data:
```

### Khởi chạy

```bash
docker-compose up -d redis
```

### Kiểm tra Redis

```bash
# Kết nối Redis CLI
docker exec -it redis redis-cli

# Test commands
127.0.0.1:6379> PING
PONG
127.0.0.1:6379> SET test "Hello Redis"
OK
127.0.0.1:6379> GET test
"Hello Redis"
```

---

## 2. Thêm Dependencies vào Spring Boot

Thêm dependencies vào file `pom.xml`:

```xml
<!-- Redis Cache -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<!-- Cache abstraction -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>

<!-- Jedis client (optional, lettuce is default) -->
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>

<!-- JSON serialization -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
```

---

## 3. Cấu hình Redis Connection

Tạo file `application-caching.yml`:

```yaml
spring:
  data:
    redis:
      # Connection Configuration
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
      database: ${REDIS_DATABASE:0}
      
      # Connection Pool (Jedis)
      jedis:
        pool:
          max-active: 8          # Số connection tối đa
          max-idle: 8            # Số idle connection
          min-idle: 0            # Số idle tối thiểu
          max-wait: 5000ms       # Thời gian chờ connection
      
      # Timeout
      timeout: 5000ms
      connect-timeout: 5000ms
  
  # Spring Cache Configuration
  cache:
    type: redis
    redis:
      time-to-live: 3600000      # TTL mặc định: 1 giờ
      cache-null-values: true    # Cache null để tránh cache penetration
      use-key-prefix: true       # Dùng prefix cho keys
      key-prefix: "app:cache:"
```

---

## 4. Tạo Redis Configuration Class

Tạo file `RedisConfig.java`:

```java
package com.example.demo.config;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.jsontype.BasicPolymorphicTypeValidator;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;

/**
 * Redis Configuration với Cache và RedisTemplate
 * 
 * Features:
 * - Cache Manager cho Spring Cache abstraction
 * - RedisTemplate với JSON serialization
 * - Custom TTL cho từng cache
 */
@Configuration
@EnableCaching
public class RedisConfig {

    /**
     * Cache Manager với cấu hình custom
     * - JSON serialization
     * - Cache null values (tránh Cache Penetration)
     * - TTL per cache
     */
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        // Cấu hình Jackson để serialize objects
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.activateDefaultTyping(
            BasicPolymorphicTypeValidator.builder()
                .allowIfBaseType(Object.class)
                .build(),
            ObjectMapper.DefaultTyping.NON_FINAL,
            JsonTypeInfo.As.PROPERTY
        );
        
        GenericJackson2JsonRedisSerializer serializer = 
            new GenericJackson2JsonRedisSerializer(objectMapper);
        
        // Cấu hình Redis Cache mặc định
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))          // TTL mặc định
            .disableCachingNullValues()             // Không cache null
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new StringRedisSerializer())
            )
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(serializer)
            );
        
        // Tạo Cache Manager với custom TTL cho từng cache
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            // Cache-specific TTL
            .withCacheConfiguration("users", 
                config.entryTtl(Duration.ofMinutes(30)))   // 30 phút
            .withCacheConfiguration("items", 
                config.entryTtl(Duration.ofMinutes(15)))   // 15 phút
            .withCacheConfiguration("short-lived", 
                config.entryTtl(Duration.ofMinutes(5)))    // 5 phút
            .build();
    }

    /**
     * RedisTemplate để thao tác trực tiếp với Redis
     * Sử dụng cho Sorted Sets, Lists, advanced operations
     */
    @Bean
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory connectionFactory) {
        
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        
        // Serializers
        StringRedisSerializer stringSerializer = new StringRedisSerializer();
        GenericJackson2JsonRedisSerializer jsonSerializer = 
            new GenericJackson2JsonRedisSerializer();
        
        // Key serializers
        template.setKeySerializer(stringSerializer);
        template.setHashKeySerializer(stringSerializer);
        
        // Value serializers
        template.setValueSerializer(jsonSerializer);
        template.setHashValueSerializer(jsonSerializer);
        
        template.afterPropertiesSet();
        return template;
    }
}
```

---

## 5. Tạo Redis Cache Service (Helper)

Tạo file `RedisCacheService.java`:

```java
package com.example.demo.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Random;
import java.util.Set;
import java.util.concurrent.TimeUnit;

/**
 * Redis Cache Service - Helper để thao tác với Redis
 * 
 * Features:
 * - Get/Set/Delete với TTL
 * - Random TTL để tránh Cache Avalanche
 * - Cache Penetration protection
 * - Batch operations
 */
@Service
public class RedisCacheService {
    
    private static final Logger logger = LoggerFactory.getLogger(RedisCacheService.class);
    private static final Random random = new Random();
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    /**
     * Lưu value vào cache với TTL cố định
     */
    public void set(String key, Object value, Duration ttl) {
        try {
            redisTemplate.opsForValue().set(key, value, ttl);
            logger.debug("Cached key: {} with TTL: {}", key, ttl);
        } catch (Exception e) {
            logger.error("Error setting cache for key: {}", key, e);
        }
    }
    
    /**
     * Lưu value với random TTL để tránh Cache Avalanche
     * TTL = baseTtl + random(0 to variance)
     * 
     * Ví dụ: baseTtl=30 phút, variance=5 phút
     * → TTL thực tế trong khoảng 30-35 phút
     */
    public void setWithRandomTtl(String key, Object value, 
                                  Duration baseTtl, Duration variance) {
        try {
            long randomSeconds = random.nextInt((int) variance.getSeconds());
            Duration finalTtl = baseTtl.plusSeconds(randomSeconds);
            redisTemplate.opsForValue().set(key, value, finalTtl);
            logger.debug("Cached key: {} with randomized TTL: {}", key, finalTtl);
        } catch (Exception e) {
            logger.error("Error setting cache with random TTL for key: {}", key, e);
        }
    }
    
    /**
     * Lấy value từ cache
     */
    public Object get(String key) {
        try {
            Object value = redisTemplate.opsForValue().get(key);
            if (value != null) {
                logger.debug("Cache hit for key: {}", key);
            } else {
                logger.debug("Cache miss for key: {}", key);
            }
            return value;
        } catch (Exception e) {
            logger.error("Error getting cache for key: {}", key, e);
            return null;
        }
    }
    
    /**
     * Xóa key khỏi cache
     */
    public void delete(String key) {
        try {
            redisTemplate.delete(key);
            logger.debug("Deleted cache key: {}", key);
        } catch (Exception e) {
            logger.error("Error deleting cache for key: {}", key, e);
        }
    }
    
    /**
     * Xóa nhiều keys theo pattern
     * Ví dụ: deletePattern("user:*") xóa tất cả user keys
     */
    public void deletePattern(String pattern) {
        try {
            Set<String> keys = redisTemplate.keys(pattern);
            if (keys != null && !keys.isEmpty()) {
                redisTemplate.delete(keys);
                logger.debug("Deleted {} keys matching pattern: {}", 
                           keys.size(), pattern);
            }
        } catch (Exception e) {
            logger.error("Error deleting keys by pattern: {}", pattern, e);
        }
    }
    
    /**
     * Kiểm tra key có tồn tại không
     */
    public boolean exists(String key) {
        try {
            Boolean exists = redisTemplate.hasKey(key);
            return exists != null && exists;
        } catch (Exception e) {
            logger.error("Error checking existence of key: {}", key, e);
            return false;
        }
    }
    
    /**
     * Set expiration cho key đã tồn tại
     */
    public boolean expire(String key, Duration ttl) {
        try {
            Boolean result = redisTemplate.expire(key, ttl);
            return result != null && result;
        } catch (Exception e) {
            logger.error("Error setting expiration for key: {}", key, e);
            return false;
        }
    }
    
    /**
     * Lấy TTL còn lại của key (seconds)
     */
    public Long getTtl(String key) {
        try {
            return redisTemplate.getExpire(key, TimeUnit.SECONDS);
        } catch (Exception e) {
            logger.error("Error getting TTL for key: {}", key, e);
            return null;
        }
    }
    
    /**
     * Increment giá trị số (dùng cho counters, rate limiting)
     */
    public Long increment(String key, long delta) {
        try {
            return redisTemplate.opsForValue().increment(key, delta);
        } catch (Exception e) {
            logger.error("Error incrementing key: {}", key, e);
            return null;
        }
    }
    
    /**
     * Cache null value để tránh Cache Penetration
     * Dùng TTL ngắn cho null values
     */
    public void cacheNullValue(String key, Duration ttl) {
        try {
            redisTemplate.opsForValue().set(key, "NULL", ttl);
            logger.debug("Cached null value for key: {} with TTL: {}", key, ttl);
        } catch (Exception e) {
            logger.error("Error caching null value for key: {}", key, e);
        }
    }
    
    /**
     * Kiểm tra value có phải là cached null không
     */
    public boolean isCachedNull(Object value) {
        return "NULL".equals(value);
    }
}
```

---

## 6. Use Case 1: Cache User/Product Information

### 6.1. Tạo Cached Service

Tạo file `CachedUserService.java`:

```java
package com.example.demo.service;

import com.example.demo.model.User;
import com.example.demo.repository.UserRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.List;
import java.util.Optional;

/**
 * Cached User Service - Service với Redis caching
 * 
 * Caching Strategy:
 * - @Cacheable: Lấy từ cache nếu có, không thì query DB và cache lại
 * - @CachePut: Luôn update cache sau khi thay đổi
 * - @CacheEvict: Xóa cache khi delete
 */
@Service
public class CachedUserService {
    
    private static final Logger logger = LoggerFactory.getLogger(CachedUserService.class);
    private static final String USER_CACHE_PREFIX = "user:";
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private RedisCacheService cacheService;

    /**
     * Lấy tất cả users với cache
     * Cache key: "users::all"
     */
    @Cacheable(value = "users", key = "'all'")
    public List<User> getAllUsers() {
        logger.info("📀 Fetching all users from database (cache miss)");
        return userRepository.findAll();
    }

    /**
     * Lấy user theo ID với cache
     * Cache key: "users::{id}"
     * 
     * Cache Penetration Protection:
     * - Nếu user không tồn tại, cache null với TTL ngắn
     */
    @Cacheable(value = "users", key = "#id")
    public Optional<User> getUserById(Long id) {
        logger.info("📀 Fetching user {} from database (cache miss)", id);
        Optional<User> user = userRepository.findById(id);
        
        // Cache Penetration Protection
        if (user.isEmpty()) {
            logger.debug("⚠️ User {} not found, caching null value", id);
            cacheService.cacheNullValue(USER_CACHE_PREFIX + id, 
                                       Duration.ofMinutes(5));
        }
        
        return user;
    }
    
    /**
     * Tạo user mới
     * Invalidate cache "all" sau khi tạo
     */
    @CacheEvict(value = "users", key = "'all'")
    public User createUser(User user) {
        if (userRepository.existsByEmail(user.getEmail())) {
            throw new RuntimeException("Email already exists");
        }
        
        User savedUser = userRepository.save(user);
        
        // Manual cache với random TTL để tránh Cache Avalanche
        String cacheKey = USER_CACHE_PREFIX + savedUser.getId();
        cacheService.setWithRandomTtl(cacheKey, savedUser, 
            Duration.ofMinutes(30), Duration.ofMinutes(5));
        
        logger.info("✅ Created and cached new user: {}", savedUser.getId());
        return savedUser;
    }

    /**
     * Update user
     * @CachePut: Update cache với return value
     * @CacheEvict: Invalidate "all" cache
     */
    @CachePut(value = "users", key = "#id")
    @CacheEvict(value = "users", key = "'all'")
    public User updateUser(Long id, User userDetails) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("User not found: " + id));

        user.setName(userDetails.getName());
        user.setEmail(userDetails.getEmail());
        user.setPhone(userDetails.getPhone());
        user.setAddress(userDetails.getAddress());

        User updatedUser = userRepository.save(user);
        logger.info("✅ Updated and cached user: {}", id);
        return updatedUser;
    }

    /**
     * Xóa user
     * Evict all caches liên quan
     */
    @CacheEvict(value = "users", allEntries = true)
    public void deleteUser(Long id) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("User not found: " + id));
        
        userRepository.delete(user);
        
        // Manual delete cache key
        cacheService.delete(USER_CACHE_PREFIX + id);
        logger.info("🗑️ Deleted user and cleared cache: {}", id);
    }
    
    /**
     * Clear all user caches
     */
    @CacheEvict(value = "users", allEntries = true)
    public void clearAllCaches() {
        cacheService.deletePattern(USER_CACHE_PREFIX + "*");
        logger.info("🧹 Cleared all user caches");
    }
    
    /**
     * Warm up cache - Preload dữ liệu vào cache
     * Gọi lúc startup hoặc sau khi clear cache
     */
    public void warmUpCache() {
        logger.info("🔥 Warming up user cache...");
        List<User> users = userRepository.findAll();
        
        for (User user : users) {
            String cacheKey = USER_CACHE_PREFIX + user.getId();
            // Random TTL để tránh Cache Avalanche
            cacheService.setWithRandomTtl(cacheKey, user, 
                Duration.ofMinutes(30), Duration.ofMinutes(10));
        }
        
        logger.info("✅ Cache warmed up with {} users", users.size());
    }
}
```

### 6.2. Tạo Controller

Tạo file `CachedUserController.java`:

```java
package com.example.demo.controller;

import com.example.demo.model.User;
import com.example.demo.service.CachedUserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;

/**
 * Cached User Controller - API để demo Redis caching
 */
@RestController
@RequestMapping("/api/cached-users")
public class CachedUserController {
    
    @Autowired
    private CachedUserService cachedUserService;
    
    /**
     * GET /api/cached-users - Lấy tất cả users (cached)
     */
    @GetMapping
    public ResponseEntity<List<User>> getAllUsers() {
        List<User> users = cachedUserService.getAllUsers();
        return ResponseEntity.ok(users);
    }
    
    /**
     * GET /api/cached-users/{id} - Lấy user theo ID (cached)
     */
    @GetMapping("/{id}")
    public ResponseEntity<User> getUserById(@PathVariable Long id) {
        Optional<User> user = cachedUserService.getUserById(id);
        return user.map(ResponseEntity::ok)
                   .orElse(ResponseEntity.notFound().build());
    }
    
    /**
     * POST /api/cached-users - Tạo user mới
     */
    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody User user) {
        User createdUser = cachedUserService.createUser(user);
        return ResponseEntity.ok(createdUser);
    }
    
    /**
     * PUT /api/cached-users/{id} - Update user
     */
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(@PathVariable Long id, 
                                           @RequestBody User user) {
        User updatedUser = cachedUserService.updateUser(id, user);
        return ResponseEntity.ok(updatedUser);
    }
    
    /**
     * DELETE /api/cached-users/{id} - Xóa user
     */
    @DeleteMapping("/{id}")
    public ResponseEntity<?> deleteUser(@PathVariable Long id) {
        cachedUserService.deleteUser(id);
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("message", "User deleted successfully");
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * POST /api/cached-users/cache/clear - Clear all caches
     */
    @PostMapping("/cache/clear")
    public ResponseEntity<?> clearCache() {
        cachedUserService.clearAllCaches();
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("message", "All caches cleared");
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * POST /api/cached-users/cache/warmup - Warm up cache
     */
    @PostMapping("/cache/warmup")
    public ResponseEntity<?> warmUpCache() {
        cachedUserService.warmUpCache();
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("message", "Cache warmed up successfully");
        
        return ResponseEntity.ok(response);
    }
}
```

---

## 7. Use Case 2: Leaderboard với Sorted Set

### 7.1. Tạo Leaderboard Service

Tạo file `LeaderboardService.java`:

```java
package com.example.demo.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.ZSetOperations;
import org.springframework.stereotype.Service;

import java.util.*;

/**
 * Leaderboard Service - Bảng xếp hạng với Redis Sorted Set
 * 
 * Use case: Game leaderboard, product ratings, trending topics
 */
@Service
public class LeaderboardService {
    
    private static final Logger logger = LoggerFactory.getLogger(LeaderboardService.class);
    private static final String LEADERBOARD_KEY = "leaderboard:global";
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    /**
     * Thêm/update score cho player
     */
    public void addScore(String playerId, double score) {
        redisTemplate.opsForZSet().add(LEADERBOARD_KEY, playerId, score);
        logger.info("✅ Added/Updated player {} with score {}", playerId, score);
    }
    
    /**
     * Tăng score cho player
     */
    public Double incrementScore(String playerId, double delta) {
        Double newScore = redisTemplate.opsForZSet()
            .incrementScore(LEADERBOARD_KEY, playerId, delta);
        logger.info("✅ Incremented player {} by {}. New score: {}", 
                   playerId, delta, newScore);
        return newScore;
    }
    
    /**
     * Lấy Top N players (highest scores)
     */
    public List<Map<String, Object>> getTopPlayers(int topN) {
        // ZREVRANGE: Lấy từ cao xuống thấp
        Set<ZSetOperations.TypedTuple<Object>> topWithScores = 
            redisTemplate.opsForZSet()
                .reverseRangeWithScores(LEADERBOARD_KEY, 0, topN - 1);
        
        List<Map<String, Object>> result = new ArrayList<>();
        int rank = 1;
        
        if (topWithScores != null) {
            for (ZSetOperations.TypedTuple<Object> item : topWithScores) {
                Map<String, Object> player = new HashMap<>();
                player.put("rank", rank++);
                player.put("playerId", item.getValue());
                player.put("score", item.getScore());
                result.add(player);
            }
        }
        
        logger.info("📊 Retrieved top {} players", topN);
        return result;
    }
    
    /**
     * Lấy rank của player (1-based)
     */
    public Long getPlayerRank(String playerId) {
        // ZREVRANK: Rank từ cao xuống thấp (0-based)
        Long rank = redisTemplate.opsForZSet()
            .reverseRank(LEADERBOARD_KEY, playerId);
        
        // Convert to 1-based rank
        return rank != null ? rank + 1 : null;
    }
    
    /**
     * Lấy score của player
     */
    public Double getPlayerScore(String playerId) {
        return redisTemplate.opsForZSet()
            .score(LEADERBOARD_KEY, playerId);
    }
    
    /**
     * Lấy thông tin đầy đủ của player
     */
    public Map<String, Object> getPlayerInfo(String playerId) {
        Double score = getPlayerScore(playerId);
        Long rank = getPlayerRank(playerId);
        
        if (score == null) {
            return null;
        }
        
        Map<String, Object> info = new HashMap<>();
        info.put("playerId", playerId);
        info.put("score", score);
        info.put("rank", rank);
        
        return info;
    }
    
    /**
     * Xóa player khỏi leaderboard
     */
    public void removePlayer(String playerId) {
        redisTemplate.opsForZSet().remove(LEADERBOARD_KEY, playerId);
        logger.info("🗑️ Removed player {} from leaderboard", playerId);
    }
    
    /**
     * Lấy tổng số players
     */
    public Long getTotalPlayers() {
        return redisTemplate.opsForZSet().size(LEADERBOARD_KEY);
    }
    
    /**
     * Lấy players trong range score
     */
    public Set<Object> getPlayersByScoreRange(double minScore, double maxScore) {
        return redisTemplate.opsForZSet()
            .rangeByScore(LEADERBOARD_KEY, minScore, maxScore);
    }
    
    /**
     * Clear toàn bộ leaderboard
     */
    public void clearLeaderboard() {
        redisTemplate.delete(LEADERBOARD_KEY);
        logger.info("🧹 Cleared leaderboard");
    }
}
```

### 7.2. Tạo Leaderboard Controller

```java
package com.example.demo.controller;

import com.example.demo.service.LeaderboardService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/leaderboard")
public class LeaderboardController {
    
    @Autowired
    private LeaderboardService leaderboardService;
    
    /**
     * POST /api/leaderboard/score - Thêm/update score
     */
    @PostMapping("/score")
    public ResponseEntity<?> addScore(@RequestBody Map<String, Object> request) {
        String playerId = (String) request.get("playerId");
        Double score = ((Number) request.get("score")).doubleValue();
        
        leaderboardService.addScore(playerId, score);
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("playerId", playerId);
        response.put("score", score);
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * POST /api/leaderboard/increment - Tăng score
     */
    @PostMapping("/increment")
    public ResponseEntity<?> incrementScore(@RequestBody Map<String, Object> request) {
        String playerId = (String) request.get("playerId");
        Double delta = ((Number) request.get("delta")).doubleValue();
        
        Double newScore = leaderboardService.incrementScore(playerId, delta);
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("playerId", playerId);
        response.put("newScore", newScore);
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * GET /api/leaderboard/top/{n} - Lấy top N
     */
    @GetMapping("/top/{n}")
    public ResponseEntity<List<Map<String, Object>>> getTopPlayers(
            @PathVariable int n) {
        List<Map<String, Object>> topPlayers = leaderboardService.getTopPlayers(n);
        return ResponseEntity.ok(topPlayers);
    }
    
    /**
     * GET /api/leaderboard/player/{id} - Lấy thông tin player
     */
    @GetMapping("/player/{id}")
    public ResponseEntity<?> getPlayerInfo(@PathVariable String id) {
        Map<String, Object> info = leaderboardService.getPlayerInfo(id);
        
        if (info == null) {
            return ResponseEntity.notFound().build();
        }
        
        return ResponseEntity.ok(info);
    }
    
    /**
     * GET /api/leaderboard/stats - Thống kê
     */
    @GetMapping("/stats")
    public ResponseEntity<?> getStats() {
        Map<String, Object> stats = new HashMap<>();
        stats.put("totalPlayers", leaderboardService.getTotalPlayers());
        
        return ResponseEntity.ok(stats);
    }
    
    /**
     * DELETE /api/leaderboard/player/{id} - Xóa player
     */
    @DeleteMapping("/player/{id}")
    public ResponseEntity<?> removePlayer(@PathVariable String id) {
        leaderboardService.removePlayer(id);
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("message", "Player removed from leaderboard");
        
        return ResponseEntity.ok(response);
    }
}
```

---

# Phần 5: Lưu ý quan trọng khi dùng Cache

## 1. TTL (Time to Live)

**Vấn đề:** Dữ liệu trong cache có thể trở nên lỗi thời (stale).

**Best Practices:**

```java
// ✅ Đặt TTL phù hợp với tính chất dữ liệu
Duration ttl;

// Hot data, ít thay đổi
ttl = Duration.ofHours(24);     // User profiles

// Medium volatility
ttl = Duration.ofMinutes(30);   // Product listings

// High volatility
ttl = Duration.ofMinutes(5);    // Stock prices, trending

// Null values (cache penetration protection)
ttl = Duration.ofMinutes(5);    // Short TTL for nulls
```

**Monitoring TTL:**

```java
public void checkTtl(String key) {
    Long ttl = cacheService.getTtl(key);
    logger.info("Key {} TTL: {} seconds", key, ttl);
}
```

---

## 2. Cache Invalidation

**Vấn đề:** "There are only two hard things in Computer Science: cache invalidation and naming things."

**Strategies:**

### a. Time-based (TTL)
```java
// Tự động expire sau TTL
@Cacheable(value = "users", key = "#id")
```

### b. Event-based
```java
// Invalidate khi update
@CacheEvict(value = "users", key = "#id")
public User updateUser(Long id, User user) {
    return userRepository.save(user);
}

// Invalidate nhiều keys
@CacheEvict(value = "users", allEntries = true)
public void clearAllUsers() {}
```

### c. Manual Invalidation
```java
// Delete specific key
cacheService.delete("user:123");

// Delete by pattern
cacheService.deletePattern("user:*");
```

### d. Write-Through Pattern
```java
// Update DB và cache cùng lúc
@CachePut(value = "users", key = "#id")
public User updateUser(Long id, User user) {
    return userRepository.save(user);
}
```

---

## 3. Cache Avalanche

**Vấn đề:** Nhiều cache keys expire cùng lúc → Database bị quá tải.

**Timeline:**
```
09:00:00 - Cache 1000 users, TTL = 30 phút
09:30:00 - Tất cả expire cùng lúc
09:30:01 - 1000 requests đổ về DB → OVERLOAD!
```

**Solutions:**

### a. Random TTL
```java
// ✅ Random TTL để phân tán expiration
public void cacheWithRandomTtl(String key, Object value) {
    Duration baseTtl = Duration.ofMinutes(30);
    Duration variance = Duration.ofMinutes(5);
    
    // TTL = 30-35 phút (random)
    cacheService.setWithRandomTtl(key, value, baseTtl, variance);
}
```

### b. Never Expire + Background Refresh
```java
// Cache không bao giờ expire, refresh background
@Scheduled(fixedRate = 25 * 60 * 1000) // 25 phút
public void refreshCache() {
    List<User> users = userRepository.findAll();
    users.forEach(user -> {
        cacheService.set("user:" + user.getId(), user, Duration.ofMinutes(30));
    });
}
```

---

## 4. Cache Penetration

**Vấn đề:** Request dữ liệu không tồn tại → Bypass cache → Query DB liên tục.

**Attack scenario:**
```
Request: GET /api/users/999999999 (không tồn tại)
→ Cache miss
→ Query DB → không có
→ Không cache gì
→ Request tiếp theo lại query DB
→ Lặp lại → DB overload
```

**Solutions:**

### a. Cache Null Values
```java
@Cacheable(value = "users", key = "#id")
public Optional<User> getUserById(Long id) {
    Optional<User> user = userRepository.findById(id);
    
    // ✅ Cache null với TTL ngắn
    if (user.isEmpty()) {
        cacheService.cacheNullValue("user:" + id, Duration.ofMinutes(5));
    }
    
    return user;
}
```

### b. Bloom Filter (Advanced)
```java
// Check Bloom Filter trước khi query
if (!bloomFilter.mightContain(userId)) {
    // Chắc chắn không tồn tại → return empty ngay
    return Optional.empty();
}

// Có thể tồn tại → proceed to cache/DB
return getUserById(userId);
```

### c. Parameter Validation
```java
// ✅ Validate input
public User getUserById(Long id) {
    if (id == null || id <= 0 || id > MAX_USER_ID) {
        throw new IllegalArgumentException("Invalid user ID");
    }
    // ...
}
```

---

## 5. Cache Stampede (Thundering Herd)

**Vấn đề:** Cache expire → Nhiều requests cùng query DB → Duplicate work.

**Timeline:**
```
09:00:00.000 - Cache expire
09:00:00.001 - Request 1: cache miss → query DB
09:00:00.002 - Request 2: cache miss → query DB
09:00:00.003 - Request 3: cache miss → query DB
...
09:00:00.100 - 100 requests đều query DB cùng lúc!
```

**Solutions:**

### a. Locking/Synchronization
```java
private final Map<String, Object> locks = new ConcurrentHashMap<>();

public User getUserById(Long id) {
    String cacheKey = "user:" + id;
    
    // Check cache
    User cached = (User) cacheService.get(cacheKey);
    if (cached != null) return cached;
    
    // ✅ Synchronized block để chỉ 1 thread query DB
    synchronized (locks.computeIfAbsent(cacheKey, k -> new Object())) {
        // Double-check cache
        cached = (User) cacheService.get(cacheKey);
        if (cached != null) return cached;
        
        // Query DB
        User user = userRepository.findById(id).orElse(null);
        if (user != null) {
            cacheService.set(cacheKey, user, Duration.ofMinutes(30));
        }
        
        return user;
    }
}
```

### b. Probabilistic Early Expiration (Beta)
```java
// Refresh cache trước khi expire (random)
public User getUser(Long id) {
    String key = "user:" + id;
    User user = (User) cacheService.get(key);
    Long ttl = cacheService.getTtl(key);
    
    if (user != null && ttl != null && ttl < 60) {
        // TTL < 60s → có xác suất refresh
        double beta = 1.0;
        double probability = beta * Math.log(Math.random());
        
        if (probability < 0.1) {
            // Refresh background
            refreshCache(id);
        }
    }
    
    return user;
}
```

---

## 6. Cache Warming

**Mục đích:** Preload cache để giảm cache miss lúc đầu.

**When to warm:**
- Application startup
- After cache clear
- Deploy new version
- Scheduled (nightly)

**Implementation:**

```java
@Component
public class CacheWarmer {
    
    @Autowired
    private CachedUserService userService;
    
    /**
     * Warm up cache khi application start
     */
    @EventListener(ApplicationReadyEvent.class)
    public void warmUpOnStartup() {
        logger.info("🔥 Starting cache warm-up...");
        
        try {
            userService.warmUpCache();
            logger.info("✅ Cache warm-up completed");
        } catch (Exception e) {
            logger.error("❌ Cache warm-up failed", e);
        }
    }
}
```

---

## 7. Monitoring và Metrics

**Key Metrics:**

```java
// Cache hit rate
double hitRate = (double) hits / (hits + misses) * 100;

// Ideal: > 80%
if (hitRate < 80) {
    logger.warn("⚠️ Low cache hit rate: {}%", hitRate);
}
```

**Redis Metrics:**

```bash
# Memory usage
redis-cli INFO memory

# Hit rate
redis-cli INFO stats | grep keyspace

# Connected clients
redis-cli INFO clients

# Operations per second
redis-cli INFO stats | grep instantaneous_ops_per_sec
```

---

# Phần 6: Testing và Demo

## 1. Test Scenarios

### Scenario 1: Cache Hit vs Cache Miss

**Test cache hit/miss behavior:**

```bash
# Lần 1: Cache miss (query DB)
curl -X GET http://localhost:8080/api/cached-users/1

# Log: 📀 Fetching user 1 from database (cache miss)
# Response time: ~50ms

# Lần 2: Cache hit (from Redis)
curl -X GET http://localhost:8080/api/cached-users/1

# Log: ✅ Cache hit for key: users::1
# Response time: ~5ms (nhanh hơn 10x!)
```

### Scenario 2: Cache Invalidation

```bash
# 1. Get user (cache)
curl -X GET http://localhost:8080/api/cached-users/1

# 2. Update user → cache invalidated
curl -X PUT http://localhost:8080/api/cached-users/1 \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Name",
    "email": "updated@email.com"
  }'

# Log: ✅ Updated and cached user: 1

# 3. Get user again → cache hit với data mới
curl -X GET http://localhost:8080/api/cached-users/1
```

### Scenario 3: Cache Penetration Protection

```bash
# Request user không tồn tại
curl -X GET http://localhost:8080/api/cached-users/999999

# Log: ⚠️ User 999999 not found, caching null value
# Response: 404 Not Found

# Request lần 2 → Hit null cache, không query DB
curl -X GET http://localhost:8080/api/cached-users/999999

# Log: ✅ Cache hit for key: user:999999
```

### Scenario 4: Leaderboard Operations

```bash
# 1. Thêm players
curl -X POST http://localhost:8080/api/leaderboard/score \
  -H "Content-Type: application/json" \
  -d '{"playerId": "player1", "score": 100}'

curl -X POST http://localhost:8080/api/leaderboard/score \
  -H "Content-Type: application/json" \
  -d '{"playerId": "player2", "score": 200}'

curl -X POST http://localhost:8080/api/leaderboard/score \
  -H "Content-Type: application/json" \
  -d '{"playerId": "player3", "score": 150}'

# 2. Tăng score
curl -X POST http://localhost:8080/api/leaderboard/increment \
  -H "Content-Type: application/json" \
  -d '{"playerId": "player1", "delta": 50}'

# 3. Lấy top 10
curl -X GET http://localhost:8080/api/leaderboard/top/10

# Response:
# [
#   {"rank": 1, "playerId": "player2", "score": 200},
#   {"rank": 2, "playerId": "player1", "score": 150},
#   {"rank": 3, "playerId": "player3", "score": 150}
# ]

# 4. Lấy thông tin player
curl -X GET http://localhost:8080/api/leaderboard/player/player1

# Response:
# {
#   "playerId": "player1",
#   "score": 150,
#   "rank": 2
# }
```

---

## 2. Performance Comparison

### Benchmark: Database vs Cache

**Setup:**
```bash
# Warm up
curl -X POST http://localhost:8080/api/cached-users/cache/warmup

# Clear cache
curl -X POST http://localhost:8080/api/cached-users/cache/clear
```

**Test Database (No Cache):**
```bash
# 100 requests
ab -n 100 -c 10 http://localhost:8080/api/users/1

# Results:
# Time per request: ~50ms
# Requests per second: ~20
```

**Test Cache:**
```bash
# 100 requests
ab -n 100 -c 10 http://localhost:8080/api/cached-users/1

# Results:
# Time per request: ~5ms (10x faster!)
# Requests per second: ~200 (10x more!)
```

---

## 3. Monitor với Redis CLI

### Xem tất cả keys

```bash
docker exec -it redis redis-cli

# Xem tất cả keys
127.0.0.1:6379> KEYS *
1) "app:cache:users::1"
2) "app:cache:users::all"
3) "leaderboard:global"

# Xem value
127.0.0.1:6379> GET "app:cache:users::1"
"{\"@class\":\"com.example.demo.model.User\",\"id\":1,\"name\":\"John\"...}"

# Xem TTL
127.0.0.1:6379> TTL "app:cache:users::1"
(integer) 1798  # 1798 seconds remaining

# Xem leaderboard
127.0.0.1:6379> ZREVRANGE leaderboard:global 0 9 WITHSCORES
1) "player2"
2) "200"
3) "player1"
4) "150"
```

### Monitor real-time

```bash
# Monitor all commands
redis-cli MONITOR

# Output:
# "GET" "app:cache:users::1"
# "SET" "app:cache:users::2" ...
# "ZADD" "leaderboard:global" ...
```

### Check memory usage

```bash
redis-cli INFO memory

# used_memory: 1.2M
# used_memory_human: 1.20M
# maxmemory: 256M
```

---

## 4. Logs Minh Họa

### Application Logs

```log
2026-02-19 10:00:00 INFO  CachedUserService - 📀 Fetching user 1 from database (cache miss)
2026-02-19 10:00:00 INFO  RedisCacheService - Cached key: user:1 with TTL: PT30M
2026-02-19 10:00:05 DEBUG RedisCacheService - Cache hit for key: user:1
2026-02-19 10:01:00 INFO  CachedUserService - ✅ Updated and cached user: 1
2026-02-19 10:02:00 INFO  LeaderboardService - ✅ Added/Updated player player1 with score 100.0
2026-02-19 10:02:10 INFO  LeaderboardService - ✅ Incremented player player1 by 50.0. New score: 150.0
2026-02-19 10:03:00 INFO  LeaderboardService - 📊 Retrieved top 10 players
2026-02-19 10:05:00 DEBUG CachedUserService - ⚠️ User 999999 not found, caching null value
2026-02-19 10:06:00 INFO  CachedUserService - 🗑️ Deleted user and cleared cache: 5
2026-02-19 10:07:00 INFO  CachedUserService - 🧹 Cleared all user caches
2026-02-19 10:08:00 INFO  CachedUserService - 🔥 Warming up user cache...
2026-02-19 10:08:01 INFO  CachedUserService - ✅ Cache warmed up with 100 users
```

---

# Phần 7: Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Client Request                          │
│                  GET /api/cached-users/1                     │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  CachedUserController                        │
│                  @RestController                             │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│               CachedUserService                              │
│            @Cacheable(value="users", key="#id")             │
└───────────────────────────┬─────────────────────────────────┘
                            │
                ┌───────────┴──────────┐
                │                      │
                ▼                      ▼
    ┌──────────────────┐   ┌──────────────────────┐
    │  Redis Cache     │   │  Database (MySQL)    │
    │  (Check first)   │   │  (If cache miss)     │
    └───────┬──────────┘   └──────────┬───────────┘
            │                         │
            │  Cache HIT              │  Cache MISS
            │  (Fast: ~5ms)           │  (Slow: ~50ms)
            │                         │
            │                         ▼
            │              ┌──────────────────────┐
            │              │ Query Database       │
            │              │ Save to Cache        │
            │              │ Return result        │
            │              └──────────┬───────────┘
            │                         │
            └─────────────────────────┘
                            │
                            ▼
                    ┌──────────────┐
                    │   Response   │
                    └──────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Redis Data Structures                     │
├─────────────────────────────────────────────────────────────┤
│  String:      app:cache:users::1 → {"id":1,"name":"John"}  │
│  Hash:        user:1:profile → {name:"John", age:30}        │
│  List:        notifications → [notif1, notif2, notif3]      │
│  Set:         tags:post:1 → {java, redis, cache}            │
│  Sorted Set:  leaderboard → {player1:150, player2:200}      │
└─────────────────────────────────────────────────────────────┘
```

---

# Phần 8: Best Practices Summary

## ✅ DO's

1. **Đặt TTL phù hợp** - Tránh stale data
2. **Random TTL** - Tránh cache avalanche
3. **Cache null values** - Tránh cache penetration với TTL ngắn
4. **Monitor cache hit rate** - Target > 80%
5. **Invalidate on write** - Sử dụng `@CacheEvict`, `@CachePut`
6. **Warm up cache** - Preload critical data
7. **Handle cache failures** - Graceful degradation (fallback to DB)
8. **Log cache operations** - Debug và monitoring
9. **Use appropriate data structures** - String, Hash, Sorted Set...
10. **Test cache behavior** - Cache hit/miss, invalidation

## ❌ DON'Ts

1. **Không cache mọi thứ** - Chỉ cache dữ liệu đọc nhiều
2. **Không set TTL quá dài** - Stale data risk
3. **Không cache sensitive data** - Security risk
4. **Không cache quá nhiều** - Memory cost
5. **Không quên invalidate** - Consistency issues
6. **Không depend hoàn toàn vào cache** - Cần fallback
7. **Không ignore cache failures** - Handle exceptions
8. **Không cache large objects** - Redis max 512MB/key
9. **Không dùng cache làm primary storage** - Cache != Database
10. **Không quên monitor metrics** - Cache hit rate, memory usage

---

# Phần 9: Troubleshooting

## Issue 1: Low Cache Hit Rate

**Triệu chứng:** Cache hit rate < 50%

**Nguyên nhân:**
- TTL quá ngắn
- Data access pattern không phù hợp cache
- Cache bị evicted do memory full

**Giải pháp:**
```bash
# Check hit rate
redis-cli INFO stats | grep keyspace_hits

# Tăng TTL
Duration.ofMinutes(30) → Duration.ofHours(1)

# Tăng memory limit
docker run -d redis --maxmemory 512mb
```

---

## Issue 2: Redis Memory Full

**Triệu chứng:** `OOM command not allowed when used memory > 'maxmemory'`

**Giải pháp:**

```bash
# Check memory
redis-cli INFO memory

# Set eviction policy
redis-cli CONFIG SET maxmemory-policy allkeys-lru

# Policies:
# - allkeys-lru: Evict least recently used
# - volatile-lru: Evict LRU với TTL
# - allkeys-lfu: Evict least frequently used
```

---

## Issue 3: Cache Stampede

**Triệu chứng:** Database spikes khi cache expire

**Log:**
```log
09:00:00.001 - Query user 1 from DB
09:00:00.002 - Query user 1 from DB
09:00:00.003 - Query user 1 from DB
... (100 duplicate queries)
```

**Giải pháp:** Implement locking (xem Phần 5)

---

## Issue 4: Stale Data

**Triệu chứng:** User thấy dữ liệu cũ sau khi update

**Giải pháp:**

```java
// ✅ Sử dụng @CachePut để update cache ngay
@CachePut(value = "users", key = "#id")
public User updateUser(Long id, User user) {
    return userRepository.save(user);
}

// Hoặc manual evict
@CacheEvict(value = "users", key = "#id")
```

---

# Phần 10: Key Takeaways

✅ **Caching giảm latency 10-100x** - Từ 50ms → 5ms  
✅ **Redis hỗ trợ nhiều data structures** - String, Hash, List, Set, Sorted Set  
✅ **Cache-Aside là strategy phổ biến nhất** - Dễ implement, reliable  
✅ **TTL quan trọng** - Cân bằng freshness và performance  
✅ **Cache invalidation khó** - Cần strategy rõ ràng  
✅ **Tránh 4 vấn đề chính** - Penetration, Avalanche, Stampede, Stale Data  
✅ **Monitor metrics** - Hit rate, memory, eviction rate  
✅ **Sorted Set mạnh cho Leaderboard** - O(log N) operations  
✅ **Spring Cache abstraction** - Dễ integrate với annotations  
✅ **Fallback to DB khi cache fail** - Graceful degradation  

---

# Phần 11: References

- [Redis Official Documentation](https://redis.io/docs/)
- [Spring Cache Documentation](https://docs.spring.io/spring-framework/reference/integration/cache.html)
- [Spring Data Redis](https://docs.spring.io/spring-data/redis/docs/current/reference/html/)
- [Redis Data Types](https://redis.io/docs/data-types/)
- [Caching Best Practices](https://aws.amazon.com/caching/best-practices/)
- [Cache Patterns](https://docs.aws.amazon.com/whitepapers/latest/database-caching-strategies-using-redis/caching-patterns.html)
