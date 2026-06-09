# 🗺️ Java Enterprise & System Design Knowledge Graph

[![Java Version](https://shields.io)](https://oracle.com)
[![Frameworks](https://shields.io)]()
[![Pattern](https://shields.io)]()

Welcome to my personal, high-density engineering knowledge base designed for **Middle+ / Senior Java Engineer** interview preparation and system architectural design.

This repository is built following the **Andrej Karpathy LLM Wiki pattern** and the **Zettelkasten methodology**—focusing on atomic notes, technical density, production trade-offs, and conceptual cross-linking rather than linear textbook definitions.

---

## 🚀 Key Architectural Principles

*   **Atomic Structure:** 1 file = 1 standalone technical concept.
*   **High Information Density:** Zero fluff. Direct explanations of underlying mechanics, internal JVM/framework telemetry, and data structures.
*   **Trade-off Oriented:** Every note explicitly breaks down what we sacrifice when adopting a technology (e.g., Active Record vs. SOLID, Service Mesh overhead, Virtual Threads edge cases).
*   **Interview & Production Traps:** Every topic contains real-world, deceptive senior-level interview questions and edge cases (e.g., Spring self-invocation, Postgres MVCC lost updates, Kafka consumer group rebalance loops).

---

## 📁 Repository Map (Core MOCs)

The repository is loosely flattened into core engineering domains. To explore the interconnected web of nodes, start from the root map of content: **`[[Index.md]]`**.

*   **`📁 01_Java_Core`** — Memory models (ZGC vs. G1), advanced collections telemetry, and modern JDK 17–25 LTS features.
*   **`📁 02_Concurrency`** — Low-level synchronizers (AQS, CAS), Project Loom Virtual Threads reality, and Scoped Values paradigm.
*   **`📁 03_Frameworks_Ecosystem`** — Deep-dive into Spring Boot 3.x IoC/AOP internals, Spring MVC vs. WebFlux, and Quarkus high-performance engines (RESTEasy Reactive).
*   **`📁 04_Databases_and_Cache`** — Relational database indexes, MVCC internals, Spring Data vs. Panache, and Redis High-Load architectures (Cache Avalanche/Stampede).
*   **`📁 05_Messaging_Kafka`** — Event-Driven Architecture, partitions tuning, and Kafka Exactly-Once Semantics (EOS).
*   **`📁 06_Microservices_Architecture`** — Distributed transactions (Saga vs. Transactional Outbox), Service Mesh (Istio/Envoy) vs. API Gateways, and MicroProfile standards.
*   **`📁 07_System_Design`** — Horizontal scaling strategies, CAP/PACELC trade-offs, Consistent Hashing, and distributed consensus (Raft/Paxos).

---

## 🛠️ Recommended Setup (Obsidian)

While these are pure Markdown files readable directly on GitHub, this repository is optimized to be opened as a local vault in **Obsidian**:

1. Clone this repository to your local machine:
   ```bash
   git clone https://github.com
   ```
2. Download and install [Obsidian](https://obsidian.md) (free).
3. Open Obsidian and select **"Open folder as vault"**, then choose the cloned repository folder.
4. Press `Ctrl + G` (or `Cmd + G` on Mac) to open the **Graph View**. 
5. In Graph View settings, enable **Tags** and group nodes by folder color to visualize your interactive engineering mind map.

---

## 🔒 License & Usage
Feel free to fork this repository to structure your own interview preparation and scale your backend architectural skills.
