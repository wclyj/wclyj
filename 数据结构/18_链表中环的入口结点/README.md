# 链表中环的入口结点

## 题目描述

给定可能带环的单向链表。若存在环，返回环的入口节点；若不存在，返回空。

## 思考

快慢指针可以判断有没有环。慢指针每次一步，快指针每次两步。若有环，快指针会从后面追上慢指针；若快指针先走到空，就没有环。

设头节点到入口有 `s` 步，入口到相遇点有 `d` 步，环长 `c`。相遇时慢指针走了 `s + d`，快指针走了两倍，多出来的距离是若干整圈，因此 `s + d` 等于整数圈。把头节点到入口的距离，和相遇点再走到入口的距离对齐：一个指针留在相遇点，另一个回到头，两者都改为一步一步走，相遇处就是入口。

## 演算示例

节点：`10 -> 20 -> 30 -> 40 -> 50`，然后 50 指回 30。入口是 30，环长是 3，头到入口是 2 步。

快慢指针：

1. 慢在 20，快在 30。
2. 慢在 30，快在 50。
3. 慢在 40，快在 40，相遇。

相遇点是 40。指针甲回到 10，指针乙留在 40，之后两者都只走一步：

- 甲到 20，乙到 50
- 甲到 30，乙到 30

两者在 30 相遇，入口就是 30。从头到入口是 2 步，40 再走回 30 也是 2 步（40 到 50，再到 30）。

## 代码

```python
class ListNode:
    def __init__(self, value):
        self.val = value
        self.next = None


class Solution:
    def EntryNodeOfLoop(self, head):
        slow = fast = head
        while fast and fast.next:
            slow = slow.next
            fast = fast.next.next
            if slow is fast:
                break
        else:
            return None
        finder = head
        while finder is not slow:
            finder = finder.next
            slow = slow.next
        return finder


if __name__ == "__main__":
    nodes = [ListNode(value) for value in (10, 20, 30, 40, 50)]
    for earlier, later in zip(nodes, nodes[1:]):
        earlier.next = later
    nodes[4].next = nodes[2]
    entry = Solution().EntryNodeOfLoop(nodes[0])
    assert entry is nodes[2]

    straight = ListNode(1)
    straight.next = ListNode(2)
    straight.next.next = ListNode(3)
    assert Solution().EntryNodeOfLoop(straight) is None
    assert Solution().EntryNodeOfLoop(None) is None

    alone = ListNode(7)
    alone.next = alone
    assert Solution().EntryNodeOfLoop(alone) is alone
    print(entry.val)
```

## 复杂度

快慢指针和寻找入口各最多走常数圈，时间 `O(n)`，额外空间 `O(1)`。把路过的节点放进集合也能找入口，空间是 `O(n)`。

## 反例与边界

- `1 -> 2 -> 3` 没有环，快指针会走到空，必须返回空，不能再从空节点取 `next`。
- 整个链表就是一个自环：唯一节点指向自己。第一次循环后快慢指针都回到它自己，入口就是该节点。
- 空链表返回空。
- 若第二个阶段两个指针速度不一致，就不会在入口相遇。上面从 10 和 40 出发时，必须都改成一步。
