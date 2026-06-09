# 🗺️ Java Enterprise & System Design: Root MOC
#index #map-of-content #senior-prep

> **LLM Wiki (Karpathy Style)** — это интерактивный граф твоих инженерных знаний. Используй эту карту как отправную точку для интервального повторения перед собеседованиями. Нажимай `Ctrl + Клик` (в Obsidian) на любую тему, чтобы провалиться в глубокий технический конспект.

---

## 📁 01. Java Core & JVM Architecture
Фундамент виртуальной машины, структуры данных и управление памятью на низком уровне.
*   [[JVM_Memory_Model]] — Стек vs Хип, Escape Analysis, типы ссылок и эволюция сборщиков мусора (G1 vs **ZGC** в Java 21+).
*   [[Collections_Deep_Dive]] — Кишки `HashMap` и эволюция `ConcurrentHashMap` (Segment Locking в Java 7 vs Fine-Grained CAS в Java 8+).

## 📁 02. Advanced Concurrency & Threading Models
Парадигмы масштабирования вычислений и управление контекстом в высоконагруженных системах.
*   [[JMM_and_Volatile]] — Спецификация Java Memory Model, барьеры памяти, эффект `Instruction Reordering` и ловушки операции `counter++`.
*   [[Virtual_Threads_Reality]] — Инсайд Project Loom. Почему виртуальные потоки «споткнулись» в Java 21-23 (**Thread Pinning**, пулы БД) и как это починили в Java 24+.
*   [[ThreadLocal_vs_Scoped]] — Почему `ThreadLocal` стал опасным антипаттерном памяти и как ему на замену пришли неизменяемые `Scoped Values`.

## 📁 03. Spring Framework & High-Performance Engines
Внутреннее устройство главного корпоративного фреймворка и его современных альтернатив.
*   [[Spring_IoC_and_Lifecycle]] — Конвейер жизненного цикла бина, точки расширения BPP и запрет циклических зависимостей в Spring Boot 3.x.
*   [[Spring_AOP_and_Proxy]] — JDK Dynamic Proxy vs CGLIB. Разбор критической ловушки `Self-Invocation` при работе с `@Transactional`.
*   [[Spring_MVC_vs_WebFlux]] — Архитектурный разбор: почему `spring-web` — это лишь базис. Сравнение классического MVC и реактивного WebFlux.
*   [[RESTEasy_vs_Spring_Web]] — HTTP-стек в Quarkus. Разница между Classic и Reactive подходами на движке Vert.x.

## 📁 04. Data Storage, Caching & Performance
Как эффективно сохранять, индексировать и временно удерживать данные под высокой нагрузкой.
*   [[SQL_Indexes_and_Isolations]] — Аномалии параллельного доступа, устройство индексов B-Tree vs LSM-Tree и архитектура **MVCC** в PostgreSQL.
*   [[Hibernate_Data_and_Panache]] — Спор об архитектуре. Как Quarkus Panache упрощает Hibernate и почему паттерн `Active Record` жестко нарушает SOLID (SRP).
*   [[Redis_Architecture]] — Однопоточный Event Loop, стратегии вытеснения данных (LRU/LFU) и кэш-катастрофы (**Cache Avalanche** и **Cache Stampede**).

## 📁 05. Distributed Systems & Messaging (Apache Kafka)
Проектирование асинхронного взаимодействия и построение событийно-ориентированной архитектуры (EDA).
*   [[Kafka_Delivery_Guarantees]] — Разбор продюсерских `acks`, обеспечение идемпотентности и реализация семантики **Exactly-Once (EOS)**.

## 📁 06. Microservices & Cloud-Native Infrastructure
Шаблоны проектирования распределенных систем, отказоустойчивость и управление внутренним сетевым трафиком.
*   [[Distributed_Transactions]] — Как обеспечить консистентность без 2PC. Паттерны **Saga (Оркестрация vs Хореография)** и **Transactional Outbox** (Debezium).
*   [[Service_Discovery_and_Mesh]] — Управление трафиком. Разница между API Gateway (North-South) и **Service Mesh / Istio** (East-West через Sidecar Envoy).
*   [[MicroProfile_and_Standards]] — Открытый индустриальный стандарт облачных микросервисов как альтернатива Spring Cloud.

## 📁 07. System Design & Large-Scale Architecture
Проектирование систем, выдерживающих миллионы RPS, и принятие глобальных архитектурных компромиссов.
*   [[Scalability_and_Sharding]] — Горизонтальное масштабирование, теоремы CAP / PACELC, **Консистентное хэширование** и ловушка Scatter-Gather запросов.

---

### 📈 Инструкция по визуализации графа (Graph View)
1. Открой эту папку в **Obsidian**.
2. Нажми комбинацию клавиш `Ctrl + G` (или выбери значок графа на левой панели).
3. В настройках графа (шестеренка) включи отображение **Tags (Тегов)** и выставь группировку по папкам (окрашивание в разные цвета).
4. Ты увидишь, как `🗺️ Index.md` станет центральным ядром, от которого расходятся лучи к доменам знаний, а темы вроде `Virtual_Threads_Reality` или `Distributed_Transactions` начнут переплетаться между собой, формируя прочные нейронные связи в твоей голове.

