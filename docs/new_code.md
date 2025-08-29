Вот вариант переделки кода для перехода с `main`/`duplicate` на `mainBalancingGroup`, учитывая код:

### 1. Модификация `AuditClientProperties`

```kotlin
data class AuditClientProperties(
    val mode: AuditClientModeEnum,
    val nodeId: String,
    val project: String,
    val mainBalancingGroup: BalancingGroup = BalancingGroup.NONE,
    // Удаляем duplicate, так как теперь это часть mainBalancingGroup
) {
    @ConstructorBinding
    constructor(
        nodeId: String?, 
        mode: AuditClientModeEnum?, 
        project: String?,
        mainBalancingGroup: BalancingGroup?
    ) : this(
        mode ?: AuditClientModeEnum.SYNC,
        requireNotNull(nodeId) { "audit.client.nodeId обязателен для заполнения" },
        requireNotNull(project) { "audit.client.project обязателен для заполнения" },
        mainBalancingGroup ?: BalancingGroup.NONE
    ) {
        require(nodeId.isNotEmpty()) { "audit.client.nodeId не должен быть пустым" }
        require(project.isNotEmpty()) { "audit.client.project не должен быть пустым" }
    }
}
```

### 2. Обновление `AuditProperties`

```kotlin
@ConfigurationProperties(prefix = "audit")
data class AuditProperties(
    val enabled: Boolean, 
    val client: AuditClientProperties, 
    // Удаляем duplicate, так как теперь это часть mainBalancingGroup
) {
    @ConstructorBinding
    constructor(
        client: AuditClientProperties?, 
        enabled: Boolean?,
        // Удаляем duplicate параметр
    ) : this(
        enabled = enabled != false, 
        client = if (enabled != false) {
            requireNotNull(client) { "блок audit.client обязателен для заполнения" }
        } else {
            AuditClientProperties(
                mode = AuditClientModeEnum.ASYNC, 
                nodeId = "disabled", 
                project = "disabled"
            )
        }
    )
}
```

### 3. Удаление `main` из `RestPropertiesBuilder` и `KafkaPropertiesBuilder`

```kotlin
@ConfigurationProperties(prefix = "audit.client.rest")
data class RestPropertiesBuilder(
    override val enabled: Boolean?,
    val url: String?,
    // Удаляем main: Boolean = true,
    val circuitBreaker: CircuitBreakerConfiguration?
) : EnablerProperties {
    fun build(): RestProperties? {
        requireNotNull(url) { "audit.client.rest.url обязателен для заполнения" }
        require(url.isNotEmpty()) { "audit.client.rest.url обязателен для заполнения" }
        requireNotNull(circuitBreaker) { "audit.client.rest.circuitBreaker обязателен для заполнения" }
        return if (enabled == null || enabled) RestProperties(
            url,
            circuitBreaker // Удаляем main
        ) else null
    }
}

data class RestProperties(
    val url: String, 
    // Удаляем main: Boolean,
    val circuitBreaker: CircuitBreakerConfiguration
)
```

Аналогично для `KafkaPropertiesBuilder`:

```kotlin
@ConfigurationProperties(prefix = "audit.client.kafka")
data class KafkaPropertiesBuilder(
    override val enabled: Boolean?,
    val servers: String?,
    val topics: TopicsPairBuilder?,
    // Удаляем main: Boolean = false,
    val circuitBreaker: CircuitBreakerConfiguration?
) : EnablerProperties {
    fun build(): KafkaProperties? {
        requireNotNull(servers) { "сервер kafka audit.client.kafka.servers должен быть обязательно указан" }
        requireNotNull(topics) { "топики kafka audit.client.kafka.topics должны быть обязательно указаны" }
        requireNotNull(circuitBreaker) { "audit.client.kafka.circuitBreaker обязателен для заполнения" }
        return if (enabled == null || enabled) KafkaProperties(
            servers,
            topics.build(),
            circuitBreaker // Удаляем main
        ) else null
    }
}

data class KafkaProperties(
    val servers: String, 
    val topicsPair: TopicsPair, 
    // Удаляем main: Boolean,
    var circuitBreaker: CircuitBreakerConfiguration
)
```

### 4. Обновление `BalancingGroupConfiguration`

Модифицируем функцию `createBalancingGroupConfiguration`:

