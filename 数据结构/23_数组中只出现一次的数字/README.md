# 数组中只出现一次的数字

## 题目描述

一个整数数组里，除了两个数只出现一次，其余每个数都出现两次。把这两个落单的值找出来。

## 思路

字典可以记出现次数，出现第二次就删掉，最后留下两个键。时间线性，空间也线性。

用异或可以把空间降到常数。相同的数异或成 0，整个数组异或完，结果就是那两个单独数 `a ^ b`。它不为 0，说明 `a` 和 `b` 至少有一位不同。取这个结果里任意一个为 1 的二进制位，按该位把数组分成两组。相同的数会进同一组，`a` 和 `b` 会分开。两组再各自异或，就得到这两个数。

## 演算示例

数组 `[4, 7, 4, 9, 2, 9]`。全体异或：

`4 ^ 7 ^ 4 ^ 9 ^ 2 ^ 9 = 7 ^ 2 = 5`，二进制 `101`。最低位是 1，按最低位分组。

- 最低位为 1：7（`111`）、9（`1001`）、9。异或得到 7。
- 最低位为 0：4（`100`）、4、2（`010`）。异或得到 2。

两个单独的数是 7 和 2。

## 代码

```python
class Solution:
    def find_by_dict(self, array):
        seen = {}
        for value in array:
            if value in seen:
                del seen[value]
            else:
                seen[value] = 1
        return sorted(seen)

    def FindNumsAppearOnce(self, array):
        mixed = 0
        for value in array:
            mixed ^= value
        # 取 mixed 最右边的 1，作为分组位
        diff_bit = mixed & -mixed
        first = second = 0
        for value in array:
            if value & diff_bit:
                first ^= value
            else:
                second ^= value
        return first, second


if __name__ == "__main__":
    solver = Solution()
    sample = [4, 7, 4, 9, 2, 9]
    assert sorted(solver.FindNumsAppearOnce(sample)) == [2, 7]
    assert solver.find_by_dict(sample) == [2, 7]
    assert sorted(solver.FindNumsAppearOnce([0, 3, 0, -1])) == [-1, 3]
    print(sorted(solver.FindNumsAppearOnce(sample)))
```

## 复杂度

异或法时间 `O(n)`，额外空间 `O(1)`。字典法时间 `O(n)`，空间 `O(n)`。

## 反例与边界

- 若分组位选错，把两个单独数分到同一组，另一组会异或成 0，得不到两个答案。分组位必须来自 `a ^ b` 里真正为 1 的那一位。
- 数组里有 0 和负数时，异或仍然成立。`[0, 3, 0, -1]` 的答案是 3 和 -1。用「非零才记录」会把 0 漏掉；本题里 0 如果只出现一次，它就是答案之一。
- 两个答案的返回顺序不固定，比较时要排序，或者按题目要求的顺序整理。
