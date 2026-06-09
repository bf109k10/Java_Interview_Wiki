# 🗺️ Map of Content: Apache Kafka & Messaging
#moc #kafka #messaging #eda

### 🎯 Зона ответственности домена
Построение событийно-ориентированной архитектуры (Event-Driven Architecture), обеспечение надежности доставки сообщений и проектирование распределенных очередей.

### 🌿 Внутренние связи (Core Links)
*   [[Kafka_Delivery_Guarantees]] — Семантика Exactly-Once (EOS), транзакции в Kafka, идемпотентность и параметры `acks`.
*   *Скоро в базе:* `[[Kafka_Core_Architecture]]` — Устройство партиций, консьюмер-группы, офсеты и процесс ребалансировки (Rebalance).

### 🧠 Архитектурный паттерн Exactly-Once
```text
[Producer] ──(acks=all + Idempotence)──> 📂 [Kafka Brokers (ISR)] ──(isolation.level=read_committed)──> [Consumer] ──> 💾 [Идемпотентный DB Sink]
```

### 🔗 Родовые связи (Parent Links)
*   [[Index]] — На главную страницу.
*   [[Distributed_Transactions]] — Связь гарантий доставки Kafka с микросервисным паттерном Transactional Outbox.
