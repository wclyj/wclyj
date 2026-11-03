# 连续子数组的最大和

## 题目描述

给定一个整数数组，求连续子数组的最大和。子数组至少包含一个元素。数组里可以有负数，不能把数组拆成不连续的几段再相加。

## 思路

两重循环可以枚举每一个起点和终点，累加后留下最大值，时间是平方级。

线性做法只保留「以当前位置结尾的最大和」。记它为 `end_here`。读到新数 `x` 时，要么把 `x` 接到原来的结尾后面，要么从 `x` 重新开始，取二者较大的一个：

`end_here = max(end_here + x, x)`。

全局答案是所有 `end_here` 里的最大值。全是负数时，这个最大值就是其中最大的那个负数，不会错误地返回 0。

## 演算示例

数组 `[2, -4, 3, 5, -1, 2, -6, 4]`。

| 当前位置 | 数值 | 以它结尾的最大和 | 全局最大 |
| --- | --- | --- | --- |
| 0 | 2 | 2 | 2 |
| 1 | -4 | -2（2 + -4 比 -4 大） | 2 |
| 2 | 3 | 3（从 3 重新开始） | 3 |
| 3 | 5 | 8 | 8 |
| 4 | -1 | 7 | 8 |
| 5 | 2 | 9 | 9 |
| 6 | -6 | 3 | 9 |
| 7 | 4 | 7 | 9 |

最大和是 9，对应子数组 `[3, 5, -1, 2]`。中间的 -1 被两边的正数补上，所以要留下。

## 代码

```python
class Solution:
    def greatest_sum_slow(self, array):
        if not array:
            return 0
        best = array[0]
        for start in range(len(array)):
            total = 0
            for value in array[start:]:
                total += value
                if total > best:
                    best = total
        return best

    def FindGreatestSumOfSubArray(self, array):
        if not array:
            return 0
        best = end_here = array[0]
        for value in array[1:]:
            end_here = value if end_here + value < value else end_here + value
            if end_here > best:
                best = end_here
        return best


if __name__ == "__main__":
    solver = Solution()
    sample = [2, -4, 3, 5, -1, 2, -6, 4]
    assert solver.FindGreatestSumOfSubArray(sample) == 9
    assert solver.greatest_sum_slow(sample) == 9
    assert solver.FindGreatestSumOfSubArray([-5, -2, -8]) == -2
    assert solver.FindGreatestSumOfSubArray([4]) == 4
    assert solver.FindGreatestSumOfSubArray([]) == 0
    print(solver.FindGreatestSumOfSubArray(sample))
```

## 复杂度

- 枚举所有子数组：时间 `O(n^2)`，空间 `O(1)`。
- 以结尾滚动：时间 `O(n)`，空间 `O(1)`。

## 反例与边界

- `[-5, -2, -8]` 的最大和是 -2，不是 0。若把「和变成负数就丢掉、答案初始成 0」，全负数数组会返回 0，但 0 并不是任何子数组的和。
- 空数组没有长度至少为 1 的子数组，这里返回 0，调用前应知道输入可能为空。
- `[2, -4, 3]` 不能把 2 和 3 拼起来得到 5，它们不连续。正确结果是 3。