```kotlin
fun createBalancingGroupConfiguration(
    breakerConfig: CircuitBreakerConfiguration,
    groupName: String,
    providerClass: Class<*>,
    isMain: Boolean, // Теперь это определяется на основе mainBalancingGroup
    priority: Int = 10
): BalancingGroupConfiguration {
    val balancingGroupConfig = BalancingGroupConfiguration()
    balancingGroupConfig.name = groupName
    balancingGroupConfig.isMain = isMain
    balancingGroupConfig.providerClass = providerClass.canonicalName
    balancingGroupConfig.breakerConfiguration = breakerConfig
    balancingGroupConfig.providerNameToWeight[providerClass.simpleName] = 1
    balancingGroupConfig.priority = priority
    
    return balancingGroupConfig
}
```

### 5. Обновление конфигурационных классов

Модифицируем `RestBalancingGroupConfiguration` и `KafkaBalancingGroupConfiguration`:

```kotlin
@Configuration
@Conditional(RestCondition::class)
class RestBalancingGroupConfiguration(
    private val auditProperties: AuditProperties,
    private val restPropertiesBuilder: RestPropertiesBuilder
) {
    @Bean
    fun httpSenderBalancingGroupConfiguration(restProperties: RestProperties) =
        createBalancingGroupConfiguration(
            restProperties.circuitBreaker,
            "HttpSender group",
            HttpSender::class.java,
            auditProperties.client.mainBalancingGroup == BalancingGroup.REST,
            if (auditProperties.client.mainBalancingGroup == BalancingGroup.REST) 
                Integer.MAX_VALUE else 10
        )
    // ... остальные бины без изменений
}

@Configuration
@Conditional(KafkaCondition::class)
class KafkaBalancingGroupConfiguration(
    private val auditProperties: AuditProperties,
    private val kafkaPropertiesBuilder: KafkaPropertiesBuilder
) {
    @Bean
    fun kafkaSenderBalancingGroupConfiguration(kafkaProperties: KafkaProperties) =
        createBalancingGroupConfiguration(
            kafkaProperties.circuitBreaker,
            "KafkaSender group",
            KafkaSender::class.java,
            auditProperties.client.mainBalancingGroup == BalancingGroup.KAFKA,
            if (auditProperties.client.mainBalancingGroup == BalancingGroup.KAFKA) 
                Integer.MAX_VALUE else 10
        )
    // ... остальные бины без изменений
}
```

### 6. Обновление `AuditSenderConfiguration`

```kotlin
@Configuration
class AuditSenderConfiguration(
    private val auditProperties: AuditProperties
) {
    @Bean
    fun pvmSdkConfiguration(
        kafkaPullProcessor: KafkaPullProcessor?,
        balancingGroups: List<BalancingGroupConfiguration>,
        pvmSdkMonitoringService: PvmSdkMonitoringService,
        pvmSignatureService: PvmSignatureService
    ): PvmSdkConfiguration =
        PvmSdkConfigurationImpl(
            balancingGroups,
            if (auditProperties.client.mainBalancingGroup == BalancingGroup.DUPLICATE) 
                MainBalancerMode.DUP else MainBalancerMode.STD,
            listOf<PvmSdkEnricher>(),
            listOfNotNull(kafkaPullProcessor),
            pvmSdkMonitoringService,
            pvmSignatureService
        )
    // ... остальные бины без изменений
}
```

### 7. Пример новой конфигурации `application.yaml`

```yaml
audit:
  enabled: true
  client:
    mode: sync
    nodeId: os_id
    project: CI123456
    mainBalancingGroup: REST # Или KAFKA/DUPLICATE
    rest:
      enabled: true
      url: http://localhost:8090
      circuitBreaker:
        recoveryTimeout: 1000
    kafka:
      enabled: true
      servers: localhost:8091
      topics:
        event: event-topic
        metamodel: metamodel-topic
      circuitBreaker:
        recoveryTimeout: 1000
  fallback:
    enabled: true
    servers: localhost:8092
    topic: fallback-topic
    initialDelay: 100
    delayPeriod: 100
    groupId: fallback
    clientId: fallback
    circuitBreaker:
      recoveryTimeout: 1000
```

### Итоговые изменения:

