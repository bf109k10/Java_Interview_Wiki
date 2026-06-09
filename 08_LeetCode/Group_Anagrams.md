# 🔠 LeetCode 49: Group Anagrams

*   **Ссылка на задачу:** [LeetCode #49](https://leetcode.com)

### 🎯 Главный инсайт (TL;DR)
> Слова являются анаграммами, если у них одинаковый состав и количество букв. Чтобы сгруппировать их за один проход, нам нужен уникальный ключ для `HashMap`. В качестве ключа можно использовать либо **отсортированную строку**, либо **строку-частотный массив** (подобие хэш-функции).

### 💻 Оптимальное решение на Java (Подход с кодированием частот)
```java
import java.util.*;

class Solution {
    public List<List<String>> groupAnagrams(String[] strs) {
        if (strs == null || strs.length == 0) return new ArrayList<>();
        
        Map<String, List<String>> map = new HashMap<>();
        
        for (String s : strs) {
            // Строим частотный массив для букв 'a'-'z' (размер 26)
            int[] count = new int[26];
            for (char c : s.toCharArray()) {
                count[c - 'a']++;
            }
            
            // Превращаем массив в уникальный ключ-строку, например: "1#0#2#0..."
            StringBuilder sb = new StringBuilder();
            for (int i = 0; i < 26; i++) {
                sb.append(count[i]).append('#');
            }
            String key = sb.toString();
            
            // Группируем в HashMap
            map.computeIfAbsent(key, _ -> new ArrayList<>()).add(s);
        }
        
        return new ArrayList<>(map.values());
    }
}
```

### ⚙️ Архитектурные метрики (Complexity)
*   **Time Complexity:** $O(N \cdot L)$ — где $N$ — количество строк, а $L$ — максимальная длина строки. Мы обходим каждую строку и каждую букву линейно. (Если вместо кодирования использовать сортировку строк `Arrays.sort()`, сложность возрастет до $O(N \cdot L \log L)$).
*   **Space Complexity:** $O(N \cdot L)$ — вся память уходит на хранение сгруппированных строк в `HashMap`.

### 🔗 Связанные темы (Links):
*   [[Collections_Deep_Dive]] — Коллизии и бакеты `HashMap`.
*   [[MOC_LeetCode]]
