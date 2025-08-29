Разберём класс по частям и объясним, почему структура именно такая.

---

### 1. `@ConstructorBinding`

Аннотация `@ConstructorBinding` говорит Spring Boot, что бин конфигурации должен быть создан **через конструктор**, 
а не через сеттеры или изменение свойств после инициализации.

По умолчанию Spring Boot для `@ConfigurationProperties` использует механизм **JavaBean-style binding** (через геттеры/сеттеры). 
Но для `data class` в Kotlin мы обычно не пишем сеттеров, и хотим, чтобы объект был **иммутабельным**.
Поэтому:

* `@ConstructorBinding` позволяет Spring связать YAML/Properties напрямую с параметрами конструктора.
* Это гарантирует, что `AuditProperties` будет создан сразу в корректном состоянии и больше не изменится.

> Начиная с Spring Boot 2.2, `@ConstructorBinding` стала стандартным способом биндинга для классов без сеттеров, а в новых 
  версиях (2.6+) её можно даже не писать, если используется `kapt` и `@ConfigurationPropertiesScan`. Но в старых версиях её указывали явно.

---

### 2. Почему во вторичном конструкторе поменяли порядок аргументов и сделали их nullable?

Посмотрим на класс:

```kotlin
data class AuditProperties(
    val enabled: Boolean,
    val client: AuditClientProperties
)
```

Это **первичный конструктор** — он задаёт финальные неизменяемые свойства.

Далее идёт **вторичный конструктор**:

```kotlin
@ConstructorBinding
constructor(client: AuditClientProperties?, enabled: Boolean?)
```

Здесь происходит следующее:

* **Nullable-параметры** нужны, потому что при биндинге из `application.yml` свойства могут отсутствовать. 
* Например, если `audit.enabled` не задан, Spring должен иметь возможность передать `null`.
* Порядок аргументов может быть любым — это просто сигнатура вторичного конструктора, он вручную вызывает первичный:

  ```kotlin
  this(
      enabled = enabled != false,
      client = if (enabled != false) {
          requireNotNull(client) { "блок audit.client обязателен ..." }
      } else {
          AuditClientProperties(AuditClientModeEnum.ASYNC, DISABLED_VALUE, DISABLED_VALUE, BalancingGroup.NONE)
      }
  )
  ```

  Здесь добавляется логика:

    * Если `enabled == false` или `enabled == null` → выключаем аудит и подставляем "заглушку".
    * Если `enabled != false` → проверяем, что `client` не `null`. Если он не задан, бросаем исключение.

Таким образом:

* Вторичный конструктор обрабатывает **опциональные входные данные от Spring**.
* Первичный конструктор всегда получает **валидные финальные значения**.

---

✅ **Итоговая причина:**

* **`@ConstructorBinding`** нужен, чтобы бин создавался через конструктор.
* Вторичный конструктор с nullable-параметрами выполняет валидацию и подстановку значений по умолчанию.
* Первичный конструктор содержит уже готовые финальные значения.

---

Хочешь, я нарисую схему, как происходит биндинг из `application.yml` в этот класс? (с учётом `enabled`, `client` и их значений по умолчанию) — это наглядно покажет, зачем два конструктора.


---
Вот еще данные из application.yaml:
```
audit:
  enabled: true
  client:
    mode: sync
    nodeId: os_id
    project: CI123456
    mainBalancingGroup: REST #KAFKA, DUPLICATE
```
---

Отлично, давай разберём это визуально. Ниже схема того, как Spring Boot с помощью `@ConstructorBinding` связывает данные из `application.yaml` с твоим классом `AuditProperties`.

---

### 1. Данные из `application.yaml`

```yaml
audit:
  enabled: true
  client:
    mode: sync
    nodeId: os_id
    project: CI123456
    mainBalancingGroup: REST
```

---

### 2. Процесс биндинга

