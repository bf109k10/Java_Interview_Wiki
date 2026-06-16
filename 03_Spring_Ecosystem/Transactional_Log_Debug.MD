# Советы по отладке и логированию транзакций в Spring
#transactional #spring

> **Источник**: [proselyte.net/transactional-annotation/](https://proselyte.net/transactional-annotation/)  
---

## 📌 Введение

Отладка транзакций — это **ключевой навык** для разработчика, работающего с базами данных. Неправильно настроенные транзакции могут привести к **потере данных**, **неконсистентному состоянию БД** или **проблемам с производительностью**. В этом документе собраны советы по отладке и логированию транзакций в Spring.

---

## 🔍 Как включить логирование транзакций

### 1. **Логирование на уровне Spring**

Spring позволяет логировать **начало, коммит и откат** транзакций. Для этого нужно настроить логгер для пакета `org.springframework.transaction`.

**Пример для `logback.xml**`:

```xml
<logger name="org.springframework.transaction" level="TRACE" />
<logger name="org.springframework.transaction.interceptor" level="TRACE" />
```

**Пример для `application.properties` (Spring Boot)**:

```properties
logging.level.org.springframework.transaction=TRACE
logging.level.org.springframework.transaction.interceptor=TRACE
```

**Что будет логироваться**:

- Начало транзакции.
- Коммит или откат.
- Уровень изоляции и `propagation`.
- Время выполнения транзакции.

---

### 2. **Логирование SQL-запросов**

Чтобы видеть, какие SQL-запросы выполняются в рамках транзакции, включите логирование для **Hibernate** или **JDBC**:

**Для Hibernate (Spring Boot)**:

```properties
# Логирование SQL-запросов
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# Логирование параметров запросов
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

**Для JDBC**:

```properties
logging.level.org.springframework.jdbc.core=TRACE
logging.level.org.springframework.jdbc.core.JdbcTemplate=DEBUG
```

**Пример вывода**:

```
Hibernate: select user0_.id as id1_0_, user0_.name as name2_0_ from users user0_ where user0_.id=?
Hibernate: update users set name=? where id=?
```

---

## 🛠 Инструменты для отладки

### 1. **Spring Boot Actuator**

**Spring Boot Actuator** предоставляет **HTTP-эндпоинты** для мониторинга транзакций и других метрик.

**Настройка**:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

**Включение эндпоинтов в `application.properties**`:

```properties
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
```

**Полезные эндпоинты**:

- `/actuator/health` — статус приложения.
- `/actuator/metrics` — метрики, включая информацию о транзакциях.
- `/actuator/loggers` — управление уровнями логирования на лету.

---

### 2. **Hibernate Statistics**

Hibernate предоставляет **статистику** по транзакциям, запросам и кэшу.

**Включение статистики**:

```properties
spring.jpa.properties.hibernate.generate_statistics=true
```

**Получение статистики в коде**:

```java
@Autowired
private SessionFactory sessionFactory;

public void printStatistics() {
    Statistics stats = sessionFactory.getStatistics();
    System.out.println("Transaction count: " + stats.getTransactionCount());
    System.out.println("Commit count: " + stats.getCommitCount());
    System.out.println("Rollback count: " + stats.getRollbackCount());
    System.out.println("Query execution count: " + stats.getQueryExecutionCount());
}
```

---

### 3. **AOP для логирования транзакций**

Можно использовать **AOP** для логирования вызовов методов с `@Transactional`.

**Пример**:

```java
@Aspect
@Component
public class TransactionLoggingAspect {

    private static final Logger logger = LoggerFactory.getLogger(TransactionLoggingAspect.class);

    @Around("@annotation(transactional)")
    public Object logTransaction(ProceedingJoinPoint joinPoint, Transactional transactional) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        logger.info("Starting transaction for method: {}", methodName);
        
        try {
            Object result = joinPoint.proceed();
            logger.info("Transaction committed for method: {}", methodName);
            return result;
        } catch (Exception e) {
            logger.error("Transaction rolled back for method: {}. Reason: {}", methodName, e.getMessage());
            throw e;
        }
    }
}
```

**Что будет логироваться**:

- Начало транзакции.
- Успешный коммит.
- Откат с причиной.

---

### 4. **Использование `TransactionSynchronization**`

Spring позволяет **регистрировать слушатели** для событий транзакций (например, коммит или откат).

**Пример**:

```java
@Transactional
public void someMethod() {
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
        @Override
        public void afterCommit() {
            logger.info("Transaction committed!");
        }

        @Override
        public void afterCompletion(int status) {
            if (status == TransactionSynchronization.STATUS_ROLLED_BACK) {
                logger.error("Transaction rolled back!");
            }
        }
    });
    
    // Логика метода
}
```

---

## 🔥 Типичные проблемы и их диагностика


| Проблема                          | Симптомы                                                             | Как диагностировать                                                                                                       | Решение                                                                     |
| --------------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| **Транзакция не создаётся**       | Изменения в БД не сохраняются, нет отката при ошибках.               | Проверьте логи на наличие сообщений о начале транзакции. Убедитесь, что метод `public` и не вызывается из того же класса. | Используйте `public` методы, избегайте self-invocation.                     |
| **Транзакция не откатывается**    | Ошибки в методе не приводят к откату.                                | Проверьте, какие исключения выбрасываются. Убедитесь, что они входят в `rollbackFor`.                                     | Явно укажите `rollbackFor` для `checked` исключений.                        |
| **Длинные транзакции**            | Timeoutы, deadlockи, низкая производительность.                      | Логируйте время выполнения транзакций.                                                                                    | Разделите операцию на несколько транзакций.                                 |
| **Deadlock**                      | Транзакции висят и не завершаются.                                   | Проверьте логи на наличие `DeadlockLoserDataAccessException`.                                                             | Измените порядок блокировок, используйте `REQUIRES_NEW`.                    |
| **Неправильный уровень изоляции** | Грязное, неповторяемое или фантомное чтение.                         | Логируйте уровень изоляции, проверяйте данные в БД.                                                                       | Установите подходящий уровень изоляции.                                     |
| **Проблемы с проксированием**     | Транзакции не работают на приватных методах или при self-invocation. | Проверьте, что метод `public` и не вызывается из того же класса.                                                          | Используйте `AopContext.currentProxy()` или вынесите логику в другой класс. |


---

## 📝 Логирование кастомных событий

### 1. **Логирование параметров транзакций**

Можно логировать **параметры транзакций** (например, уровень изоляции, `propagation`, таймаут):

```java
@Transactional(isolation = Isolation.READ_COMMITTED, propagation = Propagation.REQUIRED)
public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
    logger.info(
        "Transaction started with isolation={}, propagation={}",
        Isolation.READ_COMMITTED,
        Propagation.REQUIRED
    );
    // Логика перевода
}
```

### 2. **Логирование времени выполнения**

Используйте **AOP** или **Spring AOP** для логирования времени выполнения транзакций:

```java
@Aspect
@Component
public class TransactionTimeLoggingAspect {

