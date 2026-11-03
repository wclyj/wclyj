# 孩子们的游戏（圆圈中最后剩下的数）

## 题目描述

`n` 个孩子编号 `0` 到 `n-1`，围成一圈。从 0 号开始报数，每次数到 `m` 就让当前这个孩子离圈，然后从下一位重新从 0 报到 `m-1`。问最后留下的编号。若 `n` 或 `m` 小于 1，返回 -1。

## 思路

可以直接用列表模拟圆圈。当前位置加上 `m-1`，再对当前人数取余，就是这一轮离圈的下标。删掉之后，下一轮正好从该下标继续，因为后面的元素已经前移到这个位置。

还有递推。记 `f(1) = 0`。人数从 `k-1` 变成讨论 `k` 个人时，离圈的那个人会把后面的编号整体平移 `m` 位，因此

`f(k) = (f(k-1) + m) % k`。

从 2 推到 `n` 就能得到答案，不必真的删节点。

## 演算示例

取 `n = 6`、`m = 4`。圆圈开始是 `[0, 1, 2, 3, 4, 5]`，下标从 0 看。

1. `(0 + 3) % 6 = 3`，3 离圈，剩下 `[0, 1, 2, 4, 5]`，下一位仍是下标 3，也就是 4。
2. `(3 + 3) % 5 = 1`，1 离圈，剩下 `[0, 2, 4, 5]`。
3. `(1 + 3) % 4 = 0`，0 离圈，剩下 `[2, 4, 5]`。
4. `(0 + 3) % 3 = 0`，2 离圈，剩下 `[4, 5]`。
5. `(0 + 3) % 2 = 1`，5 离圈，留下 4。

递推核对：`f(1)=0`，`f(2)=0`，`f(3)=1`，`f(4)=1`，`f(5)=0`，`f(6)=4`。

## 代码

```python
class Solution:
    def last_remaining_simulate(self, n, m):
        if n < 1 or m < 1:
            return -1
        circle = list(range(n))
        index = 0
        while len(circle) > 1:
            index = (index + m - 1) % len(circle)
            circle.pop(index)
        return circle[0]

    def LastRemaining_Solution(self, n, m):
        if n < 1 or m < 1:
            return -1
        survivor = 0
        for size in range(2, n + 1):
            survivor = (survivor + m) % size
        return survivor


if __name__ == "__main__":
    solver = Solution()
    assert solver.LastRemaining_Solution(6, 4) == 4
    assert solver.last_remaining_simulate(6, 4) == 4
    assert solver.LastRemaining_Solution(5, 1) == 4
    assert solver.last_remaining_simulate(5, 1) == 4
    assert solver.LastRemaining_Solution(1, 3) == 0
    assert solver.LastRemaining_Solution(0, 3) == -1
    assert solver.LastRemaining_Solution(5, 0) == -1
    print(solver.LastRemaining_Solution(6, 4))
```

## 复杂度

- 列表模拟：每次删除可能搬移后面的元素，时间 `O(n·m)` 量级里最坏约 `O(n^2)`，空间 `O(n)`。
- 递推：时间 `O(n)`，空间 `O(1)`。

## 反例与边界

- `n < 1` 或 `m < 1` 返回 -1。`m = 0` 没有「数到 0 就离圈」的合法报数。
- `m = 1` 时每次都是当前位置立刻离圈，最后留下 `n-1`。`n = 5` 时答案是 4。若循环写成「先走 `m-1` 步但前驱指针没初始化」，`m = 1` 时根本不会移动，接着对空前驱改指针就会出错。
- 只剩一个孩子时不再报数，编号就是 0，对应 `n = 1`。
