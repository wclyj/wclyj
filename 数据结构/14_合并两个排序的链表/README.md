# 合并两个排序的链表

## 题目描述

两条链表都已经按节点值从小到大排好。把它们合成一条仍然不下降的链表。节点可以复用，不必复制值。

## 思考

类似归并两条有序序列。设一个哑节点当新链表的起点，再用指针 `tail` 始终指着新链表的末尾。比较两条链表当前头：较小的那个摘下来接到 `tail` 后面，对应链表前进一格。一条走完后，把另一条剩下的整段直接接上。

谁的头更小，新链表的真正头就是谁。两条里有一条本来就是空的，结果就是另一条。

## 演算示例

- 第一条：`1 -> 6 -> 10`
- 第二条：`3 -> 6 -> 8 -> 12`

比较过程：

1. 1 小于 3，接上 1。
2. 6 大于 3，接上 3。
3. 两个 6 相等，按「不大于就取第二条」接上第二条的 6，再接上第一条的 6。
4. 10 大于 8，接上 8。
5. 第二条结束，把剩下的 10 接上；10 后面没有节点，但第二条在接 8 时已经把 12 留在后面，实际剩余是第一条的 10。再看：接完 8 后第一条还在 10，第二条还在 12。10 小于 12，接 10，第一条结束，再接 12。

结果：`1 -> 3 -> 6 -> 6 -> 8 -> 10 -> 12`。

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
    def Merge(self, head1, head2):
        dummy = ListNode(0)
        tail = dummy
        while head1 and head2:
            if head1.val <= head2.val:
                tail.next = head1
                head1 = head1.next
            else:
                tail.next = head2
                head2 = head2.next
            tail = tail.next
        tail.next = head1 if head1 else head2
        return dummy.next


if __name__ == "__main__":
    solver = Solution()
    merged = solver.Merge(build_chain([1, 6, 10]), build_chain([3, 6, 8, 12]))
    assert to_list(merged) == [1, 3, 6, 6, 8, 10, 12]
    assert to_list(solver.Merge(None, build_chain([4, 5]))) == [4, 5]
    assert solver.Merge(None, None) is None
    print(to_list(merged))
```

## 复杂度

每个节点恰好被看一次，时间 `O(n + m)`，哑节点以外没有新开节点数组，额外空间 `O(1)`。

## 反例与边界

- 其中一条是空链表，应返回另一条，而不是空。
- 两条都空，返回空。
- 相等值必须稳定接上，不能互相覆盖。`6` 和 `6` 都要保留，结果长度是两条长度之和。若比较时写成赋值覆盖而不是改 `next`，会丢掉其中一个 6。
