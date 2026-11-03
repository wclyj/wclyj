# 拥有最多糖果的孩子

`candies[i]` 是第 `i` 个孩子已有的糖果数。另外还有 `extraCandies` 颗糖，可以全部给同一个孩子。对每个孩子问：如果把这份额外的糖都给他（其他人维持原状），他的糖果数能否至少并列第一。允许多个孩子同时为真。

## 思路

先找出当前最大值。某个孩子加上全部额外糖后，只要大于等于这个最大值，就有机会成为最多的人之一。不需要模拟「把糖分给不同的人」的多种方案，因为题目是对每个孩子单独假设「额外的都归他」。

## 演算

`candies = [1, 4, 2]`，`extraCandies = 2`。当前最多是 4。

- 第 0 个：`1 + 2 = 3`，比 4 少，假。
- 第 1 个：`4 + 2 = 6`，已经是最多，再给仍然最多，真。
- 第 2 个：`2 + 2 = 4`，与原来的最大值持平，真。

结果 `[false, true, true]`。

## 代码

```python
class Solution:
    def kidsWithCandies(self, candies, extraCandies):
        peak = max(candies)
        flags = []
        for owned in candies:
            flags.append(owned + extraCandies >= peak)
        return flags
```

## 复杂度

求最大值和再扫一遍都是线性的，时间 `O(n)`，结果列表空间 `O(n)`。

## 边界

大家糖果数一样时，每个人加上额外糖都仍并列第一，应全部为真。额外糖为 0 时，只有原本就达到最大值的孩子为真。比较用大于等于，持平也算「拥有最多」。