    private static final Logger logger = LoggerFactory.getLogger(TransactionTimeLoggingAspect.class);

    @Around("@annotation(transactional)")
    public Object logTransactionTime(ProceedingJoinPoint joinPoint, Transactional transactional) throws Throwable {
        long startTime = System.currentTimeMillis();
        String methodName = joinPoint.getSignature().getName();
        
        try {
            Object result = joinPoint.proceed();
            long duration = System.currentTimeMillis() - startTime;
            logger.info("Transaction for method {} completed in {} ms", methodName, duration);
            return result;
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - startTime;
            logger.error("Transaction for method {} failed after {} ms. Reason: {}", methodName, duration, e.getMessage());
            throw e;
        }
    }
}
```

---

## 💡 Советы для интервью

1. **Знайте инструменты**:
  - Как включить логирование транзакций в Spring.
  - Как использовать **Spring Boot Actuator** для мониторинга.
  - Как работать с **Hibernate Statistics**.
2. **Объясните подходы**:
  - Как логировать начало, коммит и откат транзакций.
  - Как диагностировать типичные проблемы (например, deadlock или отсутствие отката).
3. **Приведите примеры**:
  - Покажите, как настроить логирование SQL-запросов.
  - Напишите AOP-аспект для логирования транзакций.
4. **Практика**:
  - Объясните, как вы будете отлаживать проблему с транзакцией, которая не откатывается.
  - Расскажите, как вы будете искать причину deadlock.

---

## 🔗 Полезные ссылки

- [Оригинальная статья на Proselyte](https://proselyte.net/transactional-annotation/)
- [Spring Boot Actuator Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [Hibernate Statistics Documentation](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#statistics)
- [Spring AOP Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop)
- [Logging in Spring Boot](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.logging)