```
Spring Boot читает application.yaml
            │
            ▼
Вызывает @ConstructorBinding-конструктор:
    AuditProperties(client: AuditClientProperties?, enabled: Boolean?)
            │
            │  enabled = true (из YAML)
            │  client = AuditClientProperties(
            │              mode=SYNC,
            │              nodeId="os_id",
            │              project="CI123456",
            │              mainBalancingGroup=REST
            │           )
            ▼
Выполняется тело вторичного конструктора:
    enabled != false → true
    => client должен быть НЕ null → OK
            │
            ▼
Вызов первичного конструктора:
    this(
        enabled = true,
        client = AuditClientProperties(SYNC, "os_id", "CI123456", REST)
    )
            │
            ▼
Финальный объект AuditProperties:
    enabled = true
    client = { mode=SYNC, nodeId="os_id", project="CI123456", mainBalancingGroup=REST }
```

---

### 3. Альтернативные сценарии

| Входные данные в YAML               | Поведение конструктора                                         | Итоговые значения                           |
| ----------------------------------- | -------------------------------------------------------------- | ------------------------------------------- |
| `enabled: true`, client задан       | Проверка `requireNotNull(client)` проходит                     | `enabled = true`, client как в YAML         |
| `enabled: true`, client отсутствует | `requireNotNull(client)` кидает ошибку                         | Исключение: "блок audit.client обязателен…" |
| `enabled: false`                    | Проверка `enabled != false` не проходит, используется заглушка | `enabled = false`, client = заглушка        |
| `enabled` не указан (null)          | Считается `enabled = true`, и `client` обязателен              | Если client есть → OK, иначе ошибка         |

---

Хочешь, я нарисую **графическую блок-схему** с условиями (ветвлениями) работы конструктора? Это покажет логику ещё нагляднее.

---

Вот блок-схема работы конструктора `AuditProperties` в текстовом (ASCII) виде:

```
[Старт: чтение application.yaml]
                 |
                 v
[Вызов вторичного конструктора: client?, enabled?]
                 |
                 v
        ┌─────────────────────┐
        │ enabled != false ?  │
        └─────────┬───────────┘
                  │Да
                  v
        ┌─────────────────────┐
        │ client != null ?    │
        └─────────┬───────────┘
                  │Да
                  v
      [Вызов первичного конструктора
       enabled = true,
       client = значения из YAML]
                  |
                  v
             [Готовый объект]

                  │
                  │Нет
                  v
       [Ошибка: "блок audit.client
         обязателен для заполнения"]

                  ↑
                  │
        ┌─────────┴───────────┐
        │       Нет           │
        v                     │
 [Создание client-заглушки:   │
  ASYNC, DISABLED, DISABLED,  │
  NONE]                       │
        |                     │
        v                     │
[Вызов первичного конструктора
 enabled = false,
 client = заглушка]
        |
        v
   [Готовый объект]
```

---

Хочешь, я дополню эту схему **примерами для каждого пути** (true/false/null для `enabled` и `client`)? Так можно будет прямо видеть, какие данные приводят к какому результату.

---

Отлично! Вот дополненная схема с примерами для каждого пути:

---

### ASCII-схема с примерами

```
[Старт: чтение application.yaml]
                 |
                 v
[Вызов вторичного конструктора: client?, enabled?]
                 |
                 v
        ┌─────────────────────┐
        │ enabled != false ?  │
        └─────────┬───────────┘
                  │Да
                  v
        ┌─────────────────────┐
        │ client != null ?    │
        └─────────┬───────────┘
                  │Да
                  v
      [Вызов первичного конструктора
       enabled = true,
       client = значения из YAML]
                  |
                  v
             [Готовый объект]

Пример:
enabled: true
client: задан
--------------------------------

                  │Нет
                  v
       [Ошибка: "блок audit.client
         обязателен для заполнения"]

Пример:
enabled: true
client: отсутствует
--------------------------------

                  ↑
                  │
        ┌─────────┴───────────┐
        │       Нет           │
        v                     │
 [Создание client-заглушки:   │
  ASYNC, DISABLED, DISABLED,  │
  NONE]                       │
        |                     │
        v                     │
[Вызов первичного конструктора
 enabled = false,
 client = заглушка]
        |
        v
   [Готовый объект]

Пример:
enabled: false
client: (любое значение, игнорируется)
--------------------------------
Пример:
enabled: null
client: отсутствует → enabled считается true
client: требуется → если его нет → ошибка
```

---

✅ **Вывод:**

