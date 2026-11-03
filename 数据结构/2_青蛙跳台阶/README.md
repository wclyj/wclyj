# 青蛙跳台阶

## 题目描述

一只青蛙上楼梯，每次可以跨 1 级，也可以跨 2 级。问登上恰好 `n` 级台阶，一共有多少种跳法。步数相同但先后顺序不同，算不同方案。

另一问把限制放开：一次可以跨 1 级、2 级，一直到直接跨完剩下的全部 `n` 级。同样问方案总数。

## 思路

先看每次只跨 1 或 2 级。最后一跳如果是 1 级，前面的 `n-1` 级有 `f(n-1)` 种走法；最后一跳如果是 2 级，前面有 `f(n-2)` 种。两种情况互不重叠，因此

`f(n) = f(n-1) + f(n-2)`。

边界：`n = 1` 只有「跨 1 级」这 1 种；`n = 2` 有「1+1」和「2」这 2 种。它和斐波那契同形，只是起点不同，仍然用两个变量往前滚。

再看可以跨任意级的版本。登上 `n` 级时，最后一跳的长度可以是 1 到 `n`：

`g(n) = g(n-1) + g(n-2) + ... + g(1) + 1`，

最后那个 `+1` 表示一次就跳完 `n` 级。把 `g(n-1)` 按同样方式展开，会得到 `g(n) = 2·g(n-1)`，而 `g(1) = 1`，所以 `g(n) = 2^(n-1)`。

## 演算示例

普通跳法，取 `n = 4`：

1. `1+1+1+1`
2. `1+1+2`
3. `1+2+1`
4. `2+1+1`
5. `2+2`

一共 5 种。滚动过程：`a=1, b=2`，到 3 级变成 3，到 4 级变成 5。

任意跨度，仍取 `n = 4`。最后一跳分别是 1、2、3、4 级，对应前面的方案数 `g(3)+g(2)+g(1)+1 = 4+2+1+1 = 8`，也就是 `2^3`。

## 代码

```python
class Solution:
    def jump_floor(self, n):
        # 每次只允许跨 1 级或 2 级
        if n <= 0:
            return 0
        if n <= 2:
            return n
        prev, curr = 1, 2
        for _ in range(3, n + 1):
            prev, curr = curr, prev + curr
        return curr

    def jump_floor_any(self, n):
        # 一次可以跨 1..n 级，方案数是 2 的 n-1 次方
        if n <= 0:
            return 0
        return 1 << (n - 1)


if __name__ == "__main__":
    solver = Solution()
    assert solver.jump_floor(1) == 1
    assert solver.jump_floor(2) == 2
    assert solver.jump_floor(4) == 5
    assert solver.jump_floor(5) == 8
    assert solver.jump_floor_any(1) == 1
    assert solver.jump_floor_any(4) == 8
    assert solver.jump_floor(0) == 0
    assert solver.jump_floor_any(0) == 0
    print(solver.jump_floor(4), solver.jump_floor_any(4))
```

## 复杂度

- 普通跳法：时间 `O(n)`，空间 `O(1)`。
- 任意跨度：位移运算时间 `O(1)`，空间 `O(1)`。若改成循环累乘 2，时间也是 `O(n)`。

## 反例与边界

- `n = 0` 没有台阶可上，这里返回 0。若误写成返回 1，会把「停在原地」当成一种跳法，和题意不符。
- 普通跳法里 `n = 2` 是 2 不是 1。漏掉「一次跨两级」会少算。
- 任意跨度不能套用普通跳法的 `f(n-1)+f(n-2)`。`n = 3` 时普通跳法只有 3 种，任意跨度有 4 种，多出来的是一次跨 3 级。
