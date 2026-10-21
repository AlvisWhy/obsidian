## 判断两个字符串是否是 Anagram

Anagram 指两个字符串包含完全相同的字符，且每个字符出现次数一致，例如 `"listen"` 和 `"silent"`。

```python
from collections import Counter

def is_anagram(s: str, t: str) -> bool:
    return Counter(s) == Counter(t)

print(is_anagram("listen", "silent"))  # True
print(is_anagram("hello", "world"))    # False

def first_unique(s):
	counts = Count(s)
	for ch in s:
		if counts[ch] == 1:
			return ch
	return None


def lengthOfLongestSubstring(s: str) -> int:
	lastseen = {}
	left = 0
	maxLen = 0
	
	for right, ch in enumerate(s):
		if ch in lastseen and lastseen[ch]>=left:
			left = lastseen[ch] + 1
		lastseen[ch] = right
		maxLen = max(maxLen, right - left + 1)
	
```

时间复杂度通常为 \(O(n)\)，其中 \(n\) 是字符串长度。

面试记忆点： `Counter` 最常见的应用是频率统计、找 Top K 高频元素、判断两个集合型序列的元素频率是否相同，以及 LeetCode 中的字符串和数组题。