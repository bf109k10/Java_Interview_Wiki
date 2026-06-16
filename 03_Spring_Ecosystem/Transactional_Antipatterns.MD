# Типичные ошибки и анти-паттерны при использовании `@Transactional`
#transactional #spring
> **Источник**: [proselyte.net/transactional-annotation/](https://proselyte.net/transactional-annotation/)  
---

## 📌 Введение

Аннотация `@Transactional` упрощает управление транзакциями, но неправильное её использование может привести к **скрытым багам**, **проблемам с производительностью** и **непредсказуемому поведению**. В этом документе собраны типичные ошибки и анти-паттерны, которые стоит избегать.

---

## ❌ Анти-паттерны и ошибки

### 1. **Использование `@Transactional` на приватных методах**

- **Проблема**:  
Spring использует **прокси** для управления транзакциями. Прокси не может перехватить вызов приватного метода, поэтому транзакция **не будет создана**.
- **Пример ошибки**:
  ```java
  @Service
  public class UserService {
      @Transactional
      private void updateUser(Long userId, String name) {
          // Транзакция НЕ будет создана!
      }
  }
  ```
- **Решение**:  
Используйте `public` методы или переместите логику в отдельный публичный класс.

---

### 2. **Self-Invocation (Самовозов)**

- **Проблема**:  
Если метод с `@Transactional` вызывается из другого метода **того же класса**, транзакция **не сработает** (прокси не может перехватить внутренний вызов).
- **Пример ошибки**:
  ```java
  @Service
  public class UserService {
      @Transactional
      public void updateUser(Long userId, String name) {
          // Логика обновления
      }

      public void processUser(Long userId, String name) {
          updateUser(userId, name); // Транзакция НЕ будет создана!
      }
  }
  ```
- **Решение**:
  - Используйте `AopContext.currentProxy()`:
    ```java
    @Service
    public class UserService {
        @Transactional
        public void updateUser(Long userId, String name) {
            // Логика обновления
        }

        public void processUser(Long userId, String name) {
            ((UserService) AopContext.currentProxy()).updateUser(userId, name);
        }
    }
    ```
  - Вынесите вызов в другой класс.

---

### 3. **Длинные транзакции**

- **Проблема**:  
Длинные транзакции (например, с обращением к внешним API или долгими вычислениями) **блокируют ресурсы** (соединения с БД, строки в таблицах), что приводит к:
  - **Timeoutам** других транзакций.
  - **Deadlockам** (взаимным блокировкам).
  - **Снижению производительности** системы.
- **Пример ошибки**:
  ```java
  @Transactional
  public void processOrder(Order order) {
      // Сохранение заказа в БД
      orderRepository.save(order);
      
      // Обращение к внешнему API (долго)
      externalApi.call(); // Блокирует транзакцию!
      
      // Обновление статуса заказа
      order.setStatus("COMPLETED");
  }
  ```
- **Решение**:
  - Разделите операцию на несколько транзакций.
  - Вынесите долгие операции за пределы транзакции.
  - Используйте асинхронную обработку (например, `@Async`).

---

### 4. **Неправильное использование `propagation**`

- **Проблема**:  
Неправильный выбор типа `propagation` может привести к **неожиданному поведению** транзакций.

  | Анти-паттерн                                                                 | Проблема                                                                   | Решение                                                |
  | ---------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------ |
  | Использование `REQUIRED` для операций, которые должны быть изолированы.      | Если внешняя транзакция откатывается, откатываются все вложенные операции. | Используйте `REQUIRES_NEW` для критических операций.   |
  | Использование `REQUIRES_NEW` без необходимости.                              | Создаёт избыточные транзакции, что снижает производительность.             | Используйте `REQUIRED` по умолчанию.                   |
  | Использование `NESTED` в СУБД, которые не поддерживают вложенные транзакции. | Может привести к ошибкам.                                                  | Проверьте поддержку вложенных транзакций в вашей СУБД. |

- **Пример ошибки**:
  ```java
  @Transactional(propagation = Propagation.REQUIRED)
  public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
      // Списание денег
      accountRepository.withdraw(fromId, amount);
      
      // Зачисление денег (если что-то пойдёт не так, списание откатится)
      accountRepository.deposit(toId, amount);
  }
  ```
  Если `deposit` выбросит исключение, **всё списание откатится**, даже если оно было успешным.
- **Решение**:  
Используйте `REQUIRES_NEW` для операций, которые должны быть атомарными:
  ```java
  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void deposit(Long toId, BigDecimal amount) {
      accountRepository.deposit(toId, amount);
  }
  ```

---

### 5. **Игнорирование `checked` исключений**

- **Проблема**:  
По умолчанию `@Transactional` откатывает транзакцию только для **unchecked исключений** (`RuntimeException` и `Error`).  
**Checked исключения** (например, `IOException`, `SQLException`) **не вызывают откат**.
- **Пример ошибки**:
  ```java
  @Transactional
  public void importData() throws IOException {
      // Чтение файла
      File file = new File("data.csv");
      if (!file.exists()) {
          throw new IOException("Файл не найден");
      }
      // Сохранение данных в БД
      dataRepository.saveAll(data);
      // Если файл не найден, транзакция НЕ откатится!
  }
  ```
- **Решение**:  
Явно укажите `rollbackFor`:
  ```java
  @Transactional(rollbackFor = IOException.class)
  public void importData() throws IOException {
      // Логика импорта
  }
  ```

---

### 6. **Использование `@Transactional` на методах, которые не изменяют данные**

- **Проблема**:  
Если метод только **читает данные**, но помечен как `@Transactional`, это:
  - Создаёт ненужную транзакцию.
  - Может блокировать ресурсы (например, строки в БД).
  - Снижает производительность.
- **Пример ошибки**:
  ```java
  @Transactional
  public User getUserById(Long id) {
      return userRepository.findById(id).orElseThrow();
  }
  ```
- **Решение**:
  - Уберите `@Transactional` для методов только чтения.
  - Если нужно гарантировать консистентность чтения, используйте `readOnly = true`:
    ```java
    @Transactional(readOnly = true)
    public User getUserById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
    ```

---

### 7. **Неправильное использование `readOnly**`

- **Проблема**:  
Атрибут `readOnly = true` **не делает транзакцию действительно только для чтения** в некоторых СУБД (например, PostgreSQL).  
Он лишь подсказывает Spring и Hibernate оптимизировать работу (например, не открывать транзакцию для записи).
  **Опасность**:
  - Если в методе с `readOnly = true` попробовать изменить данные, это может привести к **исключениям** или **непредсказуемому поведению**.
- **Пример ошибки**:
  ```java
  @Transactional(readOnly = true)
  public void updateUserName(Long id, String name) {
      User user = userRepository.findById(id).orElseThrow();
      user.setName(name); // Может выбросить исключение!
      userRepository.save(user);
  }
  ```
- **Решение**:  
Не используйте `readOnly = true` для методов, которые изменяют данные.

---

### 8. **Отсутствие обработки исключений**

- **Проблема**:  
Если метод с `@Transactional` выбрасывает исключение, но оно **не обрабатывается**, транзакция откатится, но пользователь не узнает о причине ошибки.
- **Пример ошибки**:
  ```java
  @Transactional
  public void createUser(User user) {
      if (userRepository.existsByEmail(user.getEmail())) {
          throw new RuntimeException("Email уже существует");
      }
      userRepository.save(user);
  }
  ```
  Если email уже существует, транзакция откатится, но сообщение об ошибке может быть неясным.
- **Решение**:  
Используйте кастомные исключения и обрабатывайте их:
  ```java
  @Transactional
  public void createUser(User user) throws UserAlreadyExistsException {
      if (userRepository.existsByEmail(user.getEmail())) {
          throw new UserAlreadyExistsException("Email уже существует");
      }
      userRepository.save(user);
  }
  ```

---

### 9. **Использование `@Transactional` на интерфейсах**

- **Проблема**:  
Если `@Transactional` ставится на **интерфейсе**, а не на классе, это может привести к:
  - **Непредсказуемому поведению** при изменении реализации.
  - **Проблемам с проксированием** (если класс не реализует интерфейс).
- **Пример ошибки**:
  ```java
  public interface UserService {
      @Transactional
      void updateUser(Long id, String name);
  }

  @Service
  public class UserServiceImpl implements UserService {
      @Override
      public void updateUser(Long id, String name) {
          // Логика обновления
      }
  }
  ```
- **Решение**:  
Ставьте `@Transactional` на **классе реализации**, а не на интерфейсе.

---

### 10. **Неучёт уровней изоляции**

- **Проблема**:  
Использование **неподходящего уровня изоляции** может привести к:
  - **Грязному чтению** (`READ UNCOMMITTED`).
  - **Неповторяемому чтению** (`READ COMMITTED`).
  - **Фантомному чтению** (`REPEATABLE READ`).
- **Пример ошибки**:
  ```java
  @Transactional(isolation = Isolation.READ_UNCOMMITTED)
  public BigDecimal getAccountBalance(Long accountId) {
      return accountRepository.findBalanceById(accountId);
  }
  ```
  Если другая транзакция изменит баланс, но не закоммитит, метод может вернуть неактуальные данные.
- **Решение**:  
Используйте подходящий уровень изоляции:
  - `READ COMMITTED` — для большинства случаев.
  - `REPEATABLE READ` — если нужно избежать неповторяемого чтения.
  - `SERIALIZABLE` — для критических операций (например, финансовые транзакции).

---

## 💡 Рекомендации для интервью

1. **Знайте анти-паттерны**:
  - Self-invocation.
  - Длинные транзакции.
  - Неправильное использование `propagation` и `rollbackFor`.
2. **Объясните последствия**:
  - Почему приватные методы с `@Transactional` не работают.
  - Как длинные транзакции влияют на производительность.
3. **Приведите примеры**:
  - Покажите, как правильно использовать `REQUIRES_NEW` для критических операций.
  - Объясните, почему `readOnly = true` не делает транзакцию действительно только для чтения.
4. **Практика**:
  - Напишите код с правильной обработкой исключений.
  - Покажите, как избежать self-invocation.

---

## 🔗 Полезные ссылки

- [Оригинальная статья на Proselyte](https://proselyte.net/transactional-annotation/)
- [Spring Transaction Management Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction)
- [Common Pitfalls with `@Transactional](https://www.baeldung.com/transaction-configuration-with-jpa-and-spring)`
