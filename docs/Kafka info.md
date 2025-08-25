Разберём «по-человечески», вопросы про Kafka.

# 1) Ментальная модель Kafka

**Topic** = «дорога», **partition** = «полоса на этой дороге».
Сообщения внутри каждой *полосы/partition* упорядочены по **offset** и записываются append-only (_только для добавления_).
**Гарантия порядка** действует **только внутри одной partition**.

**Producer** при отправке выбирает partition:

* если задан **key** → `partition = hash(key) % partitions` → все события с одинаковым `key` всегда в одной `partition` (и значит — в порядке);
```java
        int partitions = 10;
        int partition1 = hash("some_key_for_test") % partitions;    // hash() = 2033546674 -> partition1 = 4
        int partition2 = hash("good weather") % partitions;         // hash() = -1018673232 -> partition2 = -2
        int partition3 = hash("have a nice day") % partitions;      // hash() = -933326065 -> partition3 = -5  
  ```
* если `key` нет → брокер распределяет «почти равномерно» (round-robin/sticky), **но без гарантий порядка между разными partition**.

# 2) Конкурентность (параллелизм) потребления

Параллелизм в Kafka задаёт **количество partition** и **число consumers в группе**.

* В одной **consumer group** **каждая partition** может быть назначена **ровно одному** consumer в группе.
* Следовательно, **максимальный параллелизм = min(число partitions, число consumers в группе)**.
* Если consumers больше, чем partitions → лишние простаивают.
* Если consumers меньше, чем partitions → один consumer тянет несколько partitions (но **внутри каждой partition** сообщения всё равно обрабатываются последовательно).

> Вывод: хотим больше параллелизма — **увеличиваем число partitions** и/или **число consumers** (инстансы сервиса/потоки), но помнить ограничение сверху = partitions.

## Порядок vs скорость

* Нужен **строгий порядок по сущности** (например, по `userId`) → делай `key = userId`. Тогда все события пользователя в одной partition → порядок сохранится. 
Цена: максимум одна «полоса» на пользователя, т.е. `обработка его событий последовательна`.
* Нужен **максимальный параллелизм** → повышай число partitions. Но порядок между разными partitions **не гарантируется**.

# 3) Репликация, лидерство и надёжность

У partition есть **leader** и **follower**-реплики (репликация ≥ 2–3). Писать/читать — через `leader`.
Настройка продьюсера `acks=all` + «`идемпотентность`» снижает риск потерь/дубликатов; транзакции нужны для «exactly-once» (_ровно один раз_)(дороже и сложнее).

# 4) Offsets и гарантии доставки

`Offsets` — «где мы остановились» — хранятся на брокерах в `__consumer_offsets` **для пары (group, partition)**.

Режимы:

* **At-most-once**: commit до обработки → быстрее, но можно потерять сообщения на крэше.
* **At-least-once** (`дефолт`): обработка → commit → возможны **повторы** при ретраях (делай обработчик идемпотентным).
* **Exactly-once**: транзакции (producer/Streams) → дороже по latency/сложности, но без дублей.

# 5) Rebalance (почему «дергается» потребление)

Когда в группу **входит/выходит** consumer, меняется подписка или количество partitions, Kafka делает **rebalance** — перераспределяет partitions.
Стратегии назначения: Range / RoundRobin / Sticky / CooperativeSticky (последняя снижает стоп-свет).

Практика: держи обработку быстрой (или выноси тяжёлую работу в worker-пулы), иначе рискуешь словить `max.poll.interval.ms` и вылет из группы → лишний ребаланс.

# 6) Как выбрать число partitions

Правило: планируй под **нужный параллелизм сейчас + запас**.

Пример: тебе нужно обрабатывать \~2 000 msg/s при скорости обработчика \~200 msg/s на поток.
Нужно \~10 потоков параллельно → сделай **минимум 10 partitions** (лучше 12–16 под запас и будущий рост).
Важно: **увеличивать partitions можно**, но **уменьшать** — практически нельзя без пересоздания топика.

# 7) Spring Kafka / Java: что значит `concurrency`

