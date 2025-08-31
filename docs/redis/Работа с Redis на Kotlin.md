# Работа с Redis на Kotlin. Redis - это мощная in-memory база данных типа "ключ-значение".

## Основные способы работы с Redis из Kotlin

### 1. Использование Jedis (синхронный клиент)

**Добавление зависимости (Gradle):**
```kotlin
dependencies {
    implementation("redis.clients:jedis:4.4.3")
}
```

**Пример использования:**
```kotlin
import redis.clients.jedis.Jedis
import redis.clients.jedis.JedisPool
import redis.clients.jedis.JedisPoolConfig

class RedisJedisExample {
    private val jedisPool: JedisPool
    
    init {
        val poolConfig = JedisPoolConfig()
        poolConfig.maxTotal = 10
        jedisPool = JedisPool(poolConfig, "localhost", 6379)
    }
    
    // Базовые операции
    fun basicOperations() {
        jedisPool.resource.use { jedis ->
            // SET/GET
            jedis.set("user:1:name", "Иван Иванов")
            val userName = jedis.get("user:1:name")
            println("Имя пользователя: $userName")
            
            // HSET/HGET (хеши)
            jedis.hset("user:1", "email", "ivan@mail.ru")
            jedis.hset("user:1", "age", "30")
            val userEmail = jedis.hget("user:1", "email")
            println("Email: $userEmail")
            
            // Списки
            jedis.lpush("recent_users", "user:1")
            jedis.lpush("recent_users", "user:2")
            val recentUsers = jedis.lrange("recent_users", 0, 10)
            println("Последние пользователи: $recentUsers")
        }
    }
    
    // Работа с TTL (время жизни ключей)
    fun ttlOperations() {
        jedisPool.resource.use { jedis ->
            // Ключ с истечением через 60 секунд
            jedis.setex("temp:session", 60, "session_data")
            
            // Установка TTL отдельно
            jedis.set("cache:data", "cached_value")
            jedis.expire("cache:data", 300) // 5 минут
        }
    }
}
```

### 2. Использование Lettuce (асинхронный клиент)

**Добавление зависимости:**
```kotlin
dependencies {
    implementation("io.lettuce:lettuce-core:6.2.4.RELEASE")
}
```

**Асинхронный пример:**
```kotlin
import io.lettuce.core.RedisClient
import io.lettuce.core.api.StatefulRedisConnection
import io.lettuce.core.api.coroutines
import kotlinx.coroutines.runBlocking

class RedisLettuceExample {
    private val redisClient: RedisClient = RedisClient.create("redis://localhost:6379")
    private val connection: StatefulRedisConnection<String, String> = redisClient.connect()
    private val commands = connection.sync()
    private val asyncCommands = connection.async()
    private val reactiveCommands = connection.reactive()
    
    // Синхронные операции
    fun syncOperations() {
        commands.set("counter", "100")
        val value = commands.get("counter")
        println("Счетчик: $value")
    }
    
    // Асинхронные операции с coroutines
    suspend fun asyncOperations() {
        val result = asyncCommands.set("async:key", "async_value").await()
        val value = asyncCommands.get("async:key").await()
        println("Асинхронный результат: $value")
    }
    
    fun close() {
        connection.close()
        redisClient.shutdown()
    }
}

// Использование
fun main() = runBlocking {
    val redis = RedisLettuceExample()
    redis.syncOperations()
    redis.asyncOperations()
    redis.close()
}
```

### 3. Spring Data Redis (для Spring Boot приложений)

**Зависимости:**
```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("io.lettuce:lettuce-core")
}
```

**Конфигурация:**
```kotlin
@Configuration
class RedisConfig {
    
    @Bean
    fun redisConnectionFactory(): RedisConnectionFactory {
        return LettuceConnectionFactory("localhost", 6379)
    }
    
    @Bean
    fun redisTemplate(): RedisTemplate<String, Any> {
        val template = RedisTemplate<String, Any>()
        template.connectionFactory = redisConnectionFactory()
        template.keySerializer = StringRedisSerializer()
        template.valueSerializer = Jackson2JsonRedisSerializer(Any::class.java)
        return template
    }
}
```

