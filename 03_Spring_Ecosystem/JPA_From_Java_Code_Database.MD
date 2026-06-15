# JPA: От Java-кода до базы данных
#jpa #jdbc #spring

> Конспект статьи: "How JPA Works: From Your Java Code to the Database"
> Дополнен деталями по Persistence Context, EntityManager и Hibernate internals.
>
> Источник:
> https://robertniestroj.hashnode.dev/how-jpa-works-from-your-java-code-to-the-database

---

# Общая схема

```text
Spring @Service
  ↓
Repository (Spring Data JPA)
  ↓
EntityManager
  ↓
Persistence Context
  ↓
Hibernate
  ↓
JDBC
  ↓
Database
```

Главная идея: когда разработчик вызывает `repository.save(entity)` или `repository.findById(id)`, между кодом и БД проходит несколько слоев абстракции.

---

# Основные компоненты

## JPA

JPA (Jakarta Persistence API) — это спецификация.

Она определяет:

- Entity
- EntityManager
- Persistence Context
- JPQL
- Транзакции

Сама по себе JPA ничего не делает.

Нужна реализация:

- Hibernate (де-факто стандарт)
- EclipseLink
- OpenJPA

---

## Spring Data JPA

Spring Data JPA предоставляет:

```java
public interface UserRepository
    extends JpaRepository<User, Long> {
}
```

Без реализации.

Во время старта Spring:

1. Сканирует интерфейсы Repository.
2. Генерирует прокси.
3. Делегирует вызовы EntityManager.

---

## EntityManager

Главный интерфейс JPA.

Основные методы:

```java
persist()
merge()
find()
remove()
flush()
clear()
detach()
```

EntityManager управляет Persistence Context.

---

# Persistence Context

## Что это такое

Persistence Context — это контейнер управляемых сущностей.

Можно считать его:

- Unit Of Work
- L1 Cache
- Контекстом отслеживания изменений

Каждый EntityManager имеет собственный Persistence Context. :contentReference[oaicite:0]{index=0}

---

## Зачем нужен

Он обеспечивает:

### 1. First Level Cache

Повторный запрос объекта не вызывает SQL.

```java
User u1 = em.find(User.class, 1L);
User u2 = em.find(User.class, 1L);
```

SQL выполнится один раз.

```java
u1 == u2 // true
```

Persistence Context гарантирует единственный экземпляр сущности с данным ID внутри контекста. :contentReference[oaicite:1]{index=1}

---

### 2. Identity Guarantee

Для одного ID внутри контекста всегда возвращается один и тот же объект.

---

### 3. Dirty Checking

Hibernate автоматически отслеживает изменения.

```java
User user = em.find(User.class, 1L);

user.setName("John");
```

Никакого:

```java
update(user);
```

не требуется.

При commit Hibernate сам сформирует UPDATE. :contentReference[oaicite:2]{index=2}

---

# Жизненный цикл Entity

## Transient

Создан через new.

```java
User user = new User();
```

Не известен JPA.

---

## Managed

После:

```java
em.persist(user);
```

или

```java
em.find(...)
```

Сущность находится под управлением Persistence Context.

---

## Detached

После:

```java
em.detach(user);
```

или

```java
em.clear();
```

или закрытия EntityManager.

Изменения больше не отслеживаются. :contentReference[oaicite:3]{index=3}

---

## Removed

После:

```java
em.remove(user);
```

Будет удалена при flush/commit.

---

# Что происходит при save()

## Новый объект

```java
User user = new User();

repository.save(user);
```

Внутри:

```java
entityManager.persist(user);
```

---

## Существующий объект

```java
repository.save(existingUser);
```

Обычно вызывается:

```java
entityManager.merge(user);
```

---

# persist() vs merge()

## persist()

```java
em.persist(user);
```

- Делает объект Managed
- Предназначен для новых сущностей
- Возвращает тот же объект

---

## merge()

```java
User managed = em.merge(user);
```

- Создает/находит Managed-копию
- Копирует состояние
- Возвращает другой объект

После merge работать нужно именно с возвращенным объектом. :contentReference[oaicite:4]{index=4}

---

# Что происходит при find()

```java
User user = em.find(User.class, 1L);
```

Алгоритм:

1. Проверка Persistence Context.
2. Проверка L2 Cache (если включен).
3. SQL SELECT.
4. Создание объекта.
5. Помещение в Persistence Context.
6. Возврат объекта. :contentReference[oaicite:5]{index=5}

---

# Dirty Checking

Одна из главных магий Hibernate.

```java
@Transactional
public void changeName() {

    User user = repository.findById(1L)
                          .orElseThrow();

    user.setName("Alex");
}
```

Нет вызова save().

Но Hibernate выполнит:

```sql
update users
set name='Alex'
where id=1;
```

при commit транзакции. :contentReference[oaicite:6]{index=6}

---

# Flush

## Что делает

Flush синхронизирует Persistence Context с БД.

```java
em.flush();
```

Он:

- генерирует SQL
- отправляет SQL в БД

Но транзакция может еще не быть закоммичена. :contentReference[oaicite:7]{index=7}

---

## Когда вызывается автоматически

### Commit

