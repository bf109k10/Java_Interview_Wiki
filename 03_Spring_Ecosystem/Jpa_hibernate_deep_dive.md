# Spring Data JPA и Hibernate

### 1. Архитектурный слой (Цепочка вызова)
Когда вы вызываете метод репозитория (например, `userRepository.findById(1L)`), запрос проходит следующий путь:
1. **Spring Data JPA Proxy**: Перехватывает вызов. Находит нужную реализацию в `SimpleJpaRepository`.
2. **Jakarta Persistence API (JPA) / EntityManager**: Служит стандартным интерфейсом (спецификацией). Обращается к текущему контексту постоянства (Persistence Context).
3. **Hibernate (ORM Provider)**: Реализует спецификацию JPA. Генерирует SQL-запрос на основе маппинга сущностей (`@Entity`) и метаданных.
4. **JDBC Driver**: Передает сгенерированный SQL-запрос непосредственно в базу данных через сетевое соединение и возвращает `ResultSet`.

---

### 2. Жизненный цикл сущностей (Entity States)
Вся магия JPA держится на **Persistence Context** (контексте постоянства), которым управляет `EntityManager`. Объект может находиться в одном из 4 состояний:

* **Transient (Временный)**
  * Новый объект в куче (heap), созданный через `new`.
  * Нет ID (первичного ключа), Hibernate о нем ничего не знает.
* **Managed / Persistent (Управляемый)**
  * Объект привязан к текущей сессии Hibernate и имеет ID.
  * Любые изменения полей объекта в Java **автоматически** запишутся в БД при коммите транзакции. Метод `save()` вызывать не нужно! Это называется **Dirty Checking** (проверка "грязных" объектов).
  * Попадает сюда через `repository.save()`, `entityManager.persist()` или при выгрузке из БД через `findById()`.
* **Detached (Отсоединенный)**
  * Объект имеет ID и есть в БД, но его сессия закрылась (транзакция завершена).
  * Изменения полей в Java больше не отслеживаются. Чтобы вернуть его в Managed, нужно вызвать `repository.save()` (под капотом выполнится `entityManager.merge()`).
* **Removed (Удаленный)**
  * Объект помечен на удаление через `repository.delete()`. Физический `DELETE` в БД уйдет только в момент вызова `flush` / коммита транзакции.

---


### 2.1 Вот если работает приложение, допустим монолит. В нем может быть несколько Persistence Context ? 
Persistence Context существует в рамках чего - операции, транзакции, класса?

Да, в работающем монолите почти всегда существует множество Persistence Context одновременно.
Ключевой момент: **Persistence Context принадлежит EntityManager'у**.
Спецификация JPA определяет это буквально как:
Один EntityManager ↔ один Persistence Context.

**В типичном Spring Boot приложении**
```java
@PersistenceContext
private EntityManager em;
```
**Что происходит при @Transactional**
```java
@Transactional
public void transferMoney() {
    ...
}
```
Spring:
Открывает транзакцию
      ↓
Создает EntityManager
      ↓
Создает Persistence Context
      ↓
Выполняет код
      ↓
flush
      ↓
commit
      ↓
закрывает EntityManager

То есть в большинстве Spring-приложений: транзакция = EntityManager = EntityManagerы

**Если 100 запросов одновременно**
и каждый попадает в свой @Transactional. Получаем
Tx1
 └─ EM1
     └─ PC1

Tx2
 └─ EM2
     └─ PC2

Tx3
 └─ EM3
     └─ PC3
То есть одновременно живут десятки или сотни Persistence Context. Они никак не пересекаются.

**Transaction-scoped Persistence Context**
Это режим по умолчанию в Spring. Жизненный цикл:
Начало транзакции
      ↓
Создание PC
      ↓
Работа
      ↓
Commit/Rollback
      ↓
Уничтожение PC

**Extended Persistence Context** Есть еще менее распространенный режим: Persistence Context живет дольше транзакции.
```java
@PersistenceContext(type = PersistenceContextType.EXTENDED)
```
Web Flow
      ↓
