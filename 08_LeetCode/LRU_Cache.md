# 🗄️ LeetCode 146: LRU Cache (Реализация кэша)

*   **Ссылка на задачу:** [LeetCode #146](https://leetcode.com)

### 🎯 Главный инсайт (TL;DR)
> Чтобы обеспечить операции `get` и `put` за строгое время **$O(1)$**, необходимо объединить две структуры данных: **`HashMap`** (дает быстрый поиск узла по ключу за $O(1)$) и кастомный **двунаправленный связный список (Doubly LinkedList)** (позволяет за $O(1)$ перемещать элементы в начало при обращении и удалять старые элементы с конца) [INDEX].

### 💻 Оптимальное решение на Java
```java
import java.util.HashMap;
import java.util.Map;

class LRUCache {
    // Внутренний класс для узла двунаправленного списка
    private static class Node {
        int key, value;
        Node prev, next;
        Node(int key, int value) {
            this.key = key;
            this.value = value;
        }
    }

    private final int capacity;
    private final Map<Integer, Node> map;
    // Фейковые граничные узлы (Голова и Хвост) для избавления от проверок на null
    private final Node head;
    private final Node tail;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new HashMap<>();
        this.head = new Node(0, 0);
        this.tail = new Node(0, 0);
        head.next = tail;
        tail.prev = head;
    }

    public int get(int key) {
        if (!map.containsKey(key)) {
            return -1;
        }
        Node node = map.get(key);
        moveToHead(node); // Элемент использован -> двигаем в начало списка
        return node.value;
    }

    public void put(int key, int value) {
        if (map.containsKey(key)) {
            Node node = map.get(key);
            node.value = value;
            moveToHead(node);
        } else {
            if (map.size() >= capacity) {
                // Вытеснение: удаляем самый старый элемент с конца списка (перед tail)
                Node lru = tail.prev;
                removeNode(lru);
                map.remove(lru.key);
            }
            Node newNode = new Node(key, value);
            addNode(newNode);
            map.put(key, newNode);
        }
    }

    // Вспомогательные операции со списком за O(1)
    private void addNode(Node node) { // Добавление сразу после фейковой головы
        node.next = head.next;
        node.next.prev = node;
        head.next = node;
        node.prev = head;
    }

    private void removeNode(Node node) { // Разрыв связей узла
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void moveToHead(Node node) {
        removeNode(node);
        addNode(node);
    }
}
```

### ⚙️ Архитектурные метрики (Complexity)
*   **Time Complexity:** $O(1)$ для `get` и `put`. Поиск в `HashMap` занимает $O(1)$, а переброска указателей в связном списке не зависит от количества элементов [INDEX].
*   **Space Complexity:** $O(C)$, где $C$ — емкость кэша (`capacity`). Мы храним в памяти строго ограниченное количество узлов.

### 💣 Ловушка на собеседовании (Interview Trap)
*   ❌ **Вопрос:** «В Java уже есть готовый класс `LinkedHashMap`. Если я просто отнаследуюсь от него и переопределю метод `removeEldestEntry()`, код займет 5 строк. Зачем мне писать двунаправленный список вручную на собеседовании?»
*   ✅ **Ответ:** С точки зрения production-кода использовать `LinkedHashMap` — это абсолютно правильный выбор. Но на секции алгоритмов интервьюер проверяет ваше понимание работы с памятью и указателями. Если вы используете готовый класс, вы не показываете, что понимаете, *почему* там нужен именно двунаправленный список (а не однонаправленный, где удаление с конца заняло бы $O(N)$, так как у нас нет ссылки на предыдущий узел). Написав решение вручную, вы доказываете глубокое понимание механики кэширования [INDEX].

### 🔗 Связанные темы (Links):
*   [[Collections_Deep_Dive]] — Разбор хэш-таблиц и коллизий [INDEX].
*   [[Redis_Architecture]] — Почему стратегия вытеснения LRU критически важна для кэш-серверов в High-Load [INDEX].
*   [[MOC_LeetCode]]