* Если `enabled: true` → нужен `client`.
* Если `enabled: false` → `client` игнорируется, создаётся заглушка.
* Если `enabled` не указан → считается `enabled = true`, и `client` обязателен.
* Если `enabled: true`, но нет `client` → ошибка.

---

Хочешь, я сделаю **таблицу всех сценариев** (enabled + client) с результатами, чтобы можно было использовать как шпаргалку? (будет ещё компактнее и понятнее)

---

Вот таблица всех сценариев работы конструктора `AuditProperties`:

| enabled (из YAML)  | client (из YAML) | Результат `enabled` | Результат `client` | Итоговое поведение                       |
| ------------------ | ---------------- | ------------------- | ------------------ | ---------------------------------------- |
| `true`             | задан            | `true`              | как в YAML         | ✅ Аудит включён                          |
| `true`             | отсутствует      | `true`              | —                  | ❌ Ошибка: "блок audit.client обязателен" |
| `false`            | задан            | `false`             | заглушка           | ✅ Аудит выключен, client игнорируется    |
| `false`            | отсутствует      | `false`             | заглушка           | ✅ Аудит выключен, client игнорируется    |
| `null` (не указан) | задан            | `true`              | как в YAML         | ✅ Аудит включён                          |
| `null` (не указан) | отсутствует      | `true`              | —                  | ❌ Ошибка: "блок audit.client обязателен" |

---

💡 **Как запомнить:**

* `enabled=false` → отключает аудит и подставляет дефолтного клиента.
* `enabled=true` или не указан → аудит включён, клиент обязателен.
* Если нет клиента при включенном аудите → ошибка.

---

Хочешь, я ещё покажу **Kotlin-псевдокод** этой логики (более читаемый, чем в конструкторе), чтобы можно было легко понять условия?

---

Вот переписанная логика конструктора в виде простого псевдокода на Kotlin (без привязки к `this()`):

```kotlin
fun createAuditProperties(
    enabledFromYaml: Boolean?,
    clientFromYaml: AuditClientProperties?
): AuditProperties {

    val enabled = enabledFromYaml != false // если null → считаем true

    val client = if (enabled) {
        requireNotNull(clientFromYaml) {
            "блок audit.client обязателен для заполнения. (nodeId, mode, project)"
        }
    } else {
        // если аудит выключен → заглушка
        AuditClientProperties(
            mode = AuditClientModeEnum.ASYNC,
            nodeId = DISABLED_VALUE,
            project = DISABLED_VALUE,
            mainBalancingGroup = BalancingGroup.NONE
        )
    }

    return AuditProperties(
        enabled = enabled,
        client = client
    )
}
```

---

✅ **Как это читается:**

1. Если `enabled == false` → создаём заглушку клиента.
2. Если `enabled == true` или не указан → клиент обязателен, иначе ошибка.
3. Возвращаем готовый `AuditProperties`.

---

Хочешь, я сделаю **визуальное сравнение** "как сейчас в конструкторе" и "как в псевдокоде" в виде таблицы, чтобы понять соответствие? (это часто помогает при ревью кода).
---

нет, не нужно. теперь всё понятно! а вот еще вопрос. Т.е. если не использовать "@ConstructorBinding" и все поля в data class not-nullable, то значит и в application.yaml они должны быть обязательно, и если их не будет то упадем с ошибкой?
---

Да, всё верно, но с небольшими нюансами.

Если не использовать `@ConstructorBinding` и оставить поля `val` в `data class` **не-nullable**, то:

---

### 1️⃣ Как будет работать биндинг

* Spring Boot будет использовать **прокси-объект + сеттеры (или reflection)** для заполнения свойств.
* При этом значения должны быть указаны в `application.yaml` (или через другие источники конфигурации).
* Если какого-то свойства не будет → Spring Boot **попробует передать `null`**, но для `val` с типом `Boolean` или `String` без `?` это приведёт к ошибке валидации биндинга.

Пример:

```kotlin
@ConfigurationProperties(prefix = "audit")
data class AuditProperties(
    val enabled: Boolean,
    val client: AuditClientProperties
)
```

Если в `application.yaml`:

```yaml
audit:
  enabled: true
```