Persistence Context
      ↓
Tx1
Tx2
Tx3
Он может переживать несколько транзакций. Чаще встречался в старых JSF/EJB приложениях. В Spring Boot почти не используется.


### 3. Топ вопросов на интервью по JPA
* **Что такое Dirty Checking?** Механизм Hibernate, который при коммите транзакции сравнивает текущее состояние объекта с его исходным снимком (snapshot), сделанным при загрузке. Если есть изменения, Hibernate сам генерирует `UPDATE`.
* **В чем разница между `persist()` и `merge()`?** `persist()` берет Transient объект, делает его Managed (если у него уже есть ID, будет исключение). `merge()` берет Detached объект, копирует его состояние в *новый* Managed объект, который достает из базы (или создает), и возвращает этот новый объект.
* **Как поймать LazyInitializationException?** Попытаться обратиться к ленивой коллекции (связи `@OneToMany(fetch = FetchType.LAZY)`) за пределами открытой транзакции (когда сессия Hibernate уже закрыта, а объект стал Detached).

### 4. Практические кейсы: Dirty Checking и LazyInitializationException

#### Кейс А: Демонстрация Dirty Checking (Магия Managed-состояния)
Обрати внимание: в коде ниже **нет** вызова `userRepository.save(user)`.

```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    @Transactional // 1. Открывается транзакция и сессия Hibernate
    public void updateUserEmail(Long userId, String newEmail) {
        // 2. Объект загружается из БД и становится MANAGED
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new EntityNotFoundException();

        // 3. Меняем поле в Java-объекте
        user.setEmail(newEmail); 
        
        // К концу метода транзакция завершается (commit).
        // Hibernate сравнивает объект со snapshot'ом, видит изменение 
        // и САМ выполняет: UPDATE users SET email = '...' WHERE id = ...
    } 
}
```

**Что такое Persistence Context?**
Persistence Context — это набор управляемых сущностей, связанный с EntityManager. Он реализует First-Level Cache, 
обеспечивает Identity Map и используется Hibernate для Dirty Checking и управления жизненным циклом Entity.


**L2 Cache:**
EntityManager #1
        │
EntityManager #2
        │
EntityManager #3
        ▼
   Shared Cache
Общий для всех EntityManager одного EntityManagerFactory.
Например через:

Hibernate Second Level Cache
Ehcache
Caffeine
Infinispan

**Как Hibernate “думает”**
1. Ты загрузил объект
2. Он его запомнил
3. Ты поменял поля
4. Он сам понял, что изменилось
5. Он сам сгенерировал SQL
6. Он сам отправил в БД

**Dirty Checking (грязная проверка)** — это механизм Hibernate/JPA, который автоматически определяет, изменился ли объект-сущность, и при необходимости генерирует UPDATE без вызова save().
Dirty Checking — это процесс, при котором Hibernate сравнивает текущее состояние Entity с её исходным состоянием и сам решает, нужно ли обновлять БД.
Работает только внутри Persistence Context
Основан на сравнении snapshot vs current state
Триггерится на flush/commit

**Hibernate**
Hibernate — это Java-фреймворк, который реализует ORM (Object-Relational Mapping), то есть связывает Java-объекты с таблицами в базе данных.
Hibernate — это ORM-движок, который автоматически переводит операции с Java-объектами в SQL и обратно.
Где он находится в архитектуре
Java Code
   ↓
Spring Data JPA
   ↓
EntityManager (JPA API)
   ↓
Hibernate (реализация JPA)
   ↓
JDBC
   ↓
Database
Hibernate — это ORM-фреймворк для Java, который автоматически отображает объекты в реляционные таблицы, управляет их жизненным циклом, выполняет SQL-генерацию, кеширование, lazy loading и dirty checking.


L1 cache живет в EntityManager L2 cache живет в SessionFactory
1. L1 cache (EntityManager)
2. L2 cache (SessionFactory)
3. Database