1. Удалены поля `main` из REST и Kafka конфигураций
2. Удалено поле `duplicate` из корневой конфигурации
3. Добавлено поле `mainBalancingGroup` в `AuditClientProperties`
4. Обновлена логика определения `isMain` и приоритетов в балансировочных группах
5. Обновлен механизм определения `MainBalancerMode` (DUP/STD) на основе `mainBalancingGroup`
6. Сохранена вся существующая функциональность, но с более понятной конфигурацией

Такой подход делает конфигурацию:
- Более понятной и явной
- Менее подверженной ошибкам
- Легче расширяемой для новых транспортов
- Сохраняющей всю существующую функциональность


Вот как можно реализовать валидацию конфигурации при старте приложения, используя Spring Boot's `ApplicationListener` и кастомные валидаторы:

### 1. Создаем класс для валидации конфигурации

```kotlin
import org.springframework.boot.context.event.ApplicationReadyEvent
import org.springframework.context.ApplicationListener
import org.springframework.core.env.Environment
import org.springframework.stereotype.Component

@Component
@ConditionalOnProperty("audit.enabled", havingValue = "true", matchIfMissing = true)
class AuditConfigValidator(
    private val environment: Environment,
    private val auditProperties: AuditProperties
) : ApplicationListener<ApplicationReadyEvent> {

    override fun onApplicationEvent(event: ApplicationReadyEvent) {
        validateConfiguration()
    }

    private fun validateConfiguration() {
        when (auditProperties.client.mainBalancingGroup) {
            BalancingGroup.REST -> validateRestConfig()
            BalancingGroup.KAFKA -> validateKafkaConfig()
            BalancingGroup.DUPLICATE -> validateDuplicateConfig()
            BalancingGroup.NONE -> {} // Ничего не проверяем
        }
    }

    private fun validateRestConfig() {
        require(auditProperties.client.rest.enabled) {
            "Для mainBalancingGroup=REST необходимо включить REST конфигурацию (audit.client.rest.enabled=true)"
        }
        validateRestProperties()
    }

    private fun validateKafkaConfig() {
        require(auditProperties.client.kafka.enabled) {
            "Для mainBalancingGroup=KAFKA необходимо включить Kafka конфигурацию (audit.client.kafka.enabled=true)"
        }
        validateKafkaProperties()
    }

    private fun validateDuplicateConfig() {
        require(auditProperties.client.rest.enabled && auditProperties.client.kafka.enabled) {
            "Для mainBalancingGroup=DUPLICATE необходимо включить и REST и Kafka конфигурации " +
            "(audit.client.rest.enabled=true и audit.client.kafka.enabled=true)"
        }
        validateRestProperties()
        validateKafkaProperties()
    }

    private fun validateRestProperties() {
        val rest = auditProperties.client.rest
        require(rest.url.isNotBlank()) {
            "Не указан URL для REST (audit.client.rest.url)"
        }
        // Дополнительные проверки для REST
    }

    private fun validateKafkaProperties() {
        val kafka = auditProperties.client.kafka
        require(kafka.servers.isNotBlank()) {
            "Не указаны серверы Kafka (audit.client.kafka.servers)"
        }
        require(kafka.topicsPair.event.isNotBlank()) {
            "Не указана тема для событий (audit.client.kafka.topics.event)"
        }
        require(kafka.topicsPair.metamodel.isNotBlank()) {
            "Не указана тема для метамодели (audit.client.kafka.topics.metamodel)"
        }
        // Дополнительные проверки для Kafka
    }
}
```

### 2. Модифицируем `AuditClientProperties` для более строгой валидации

```kotlin
data class AuditClientProperties(
    val mode: AuditClientModeEnum,
    val nodeId: String,
    val project: String,
    val mainBalancingGroup: BalancingGroup = BalancingGroup.NONE,
    val rest: RestProperties,
    val kafka: KafkaProperties
) {
    @ConstructorBinding
    constructor(
        nodeId: String?,
        mode: AuditClientModeEnum?,
        project: String?,
        mainBalancingGroup: BalancingGroup?,
        rest: RestProperties?,
        kafka: KafkaProperties?
    ) : this(
        mode ?: AuditClientModeEnum.SYNC,
        requireNotNull(nodeId) { "audit.client.nodeId обязателен для заполнения" },
        requireNotNull(project) { "audit.client.project обязателен для заполнения" },
        mainBalancingGroup ?: BalancingGroup.NONE,
        rest ?: RestProperties("", CircuitBreakerConfiguration()),
        kafka ?: KafkaProperties("", TopicsPair("", ""), CircuitBreakerConfiguration())
    ) {
        validateInitialConfiguration()
    }

    private fun validateInitialConfiguration() {
        require(nodeId.isNotEmpty()) { "audit.client.nodeId не должен быть пустым" }
        require(project.isNotEmpty()) { "audit.client.project не должен быть пустым" }
    }
}
```