```java
transaction.commit();
```

### Перед некоторыми запросами

Чтобы результаты запроса учитывали текущие изменения.

---

# Что делает Hibernate внутри

## Snapshot

Когда сущность становится Managed:

```java
User user = em.find(...);
```

Hibernate сохраняет снимок состояния.

```text
name = Bob
age  = 25
```

---

## Изменение объекта

```java
user.setAge(26);
```

---

## При flush

Hibernate сравнивает:

```text
Было: 25
Стало: 26
```

Если есть отличия:

```sql
update user
set age = 26
where id = ?
```

Это и есть Dirty Checking. :contentReference[oaicite:8]{index=8}

---

# Транзакция целиком

```java
@Transactional
public void updateUser() {

    User user =
        repository.findById(1L)
                  .orElseThrow();

    user.setName("Alice");
}
```

Фактически происходит:

```text
BEGIN

SELECT ...

Persistence Context:
User(id=1, name=Bob)

Изменение объекта

Dirty Checking

UPDATE ...

COMMIT
```

---

# Когда SQL реально выполняется

Многие считают:

```java
repository.save(user);
```

сразу делает INSERT.

На практике часто происходит:

```text
persist()
    ↓
Persistence Context
    ↓
flush()
    ↓
SQL
    ↓
commit
```

Hibernate активно использует write-behind стратегию. :contentReference[oaicite:9]{index=9}

---

# Lazy Loading

## Идея

```java
@ManyToOne(fetch = LAZY)
private Department department;
```

Сначала загружается только User.

---

При обращении:

```java
user.getDepartment();
```

Hibernate может выполнить дополнительный SQL.

---

Для этого используются Proxy-объекты.

---

# Частые ошибки

## 1. LazyInitializationException

```java
User user = repository.findById(id);

return user.getDepartment().getName();
```

Если Persistence Context уже закрыт — будет исключение.

---

## 2. Работа с Detached Entity

```java
User user = em.find(...);

em.detach(user);

user.setName("New");
```

Изменения не сохранятся. :contentReference[oaicite:10]{index=10}

---

## 3. Лишний save()

Часто встречается:

```java
user.setName("Alex");

repository.save(user);
```

Если объект Managed внутри транзакции, save обычно не нужен.

Dirty Checking всё сделает сам. :contentReference[oaicite:11]{index=11}

---

# Что любят спрашивать на собеседованиях

## Базовые вопросы

### Что такое JPA?

Спецификация ORM для Java.

---

### Что такое Hibernate?

Самая популярная реализация JPA.

---

### Что такое EntityManager?

Интерфейс управления Persistence Context.

---

### Что такое Persistence Context?

Контекст управляемых сущностей, содержащий L1 Cache и механизм Dirty Checking.

---

### Чем отличается JPA от Hibernate?

JPA — спецификация.

Hibernate — реализация.

---

# Вопросы среднего уровня

### Что такое Dirty Checking?

Автоматическое обнаружение изменений Managed Entity и генерация UPDATE.

---

### Что делает flush()?

Синхронизирует Persistence Context с БД.

---

### Когда вызывается flush() автоматически?

- commit
- перед некоторыми запросами

---

### Чем persist отличается от merge?

| persist | merge |
|----------|--------|
| Для новых сущностей | Для detached/new |
| Возвращает тот же объект | Возвращает managed копию |
| Делает объект Managed | Копирует состояние |

---

### Какие состояния Entity существуют?

- Transient
- Managed
- Detached
- Removed

---

### Что такое L1 Cache?

Persistence Context.

---

### Что такое L2 Cache?

Кэш уровня SessionFactory/EntityManagerFactory, разделяемый между сессиями.

---

# Вопросы Senior уровня

### Почему save() часто не нужен внутри @Transactional?

Потому что Managed Entity отслеживается через Dirty Checking.

---

### Почему find() дважды может не делать SQL?

Сработает L1 Cache.

---

### Что будет если изменить Detached Entity?

Изменения не попадут в БД.

---

### Почему merge может быть дорогим?

Потому что Hibernate:

1. Ищет сущность.
2. Создает Managed экземпляр.
3. Копирует состояние.
4. Выполняет Dirty Checking.

---

### Что такое transactional write-behind?

Изменения накапливаются в Persistence Context и отправляются в БД только при flush.

---

# Краткое резюме

Запомнить цепочку:

```text
Repository
    ↓
EntityManager
    ↓
Persistence Context
    ↓
Dirty Checking
    ↓
Flush
    ↓
Hibernate
    ↓
JDBC
    ↓
Database
```

Для большинства вопросов на собеседовании по JPA понимание:

- Persistence Context
- Entity Lifecycle
- Dirty Checking
- Flush
- persist vs merge

Любимый вопрос на Senior-собеседовании

Что вернет: (true)
```java
@Transactional
public void test() {

    User u1 = em.find(User.class, 1L);
    User u2 = em.find(User.class, 1L);

    System.out.println(u1 == u2);
}
```

Ответ: true

А если:
```java
User u1 = service1.load();
User u2 = service2.load();
```
где каждый вызов открыл свою транзакцию, то: u1 == u2 скорее всего: false

