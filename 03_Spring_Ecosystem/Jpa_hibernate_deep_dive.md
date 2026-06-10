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
