# 反转链表

## 题目描述

给定单向链表，把指针方向全部反过来，返回新的头节点。不要新建一份节点，只改原来的 `next`。

## 思考

头节点最终要变成尾节点，它的 `next` 应是空。从第二个节点起，每个节点都改成指向自己原来的前驱。因此手上要同时拿着三个位置：已经反转好的前驱、当前节点、当前原来的后继。先把后继存下来，再把当前的 `next` 拨到前驱，然后三个位置一起前移。

## 演算示例

原链表 `4 -> 7 -> 1 -> 5`。

| 步骤 | 前驱 | 当前 | 后继 | 当前改完后的链 |
| --- | --- | --- | --- | --- |
| 初始 | 空 | 4 | 7 | 4 -> 空 |
| 1 | 4 | 7 | 1 | 7 -> 4 -> 空 |
| 2 | 7 | 1 | 5 | 1 -> 7 -> 4 -> 空 |
| 3 | 1 | 5 | 空 | 5 -> 1 -> 7 -> 4 -> 空 |

新的头是 5。

## 代码

```python
class ListNode:
    def __init__(self, value):
        self.val = value
        self.next = None


def build_chain(values):
    dummy = ListNode(0)
    cursor = dummy
    for value in values:
        cursor.next = ListNode(value)
        cursor = cursor.next
    return dummy.next


def to_list(head):
    values = []
    while head:
        values.append(head.val)
        head = head.next
    return values


class Solution:
    def ReverseList(self, head):
        previous = None
        current = head
        while current:
            following = current.next
            current.next = previous
            previous = current
            current = following
        return previous


if __name__ == "__main__":
    solver = Solution()
    assert to_list(solver.ReverseList(build_chain([4, 7, 1, 5]))) == [5, 1, 7, 4]
    assert solver.ReverseList(None) is None
    single = solver.ReverseList(build_chain([6]))
    assert to_list(single) == [6]
    print(to_list(solver.ReverseList(build_chain([4, 7, 1, 5]))))
```

## 复杂度

每个节点只改一次指针，时间 `O(n)`，只用三个引用，额外空间 `O(1)`。

## 反例与边界

- 空链表返回空。
- 只有一个节点时，循环把它的 `next` 设为空，头还是它自己。
- 如果先改 `current.next` 再去读原来的后继，后继就丢了，后面的节点全部断裂。必须先把后继存进临时变量。
