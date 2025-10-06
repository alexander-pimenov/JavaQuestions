Разберемся в разнице между типами тестов с примерами на Kotlin! 😊

## Юнит-тесты (Unit Tests)
**Что тестируют:** Отдельные изолированные модули кода (функции, методы, классы)
**Скорость:** Быстрые
**Зависимости:** Mock/Stub для внешних зависимостей

```kotlin
// Простой класс для тестирования
class Calculator {
    fun add(a: Int, b: Int): Int = a + b
    fun divide(a: Int, b: Int): Int {
        if (b == 0) throw IllegalArgumentException("Cannot divide by zero")
        return a / b
    }
}

// Юнит-тесты
class CalculatorUnitTest {
    
    @Test
    fun `test addition`() {
        // Arrange
        val calculator = Calculator()
        
        // Act
        val result = calculator.add(2, 3)
        
        // Assert
        assertEquals(5, result)
    }
    
    @Test
    fun `test division by zero throws exception`() {
        // Arrange
        val calculator = Calculator()
        
        // Act & Assert
        assertThrows<IllegalArgumentException> {
            calculator.divide(10, 0)
        }
    }
}
```

## Модульные тесты (Module Tests)
**Что тестируют:** Группу связанных компонентов вместе
**Скорость:** Средние
**Зависимости:** Частично изолированы

```kotlin
// Сервис с зависимостью
class UserService(private val userRepository: UserRepository) {
    fun createUser(name: String, email: String): User {
        if (userRepository.existsByEmail(email)) {
            throw IllegalArgumentException("User with this email already exists")
        }
        return userRepository.save(User(name = name, email = email))
    }
}

interface UserRepository {
    fun save(user: User): User
    fun existsByEmail(email: String): Boolean
}

// Модульный тест
class UserServiceModuleTest {
    
    @Test
    fun `test user creation with repository`() {
        // Arrange - используем реальный репозиторий, но в памяти
        val userRepository = InMemoryUserRepository()
        val userService = UserService(userRepository)
        
        // Act
        val user = userService.createUser("John", "john@test.com")
        
        // Assert
        assertEquals("John", user.name)
        assertEquals("john@test.com", user.email)
        assertTrue(userRepository.existsByEmail("john@test.com"))
    }
}

// In-memory реализация для тестов
class InMemoryUserRepository : UserRepository {
    private val users = mutableListOf<User>()
    
    override fun save(user: User): User {
        users.add(user)
        return user
    }
    
    override fun existsByEmail(email: String): Boolean {
        return users.any { it.email == email }
    }
}
```

## Интеграционные тесты (Integration Tests)
**Что тестируют:** Взаимодействие между несколькими системами/компонентами
**Скорость:** Медленные
**Зависимости:** Реальные базы данных, внешние сервисы

```kotlin
// Интеграционный тест с реальной БД
@SpringBootTest
@TestPropertySource(locations = ["classpath:application-test.properties"])
class UserIntegrationTest {
    
    @Autowired
    private lateinit var userService: UserService
    
    @Autowired
    private lateinit var userRepository: UserRepository
    
    @Autowired
    private lateinit var emailService: EmailService
    
    @Test
    fun `test complete user registration flow`() {
        // Arrange
        val userDto = CreateUserDto("Alice", "alice@test.com")
        
        // Act - тестируем полный поток
        val user = userService.registerUser(userDto)
        
        // Assert - проверяем взаимодействие всех компонентов
        assertNotNull(user.id)
        assertTrue(userRepository.existsById(user.id!!))
        // Проверяем, что email был отправлен
        verify(emailService).sendWelcomeEmail(user.email)
    }
    
    @Test
    fun `test user registration with duplicate email`() {
        // Arrange
        val userDto = CreateUserDto("Bob", "bob@test.com")
        userService.registerUser(userDto)
        
        // Act & Assert
        assertThrows<DuplicateEmailException> {
            userService.registerUser(userDto.copy(name = "Another Bob"))
        }
    }
}
```

## Сравнение в таблице

| Критерий | Юнит-тесты | Модульные тесты | Интеграционные тесты |
|----------|-------------|-----------------|---------------------|
| **Объем** | Один класс/метод | Группа классов | Система целиком |
| **Скорость** | Очень быстрые | Быстрые | Медленные |
| **Зависимости** | Mock/Stub | Fake/In-memory | Реальные |
| **Цель** | Проверить логику | Проверить взаимодействие | Проверить работу системы |

## Практические рекомендации:

```kotlin
// Тестовая пирамида - идеальное соотношение:
class TestPyramid {
    // 70% - Юнит-тесты (быстрые, много)
    // 20% - Модульные тесты (средние, меньше)
    // 10% - Интеграционные тесты (медленные, мало)
}

// Пример аннотаций в Kotlin/Spring
@ExtendWith(MockKExtension::class)
class ServiceUnitTest {
    
    @MockK
    private lateinit var repository: UserRepository
    
    @InjectMockKs
    private lateinit var service: UserService
    
    @Test
    fun `unit test with mocks`() {
        // Given
        every { repository.existsByEmail(any()) } returns false
        every { repository.save(any()) } returns User("test", "test@test.com")
        
        // When
        val result = service.createUser("test", "test@test.com")
        
        // Then
        assertEquals("test", result.name)
    }
}
```

Итог:
- `Юнит-тесты` - для изолированной логики, `интеграционные` - для проверки работы системы в целом, а `модульные` - что-то посередине. 🚀