**L2 cache (Second-Level Cache) в Hibernate** — это общий кэш между разными Persistence Context, 
который живёт дольше транзакции и помогает уменьшить количество обращений к базе. 
L2 cache в Hibernate — это опциональный кэш уровня SessionFactory, который хранит состояние сущностей между транзакциями и снижает количество обращений к базе данных, но не гарантирует одинаковые Java-объекты.

**Где он находится в архитектуре**
Request 1        Request 2
   ↓                ↓
EntityManager   EntityManager
   ↓                ↓
 L1 Cache        L1 Cache
      \            /
       \          /
        ↓        ↓
      L2 Cache (общий)
           ↓
        Database

**Что именно кэшируется**

В L2 cache хранятся:
Entity по ID
иногда ассоциации (collections)
иногда query results (если включено)

User u1 = em1.find(User.class, 1L);

Что происходит:

Проверка L1 cache (пусто)
Проверка L2 cache (пусто)
SQL в БД
Результат кладётся:
в L1 cache
в L2 cache

**Где хранится L2 cache**
Hibernate сам не хранит данные.
Он использует провайдеры:

Ehcache
Infinispan
Caffeine
Hazelcast
Redis (через интеграции)

**Когда L2 cache полезен**
Хорошие кейсы:
часто читаемые справочники
страны
валюты
роли
редко изменяемые данные
read-heavy системы

**Плохие кейсы:**
часто обновляемые сущности
high-frequency trading / real-time данные
большие write-heavy системы

**Важный момент (очень любят на собеседованиях)**
L2 cache хранит НЕ Java объект. Он хранит состояние сущности
L1 cache = готовая чашка кофе, L2 cache = рецепт кофе
L2 cache хранит не Java объекты, а сериализованное состояние сущностей, поэтому при получении данных из L2 cache Hibernate всегда создаёт новый объект, и identity (==) не сохраняется.

**Как делают “общий L2 cache” в кластере**
Чтобы L2 cache работал между нодами, подключают distributed cache provider.
1. Hazelcast
2. Infinispan (очень популярно в JBoss/WildFly мире)
3. Ehcache + RMI/JMS replication (старый подход)
4. Redis (через интеграцию, не native Hibernate)

Но даже в distributed L2 cache есть проблема
Консистентность (главная боль)
Решение - инвалидирование

> **Cache invalidation problem** — это проблема обеспечения консистентности данных между базой данных и кэшами (L1/L2/distributed) в условиях распределённой системы, где обновления данных должны корректно отражаться во всех кэшированных копиях, иначе возникает риск устаревших данных (stale data).

**Почему в микросервисах L2 cache почти не любят**
Потому что:
много нод
сложная invalidation
риск stale data
сложность дебага

#### Кейс Б: Ловим и чиним LazyInitializationException
Эта ошибка происходит, когда вы пытаетесь загрузить ленивое поле (`LAZY`) у объекта в состоянии `DETACHED` (вне транзакции).

```java
// Сущность с ленивой загрузкой
@Entity
public class User {
    @Id @GeneratedValue int id;
    
    @OneToMany(fetch = FetchType.LAZY) // По умолчанию для @OneToMany
    private List<Order> orders;
}

// REST-Контроллер (Вне транзакции!)
@GetMapping("/users/{id}")
public UserDto getUser(@PathVariable Long id) {
    User user = userService.findById(id); // Метод сервиса @Transactional закрылся, объект DETACHED
    
    // БУМ! Исключение LazyInitializationException: 
    // "could not initialize proxy - no Session"
    int orderCount = user.getOrders().size(); 
    return new UserDto(user, orderCount);
}
```

**Как правильно починить (Использование `JOIN FETCH`):**
Вместо изменения `LAZY` на `EAGER` (что убьет производительность во всем приложении), пишем точечный запрос в репозитории:

```java
public interface UserRepository extends JpaRepository<User, Long> {
    
    // За один SQL-запрос через INNER JOIN вытягиваем и юзера, и его заказы
    @Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id = :id")
    Optional<User> findByIdWithOrders(@Param("id") Long id);
}
```
---
