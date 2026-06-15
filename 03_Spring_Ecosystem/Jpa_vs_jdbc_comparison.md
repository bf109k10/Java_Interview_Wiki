# Сравнение: Spring Data JPA (Hibernate) vs Spring Data JDBC
#jpa #jdbc #spring
Шпаргалка для быстрого ответа на архитектурные вопросы на собеседовании.

### 1. Сравнительная таблица

| Критерий | Spring Data JPA (Hibernate) | Spring Data JDBC |
| :--- | :--- | :--- |
| **Управление состояниями** | Сложное (Transient, Managed, Detached...) | Отсутствует (Объект = просто данные) |
| **Кэширование** | Да (L1 в рамках сессии, L2 на уровне приложения) | Нет (Каждый вызов — реальный запрос в БД) |
| **Загрузка данных** | Раздельная (Lazy / Eager загрузка связей) | Только Eager (Всё или ничего в рамках агрегата) |
| **Контроль над SQL** | Скрытый (Сложно предсказать точный SQL без логов) | Явный (Выполняется ровно то, что вызвано) |
| **Связи таблиц** | Любые (`@ManyToMany`, `@ManyToOne` и т.д.) | Ограничены концепцией Aggregate Root |
| **Идеально подходит для** | Сложных монолитов, ERP-систем, богатых моделей | Микросервисов, CQRS-архитектур, Serverless |

---

### 2. Главные компромиссы (Trade-offs)

* **Производительность vs Скорость разработки**:
  * **JPA** ускоряет написание CRUD для систем со 100+ таблицами и сложными связями, но требует высокой квалификации, чтобы не уронить базу запросами N+1.
  * **JDBC** требует больше явного проектирования и ручной обработки связей, но гарантирует стабильную, предсказуемую скорость работы.

* **Память и Прогрев (Cold Start)**:
  * **JPA** долго инициализируется и потребляет много памяти из-за метамодели Hibernate. Плохо подходит для Cloud Native / GraalVM приложений.
  * **JDBC** "взлетает" моментально, потребляет минимум ресурсов, идеален для масштабирования микросервисов в Kubernetes.

---

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

---

### 3. Шпаргалка: Что отвечать на вопрос «Что выберешь для нового проекта?»
1. *«Я выберу **Spring Data JDBC**, если мы пишем изолированный микросервис с простой доменной моделью, где важна предсказуемость SQL-запросов, высокая скорость запуска (например, в Serverless или GraalVM) и нет запутанных связей типа Many-to-Many между разными сущностями».*
2. *«Я выберу **Spring Data JPA**, если проект представляет собой крупный домен с тяжелыми бизнес-правилами, огромным количеством связанных сущностей, где критически важен механизм Dirty Checking и кэширование данных для уменьшения нагрузки при повторных запросах».*

### 4. Инженерный блиц-опрос (Senior level)

* **Вопрос**: В JPA мы можем написать `user.setName("Alex")` и забыть. В Spring Data JDBC нам обязательно вызывать `repo.save(user)`. Почему это кардинально меняет подход к написанию юнит-тестов?
* **Ответ**: В JPA тесты бизнес-логики часто требуют `@DataJpaTest` или мока `EntityManager`, потому что изменения происходят неявно в контексте. В Spring Data JDBC методы сервисов становятся "чистыми функциями": мы можем протестировать всю логику обычными JUnit тестами без поднятия базы данных, просто проверяя, был ли в конце вызван метод `.save()` с нужными параметрами.

* **Вопрос**: Как ведет себя первичный ключ (`@Id`) при генерации стратегий?
* **Ответ**: Hibernate (JPA) умеет работать со стратегией `GenerationType.SEQUENCE` эффективно, забирая пачку ID из базы заранее (оптимизация `pooled-lo`). Spring Data JDBC при генерации ID полагается исключительно на механизмы самой БД (например, `IDENTITY` автоинкремент) или требует ручной настройки `BeforeConvertCallback`, что делает генерацию ID в коде чуть менее гибкой из коробки.
```

### 🔗 Связанные темы (Links):
*   [[Spring_data_jdbc]]