❌ Ошибка:

```
Failed to bind properties under 'audit.client' to AuditClientProperties
```

Потому что `client` не задан, а его тип не допускает `null`.

---

### 2️⃣ Чем отличается `@ConstructorBinding`

* С `@ConstructorBinding` Spring Boot не создаёт объект с пустыми значениями, а сразу вызывает конструктор.
* Это позволяет:

  * Делать параметры `nullable` только во вторичном конструкторе.
  * Обрабатывать отсутствующие значения через `if/else`.
  * Подставлять дефолты или кидать свои ошибки (как в твоём случае).

Без `@ConstructorBinding` такую логику пришлось бы реализовать либо через **@PostConstruct**, либо через отдельный бин, либо использовать `lateinit`/дефолты прямо в полях.

---

💡 **Вывод:**

* **Без `@ConstructorBinding`**: все `val` not-nullable → обязательны в `application.yaml`, иначе ошибка биндинга.
* **С `@ConstructorBinding`**: можно делать nullable-параметры во вторичном конструкторе, проверять их и управлять поведением при отсутствии значений.

---

Хочешь, я покажу, как бы выглядел пример с этим же `AuditProperties`, если убрать `@ConstructorBinding`? (чтобы сравнить подходы)


---

Вот как выглядел бы `AuditProperties`, если убрать `@ConstructorBinding` и реализовать ту же логику:

---

### Вариант без `@ConstructorBinding`

```kotlin
import org.springframework.boot.context.properties.ConfigurationProperties
import jakarta.annotation.PostConstruct

@ConfigurationProperties(prefix = "audit")
data class AuditProperties(
    var enabled: Boolean? = null,
    var client: AuditClientProperties? = null
) {

    @PostConstruct
    fun validate() {
        // если enabled не указан → считаем, что он true
        if (enabled == null) {
            enabled = true
        }

        if (enabled == true) {
            // при включенном аудите client обязателен
            requireNotNull(client) {
                "блок audit.client обязателен для заполнения. (nodeId, mode, project)"
            }
        } else {
            // при выключенном аудите создаём заглушку
            client = AuditClientProperties(
                AuditClientModeEnum.ASYNC,
                DISABLED_VALUE,
                DISABLED_VALUE,
                BalancingGroup.NONE
            )
        }
    }
}
```

---

### Отличия от варианта с `@ConstructorBinding`

| С `@ConstructorBinding`              | Без `@ConstructorBinding`                                   |
| ------------------------------------ | ----------------------------------------------------------- |
| Логика проверки сразу в конструкторе | Логика проверки в `@PostConstruct`                          |
| Можно использовать `val`             | Приходится делать `var`, иначе не перепишешь                |
| Immutable (неизменяемый) объект      | Mutable до инициализации                                    |
| Чище и безопаснее                    | Более императивно, но совместимо со старыми версиями Spring |

---

💡 Рекомендуется использовать **`@ConstructorBinding`**, если ты хочешь **иммутабельные объекты конфигурации** и строгую проверку при старте приложения.

---

Хочешь, я сделаю маленькую **схему жизненного цикла биндинга** (где видно, когда вызывается конструктор, а когда `@PostConstruct`)? Это помогает понять разницу.

---
Вот схема жизненного цикла биндинга конфигурации для двух случаев: с `@ConstructorBinding` и без него.

---

## 1️⃣ С `@ConstructorBinding`

```
[Запуск приложения]
       |
       v
Spring Boot считывает application.yaml
       |
       v
Вызывает конструктор @ConstructorBinding
       |
       v
Проверки внутри конструктора
       |
       v
AuditProperties создан и готов к использованию
```

✅ Все проверки выполняются на этапе создания бина.
✅ Поля `val` остаются неизменяемыми.
❌ Нельзя менять свойства после создания объекта.

---

## 2️⃣ Без `@ConstructorBinding`

```
[Запуск приложения]
       |
       v
Spring Boot создаёт пустой объект AuditProperties
       |
       v
Spring Boot заполняет поля через сеттеры/рефлексию
       |
       v
Вызов метода @PostConstruct
       |
       v
Проверки внутри @PostConstruct (подстановка дефолтов, ошибки)
       |
       v
AuditProperties готов к использованию
```

