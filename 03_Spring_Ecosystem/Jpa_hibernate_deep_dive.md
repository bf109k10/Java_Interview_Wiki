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

### Дополнение в Файл 2: `03_Spring_Ecosystem/02_spring_data_jdbc.md`
*(Вставь этот блок в раздел про связи между таблицами)*

```markdown
### 5. Паттерн AggregateReference (Реализация связей без боли)

Так как в Spring Data JDBC нет управляемых прокси и ленивой загрузки, классические связи `@ManyToOne` привели бы к тому, что при загрузке одной сущности из базы вытягивался бы весь интернет. 

Чтобы связать сущности из **разных** агрегатов, используется `AggregateReference`.

#### Пример: Связь «Заказ» (Order) и «Покупатель» (Customer)
Покупатель и Заказ — это два разных бизнес-контекста (два разных Aggregate Root). Заказ должен знать, чей он, но не должен тянуть за собой весь объект покупателя с его адресами и историей.

```java
// Агрегат Покупателя
@Table("customers")
public class Customer {
    @Id Long id;
    String name;
}

// Агрегат Заказа
@Table("orders")
public class Order {
    @Id Long id;
    LocalDateTime creationDate;
    
    // Вместо объекта Customer используем обертку над его ID
    private AggregateReference<Customer, Long> customerId; 
    
    public void setCustomer(Customer customer) {
        this.customerId = AggregateReference.to(customer.getId());
    }
}
```

**Как это работает на практике:**
1. При сохранении заказа Spring Data JDBC просто вставит `customer_id` в таблицу `orders`.
2. Если вам при обработке заказа понадобятся данные покупателя, вы берете `order.getCustomerId().getId()` и идете в `CustomerRepository.findById(...)`. 
3. Это обеспечивает **полную изоляцию микросервисов** и чистоту архитектуры.
```

---

### Дополнение в Файл 3: `03_Spring_Ecosystem/03_jpa_vs_jdbc_comparison.md`
*(Вставь этот блок в самый конец как секцию для закрепления)*

```markdown
### 4. Инженерный блиц-опрос (Senior level)

* **Вопрос**: В JPA мы можем написать `user.setName("Alex")` и забыть. В Spring Data JDBC нам обязательно вызывать `repo.save(user)`. Почему это кардинально меняет подход к написанию юнит-тестов?
* **Ответ**: В JPA тесты бизнес-логики часто требуют `@DataJpaTest` или мока `EntityManager`, потому что изменения происходят неявно в контексте. В Spring Data JDBC методы сервисов становятся "чистыми функциями": мы можем протестировать всю логику обычными JUnit тестами без поднятия базы данных, просто проверяя, был ли в конце вызван метод `.save()` с нужными параметрами.

* **Вопрос**: Как ведет себя первичный ключ (`@Id`) при генерации стратегий?
* **Ответ**: Hibernate (JPA) умеет работать со стратегией `GenerationType.SEQUENCE` эффективно, забирая пачку ID из базы заранее (оптимизация `pooled-lo`). Spring Data JDBC при генерации ID полагается исключительно на механизмы самой БД (например, `IDENTITY` автоинкремент) или требует ручной настройки `BeforeConvertCallback`, что делает генерацию ID в коде чуть менее гибкой из коробки.
```

---

Теперь твоя Wiki-база по Spring Data полностью укомплектована боевыми примерами. 

Куда двинемся дальше в рамках подготовки к интервью? Можем разобрать:
* Как работает **механизм проксирования в Spring** (`JDK Dynamic Proxy` vs `CGLIB`), который стоит за `@Transactional`?
* Как правильно готовить **индексы и транзакции** (уровни изоляции `Isolation` и `Propagation`)? 
* Или перейдем к вопросам по **Spring Security / Spring Cloud**?
