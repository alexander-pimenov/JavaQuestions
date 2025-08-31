# Принципы Domain-Driven Design (DDD) для разбиения системы на домены и поддомены.

## Основные концепции DDD

### 1. Домен (Domain)
**Домен** - это предметная область, которую охватывает ваше приложение. Это вся бизнес-логика и процессы, которые вы автоматизируете.

Пример: для банковского приложения домен - это "Банковские операции".

### 2. Поддомены (Subdomains)
Домен делится на поддомены - более мелкие, focused области:

#### Типы поддоменов:
- **Core Domain** - основная ценность бизнеса
- **Supporting Subdomain** - поддерживающие функции
- **Generic Subdomain** - стандартные функции, которые можно вынести наружу

## Пример разбиения банковской системы

```kotlin
// Core Domain - то, что делает банк уникальным
object CoreDomain {
    // Кредитные операции
    object Lending {
        fun approveLoan(application: LoanApplication) {}
        fun calculateInterest(loan: Loan) {}
    }
    
    // Инвестиционные продукты
    object Investment {
        fun createPortfolio(customer: Customer) {}
        fun executeTrade(order: TradeOrder) {}
    }
}

// Supporting Subdomain - поддерживающие функции
object SupportingDomain {
    // Управление клиентами
    object CustomerManagement {
        fun registerCustomer(customer: Customer) {}
        fun updateProfile(customerId: String, profile: Profile) {}
    }
    
    // Отчетность
    object Reporting {
        fun generateStatement(account: Account, period: Period) {}
    }
}

// Generic Subdomain - можно вынести в сторонние сервисы
object GenericDomain {
    // Email нотификации
    object Notifications {
        fun sendEmail(to: String, template: EmailTemplate) {}
    }
    
    // Файловое хранилище
    object FileStorage {
        fun uploadDocument(document: Document) {}
        fun getDocument(id: String) {}
    }
}
```

## Принципы разбиения на поддомены

### 1. Анализ бизнес-процессов
```java
public class DomainAnalysis {
    public void identifySubdomains() {
        // 1. Картируйте бизнес-процессы
        Map<String, List<BusinessProcess>> businessProcesses = mapBusinessProcesses();
        
        // 2. Группируйте по функциональности
        Map<String, BusinessCapability> capabilities = groupByCapability(businessProcesses);
        
        // 3. Определяйте зависимости
        analyzeDependencies(capabilities);
    }
}
```

### 2. Выявление ограниченных контекстов (Bounded Contexts)
```kotlin
// Bounded Context - четко очерченная граница, где модель имеет определенное значение
data class BoundedContext(
    val name: String,
    val domain: String,
    val responsibilities: List<String>,
    val ubiquitousLanguage: Map<String, String>
)

// Пример для банковского контекста
val lendingContext = BoundedContext(
    name = "Lending",
    domain = "Core",
    responsibilities = listOf("Loan approval", "Risk assessment", "Payment processing"),
    ubiquitousLanguage = mapOf(
        "Loan" to "Credit agreement",
        "Applicant" to "Customer seeking loan",
        "Collateral" to "Asset securing the loan"
    )
)
```

## Практические шаги реализации

### 1. Event Storming для выявления доменов
```java
public class EventStormingSession {
    public DomainModel discoverDomains() {
        // Шаг 1: Выявление domain events
        List<DomainEvent> events = discoverEvents();
        
        // Шаг 2: Группировка команд и агрегатов
        Map<String, Aggregate> aggregates = groupAggregates(events);
        
        // Шаг 3: Определение bounded contexts
        return defineBoundedContexts(aggregates);
    }
}
```

### 2. Создание доменной модели на Kotlin
```kotlin
// Aggregate Root - основной объект домена
class LoanApplication(
    val id: LoanApplicationId,
    val applicant: Customer,
    val amount: Money,
    val status: ApplicationStatus
) {
    fun submit() {
        require(amount > Money.ZERO) { "Amount must be positive" }
        this.status = ApplicationStatus.SUBMITTED
        DomainEventPublisher.publish(LoanApplicationSubmitted(this))
    }
    
    fun approve() {
        require(status == ApplicationStatus.SUBMITTED) { "Application must be submitted" }
        this.status = ApplicationStatus.APPROVED
        DomainEventPublisher.publish(LoanApplicationApproved(this))
    }
}

// Value Object
@JvmInline value class LoanApplicationId(val value: String)

// Domain Event
data class LoanApplicationSubmitted(val application: LoanApplication) : DomainEvent
```

### 3. Организация модулей (Java)
```java
// Структура пакетов по DDD
src/main/java/
└── com/company/bank/
    ├── core/                    // Core Domain
    │   ├── lending/             // Поддомен кредитования
    │   │   ├── application/     // Application Layer
    │   │   ├── domain/          // Domain Layer
    │   │   └── infrastructure/  // Infrastructure Layer
    │   └── investment/          // Поддомен инвестиций
    ├── support/                 // Supporting Subdomains
    │   ├── customermanagement/
    │   └── reporting/
    └── shared/                  // Shared Kernel
        ├── kernel/
        └── common/
```

## Ключевые принципы при разбиении

### 1. Принцип единственной ответственности
Каждый поддомен должен отвечать за одну четко определенную бизнес-способность.

### 2. Минимизация зависимостей
```kotlin
interface LoanRepository {
    fun findById(id: LoanApplicationId): LoanApplication?
    fun save(application: LoanApplication)
}

// Внедрение зависимостей через интерфейсы
class LoanService(
    private val loanRepository: LoanRepository,
    private val riskAssessment: RiskAssessmentService // отдельный сервис
) {
    fun processApplication(application: LoanApplication) {
        val riskScore = riskAssessment.evaluate(application)
        if (riskScore < THRESHOLD) {
            application.approve()
            loanRepository.save(application)
        }
    }
}
```

### 3. Ubiquitous Language
Используйте единый язык throughout всех слоев приложения.

## Инструменты и практики

- **Event Storming** - для выявления доменных событий
- **Context Mapping** - для определения отношений между контекстами
- **Hexagonal Architecture** - для изоляции доменной логики

DDD - это не просто технический подход, а методология, требующая тесного сотрудничества с бизнес-экспертами. 
Начинайте с малого, фокусируйтесь на core domain и постепенно расширяйте!

---