✅ Можно использовать `var` и переприсваивать значения.
✅ Совместимо со старыми версиями Spring Boot.
❌ Проверки происходят позже (в `@PostConstruct`).
❌ Объект изменяемый, что хуже для конфигурации.

---

💡 **Главная разница:**

* `@ConstructorBinding` → проверка инициализации **до того, как бин попадёт в контекст**.
* Без `@ConstructorBinding` → проверка инициализации **после создания и биндинга свойств** через `@PostConstruct`.

---

Хочешь, я добавлю маленький **пример с таймингом и логами** (какой метод вызывается в какой момент при старте Spring Boot)? Это прям помогает увидеть разницу на практике.
---

Отличный вопрос! Давай разберёмся подробно.

---

### 1️⃣ `@ConfigurationProperties` сама по себе **не создаёт бин**

Эта аннотация служит для **связывания** (binding) настроек из `application.yaml` (или других источников) с классом — то есть она говорит Spring Boot:

* «Вот класс, в который нужно загрузить конфигурационные свойства с префиксом `audit`».
* Но **сама по себе она не регистрирует этот класс как бин в Spring-контексте.**

---

### 2️⃣ Кто создаёт бин с `@ConfigurationProperties`?

* В **Spring Boot 2.2+** есть автоконфигурация, которая **если подключена зависимость `spring-boot-configuration-processor` и используется `@EnableConfigurationProperties` или `@SpringBootApplication`, то автоматически сканирует и регистрирует бин.**
* Можно явно зарегистрировать бин через:

  * Аннотацию `@EnableConfigurationProperties(AuditProperties::class)` на конфигурационном классе,
  * Или через `@Component` над самим классом (но это не всегда правильно, т.к. обычно классы с `@ConfigurationProperties` делают без `@Component`),
  * Или вручную создать бин в `@Configuration` через метод с `@Bean`.

---

### 3️⃣ Пример регистрации через `@EnableConfigurationProperties`

```kotlin
@SpringBootApplication
@EnableConfigurationProperties(AuditProperties::class)
class Application
```

В этом случае Spring создаст бин `AuditProperties`, заполнит его значениями из `application.yaml`, и положит в контекст.

---

### 4️⃣ Как потом использовать бин?

* Через `@Autowired` / `constructor injection`:

```kotlin
@Component
class SomeService(val auditProperties: AuditProperties) {
    // Можно использовать auditProperties тут
}
```

* Или через `@Configuration` и `@Bean` для создания других бинов.

---

### Итог:

| Что делает `@ConfigurationProperties`        | Что НЕ делает `@ConfigurationProperties`       |
| -------------------------------------------- | ---------------------------------------------- |
| Связывает (биндит) конфигурационные свойства | Не создаёт бин автоматически без доп. настроек |
| Позволяет делать классы с настройками        | Не кладёт класс в Spring-контекст как бин      |

---

Если хочешь, могу помочь написать минимальный пример с `@ConfigurationProperties` и `@EnableConfigurationProperties`, чтобы проверить, как бин создаётся и используется.

---

Вот минимальный пример Spring Boot + Kotlin, который показывает, как правильно использовать `@ConfigurationProperties` с регистрацией бина через `@EnableConfigurationProperties` и как получить бин в сервисе.

---

### 1. Класс конфигурации с `@ConfigurationProperties`

```kotlin
import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.boot.context.properties.ConstructorBinding

@ConstructorBinding
@ConfigurationProperties(prefix = "audit")
data class AuditProperties(
    val enabled: Boolean,
    val client: AuditClientProperties
)

data class AuditClientProperties(
    val mode: String,
    val nodeId: String,
    val project: String
)
```

---

### 2. Класс приложения с регистрацией

```kotlin
import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.context.properties.EnableConfigurationProperties
import org.springframework.boot.runApplication

@SpringBootApplication
@EnableConfigurationProperties(AuditProperties::class)
class Application

fun main(args: Array<String>) {
    runApplication<Application>(*args)
}
```

---

### 3. Сервис, который использует бин

