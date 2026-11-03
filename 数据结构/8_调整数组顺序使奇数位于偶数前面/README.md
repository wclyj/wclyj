# 把奇数整体挪到偶数前面

## 题目描述

给定整数数组，调整顺序，使所有奇数都在偶数前面。奇数与奇数之间、偶数与偶数之间，原来的相对次序必须保持不变。

## 思考

稳定地分开两类数，最直接的办法是走一遍数组：奇数按遇到的顺序放进第一条列表，偶数放进第二条，最后把偶数接在奇数后面。每一类内部都是按原顺序追加的，所以相对次序不会变。

如果允许打乱同类的次序，可以用左右指针交换，额外空间是常数。本题要求稳定，交换法会把先出现的奇数换到后面。

## 演算示例

输入 `[7, 2, 9, 4, 3, 8, 1]`。

- 奇数桶按扫描顺序：`7, 9, 3, 1`
- 偶数桶按扫描顺序：`2, 4, 8`
- 拼接：`[7, 9, 3, 1, 2, 4, 8]`

## 代码

```python
class Solution:
    def reOrderArray(self, array):
        odds = []
        evens = []
        for value in array:
            if value % 2 == 0:
                evens.append(value)
            else:
                odds.append(value)
        return odds + evens


if __name__ == "__main__":
    solver = Solution()
    assert solver.reOrderArray([7, 2, 9, 4, 3, 8, 1]) == [7, 9, 3, 1, 2, 4, 8]
    assert solver.reOrderArray([]) == []
    assert solver.reOrderArray([2, 4, 6]) == [2, 4, 6]
    assert solver.reOrderArray([1, 3, 5]) == [1, 3, 5]
    assert solver.reOrderArray([-3, -2, 0, 4, -1]) == [-3, -1, -2, 0, 4]
    print(solver.reOrderArray([7, 2, 9, 4, 3, 8, 1]))
```

## 复杂度

时间 `O(n)`，需要两块辅助列表，空间 `O(n)`。

## 反例与边界

- 两端向中间交换不保证稳定。`[2, 1, 4, 3]` 若把左端偶数 2 和右端奇数 3 对调，得到 `[3, 1, 4, 2]`。奇数原来的顺序是 1 然后 3，结果变成 3 然后 1，不符合题意。
- 全奇数或全偶数时，结果应与原数组相同。
- 负数按余数判断：`-3` 是奇数，`-2` 和 `0` 是偶数。若用「除以 2 的符号」而不是余数，0 和负数容易分错。
