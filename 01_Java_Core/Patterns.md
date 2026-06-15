# 🧠 Patterns
#java-core #patterns

### 🎯 Главный инсайт (TL;DR)
> Паттерны

### ⚙️ 

#### 1. Декоратор
1.  **Основная цель: Расширение поведения объекта на лету без изменения его кода.
2.  **Что на входе?: Уже существующий готовый объект.
3.  **Что на выходе?: Тот же самый интерфейс, но с новой оберткой.
4.  **Жизненный цикл: Используется во время работы программы (Runtime).
5.  **Структура классов: классовКласс-декоратор реализует тот же интерфейс, что и объект, который он оборачивает.
6.  **Аналогия: Одежда. Вы можете надеть куртку, поверх нее — дождевик, а сверху — светоотражающий жилет.

**Главный нюанс синтаксиса Java - цепочка декораторов визуально выглядит как классическая вложенность:
```java
Notifier fullNotifier = new LoggingDecorator(
                            new TelegramDecorator(
                                new SmsNotifier()
                            )
                        );
```

Или так:
```java
// У нас уже есть базовый объект
IPizza myPizza = new TomatoPizza(); 

// Мы оборачиваем его в декораторы (матрешка), которые меняют поведение
myPizza = new CheeseDecorator(myPizza);   // Добавили сыр, цена выросла
myPizza = new MushroomDecorator(myPizza); // Добавили грибы, цена выросла еще

// Интерфейс остался прежним, но метод GetCost() теперь считает сумму всей цепочки
double finalPrice = myPizza.GetCost();
```java

#### 2.Строитель (Builder)
Определяют, как GC будет взаимодействовать с объектом:
1.  **Основная цель:  Пошаговое создание сложного объекта или конфигурации.
2.  **Что на входе?:  Набор параметров, конфигурационные данные.
3.  **Что на выходе?: Новой оберткой.Новый, полностью собранный целевой объект.
4.  **Жизненный цикл: Используется один раз в момент инициализации объекта.
5.  **Структура классов: Строитель имеет свои уникальные методы настройки, отличные от целевого объекта.

// Builder пошагово собирает свойства будущего объекта
```java
Pizza pizza = new PizzaBuilder() // Fluent API 
    .SetSize(30)
    .SetDough("Thick")
    .SetSauce("Tomato")
    .Build(); // Метод Build() возвращает саму Пиццу, а не Builder
```

### 💣Шпаргалка: 
Используйте **Builder**, если у вас есть класс со слишком большим конструктором (10+ параметров) или вам нужно создавать разные вариации одного и того же объекта.
Используйте **Decorator**, если у вас есть готовый класс (например, отправка уведомлений SmsSender), и вы хотите добавить к нему логирование (LogSender) или шифрование (CryptoSender), 
не переписывая исходный класс и не создавая под каждый случай новый подкласс (CryptoLogSmsSender).
Декоратор Используется во время работы программы (Runtime).
Строитель Используется один раз в момент инициализации объекта.

#### 3. Стратегия

```java
// 1. Интерфейс стратегии
@FunctionalInterface
interface PaymentStrategy {
    void pay(int amount);
}

// 2. Конкретные реализации стратегий
class CreditCardStrategy implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        System.out.println("Оплачено " + amount + " руб. с помощью Credit Card.");
    }
}

class PayPalStrategy implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        System.out.println("Оплачено " + amount + " руб. с помощью PayPal.");
    }
}

// 3. Контекст (Класс, который использует стратегию)
class Order {
    private PaymentStrategy paymentStrategy;

    // Метод для динамической смены стратегии
    public void setPaymentStrategy(PaymentStrategy paymentStrategy) {
        this.paymentStrategy = paymentStrategy;
    }
    public void checkout(int total) {
        if (paymentStrategy == null) {
            throw new IllegalStateException("Стратегия оплаты не задана!");
        }
        paymentStrategy.pay(total);
    }
}

// Использование:
public class Main {
    public static void main(String[] args) {
        Order order = new Order();

        // Вариант А: Передаем обычный класс
        order.setPaymentStrategy(new CreditCardStrategy());
        order.checkout(1500);

        // Вариант Б: Меняем стратегию на лету
        order.setPaymentStrategy(new PayPalStrategy());
        order.checkout(2300);
        order.checkout(500);
    }
}
```

### 💣 Шпаргалка для выбора
Выбирайте **Стратегию**, если у вас есть один объект, но алгоритм его работы должен кардинально меняться в зависимости от условий (например, разные алгоритмы сжатия файлов Zip, Rar, Tar). 
Класс знает: «Я делаю это вот таким способом».
Выбирайте **Декоратор**, если у вас есть базовое поведение, и вы хотите прозрачно для остального кода добавлять к нему новые «фичи» или обязанности (например, логирование, кэширование, шифрование данных перед записью). 
Класс внутри не меняется, но его возможности растут за счет внешних оберток.

### 💣 Ловушки на собеседованиях (Interview Traps)


### 🔗 Связанные темы (Links):
*   [[Java]]