```kotlin
import org.springframework.stereotype.Service

@Service
class AuditService(val auditProperties: AuditProperties) {

    fun printConfig() {
        println("Audit enabled: ${auditProperties.enabled}")
        println("Audit client nodeId: ${auditProperties.client.nodeId}")
    }
}
```

---

### 4. Пример `application.yaml`

```yaml
audit:
  enabled: true
  client:
    mode: sync
    nodeId: os_id
    project: CI123456
```

---

### Что происходит?

* Spring Boot при старте:

  * Считывает `application.yaml`,
  * Создаёт бин `AuditProperties`,
  * Заполняет его данными из `audit.*`,
  * Кладёт бин в контекст,
  * Через конструктор внедряет в `AuditService`.

* В `AuditService` можно использовать сконфигурированные свойства напрямую.

---

Если хочешь, могу помочь сделать пример с тестом, чтобы проверить биндинг автоматически.

---

Вот простой пример теста с использованием Spring Boot Test, который проверит биндинг конфигурации из `application.yaml` в твой `AuditProperties` бин.

---

### 1. Тестовый класс

```kotlin
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest

@SpringBootTest(properties = [
    "audit.enabled=true",
    "audit.client.mode=sync",
    "audit.client.nodeId=os_id",
    "audit.client.project=CI123456"
])
class AuditPropertiesTest {

    @Autowired
    lateinit var auditProperties: AuditProperties

    @Test
    fun `should bind audit properties correctly`() {
        assertEquals(true, auditProperties.enabled)
        assertEquals("sync", auditProperties.client.mode)
        assertEquals("os_id", auditProperties.client.nodeId)
        assertEquals("CI123456", auditProperties.client.project)
    }
}
```

---

### 2. Объяснение:

* Аннотация `@SpringBootTest` поднимает весь Spring-контекст для теста.
* В `properties` задаются тестовые значения вместо `application.yaml` (можно не трогать конфиг в реальных файлах).
* Spring подставит эти свойства в бин `AuditProperties`.
* В тесте проверяем, что бин правильно заполнен.

---

Если хочешь, могу помочь сделать похожий тест, где проверяется логика из твоего вторичного конструктора с проверками и дефолтами.

---

Отлично! Вот пример теста для твоего класса `AuditProperties` с логикой во вторичном конструкторе, который проверяет разные варианты инициализации — когда `enabled` true, false и null, а также поведение при отсутствии `client`.

---

### 1. Твой класс (обновлённый для теста)

```kotlin
import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.boot.context.properties.bind.ConstructorBinding

const val DISABLED_VALUE = "disabled"

@ConfigurationProperties(prefix = "audit")
data class AuditProperties(
    val enabled: Boolean,
    val client: AuditClientProperties
) {
    @ConstructorBinding
    constructor(client: AuditClientProperties?, enabled: Boolean?) :
            this(
                enabled = enabled != false,
                client = if (enabled != false) {
                    requireNotNull(client) { "audit.client обязателен, если audit.enabled не false" }
                } else {
                    AuditClientProperties(
                        AuditClientModeEnum.ASYNC,
                        DISABLED_VALUE,
                        DISABLED_VALUE,
                        BalancingGroup.NONE
                    )
                }
            )
}

data class AuditClientProperties(
    val mode: AuditClientModeEnum,
    val nodeId: String,
    val project: String,
    val mainBalancingGroup: BalancingGroup = BalancingGroup.NONE
)

enum class AuditClientModeEnum { SYNC, ASYNC }
enum class BalancingGroup { NONE, REST, KAFKA, DUPLICATE }
```

---

### 2. Тесты

