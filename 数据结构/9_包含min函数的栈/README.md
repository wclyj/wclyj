# 包含 min 函数的栈

## 题目描述

实现一个栈，除了压入、弹出、查看栈顶，还要能在 `O(1)` 时间得到当前栈里的最小值。测试数据不会在栈空时调用弹出、栈顶或取最小值。

## 思路

普通栈取最小值要扫一遍，做不到常数时间。再维护一条最小值栈：只有新元素小于或等于当前最小值时才压入。弹出时，若弹出的值正好是最小值栈的栈顶，最小值栈也弹出。这样最小值栈顶始终是「当前还在栈里的最小值」。

等于最小值时也要压入。否则连续两个相同最小值，弹出一个之后会把另一个也丢掉。

## 演算示例

依次压入 `5, 3, 7, 2, 2, 6`。

| 操作 | 数据栈 | 最小值栈 | 当前最小值 |
| --- | --- | --- | --- |
| push 5 | 5 | 5 | 5 |
| push 3 | 5,3 | 5,3 | 3 |
| push 7 | 5,3,7 | 5,3 | 3 |
| push 2 | 5,3,7,2 | 5,3,2 | 2 |
| push 2 | 5,3,7,2,2 | 5,3,2,2 | 2 |
| push 6 | 5,3,7,2,2,6 | 5,3,2,2 | 2 |
| pop | 5,3,7,2,2 | 5,3,2,2 | 2 |
| pop | 5,3,7,2 | 5,3,2 | 2 |
| pop | 5,3,7 | 5,3 | 3 |

弹出第二个 2 之后，最小值仍然是 2；再弹出这个 2，最小值回到 3。

## 代码

```python
class MinStack:
    def __init__(self):
        self.data = []
        self.mins = []

    def push(self, value):
        self.data.append(value)
        if not self.mins or value <= self.mins[-1]:
            self.mins.append(value)

    def pop(self):
        if not self.data:
            return None
        gone = self.data.pop()
        if self.mins and gone == self.mins[-1]:
            self.mins.pop()
        return gone

    def top(self):
        if not self.data:
            return None
        return self.data[-1]

    def min(self):
        if not self.mins:
            return None
        return self.mins[-1]


if __name__ == "__main__":
    stack = MinStack()
    for value in (5, 3, 7, 2, 2, 6):
        stack.push(value)
    assert stack.min() == 2
    assert stack.top() == 6
    assert stack.pop() == 6
    assert stack.pop() == 2
    assert stack.min() == 2
    assert stack.pop() == 2
    assert stack.min() == 3
    empty = MinStack()
    assert empty.pop() is None
    assert empty.min() is None
    print(stack.min())
```

## 复杂度

压入、弹出、栈顶、取最小值都是 `O(1)`。最坏情况下每个元素都进入最小值栈，额外空间 `O(n)`。

## 反例与边界

- 只在「严格小于」时记录最小值，会在重复最小值上出错。压入 `3, 3` 再弹出一次，最小值仍应是 3。若第二次 3 没进最小值栈，弹出时会把唯一的 3 清掉，栈里其实还有一个 3。
- 空栈取最小值、弹出都没有意义。题目保证测试不会这样做；实现里仍返回 `None`，避免直接对空列表取下标。
