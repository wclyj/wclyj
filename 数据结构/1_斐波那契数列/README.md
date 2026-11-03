# 斐波那契数列

## 题目描述

给出非负整数 `n`（题目限制 `n` 不超过 39），返回斐波那契数列的第 `n` 项。下标从 0 起算：第 0 项是 0，第 1 项是 1，从第 2 项开始，每一项都是紧挨着的前两项相加。

## 思路

递推关系可以写成 `F(k) = F(k-1) + F(k-2)`，边界是 `F(0) = 0`、`F(1) = 1`。

直接按这个式子递归，会把同一个下标算很多遍。比如求 `F(6)` 时，`F(4)` 会从 `F(6)` 和 `F(5)` 两条支路各算一次，支路再往下还会重复。调用次数大约按黄金比例增长，时间是指数级，空间则是递归深度 `O(n)`。

既然每一项只依赖前两项，就从 0 和 1 顺着往后推，只留下最近两个结果。这样每个下标只算一次。

## 演算示例

取 `n = 7`（与常见演示里直接打印第 10 项不同）。手工展开：

| 下标 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 数值 | 0 | 1 | 1 | 2 | 3 | 5 | 8 | 13 |

滚动变量的过程：

- 起步：`prev = 0`，`curr = 1`
- 下标 2：`prev, curr = 1, 0+1 = 1`
- 下标 3：`prev, curr = 1, 1+1 = 2`
- 下标 4：`prev, curr = 2, 1+2 = 3`
- 下标 5：`prev, curr = 3, 2+3 = 5`
- 下标 6：`prev, curr = 5, 3+5 = 8`
- 下标 7：`prev, curr = 8, 5+8 = 13`

所以 `F(7) = 13`。

## 代码

```python
class Solution:
    def fib_recursive(self, n):
        # 只适合很小的 n，用来对照递推式
        if n < 0:
            return None
        if n < 2:
            return n
        return self.fib_recursive(n - 1) + self.fib_recursive(n - 2)

    def Fibonacci(self, n):
        # 线性递推：只保留前一项和当前项
        if n < 0:
            return None
        if n < 2:
            return n
        prev, curr = 0, 1
        for _ in range(2, n + 1):
            prev, curr = curr, prev + curr
        return curr


if __name__ == "__main__":
    solver = Solution()
    assert solver.Fibonacci(0) == 0
    assert solver.Fibonacci(1) == 1
    assert solver.Fibonacci(7) == 13
    assert solver.fib_recursive(7) == 13
    assert solver.Fibonacci(-1) is None
    print(solver.Fibonacci(7))
```

## 复杂度

- 递归：时间约 `O(φ^n)`，调用栈空间 `O(n)`。
- 循环：时间 `O(n)`，额外空间 `O(1)`。

## 反例与边界

- `n = 0` 必须返回 0。若有人把「第一项」理解成 1，会把下标整体错一位。
- 负数没有定义在这道题里。循环实现遇到 `n < 0` 返回 `None`，避免把负下标当成合法项。
- 递归版本在 `n = 39` 时重复计算过多，容易超时；循环版本仍然只做几十次加法。
