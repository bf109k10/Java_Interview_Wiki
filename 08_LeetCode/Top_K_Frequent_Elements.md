# 📊 LeetCode 347: Top K Frequent Elements

*   **Ссылка на задачу:** [LeetCode #347](https://leetcode.com)

### 🎯 Главный инсайт (TL;DR)
> Брутфорс (подсчет частот и полная сортировка массива) занимает $O(N \log N)$ времени. Оптимальное решение использует **Мин-Кучу (Min-Heap / `PriorityQueue` в Java)** фиксированного размера $K$. Мы удерживаем в куче только $K$ самых частых элементов, что снижает сложность до **$O(N \log K)$**.

### 💻 Оптимальное решение на Java
```java
import java.util.HashMap;
import java.util.Map;
import java.util.PriorityQueue;
import java.util.Queue;

class Solution {
    public int[] topKFrequent(int[] nums, int k) {
        // 1. Считаем частоту каждого числа через HashMap — O(N)
        Map<Integer, Integer> countMap = new HashMap<>();
        for (int num : nums) {
            countMap.put(num, countMap.getOrDefault(num, 0) + 1);
        }

        // 2. Инициализируем Мин-Кучу по значению частоты (Map.Entry.getValue)
        Queue<Map.Entry<Integer, Integer>> minHeap = new PriorityQueue<>(
            (a, b) -> Integer.compare(a.getValue(), b.getValue())
        );

        // 3. Обходим карту и держим размер кучи <= K — O(N log K)
        for (Map.Entry<Integer, Integer> entry : countMap.entrySet()) {
            minHeap.add(entry);
            if (minHeap.size() > k) {
                minHeap.poll(); // Удаляем элемент с наименьшей частотой
            }
        }

        // 4. Формируем итоговый массив результата — O(K log K)
        int[] result = new int[k];
        for (int i = k - 1; i >= 0; i--) {
            result[i] = minHeap.poll().getKey();
        }
        return result;
    }
}
```

### ⚙️ Архитектурные метрики (Complexity)
*   **Time Complexity:** $O(N \log K)$ — где $N$ — число элементов в массиве. Операции добавления/удаления из кучи занимают $\log K$, так как размер кучи никогда не превышает $K$. При $K \ll N$ это работает намного быстрее полной сортировки.
*   **Space Complexity:** $O(N + K)$ — память под `HashMap` (до $N$ уникальных элементов) и под `PriorityQueue` (строго $K$ элементов).

### 💣 Ловушка на собеседовании (Interview Trap)
*   ❌ **Вопрос:** «Существует ли решение этой задачи за линейное время $O(N)$ без использования кучи?»
*   ✅ **Ответ:** **Да, существует.** Его можно решить через **Блочную Сортировку (Bucket Sort)**. Вместо сортировки элементов мы создаем массив списков (buckets), где индекс массива — это *частота появления*, а значение — список чисел с такой частотой. Пройдя по этому массиву с конца, мы соберем топ-K элементов ровно за $O(N)$ по времени и $O(N)$ по памяти. На собеседовании Senior-уровня важно озвучить оба варианта и объяснить этот Trade-off.

### 🔗 Связанные темы (Links):
*   [[Collections_Deep_Dive]] — Особенности работы очередей с приоритетом.
*   [[MOC_LeetCode]]