В Spring Kafka `@KafkaListener(concurrency = "N")` создаёт **N consumer-потоков** (внутри контейнера). Но параллелизм всё равно ограничен **числом partitions**.

Минимальный каркас:

```java
@EnableKafka
@Configuration
class KafkaConfig {

  @Bean
  ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(
      ConsumerFactory<String, String> cf) {
    var f = new ConcurrentKafkaListenerContainerFactory<String, String>();
    f.setConsumerFactory(cf);
    // Пара полезных настроек:
    // f.getContainerProperties().setAckMode(ContainerProperties.AckMode.RECORD); // commit после записи
    // f.setCommonErrorHandler(new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3L)));
    return f;
  }
}

@Service
class MyListener {

  // Параллелизм потоков потребления внутри инстанса
  @KafkaListener(topics = "orders", groupId = "billing", concurrency = "6")
  public void onMessage(ConsumerRecord<String, String> rec) {
    // ВАЖНО: метод вызывается конкурентно → код должен быть потокобезопасным.
    // Порядок для одного key сохранится ТОЛЬКО если все его записи в одной partition.
  }
}
```

**Важно помнить:**

* `concurrency` > `partitions` → часть потоков будет простаивать.
* Один поток может обслуживать **несколько** partitions, но **сообщения в каждой из них** обрабатываются по очереди.
* Длинная обработка? Либо подними `max.poll.interval.ms`, либо **делегируй** работу в executor и используй `pause()/resume()` partitions, либо включай 
  батчи (`BatchListener` + `max.poll.records`) — и **коммить** осознанно.
* Для гарантированного порядка по key — **не параллелься внутри одной partition**.

Если нужно жёстко закрепить разделы за слушателем (редкий кейс), можно так:

```java
@KafkaListener(groupId = "analytics",
  topicPartitions = {
    @TopicPartition(topic = "events", partitions = {"0","1","2"})
})
public void on(ConsumerRecord<String, String> rec) { ... }
```

# 8) Частые вопросы на собеседованиях (и короткие ответы)

**Q: Что даёт partition?**
A: Масштабирование и изоляция порядка: порядок гарантируется только внутри одной partition.

**Q: Сколько максимум параллелизма?**
A: `min(#partitions, #consumers в группе)`.

**Q: Как сохранить порядок по пользователю?**
A: Ключуй сообщения `key = userId`, чтобы все они попали в одну partition.

**Q: Почему при увеличении consumers ничего не ускорилось?**
A: Потому что упёрлись в число partitions; лишние consumers простаивают.

**Q: Где хранятся offsets?**
A: В специальном топике `__consumer_offsets`, по (group, partition).

**Q: Как избежать дублей при at-least-once?**
A: Сделать обработчик идемпотентным (например, upsert по ключу, дедупликация по eventId).

**Q: Что такое ребаланс и как его смягчить?**
A: Перераспределение partitions при изменениях в группе; использовать sticky/cooperative assignor, избегать долгих обработок/stop-the-world.

**Q: Как подобрать #partitions?**
A: От требуемого параллелизма/пропускной способности сейчас + запас (учти, уменьшать нельзя).

---

🔥 Шпаргалка по **Kafka + Spring Kafka**, чтобы можно было быстро освежить знания.

---

# 📝 Kafka + Spring Kafka шпаргалка

---

## 1. Базовые термины

* **Topic** — логический канал сообщений.
* **Partition** — «полоса» внутри топика, упорядоченная по offset.
* **Consumer group** — группа потребителей, которые совместно читают топик (каждый partition → только одному consumer в группе).
* **Offset** — позиция в partition.

---

## 2. Параллелизм (concurrency)

* Параллелизм ограничен:

```
  concurrency = min(#partitions, #consumers в group)
```

* Внутри одной partition сообщения **строго по порядку**.
* Порядок между partition **не гарантируется**.

👉 Чтобы все сообщения одного пользователя обрабатывались по порядку → `key = userId`.

---

## 3. Producer (отправка сообщений)

Основные настройки:

```yaml
spring:
  kafka:
    producer:
      bootstrap-servers: localhost:9092
      acks: all        # ждём подтверждения от всех реплик
      retries: 3       # ретраи при ошибке
      properties:
        enable.idempotence: true   # идемпотентность (нет дублей)
        max.in.flight.requests.per.connection: 1
```

Код:

```java
@Service
class Producer {
  private final KafkaTemplate<String, String> template;

  public Producer(KafkaTemplate<String, String> template) {
    this.template = template;
  }

  public void send(String key, String value) {
    template.send("orders", key, value);
  }
}
```

---

## 4. Consumer (чтение сообщений)

Настройки:

```yaml
spring:
  kafka:
    consumer:
      bootstrap-servers: localhost:9092
      group-id: billing-service
      enable-auto-commit: false
      auto-offset-reset: earliest
      max-poll-records: 50  # за раз читаем до 50 сообщений
```

Код:

```java
@Service
class MyListener {

  // concurrency = кол-во consumer-потоков внутри приложения
  @KafkaListener(topics = "orders", concurrency = "4")
  public void onMessage(ConsumerRecord<String, String> rec) {
    // Обработка сообщения
    System.out.println("Got: " + rec.value());
  }
}
```

---

## 5. Commit offsets

* **At-most-once**: commit до обработки (быстро, но возможна потеря).
* **At-least-once (дефолт)**: обработка → commit (надёжно, но возможны дубли).
* **Exactly-once**: транзакции (дороже, редко надо).

Пример ручного коммита:

```java
@KafkaListener(topics = "orders")
public void listen(ConsumerRecord<String, String> rec, Acknowledgment ack) {
    try {
        process(rec.value());
        ack.acknowledge(); // коммит offset
    } catch (Exception e) {
        // не коммитим, сообщение вернется
    }
}
```

---

## 6. Ошибки и ретраи

* **DefaultErrorHandler** — обработка ошибок + ретраи.
* **DLT (Dead Letter Topic)** — топик для «плохих» сообщений.