### 3. Обновляем `RestPropertiesBuilder` и `KafkaPropertiesBuilder`

```kotlin
@ConfigurationProperties(prefix = "audit.client.rest")
data class RestPropertiesBuilder(
    override val enabled: Boolean?,
    val url: String?,
    val circuitBreaker: CircuitBreakerConfiguration?
) : EnablerProperties {
    fun build(): RestProperties {
        require(enabled != false) { "REST transport cannot be disabled when used in main balancing group" }
        requireNotNull(url) { "audit.client.rest.url обязателен для заполнения" }
        require(url.isNotBlank()) { "audit.client.rest.url не должен быть пустым" }
        requireNotNull(circuitBreaker) { "audit.client.rest.circuitBreaker обязателен для заполнения" }
        
        return RestProperties(url!!, circuitBreaker)
    }
}

@ConfigurationProperties(prefix = "audit.client.kafka")
data class KafkaPropertiesBuilder(
    override val enabled: Boolean?,
    val servers: String?,
    val topics: TopicsPairBuilder?,
    val circuitBreaker: CircuitBreakerConfiguration?
) : EnablerProperties {
    fun build(): KafkaProperties {
        require(enabled != false) { "Kafka transport cannot be disabled when used in main balancing group" }
        requireNotNull(servers) { "audit.client.kafka.servers обязателен для заполнения" }
        require(servers.isNotBlank()) { "audit.client.kafka.servers не должен быть пустым" }
        requireNotNull(topics) { "audit.client.kafka.topics обязателен для заполнения" }
        requireNotNull(circuitBreaker) { "audit.client.kafka.circuitBreaker обязателен для заполнения" }
        
        return KafkaProperties(servers!!, topics!!.build(), circuitBreaker)
    }
}
```

### 4. Добавляем обработчик ошибок для красивого вывода

```kotlin
@RestControllerAdvice
class AuditConfigValidationHandler {

    @ExceptionHandler(IllegalStateException::class)
    fun handleIllegalStateException(ex: IllegalStateException): ResponseEntity<ErrorResponse> {
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(ErrorResponse("Ошибка конфигурации аудита", ex.message))
    }

    data class ErrorResponse(val error: String, val message: String?)
}
```

### 5. Примеры валидных и невалидных конфигураций

**Валидная конфигурация для REST:**
```yaml
audit:
  enabled: true
  client:
    mainBalancingGroup: REST
    rest:
      enabled: true
      url: http://localhost:8090
    kafka:
      enabled: false
```

**Невалидная (выбросит исключение):**
```yaml
audit:
  enabled: true
  client:
    mainBalancingGroup: REST
    rest:
      enabled: false # Ошибка: должен быть true
    kafka:
      enabled: true
```

**Валидная конфигурация для DUPLICATE:**
```yaml
audit:
  enabled: true
  client:
    mainBalancingGroup: DUPLICATE
    rest:
      enabled: true
      url: http://localhost:8090
    kafka:
      enabled: true
      servers: localhost:9092
      topics:
        event: events
        metamodel: metamodel
```

### 6. Дополнительные улучшения

1. **Логирование** - добавьте логирование успешной проверки конфигурации
2. **Метрики** - можно добавить метрики для мониторинга используемых транспортов
3. **Тесты** - напишите тесты для всех возможных комбинаций конфигурации

