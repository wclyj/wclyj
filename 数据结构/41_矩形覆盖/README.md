# 矩形覆盖

## 题目描述

用 `n` 个 `2×1` 的小矩形，不重叠也不越界地铺满一个 `2×n` 的大矩形。小矩形可以竖着放，也可以横着放。问有多少种铺法。`n = 0` 时没有可以铺的大矩形，返回 0。

## 思考

竖着放时，一块小矩形占满一列的两行。横着放时，一块只盖住一行里的两列，所以横放必须上下各一块，一次占掉两列。

记 `f(n)` 为铺满 `2×n` 的方法数。

- 最右边一列如果竖放一块，左边是 `f(n-1)` 种。
- 最右边两列如果用两块横放的小矩形盖住，左边是 `f(n-2)` 种。

这两种情况不重叠，因此 `f(n) = f(n-1) + f(n-2)`。

边界要单独数：`n = 1` 只能竖放一块，1 种；`n = 2` 可以两块都竖放，或者两块横放叠在一起，2 种。它和斐波那契同形，但 `f(2)` 是 2 不是 3。

## 演算示例

`n = 3` 的三种铺法：

1. 三块都竖放。
2. 左边一块竖放，右边两列用一对横放盖住。
3. 左边两列用一对横放盖住，最右一块竖放。

没有第四种：三列的宽度是奇数，不能全部用横放，因为横放一次要占两列。

`n = 4` 在 `f(3)` 和 `f(2)` 上加起来，是 `3 + 2 = 5`。

## 代码

```python
class Solution:
    def rectCover(self, number):
        if number <= 0:
            return 0
        if number <= 2:
            return number
        previous, current = 1, 2
        for _ in range(3, number + 1):
            previous, current = current, previous + current
        return current


if __name__ == "__main__":
    solver = Solution()
    assert solver.rectCover(1) == 1
    assert solver.rectCover(2) == 2
    assert solver.rectCover(3) == 3
    assert solver.rectCover(4) == 5
    assert solver.rectCover(5) == 8
    assert solver.rectCover(0) == 0
    assert solver.rectCover(-2) == 0
    print(solver.rectCover(4))
```

## 复杂度

循环到 `n`，时间 `O(n)`，只用两个变量，空间 `O(1)`。

## 反例与边界

- `n = 2` 是 2 种，不是 3 种。把两块横放再左右对调并不产生新的覆盖，它们占的是同一对格子。
- `n = 0` 和负数返回 0。若从 `f(0) = 1` 起推并把 0 当成一种「空铺法」，会和「没有大矩形」的题意不一致。
- `n = 3` 只有上面三种。若允许小矩形重叠或伸出边界，会多出不合法的摆法。