Пример:

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> template) {
    var recoverer = new DeadLetterPublishingRecoverer(template);
    var backOff = new FixedBackOff(1000L, 3L); // 3 ретрая с паузой 1с
    return new DefaultErrorHandler(recoverer, backOff);
}
```

---

## 7. Пауза / резюмирование (pause / resume)

Если обработка тяжёлая:

```java
@KafkaListener(id = "orders", topics = "orders")
public void listen(ConsumerRecord<String, String> rec, 
                   Acknowledgment ack, 
                   Consumer<?,?> consumer) {
    consumer.pause(Collections.singleton(new TopicPartition("orders", rec.partition())));
    try {
        process(rec.value());
        ack.acknowledge();
    } finally {
        consumer.resume(Collections.singleton(new TopicPartition("orders", rec.partition())));
    }
}
```

---

## 8. Полезные пропсы

* `max.poll.records` — сколько сообщений за раз тянем.
* `max.poll.interval.ms` — макс. время между poll() (увеличивай при долгой обработке).
* `session.timeout.ms` — время, через которое consumer считается мёртвым.
* `fetch.min.bytes` / `fetch.max.wait.ms` — управление размером/задержкой пакетов.

---

## 9. Частые собеседовательные вопросы

* **Q: Что даёт partition?**
  Масштабируемость и порядок (внутри partition).
* **Q: Почему лишние consumers простаивают?**
  Потому что consumers > partitions.
* **Q: Как сохранить порядок по пользователю?**
  Использовать key → все события этого key в одной partition.
* **Q: Где хранятся offsets?**
  В топике `__consumer_offsets` (по группе и partition).
* **Q: Как избежать дублей?**
  Сделать обработку идемпотентной (например, по eventId).
* **Q: Что такое rebalance?**
  Перераспределение partitions между consumers (при входе/выходе участников).

---

📌 **Главное правило**:

* Больше partitions → выше параллелизм, но сложнее порядок.
* Порядок по `key` сохраняется только в рамках одной `partition`.
* Дубли → нормально, делай обработчик идемпотентным.

---

Хочешь, я соберу это в **одну таблицу-«шпаргалку»** (в виде коротких пунктов/сравнений), чтобы можно было за пару минут пробежаться перед интервью?


---

Супер 🚀 Вот тебе компактная **таблица-шпаргалка по Kafka** — можно пробежать глазами прямо перед собеседованием:

---

# 📝 Kafka шпаргалка (для собесов)

| Тема                    | Ключевые моменты                                                                                                                                                                              |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Topic**               | Логический канал сообщений                                                                                                                                                                    |
| **Partition**           | «Полоса» внутри топика, гарантирует порядок сообщений **только внутри себя**                                                                                                                  |
| **Offset**              | Позиция сообщения в partition                                                                                                                                                                 |
| **Consumer Group**      | Совместное чтение топика; **одна partition → только одному consumer в группе**                                                                                                                |
| **Макс. параллелизм**   | `min(#partitions, #consumers в группе)`                                                                                                                                                       |
| **Producer (отправка)** | - `acks=all` (надёжно)<br>- `enable.idempotence=true` (нет дублей)<br>- ключ (`key`) определяет partition                                                                                     |
| **Consumer (чтение)**   | - `enable-auto-commit=false` (лучше ручной commit)<br>- `max.poll.records` — сколько сообщений за раз<br>- `max.poll.interval.ms` — увеличить при долгой обработке                            |
| **Commit offset**       | - *at-most-once*: commit до обработки (потери)<br>- *at-least-once*: commit после обработки (дубли возможны)<br>- *exactly-once*: транзакции (дорого, редко)                                  |
| **Ретраи и ошибки**     | `DefaultErrorHandler` + `DeadLetterPublishingRecoverer` → DLT (топик для «плохих» сообщений)                                                                                                  |
| **Порядок сообщений**   | - Сохраняется только в одной partition<br>- Чтобы порядок был по пользователю → `key = userId`                                                                                                |
| **Rebalance**           | Перераспределение partitions при изменении числа consumers/partitions                                                                                                                         |
| **Как масштабировать**  | - Увеличивать partitions<br>- Поднимать больше consumers (но не больше partitions)                                                                                                            |
| **Практические советы** | - Долгие задачи: либо увеличь `max.poll.interval.ms`, либо вынеси в worker pool<br>- Обработчик делай идемпотентным (eventId, upsert)<br>- Партиций лучше задать с запасом (уменьшить нельзя) |

---

⚡ **Сверхкороткие ответы для собеса**:

* *Почему UUID уникален?* → 2¹²⁸ комбинаций, вероятность коллизии ≈ 0.
* *Что даёт partition?* → масштабируемость и порядок внутри одной линии.
* *Какой макс. параллелизм?* → ограничен числом partitions.
* *Как сохранить порядок по userId?* → key=userId.
* *Что делать с дублями?* → идемпотентная обработка.
* *Что такое rebalance?* → перераспределение partitions при изменении группы.

---


Очень правильный вопрос 👍
Разберём по шагам.

---

## 🔹 Что такое `max.poll.interval.ms`

* Это **consumer-side настройка** (для клиента, который читает из Kafka).
* Она определяет **максимальное время между двумя вызовами `poll()`**.
* Если consumer **слишком долго** обрабатывает сообщения и не вызывает `poll()` снова, брокер считает его **«упавшим»** и запускает **rebalance** → partition уходит другому consumer.

👉 То есть, **если у тебя обработка долгих сообщений** (например, 30 сек, 1 мин), надо увеличить `max.poll.interval.ms`, чтобы consumer не вылетал из группы.

---

## 🔹 Для кого это настраивается

Это параметр **Kafka consumer**, значит:

* Работает на **потребителях** (consumers).
* Настраивается на стороне приложения (Spring Kafka → `application.yaml` или `ConsumerFactory`).

---

## 🔹 Как задать в `application.yaml`

Да, можно! В Spring Boot всё просто:

```yaml
spring:
  kafka:
    consumer:
      properties:
        max.poll.interval.ms: 600000   # 10 минут
```

или через Java Config:

```java
props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 600000);
```

---

## 🔹 Важный нюанс

* Увеличение `max.poll.interval.ms` **не увеличивает время heartbeat**.
  Чтобы consumer не выпал из группы, он продолжает слать heartbeat (через `heartbeat.interval.ms`) параллельно обработке.
* Но если ты «зависнешь» дольше, чем `max.poll.interval.ms` без вызова `poll()` → consumer считается застрявшим, идёт ребаланс.

---

## 🔹 Практический совет

1. Если у тебя **долгие тяжёлые операции** → подними `max.poll.interval.ms` (например, до 5–10 минут).
2. Если можно — **дели работу** на чанки и обрабатывай быстрее, чтобы не приходилось дёргать параметр.
3. Ещё лучше: вынеси тяжёлую обработку в отдельный пул потоков (worker pool), а consumer только **быстро подтверждает offset**.

---

📌 На собесе хорошо ответить так:

> "`max.poll.interval.ms` — это настройка consumer, которая определяет максимальное время между poll(). Если consumer долго обрабатывает сообщения, но не вызывает poll, он выпадет из группы. В Spring Boot это можно задать в `application.yaml` через `spring.kafka.consumer.properties.max.poll.interval.ms`. Обычно его увеличивают при долгих обработках или выносят тяжёлую работу в отдельный пул потоков."

---

Хочешь, я ещё покажу пример конфигурации, где одновременно выставлены **`max.poll.interval.ms`** и **`max.poll.records`**, чтобы оптимизировать потребление при тяжёлой обработке?


---

Отличный запрос 👍
Эти два параметра как раз часто настраивают вместе:

* **`max.poll.records`** — сколько сообщений consumer забирает за один `poll()`.
* **`max.poll.interval.ms`** — сколько максимально может длиться обработка этой пачки сообщений.

Идея: если у тебя тяжёлая обработка (например, каждое сообщение обрабатывается по 2–3 секунды), то:

* уменьшить `max.poll.records`, чтобы за один `poll()` не тянуть слишком много;
* увеличить `max.poll.interval.ms`, чтобы обработать пачку и не вылететь из группы.

---

## 🔹 Пример в `application.yaml`

```yaml
spring:
  kafka:
    consumer:
      bootstrap-servers: localhost:9092
      group-id: heavy-worker
      enable-auto-commit: false
      auto-offset-reset: earliest
      properties:
        max.poll.records: 10         # берём максимум 10 сообщений за раз
        max.poll.interval.ms: 300000 # даём до 5 минут на их обработку
```

---

## 🔹 Java-конфиг с ConsumerFactory

```java
@Configuration
@EnableKafka
public class KafkaConfig {

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "heavy-worker");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        // 🔑 наши параметры:
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 10);         // меньше сообщений за раз
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000); // 5 минут на обработку пачки

        return new DefaultKafkaConsumerFactory<>(props, new StringDeserializer(), new StringDeserializer());
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
}
```

---

## 🔹 Как это работает на практике

Допустим:

* каждое сообщение обрабатывается \~2 сек,
* в пачке (`max.poll.records = 10`) будет максимум 10 сообщений,
* значит worst-case обработка пачки = \~20 сек.

Мы задали `max.poll.interval.ms = 300000` (5 минут), что с запасом покрывает такой сценарий.

И брокер не выкинет consumer из группы, даже если он «задержался» на 20–30 секунд.

---

👉 На собесе хорошо сказать так:

> "Чтобы настроить consumer под тяжёлую обработку, мы уменьшаем `max.poll.records`, чтобы не получать слишком много сообщений за раз, и увеличиваем `max.poll.interval.ms`, чтобы consumer успел обработать пачку и не выпасть из группы. В Spring Boot это можно задать прямо в `application.yaml`."

---

Хочешь, я соберу тебе **рецепты для разных сценариев** (короткие/долгие задачи, батчи, параллельная обработка в пуле потоков), чтобы сразу иметь готовые паттерны?

---

Отлично! 🚀
Тогда давай сделаем **шпаргалку с рецептами конфигурации Kafka consumer под разные сценарии**.

---

# 📝 Рецепты настройки Kafka Consumer

---

## 1. ⚡ Быстрая обработка (легкие сообщения)

* Цель: максимальная скорость, минимальная задержка.
* Сообщения обрабатываются <100 мс.

```yaml
spring:
  kafka:
    consumer:
      group-id: fast-worker
      enable-auto-commit: true      # можно оставить авто-коммит
      properties:
        max.poll.records: 500       # побольше сообщений за раз
        max.poll.interval.ms: 300000 # дефолт (5 мин) достаточно
```

## 👉 Применяется для логирования, метрик, лёгких ивентов.

## 2. 🛠 Тяжёлая обработка (CPU / DB операции)

* Цель: не перегружать consumer, дать время обработать пачку.
* Каждое сообщение занимает секунды.

```yaml
spring:
  kafka:
    consumer:
      group-id: heavy-worker
      enable-auto-commit: false
      properties:
        max.poll.records: 10        # мало сообщений за раз
        max.poll.interval.ms: 600000 # 10 минут на обработку пачки
```

## 👉 Хорошо для тяжёлых бизнес-процессов (например, расчёт отчётов, сложные запросы).

## 3. 📦 Батч-обработка (bulk)

* Цель: обработка пачки сообщений одним запросом к БД / API.

```yaml
spring:
  kafka:
    listener:
      type: batch  # включаем батч-листенер
    consumer:
      group-id: batch-worker
      enable-auto-commit: false
      properties:
        max.poll.records: 100       # берём по 100 сообщений
        max.poll.interval.ms: 300000
```

Код:

```java
@KafkaListener(topics = "events", containerFactory = "batchFactory")
public void listen(List<ConsumerRecord<String, String>> records) {
    // пачка сообщений
    bulkInsert(records.stream().map(ConsumerRecord::value).toList());
}
```

👉 Подходит, если выгоднее писать в БД/отправлять API **сразу пачкой**.

---

## 4. 🔄 Параллельная обработка (worker pool)

* Цель: consumer быстро «вытаскивает» сообщения, а тяжёлая работа уходит в отдельный пул потоков.

```yaml
spring:
  kafka:
    consumer:
      group-id: async-worker
      enable-auto-commit: false
      properties:
        max.poll.records: 50
        max.poll.interval.ms: 600000
```

Код:

```java
@Service
class AsyncListener {

    private final ExecutorService pool = Executors.newFixedThreadPool(10);

    @KafkaListener(topics = "orders", concurrency = "3")
    public void listen(ConsumerRecord<String, String> rec, Acknowledgment ack) {
        pool.submit(() -> {
            try {
                process(rec.value()); // тяжёлая работа
                ack.acknowledge();
            } catch (Exception e) {
                // retry / dlt
            }
        });
    }
}
```

👉 Подходит для тяжёлых задач, где нужно максимизировать throughput.

---

## 5. ❌ Работа с ошибками и DLT

* Цель: не терять сообщения, но и не блокировать поток.

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> template) {
    var recoverer = new DeadLetterPublishingRecoverer(template);
    var backOff = new FixedBackOff(2000L, 3L); // 3 ретрая с паузой 2 сек
    return new DefaultErrorHandler(recoverer, backOff);
}
```

👉 Сообщения после 3 неудач отправятся в **Dead Letter Topic** (например, `orders.DLT`).

---

# 📌 Краткий чек-лист перед собесом

1. **Лёгкие задачи** → `max.poll.records` побольше, можно авто-коммит.
2. **Тяжёлые задачи** → `max.poll.records` уменьшить, `max.poll.interval.ms` увеличить.
3. **Батчи** → `listener.type=batch` + `max.poll.records` под размер пачки.
4. **Worker pool** → consumer быстро тянет, обработка уходит в отдельные потоки.
5. **Ошибки** → DefaultErrorHandler + DLT, чтобы не терять сообщения.

---

🔥 Это прям готовый набор для любого вопроса про «как настроить consumer под разные сценарии».

Хочешь, я оформлю это в **.docx шпаргалку** (таблицы + примеры кода), чтобы удобно держать под рукой при подготовке?

---

