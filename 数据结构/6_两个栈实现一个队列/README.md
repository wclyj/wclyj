# 两个栈实现一个队列

## 题目描述

只能使用两个栈，完成队列的入队和出队。队列里的元素是整数。入队把新元素放到队尾，出队拿走最早进入的那个元素。

## 思考

栈是后进先出，队列是先进先出。一个栈单独用，弹出顺序和进入顺序相反，做不成队列。

准备两个栈：

- `inbox` 专门接收新元素，入队只往这里压。
- `outbox` 专门出队。它空了的时候，把 `inbox` 里的元素逐个弹出再压进来。经过这一倒，最先进入的元素会落在 `outbox` 栈顶。

`outbox` 里还有元素时不要倒栈。否则会把新旧两段的顺序搅乱。

## 演算示例

依次入队 7、8、9，再出队两次，然后入队 4，再出队。

1. 入队后 `inbox = [7, 8, 9]`，`outbox` 空。
2. 第一次出队：把 inbox 倒到 outbox，得到 `outbox = [9, 8, 7]`，弹出 7。
3. 第二次出队：outbox 非空，直接弹出 8。此时 `outbox = [9]`。
4. 入队 4：只压进 inbox，`inbox = [4]`，不要去动 outbox。
5. 第三次出队：outbox 里还有 9，弹出 9。
6. 第四次出队：outbox 空了，再把 inbox 的 4 倒过去并弹出。

出队序列是 `7, 8, 9, 4`，符合先进先出。

## 代码

```python
class QueueByTwoStacks:
    def __init__(self):
        self.inbox = []
        self.outbox = []

    def push(self, value):
        self.inbox.append(value)

    def pop(self):
        if not self.outbox:
            while self.inbox:
                self.outbox.append(self.inbox.pop())
        if not self.outbox:
            return None
        return self.outbox.pop()


if __name__ == "__main__":
    queue = QueueByTwoStacks()
    queue.push(7)
    queue.push(8)
    queue.push(9)
    assert queue.pop() == 7
    assert queue.pop() == 8
    queue.push(4)
    assert queue.pop() == 9
    assert queue.pop() == 4
    assert queue.pop() is None
    print("ok")
```

## 复杂度

单次入队是 `O(1)`。出队在需要倒栈时是 `O(n)`，但每个元素最多从 inbox 搬到 outbox 一次，均摊 `O(1)`。两个栈合计空间 `O(n)`。

## 反例与边界

- 两个栈都空时出队，没有元素可拿，返回 `None`。
- 若每次出队都把 outbox 倒回 inbox 再重新倒一次，上面例子在弹出 7 之后再入队 4，可能先弹出 4 而不是 8。倒栈只能发生在 outbox 为空的时候。
- 只用一个栈，入队 7、8 后弹出的是 8，已经不是队列。
