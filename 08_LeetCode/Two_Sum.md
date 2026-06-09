# 🟢 LeetCode 1: Two Sum
#leetcode #hashmap #arrays #easy

*   **Ссылка на задачу:** [LeetCode #1](https://leetcode.com)

### 🎯 Главный инсайт (TL;DR)
> Брутфорс-решение (два цикла) требует $O(N^2)$ времени. Использование `HashMap` позволяет найти пару за **один проход по массиву $O(N)$**, меняя процессорное время на память (расход памяти увеличивается до $O(N)$).

### 💻 Оптимальное решение на Java
```java
import java.util.HashMap;
import java.util.Map;

class Solution {
    public int[] twoSum(int[] nums, int[] target) {
        Map<Integer, Integer> map = new HashMap<>();
        
        for (int i = 0; i < nums.length; i++) {
            int complement = target - nums[i];
            
            // Если в карте есть дополняющее число, мы нашли пару
            if (map.containsKey(complement)) {
                return new int[] { map.get(complement), i };
            }
            
            // Сохраняем: [Значение -> Индекс]
            map.put(nums[i], i);
        }
        throw new IllegalArgumentException("No two sum solution");
    }
}
```

### ⚙️ Архитектурные метрики (Complexity)
*   **Time Complexity:** $O(N)$ — мы гарантированно обходим массив всего один раз. Поиск в `HashMap` (`containsKey`, `get`) занимает $O(1)$ в среднем.
*   **Time Complexity:** $O(N)$ — подробнее о расчете скоростей роста смотри в [[Asymptotic_Analysis_O_Big]].
*   **Space Complexity:** $O(N)$ — в худшем случае нам придется положить все элементы массива в `HashMap`.

### 🔗 Связанные темы (Links):
*   [[Collections_Deep_Dive]] — Как внутри устроена `HashMap`.
*   [[MOC_LeetCode]]