```kotlin
@SpringBootTest
class AuditConfigValidationTest {

    @Test
    fun `should fail when DUPLICATE mode but kafka disabled`() {
        assertThrows<BeanCreationException> {
            SpringApplicationBuilder()
                .sources(YourApplication::class.java)
                .properties(
                    "audit.enabled=true",
                    "audit.client.mainBalancingGroup=DUPLICATE",
                    "audit.client.rest.enabled=true",
                    "audit.client.rest.url=http://localhost",
                    "audit.client.kafka.enabled=false"
                )
                .run()
        }
    }

    @Test
    fun `should pass when DUPLICATE mode and both transports enabled`() {
        assertDoesNotThrow {
            val ctx = SpringApplicationBuilder()
                .sources(YourApplication::class.java)
                .properties(
                    "audit.enabled=true",
                    "audit.client.mainBalancingGroup=DUPLICATE",
                    "audit.client.rest.enabled=true",
                    "audit.client.rest.url=http://localhost",
                    "audit.client.kafka.enabled=true",
                    "audit.client.kafka.servers=localhost:9092",
                    "audit.client.kafka.topics.event=events",
                    "audit.client.kafka.topics.metamodel=metamodel"
                )
                .run()
            ctx.close()
        }
    }
}
```

Эта реализация обеспечит:
- Проверку конфигурации при старте приложения
- Четкие сообщения об ошибках
- Гибкость в настройке транспортов
- Соответствие требованиям для разных режимов работы (REST/KAFKA/DUPLICATE)


Да, вы абсолютно правы! Конфигурация должна быть валидной и без блока `kafka`, когда используется `mainBalancingGroup: REST`. Вот как нужно модифицировать код для поддержки этого случая:

### 1. Обновляем `AuditClientProperties`

```kotlin
data class AuditClientProperties(
    val mode: AuditClientModeEnum,
    val nodeId: String,
    val project: String,
    val mainBalancingGroup: BalancingGroup = BalancingGroup.NONE,
    val rest: RestProperties? = null,  // Делаем nullable
    val kafka: KafkaProperties? = null // Делаем nullable
) {
    @ConstructorBinding
    constructor(
        nodeId: String?,
        mode: AuditClientModeEnum?,
        project: String?,
        mainBalancingGroup: BalancingGroup?,
        rest: RestProperties?,
        kafka: KafkaProperties?
    ) : this(
        mode ?: AuditClientModeEnum.SYNC,
        requireNotNull(nodeId) { "audit.client.nodeId обязателен для заполнения" },
        requireNotNull(project) { "audit.client.project обязателен для заполнения" },
        mainBalancingGroup ?: BalancingGroup.NONE,
        rest,
        kafka
    ) {
        validateInitialConfiguration()
    }

    private fun validateInitialConfiguration() {
        require(nodeId.isNotEmpty()) { "audit.client.nodeId не должен быть пустым" }
        require(project.isNotEmpty()) { "audit.client.project не должен быть пустым" }
    }
}
```

### 2. Модифицируем валидатор

```kotlin
private fun validateRestConfig() {
    require(auditProperties.client.rest != null) {
        "Для mainBalancingGroup=REST необходимо настроить REST конфигурацию"
    }
    require(auditProperties.client.rest.enabled) {
        "Для mainBalancingGroup=REST необходимо включить REST конфигурацию (audit.client.rest.enabled=true)"
    }
    validateRestProperties()
}

private fun validateKafkaConfig() {
    require(auditProperties.client.kafka != null) {
        "Для mainBalancingGroup=KAFKA необходимо настроить Kafka конфигурацию"
    }
    require(auditProperties.client.kafka.enabled) {
        "Для mainBalancingGroup=KAFKA необходимо включить Kafka конфигурацию (audit.client.kafka.enabled=true)"
    }
    validateKafkaProperties()
}

private fun validateDuplicateConfig() {
    require(auditProperties.client.rest != null && auditProperties.client.kafka != null) {
        "Для mainBalancingGroup=DUPLICATE необходимо настроить и REST и Kafka конфигурации"
    }
    require(auditProperties.client.rest.enabled && auditProperties.client.kafka.enabled) {
        "Для mainBalancingGroup=DUPLICATE необходимо включить и REST и Kafka конфигурации"
    }
    validateRestProperties()
    validateKafkaProperties()
}
```

### 3. Обновляем builders для поддержки optional конфигураций

```kotlin
@ConfigurationProperties(prefix = "audit.client.rest")
data class RestPropertiesBuilder(
    override val enabled: Boolean? = null, // Делаем optional
    val url: String? = null,
    val circuitBreaker: CircuitBreakerConfiguration? = null
) : EnablerProperties {
    fun build(): RestProperties? {
        if (enabled == false) return null
        requireNotNull(url) { "audit.client.rest.url обязателен для заполнения" }
        require(url.isNotBlank()) { "audit.client.rest.url не должен быть пустым" }
        requireNotNull(circuitBreaker) { "audit.client.rest.circuitBreaker обязателен для заполнения" }
        
        return RestProperties(url!!, circuitBreaker)
    }
}

// Аналогично для KafkaPropertiesBuilder
```

