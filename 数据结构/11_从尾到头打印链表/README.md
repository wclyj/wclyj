# 从尾到头打印链表

## 题目描述

给定一条单向链表，按从尾到头的顺序返回各节点的值，结果放在列表里。

## 思考

链表只能顺着 `next` 往下走，不能直接从尾部开始。可以一边走一边把值插到结果列表的最前面，但每次插入都要把已有元素后移，总时间会变成平方级。

更合适的是用一个栈：从头走到尾，把值依次压栈，走完再依次弹出。弹出顺序自然是从尾到头。递归也能先走到尾再回溯收集，只是链表很长时调用栈可能过深。

## 演算示例

链表 `8 -> 3 -> 6 -> 1`。

压栈过程：`[] -> [8] -> [8, 3] -> [8, 3, 6] -> [8, 3, 6, 1]`。

再弹出：先得到 1，然后 6、3、8。返回 `[1, 6, 3, 8]`。

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


class Solution:
    def printListFromTailToHead(self, head):
        stack = []
        cursor = head
        while cursor:
            stack.append(cursor.val)
            cursor = cursor.next
        output = []
        while stack:
            output.append(stack.pop())
        return output


if __name__ == "__main__":
    solver = Solution()
    assert solver.printListFromTailToHead(build_chain([8, 3, 6, 1])) == [1, 6, 3, 8]
    assert solver.printListFromTailToHead(None) == []
    assert solver.printListFromTailToHead(build_chain([4])) == [4]
    print(solver.printListFromTailToHead(build_chain([8, 3, 6, 1])))
```

## 复杂度

时间 `O(n)`，栈和结果列表各占 `O(n)` 空间。若边走边 `insert(0, value)`，时间变成 `O(n^2)`。

## 反例与边界

- 空链表返回空列表，不能对空头节点取 `val`。
- 只有一个节点时，正序和倒序相同，都是只含该值的列表。
- 递归写法在极长链表上可能超出调用深度；循环压栈没有这个问题。
