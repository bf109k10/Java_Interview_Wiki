# 🗄️ Quarkus Panache & Дилемма SOLID
#quarkus

### 📌 Что это такое
Аналог **Spring Data JPA** для Quarkus, радикально сокращающий boilerplate-код работы с СУБД.

### ⚔️ Нарушение SOLID (Паттерн Active Record)
В Panache популярен подход **Active Record**, где сущность наследуется от `PanacheEntity`.

```java
// Использование в коде
User user = User.findById(123L);
user.name = "Alex";
user.persist(); 
```

* **Нарушение SRP (Single Responsibility Principle):** Класс `User` отвечает одновременно и за хранение данных (Domain), и за сохранение себя в базу (Infrastructure).
* **Компромисс (Trade-off):** Разработчики жертвуют чистотой SOLID ради скорости разработки (Time-to-Market) в мелких микросервисах.

### 🛠️ Альтернатива по SOLID
Если нужна чистая архитектура (DDD) без нарушения SRP, Panache позволяет использовать классический паттерн **Repository** через интерфейс `PanacheRepository<User>`.

### ❓ Вопросы на собеседовании:
* *В чем архитектурный недостаток паттерна Active Record?* (Ответ: Нарушение SRP, сильная связанность домена и инфраструктуры, сложность юнит-тестирования без мока базы).

### 🔗 Связанные темы (Links):
*   [[Reactive_Stack_WebFlux_vs_Quarkuss]]
