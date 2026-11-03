# 链表中倒数第 K 个节点

## 题目描述

给定单向链表和正整数 `k`，返回倒数第 `k` 个节点。`k` 从 1 开始，倒数第 1 个就是尾节点。`k` 不合法或比链表还长时，没有对应节点。

## 思路

可以把节点先收进列表，再取下标 `长度 - k`。这能做对，但多占一份线性空间。

两个指针只需走一趟。`ahead` 先独自前进 `k` 步，然后 `ahead` 和 `behind` 同步移动。`ahead` 走到末尾空位时，`behind` 正好停在倒数第 `k` 个节点。若 `ahead` 还没走满 `k` 步就到了末尾，说明 `k` 比长度大。

## 演算示例

链表 `9 -> 4 -> 6 -> 2 -> 8`，求倒数第 2 个。尾节点 8 是倒数第 1，所以答案应是 2。

1. `ahead` 先走 2 步，停在 6；`behind` 仍在 9。
2. 同步移动：`ahead` 到 2，`behind` 到 4。
3. `ahead` 到 8，`behind` 到 6。
4. `ahead` 再到空，`behind` 到 2。返回值为 2 的节点。

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
    def FindKthToTail(self, head, k):
        if k <= 0 or head is None:
            return None
        ahead = head
        for _ in range(k):
            if ahead is None:
                return None
            ahead = ahead.next
        behind = head
        while ahead:
            ahead = ahead.next
            behind = behind.next
        return behind


if __name__ == "__main__":
    solver = Solution()
    head = build_chain([9, 4, 6, 2, 8])
    node = solver.FindKthToTail(head, 2)
    assert node is not None and node.val == 2
    assert solver.FindKthToTail(head, 1).val == 8
    assert solver.FindKthToTail(head, 5).val == 9
    assert solver.FindKthToTail(head, 6) is None
    assert solver.FindKthToTail(head, 0) is None
    assert solver.FindKthToTail(None, 1) is None
    print(node.val)
```

## 复杂度

双指针时间 `O(n)`，只用常数额外空间。先存进列表再取下标，时间同样 `O(n)`，空间 `O(n)`。

## 反例与边界

- `k = 0` 或负数没有「倒数第 0 个」这种位置，返回空。
- `k` 比节点数大，例如上面长度为 5 却要倒数第 6 个，`ahead` 在预走阶段就会变成空，返回空。
- 若两个指针一开始就同步走、没有先让一个指针领先 `k` 步，结束时后指针仍在头节点，倒数第 2 个会被错认成 9。
