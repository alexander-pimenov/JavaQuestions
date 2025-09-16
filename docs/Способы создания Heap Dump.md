Отлично! **Heap Dump** — это моментальный снимок памяти Java-процесса, который можно проанализировать в Eclipse MAT (Memory Analyzer Tool). Вот все способы собрать его.

## 📋 Способы создания Heap Dump

### 1. **Автоматически при OutOfMemoryError (Рекомендуется)**
```bash
java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./heapdump.hprof -jar your-app.jar
```
**Преимущества:**
- Автоматически создается при падении
- Не требует ручного вмешательства
- Идеально для продакшена

### 2. **Командой jmap (Во время работы)**
```bash
# Найти PID процесса
jps -l
# или
ps aux | grep java

# Создать heap dump
jmap -dump:live,format=b,file=heapdump.hprof <PID>

# Пример:
jmap -dump:live,format=b,file=heapdump_2024.hprof 12345
```
**Параметры:**
- `live` - только живые объекты (рекомендуется)
- `format=b` - бинарный формат
- `file=` - путь к файлу

### 3. **Через JConsole / VisualVM**
1. Запустите `jconsole` или `jvisualvm`
2. Подключитесь к процессу
3. Во вкладке "Monitor" → "Heap Dump" → "Save"

### 4. **Программно из кода**
```kotlin
import com.sun.management.HotSpotDiagnosticMXBean
import java.lang.management.ManagementFactory
import javax.management.MBeanServer

class HeapDumper {
    companion object {
        @JvmStatic
        fun dumpHeap(filePath: String, live: Boolean = true) {
            val server = ManagementFactory.getPlatformMBeanServer()
            val mxBean = ManagementFactory.newPlatformMXBeanProxy(
                server,
                "com.sun.management:type=HotSpotDiagnostic",
                HotSpotDiagnosticMXBean::class.java
            )
            mxBean.dumpHeap(filePath, live)
            println("Heap dump created: $filePath")
        }
    }
}

// Использование
HeapDumper.dumpHeap("./memory_leak.hprof", true)
```

## 🛠️ Практические примеры

### Для Spring Boot приложения:
```bash
# Способ 1: При запуске
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=./heapdumps/ \
     -Xmx512m \
     -jar application.jar

# Способ 2: Для запущенного процесса
jps -l | grep application.jar
# 45678 application.jar
jmap -dump:live,format=b,file=spring_boot_heap.hprof 45678
```

### Для Docker контейнера:
```bash
# Найти container ID
docker ps

# Создать dump внутри контейнера
docker exec <container_id> jmap -dump:live,format=b,file=/tmp/heap.hprof 1

# Скопировать на хост
docker cp <container_id>:/tmp/heap.hprof ./heap.hprof
```

## 📊 Анализ в Eclipse MAT

### Установка и настройка:
1. **Скачайте Eclipse MAT**: https://www.eclipse.org/mat/
2. **Запустите**: `mat` (на Linux/Mac) или `MemoryAnalyzer.exe` (Windows)
3. **Откройте heap dump**: File → Open Heap Dump

### Что искать в MAT:

| Вкладка | Что показывает |
|---------|----------------|
| **Overview** | Общая статистика по памяти |
| **Histogram** | Количество объектов по классам |
| **Dominator Tree** | Крупнейшие объекты, удерживающие память |
| **Leak Suspects** | Автоматический анализ утечек |
| **Top Components** | Крупнейшие компоненты |

### Пример анализа:
1. Откройте `heapdump.hprof`
2. Перейдите в **Leak Suspects**
3. Ищите:
    - **Большие массивы** (`byte[]`, `char[]`)
    - **Дублирующиеся строки** (`String`)
    - **Коллекции** (`HashMap`, `ArrayList`)
    - **Собственные объекты** вашего приложения

## 🚀 Продвинутые настройки

### Несколько дампов с таймстампами:
```bash
#!/bin/bash
# script dump_heap.sh
PID=$1
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DUMP_FILE="heapdump_${TIMESTAMP}.hprof"

jmap -dump:live,format=b,file=${DUMP_FILE} ${PID}
echo "Dump created: ${DUMP_FILE}"
```

### Автоматический дамп при высокой памяти:
```bash
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=./heapdumps/ \
     -XX:OnOutOfMemoryError="curl -X POST http://monitor/alert" \
     -jar app.jar
```

## 💡 Советы по анализу

1. **Сравнивайте дампы** - делайте 2 дампа с интервалом в 5-10 минут
2. **Ищите рост** - сравнивайте Histogram в двух дампах
3. **Обращайте внимание на:**
    - `java.lang.Thread` - утечки потоков
    - `java.lang.Class` - загрузчики классов
    - Ваши сервисы и компоненты

## ⚠️ Важные предупреждения

1. **Heap dump останавливает JVM** на время создания (несколько секунд)
2. **Файлы большие** (1-4GB+), убедитесь в наличии места
3. **Не используйте в production** без необходимости
4. **Тестируйте на staging** сначала

## Пример полного workflow:

```bash
# 1. Запуск с мониторингом
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=./heapdumps/ \
     -Xmx1g \
     -jar my-app.jar

# 2. Если есть подозрение на утечку
jps -l
# 12345 my-app.jar

jmap -dump:live,format=b,file=investigation.hprof 12345

# 3. Анализ в MAT
mat investigation.hprof
```

Теперь вы можете собрать heap dump и найти утечки памяти!

---

