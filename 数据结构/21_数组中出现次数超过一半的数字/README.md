# 找出超过半数的那个数

## 题目描述

数组里如果有一个数出现的次数严格大于长度的一半，把它找出来。没有这样的数就返回 0。

## 思路

用字典累计每个值的次数，边累计边判断是否已经超过 `长度 // 2`。时间、空间都是线性的。

只留常数额外空间时，用抵消法。维护一个候选值和一个票数：遇到相同的数就加一票，遇到不同的数就减一票，票数归零就改投当前数。超过一半的数不会被完全抵消。抵消结束后还要再数一遍候选值，因为「最后剩下」不等于「一定超过一半」。

## 演算示例

数组 `[6, 1, 6, 3, 6, 2, 6]`，长度 7，超过一半意味着至少 4 次。

| 读到 | 候选 | 票数 | 原因 |
| --- | --- | --- | --- |
| 6 | 6 | 1 | 票数为 0，改投 6 |
| 1 | 6 | 0 | 不同，抵消 |
| 6 | 6 | 1 | 重新开始 |
| 3 | 6 | 0 | 抵消 |
| 6 | 6 | 1 | 重新开始 |
| 2 | 6 | 0 | 抵消 |
| 6 | 6 | 1 | 重新开始 |

候选是 6。再数一遍，6 出现 4 次，`4 > 3`，返回 6。

## 代码

```python
class Solution:
    def more_than_half_by_count(self, numbers):
        if not numbers:
            return 0
        limit = len(numbers) // 2
        seen = {}
        for value in numbers:
            seen[value] = seen.get(value, 0) + 1
            if seen[value] > limit:
                return value
        return 0

    def MoreThanHalfNum_Solution(self, numbers):
        if not numbers:
            return 0
        candidate = None
        votes = 0
        for value in numbers:
            if votes == 0:
                candidate = value
                votes = 1
            elif value == candidate:
                votes += 1
            else:
                votes -= 1
        if votes == 0 or candidate is None:
            return 0
        if numbers.count(candidate) > len(numbers) // 2:
            return candidate
        return 0


if __name__ == "__main__":
    solver = Solution()
    sample = [6, 1, 6, 3, 6, 2, 6]
    assert solver.MoreThanHalfNum_Solution(sample) == 6
    assert solver.more_than_half_by_count(sample) == 6
    assert solver.MoreThanHalfNum_Solution([6, 1, 6, 3]) == 0
    assert solver.MoreThanHalfNum_Solution([]) == 0
    assert solver.MoreThanHalfNum_Solution([8]) == 8
    print(solver.MoreThanHalfNum_Solution(sample))
```

## 复杂度

- 字典：时间 `O(n)`，空间 `O(n)`。
- 抵消再验证：时间 `O(n)`，额外空间 `O(1)`。

## 反例与边界

- `[6, 1, 6, 3]` 里 6 出现两次，长度 4，两次并不大于一半。抵消后票数回到 0，应返回 0。若不做第二次计数，可能把某个偶然剩下的数当成答案。
- 空数组返回 0。
- 只有一个元素时，它出现 1 次，大于 `1 // 2 = 0`，答案就是它自己。
