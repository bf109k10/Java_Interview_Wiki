# 🔄 LeetCode 125: Valid Palindrome

*   **Ссылка на задачу:** [LeetCode #125](https://leetcode.com)

### 🎯 Главный инсайт (TL;DR)
> Попытка создать очищенную реверсивную строку через `StringBuilder.reverse()` требует $O(N)$ дополнительной памяти. Паттерн **Два указателя (Two Pointers)** позволяет проверить строку на палиндром **in-place (без выделения памяти) за $O(1)$**, двигая указатели с начала и конца навстречу друг другу.

### 💻 Оптимальное решение на Java
```java
class Solution {
    public boolean isPalindrome(String s) {
        if (s == null) return false;
        
        int left = 0;
        int right = s.length() - 1;
        
        while (left < right) {
            // Пропускаем нецифробуквенные символы слева
            while (left < right && !Character.isLetterOrDigit(s.charAt(left))) {
                left++;
            }
            // Пропускаем нецифробуквенные символы справа
            while (left < right && !Character.isLetterOrDigit(s.charAt(right))) {
                right--;
            }
            
            // Сравниваем символы в нижнем регистре
            if (Character.toLowerCase(s.charAt(left)) != Character.toLowerCase(s.charAt(right))) {
                return false;
            }
            
            left++;
            right--;
        }
        
        return true;
    }
}
```

### ⚙️ Архикетурные метрики (Complexity)
*   **Time Complexity:** $O(N)$ — в худшем случае мы пройдем по строке ровно один раз.
*   **Space Complexity:** $O(1)$ — мы используем только две переменные-индекса (`left` и `right`), память кучи не утилизируется.

### 🔗 Связанные темы (Links):
*   [[MOC_LeetCode]]