### 4. Теперь поддерживаются оба варианта конфигурации:

**Вариант 1 (с явным отключением Kafka):**
```yaml
audit:
  enabled: true
  client:
    mainBalancingGroup: REST
    rest:
      enabled: true
      url: http://localhost:8090
    kafka:
      enabled: false # Явное отключение
```

**Вариант 2 (полное отсутствие блока Kafka):**
```yaml
audit:
  enabled: true
  client:
    mainBalancingGroup: REST
    rest:
      enabled: true
      url: http://localhost:8090
    # Блок kafka полностью отсутствует
```

### 5. Проверка в валидаторе:

```kotlin
private fun validateConfiguration() {
    when (auditProperties.client.mainBalancingGroup) {
        BalancingGroup.REST -> {
            require(auditProperties.client.rest != null) {
                "Для mainBalancingGroup=REST необходимо настроить REST конфигурацию"
            }
            validateRestConfig()
            // Kafka может быть null или enabled=false
        }
        BalancingGroup.KAFKA -> {
            require(auditProperties.client.kafka != null) {
                "Для mainBalancingGroup=KAFKA необходимо настроить Kafka конфигурацию"
            }
            validateKafkaConfig()
            // REST может быть null или enabled=false
        }
        BalancingGroup.DUPLICATE -> {
            require(auditProperties.client.rest != null && auditProperties.client.kafka != null) {
                "Для mainBalancingGroup=DUPLICATE необходимо настроить и REST и Kafka конфигурации"
            }
            validateDuplicateConfig()
        }
        BalancingGroup.NONE -> {} // Ничего не проверяем
    }
}
```

### 6. Примеры обработки:

1. **Если блок `kafka` отсутствует** - `auditProperties.client.kafka` будет `null`
2. **Если `kafka.enabled: false`** - `auditProperties.client.kafka` будет не-null, но с `enabled = false`
3. **Для `mainBalancingGroup: REST`** - проверяется только REST конфигурация, Kafka игнорируется

Таким образом, ваша конфигурация теперь корректно обрабатывает все случаи:
- Полное отсутствие неиспользуемого транспорта
- Явное отключение неиспользуемого транспорта (`enabled: false`)
- Четкие сообщения об ошибках, если конфигурация не соответствует выбранному режиму


Аналогично должно быть и Для транспорта Kafka. 
**Вариант 1 (с явным отключением Rest):**
```
audit:
  enabled: true
  client:
    mainBalancingGroup: KAFKA
    rest:
      enabled: false # Явное отключение
    kafka:
      enabled: true
      servers: localhost:8091
      topics:
        event: event-topic
        metamodel: metamodel-topic
      circuitBreaker:
        recoveryTimeout: 1000
```

**Вариант 2 (полное отсутствие блока Rest):**
```
audit:
  enabled: true
  client:
    mainBalancingGroup: KAFKA
    # Блок Rest полностью отсутствует
    kafka:
      enabled: true
      servers: localhost:8091
      topics:
        event: event-topic
        metamodel: metamodel-topic
      circuitBreaker:
        recoveryTimeout: 1000
```
Это так в коде?


Да, абсолютно верно! Для режима `KAFKA` должна работать аналогичная логика: как с явным `rest.enabled: false`, так и при полном отсутствии блока `rest`. Вот как это реализовано в коде:

### 1. Финальная версия валидатора (`AuditConfigValidator`)