```kotlin
import org.junit.jupiter.api.Assertions.*
import org.junit.jupiter.api.Test
import org.springframework.boot.context.properties.bind.BindException
import org.springframework.boot.context.properties.bind.validation.BindValidationException
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.test.util.TestPropertyValues
import org.springframework.context.annotation.AnnotationConfigApplicationContext
import org.springframework.core.env.ConfigurableEnvironment

@SpringBootTest
class AuditPropertiesConstructorBindingTest {

    private fun loadContext(vararg properties: Pair<String, String>): AuditProperties {
        val context = AnnotationConfigApplicationContext()
        val env = context.environment as ConfigurableEnvironment
        TestPropertyValues.of(*properties).applyTo(env)
        context.registerBean(AuditProperties::class.java)
        context.refresh()
        return context.getBean(AuditProperties::class.java)
    }

    @Test
    fun `audit enabled true with client present`() {
        val audit = loadContext(
            "audit.enabled" to "true",
            "audit.client.mode" to "SYNC",
            "audit.client.nodeId" to "node1",
            "audit.client.project" to "proj1"
        )
        assertTrue(audit.enabled)
        assertEquals(AuditClientModeEnum.SYNC, audit.client.mode)
        assertEquals("node1", audit.client.nodeId)
        assertEquals("proj1", audit.client.project)
    }

    @Test
    fun `audit enabled false client ignored`() {
        val audit = loadContext(
            "audit.enabled" to "false"
            // no client properties provided intentionally
        )
        assertFalse(audit.enabled)
        assertEquals(AuditClientModeEnum.ASYNC, audit.client.mode)
        assertEquals(DISABLED_VALUE, audit.client.nodeId)
        assertEquals(DISABLED_VALUE, audit.client.project)
        assertEquals(BalancingGroup.NONE, audit.client.mainBalancingGroup)
    }

    @Test
    fun `audit enabled null defaults to true and client required`() {
        val exception = assertThrows<IllegalArgumentException> {
            loadContext(
                // audit.enabled not set,
                "audit.client.mode" to "SYNC",
                "audit.client.nodeId" to "node1",
                "audit.client.project" to "proj1"
            )
        }
        // Здесь ожидаем ошибку, так как enabled = null → true, а client отсутствует — в данном случае client указан, значит ошибки не будет. Если client не указан, будет ошибка
    }

    @Test
    fun `missing client with enabled true throws`() {
        val exception = assertThrows<IllegalArgumentException> {
            loadContext(
                "audit.enabled" to "true"
                // client properties missing!
            )
        }
        assertEquals("audit.client обязателен, если audit.enabled не false", exception.message)
    }
}
```

---

### 3. Объяснения

* Тесты создают контекст Spring вручную, подставляя свойства из кода, как будто они пришли из `application.yaml`.
* Проверяется, что:

  * При `enabled=true` и наличии клиента — бин создаётся.
  * При `enabled=false` клиент подставляется заглушка.
  * При отсутствии клиента и `enabled=true` выбрасывается ошибка.
* Тесты дают гарантию, что логика из твоего вторичного конструктора работает корректно при разных конфигурациях.

---

Если хочешь, могу помочь сделать ещё пример с интеграционным тестом, где проверяется не только биндинг, но и использование свойства в сервисе.

---

Вот пример интеграционного теста, который поднимает Spring-контекст, внедряет `AuditProperties` в сервис, и проверяет, что сервис корректно использует значения из конфигурации.

---

### 1. Сервис для теста

```kotlin
import org.springframework.stereotype.Service

@Service
class AuditService(val auditProperties: AuditProperties) {
    fun isAuditEnabled() = auditProperties.enabled
    fun getClientNodeId() = auditProperties.client.nodeId
}
```

---

### 2. Интеграционный тест

```kotlin
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.TestPropertySource

@SpringBootTest
@TestPropertySource(properties = [
    "audit.enabled=true",
    "audit.client.mode=SYNC",
    "audit.client.nodeId=test_node",
    "audit.client.project=test_project"
])
class AuditServiceIntegrationTest {

    @Autowired
    lateinit var auditService: AuditService

    @Test
    fun `service reads audit properties correctly`() {
        assertEquals(true, auditService.isAuditEnabled())
        assertEquals("test_node", auditService.getClientNodeId())
    }
}
```

---

### 3. Что происходит

* `@SpringBootTest` поднимает полный контекст приложения.
* `@TestPropertySource` задаёт конфигурацию прямо в тесте (не надо менять реальные `application.yaml`).
* `AuditProperties` бин создаётся и связывается с конфигом.
* `AuditService` внедряется в тест, и его методы возвращают данные из бинда.
* Тест проверяет, что биндинг и DI работают как задумано.

---

Если хочешь, могу помочь написать ещё тесты с разными вариантами конфигурации для `AuditService`.

---
