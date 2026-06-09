# 🧵 LeetCode 1114: Print in Order
#leetcode #concurrency #latch #aqs #easy

*   **Ссылка на задачу:** [LeetCode #1114](https://leetcode.com)

### 🎯 Главный инсайт (TL;DR)
> Три разных потока вызывают методы `first()`, `second()` и `third()` параллельно. Мы не можем контролировать планировщик потоков ОС, но мы можем использовать легковесный примитив синхронизации **`CountDownLatch`** (работающий на базе `AQS`), чтобы заблокировать выполнение второго и третьего потока до тех пор, пока предыдущие не завершат работу.

### 💻 Оптимальное решение на Java
```java
import java.util.concurrent.CountDownLatch;

class Foo {
    // Две защелки для координации шагов
    private final CountDownLatch latch1;
    private final CountDownLatch latch2;

    public Foo() {
        // Инициализируем счетчик единицей. Поток заблокируется на await(),
        // пока счетчик не сбросится в 0 через countDown()
        this.latch1 = new CountDownLatch(1);
        this.latch2 = new CountDownLatch(1);
    }

    public void first(Runnable printFirst) throws InterruptedException {
        printFirst.run(); // Печатаем "first"
        latch1.countDown(); // Сбрасываем первую защелку (счетчик = 0)
    }

    public void second(Runnable printSecond) throws InterruptedException {
        latch1.await(); // Ждем, пока первый поток вызовет countDown()
        printSecond.run(); // Печатаем "second"
        latch2.countDown(); // Сбрасываем вторую защелку
    }

    public void third(Runnable printThird) throws InterruptedException {
        latch2.await(); // Ждем, пока второй поток вызовет countDown()
        printThird.run(); // Печатаем "third"
    }
}
```

### ⚙️ Архитектурные метрики (Complexity)
*   **Time Complexity:** $O(1)$ — каждый метод выполняется за фиксированное время. Потоки просто ждут сигнала.
*   **Space Complexity:** $O(1)$ — мы тратим фиксированную память на создание двух объектов `CountDownLatch`.

### 💣 Ловушка на собеседовании (Interview Trap)
*   ❌ **Вопрос:** «Почему бы просто не использовать ключевое слово `volatile` для флагов `int step = 1` вместо `CountDownLatch`? Ведь `volatile` гарантирует видимость переменной между потоками?»
*   ✅ **Ответ:** Использование `volatile` в цикле типа `while(step != 2) {}` (подход *Busy Waiting* / активное ожидание) — это грубейшая ошибка на High-Load проектах. Поток не засыпает, а непрерывно крутится в цикле, нагружая ядро CPU на 100% абсолютно пустой работой. `CountDownLatch` под капотом использует **AQS**, который при вызове `await()` эффективно **паркует поток** (`LockSupport.park()`), снимая его с процессора и освобождая ресурсы для бизнес-логики.

### 🔗 Связанные темы (Links):
*   [[Synchronizers_and_Locks]] — Как `CountDownLatch` использует `AQS` и `state` под капотом.
*   [[JMM_and_Volatile]] — Почему видимости флагов недостаточно для эффективной синхронизации.
*   [[MOC_LeetCode]]