```kotlin
private fun validateConfiguration() {
    when (auditProperties.client.mainBalancingGroup) {
        BalancingGroup.REST -> {
            require(auditProperties.client.rest != null) {
                "Для mainBalancingGroup=REST необходимо настроить REST конфигурацию"
            }
            validateRestConfig()
            // Kafka может быть null или enabled=false
        }
        BalancingGroup.KAFKA -> {
            require(auditProperties.client.kafka != null) {
                "Для mainBalancingGroup=KAFKA необходимо настроить Kafka конфигурацию"
            }
            validateKafkaConfig()
            // REST может быть null или enabled=false
        }
        BalancingGroup.DUPLICATE -> {
            require(auditProperties.client.rest != null && auditProperties.client.kafka != null) {
                "Для mainBalancingGroup=DUPLICATE необходимо настроить и REST и Kafka конфигурации"
            }
            validateDuplicateConfig()
        }
        BalancingGroup.NONE -> {} // Ничего не проверяем
    }
}

private fun validateKafkaConfig() {
    // Проверяем только Kafka, REST не проверяем вообще
    require(auditProperties.client.kafka!!.enabled) {
        "Для mainBalancingGroup=KAFKA необходимо включить Kafka конфигурацию (audit.client.kafka.enabled=true)"
    }
    validateKafkaProperties()
}
```

### 2. Поддерживаемые конфигурации для KAFKA-режима

**Вариант 1 (с явным отключением REST):**
```yaml
audit:
  enabled: true
  client:
    mainBalancingGroup: KAFKA
    rest:
      enabled: false # Явное отключение
    kafka:
      enabled: true
      servers: localhost:9092
      topics:
        event: events
        metamodel: metamodel
```

**Вариант 2 (блок REST отсутствует):**
```yaml
audit:
  enabled: true
  client:
    mainBalancingGroup: KAFKA
    # Блок rest полностью отсутствует
    kafka:
      enabled: true
      servers: localhost:9092
      topics:
        event: events
        metamodel: metamodel
```

### 3. Код обработки nullable-конфигураций

В `AuditClientProperties`:

```kotlin
data class AuditClientProperties(
    // ... другие поля
    val rest: RestProperties? = null,  // Может быть null
    val kafka: KafkaProperties? = null // Может быть null
) {
    // Конструктор и валидация без изменений
}
```

### 4. Автоконфигурация транспортов

В конфигурационных классах (`RestBalancingGroupConfiguration` и `KafkaBalancingGroupConfiguration`):

```kotlin
@Configuration
@Conditional(RestCondition::class) // Бин создается только если rest.enabled != false
class RestBalancingGroupConfiguration {
    // ...
}

@Configuration
@Conditional(KafkaCondition::class) // Бин создается только если kafka.enabled != false
class KafkaBalancingGroupConfiguration {
    // ...
}
```

### 5. Проверка в тестах

```kotlin
@Test
fun `should accept KAFKA mode with missing rest block`() {
    assertDoesNotThrow {
        val ctx = SpringApplicationBuilder()
            .sources(YourApplication::class.java)
            .properties(
                "audit.enabled=true",
                "audit.client.mainBalancingGroup=KAFKA",
                "audit.client.kafka.enabled=true",
                "audit.client.kafka.servers=localhost:9092",
                "audit.client.kafka.topics.event=events",
                "audit.client.kafka.topics.metamodel=metamodel"
                // Блок rest отсутствует
            )
            .run()
        ctx.close()
    }
}

@Test
fun `should accept KAFKA mode with rest disabled`() {
    assertDoesNotThrow {
        val ctx = SpringApplicationBuilder()
            .sources(YourApplication::class.java)
            .properties(
                "audit.enabled=true",
                "audit.client.mainBalancingGroup=KAFKA",
                "audit.client.rest.enabled=false", // Явное отключение
                "audit.client.kafka.enabled=true",
                "audit.client.kafka.servers=localhost:9092",
                "audit.client.kafka.topics.event=events",
                "audit.client.kafka.topics.metamodel=metamodel"
            )
            .run()
        ctx.close()
    }
}
```

### Итоговые правила валидации:

| Режим              | REST конфигурация           | Kafka конфигурация          |
|--------------------|----------------------------|----------------------------|
| `REST`             | Обязательна и enabled=true | Может отсутствовать или enabled=false |
| `KAFKA`            | Может отсутствовать или enabled=false | Обязательна и enabled=true |
| `DUPLICATE`        | Обязательна и enabled=true | Обязательна и enabled=true |
| `NONE` (по умолчанию) | Не проверяется         | Не проверяется             |

Таким образом, ваш код полностью поддерживает оба варианта конфигурации для KAFKA-режима:
1. С явным отключением REST (`enabled: false`)
2. С полным отсутствием блока REST

Аналогично и для REST-режима с блоком Kafka.


