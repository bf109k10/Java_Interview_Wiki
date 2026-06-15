# 🗺️ Map of Content: Java Core & JVM
#moc #java-core #jvm

### 🎯 Зона ответственности домена
Понимание внутреннего устройства виртуальной машины Java, управления памятью, оптимизаций компилятора (JIT) и эффективного использования структур данных под капотом.

### 🌿 Внутренние связи (Core Links)
*   [[JVM_Memory_Model]] — Стек, Хип, Metaspace, типы ссылок и флагманский сборщик мусора **ZGC**.
*   [[Collections_Deep_Dive]] — Эволюция структур данных: `HashMap` (Java 8 Treeify) и `ConcurrentHashMap` (CAS блокировки).
*   [[Java_17_to_25_Features]] — Эволюция синтаксиса (Records, Pattern Matching, Sealed classes).

### 🧠 Ментальная карта темы (Mental Model)
```text
                  ┌──> Stack (Локальные переменные, фреймы методов)
                  ├──> Metaspace (Метаданные классов, Native Memory)
[JVM Memory] ─────┤
                  └──> Heap ──> Young Gen (Eden, S0, S1)
                              └──> Old Gen ──> GC (G1 / ZGC)
                                                 │
                                                 └──> Оптимизация: Escape Analysis
```

### 🔗 Родовые связи (Parent Links)
*   [[Index]] — На главную страницу.

### 🔗 Связанные темы (Links):
*   [[MOC_Java_Core]]
*   [[Reflection_VS_Dynamic_Proxies_VS_CGLIB]]