**Сервис для работы с Redis:**
```kotlin
@Service
class UserCacheService(
    private val redisTemplate: RedisTemplate<String, Any>
) {
    companion object {
        private const val USER_CACHE_PREFIX = "user:"
        private const val USER_LIST_KEY = "users:all"
        private const val CACHE_TTL_HOURS = 24L
    }
    
    // Сохранение пользователя
    fun saveUser(user: User) {
        val userKey = "$USER_CACHE_PREFIX${user.id}"
        redisTemplate.opsForValue().set(userKey, user, CACHE_TTL_HOURS, TimeUnit.HOURS)
        redisTemplate.opsForSet().add(USER_LIST_KEY, user.id.toString())
    }
    
    // Получение пользователя
    fun getUser(userId: Long): User? {
        val userKey = "$USER_CACHE_PREFIX$userId"
        return redisTemplate.opsForValue().get(userKey) as? User
    }
    
    // Получение всех ID пользователей
    fun getAllUserIds(): Set<String> {
        return redisTemplate.opsForSet().members(USER_LIST_KEY)
            ?.map { it.toString() }
            ?.toSet() ?: emptySet()
    }
    
    // Удаление пользователя
    fun deleteUser(userId: Long) {
        val userKey = "$USER_CACHE_PREFIX$userId"
        redisTemplate.delete(userKey)
        redisTemplate.opsForSet().remove(USER_LIST_KEY, userId.toString())
    }
}

// Модель данных
data class User(
    val id: Long,
    val name: String,
    val email: String,
    val createdAt: Instant = Instant.now()
)
```

### 4. Работа с Redis Streams (для обработки событий)

```kotlin
class RedisStreamsExample(private val jedisPool: JedisPool) {
    
    // Отправка события в stream
    fun sendEvent(streamKey: String, event: Map<String, String>) {
        jedisPool.resource.use { jedis ->
            jedis.xadd(streamKey, null, event)
        }
    }
    
    // Чтение событий из stream
    fun readEvents(streamKey: String, consumerGroup: String, consumerName: String) {
        jedisPool.resource.use { jedis ->
            // Создание consumer group если не существует
            try {
                jedis.xgroupCreate(streamKey, consumerGroup, null, false)
            } catch (e: Exception) {
                // Group уже существует - это нормально
            }
            
            // Чтение событий
            val events = jedis.xreadGroup(
                consumerGroup,
                consumerName,
                1,
                0,
                false,
                mapOf(streamKey to ">")
            )
            
            events?.forEach { (stream, messages) ->
                messages.forEach { (id, fields) ->
                    println("Получено событие $id: $fields")
                    // Подтверждение обработки
                    jedis.xack(streamKey, consumerGroup, id)
                }
            }
        }
    }
}
```

### 5. Практический пример: Кэширование данных

```kotlin
class CacheManager(private val jedisPool: JedisPool) {
    
    suspend fun <T> withCache(
        key: String,
        ttlSeconds: Long = 3600,
        supplier: suspend () -> T
    ): T {
        return jedisPool.resource.use { jedis ->
            // Пытаемся получить из кэша
            val cached = jedis.get(key)
            if (cached != null) {
                Json.decodeFromString(cached)
            } else {
                // Если нет в кэше - получаем из источника
                val result = supplier()
                val jsonResult = Json.encodeToString(result)
                jedis.setex(key, ttlSeconds, jsonResult)
                result
            }
        }
    }
    
    fun invalidateCache(key: String) {
        jedisPool.resource.use { jedis ->
            jedis.del(key)
        }
    }
    
    fun getCacheStats(): Map<String, String> {
        return jedisPool.resource.use { jedis ->
            val info = jedis.info()
            info.split("\r\n")
                .filter { it.contains(":") }
                .associate {
                    val parts = it.split(":", limit = 2)
                    parts[0] to parts[1]
                }
        }
    }
}
```

## Рекомендации по использованию:

1. **Выбор клиента**:
    - Jedis - для простых синхронных операций
    - Lettuce - для асинхронных/реактивных приложений
    - Spring Data Redis - для Spring Boot приложений

2. **Пуллинг соединений**: Всегда используйте пул соединений
3. **Сериализация**: Используйте эффективные сериализаторы (JSON, Protobuf)
4. **Обработка ошибок**: Добавляйте retry logic для сетевых ошибок
5. **Мониторинг**: Следите за использованием памяти и производительностью

Какой именно use case вас интересует? Я могу показать более конкретные примеры!

---

