# 两个链表的公共结点

## 题目描述

两条单向链表可能在某个节点汇合，汇合之后共享同一段后缀。找出第一个被两条链表共同指向的节点。这里比的是节点本身，不是节点里的数值碰巧相等。

## 思考

公共后缀的起点，到两条链表结尾的距离相同。先分别量出长度，让较长的那条先走掉多出来的步数，使两条剩下的长度相等，再同步前进。第一次相遇的节点就是入口。

若从头一起走，短链表先结束，长链表还剩的那一段正好等于长度差。记下这个差值再重走，效果和先对齐长度一样。

## 演算示例

- 链表甲：`3 -> 6 -> 9 -> 12 -> 15`
- 链表乙：`8 -> 12 -> 15`

12 和 15 是同一对节点，8 单独指向 12。长度差是 `5 - 3 = 2`。甲先走两步，从 9 开始与乙的 8 对齐：

1. 9 与 8，不同。
2. 12 与 12，是同一个节点，返回它。

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
    def _length(self, head):
        count = 0
        while head:
            count += 1
            head = head.next
        return count

    def FindFirstCommonNode(self, head1, head2):
        len1 = self._length(head1)
        len2 = self._length(head2)
        if len1 < len2:
            head1, head2 = head2, head1
            len1, len2 = len2, len1
        for _ in range(len1 - len2):
            head1 = head1.next
        while head1 and head2:
            if head1 is head2:
                return head1
            head1 = head1.next
            head2 = head2.next
        return None


if __name__ == "__main__":
    shared_12 = ListNode(12)
    shared_15 = ListNode(15)
    shared_12.next = shared_15

    first = ListNode(3)
    first.next = ListNode(6)
    first.next.next = ListNode(9)
    first.next.next.next = shared_12

    second = ListNode(8)
    second.next = shared_12

    found = Solution().FindFirstCommonNode(first, second)
    assert found is shared_12

    # 值相同但不是同一个节点
    lone_a = build_chain([3, 6])
    lone_b = build_chain([9, 6])
    assert Solution().FindFirstCommonNode(lone_a, lone_b) is None
    assert Solution().FindFirstCommonNode(first, first) is first
    assert Solution().FindFirstCommonNode(None, second) is None
    print(found.val)
```

## 复杂度

量长度和同步扫描都是线性的，时间 `O(n + m)`，额外空间 `O(1)`。若对第一条的每个节点都去第二条里找，时间是 `O(n·m)`。

## 反例与边界

- `3 -> 6` 和 `9 -> 6` 的末尾值都是 6，但节点对象不同，应返回空。用值相等代替身份相等会误判。
- 两条链表是同一条时，第一个公共节点就是头节点。
- 任一条为空，不可能有公共节点，返回空。
