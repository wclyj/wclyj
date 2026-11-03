# 复杂链表的复制

## 题目描述

复杂链表的每个节点有一个值、一个 `next`，还有一个 `random`。`random` 可以指向链表中任意节点，也可以为空。要深拷贝整条链表：新节点的 `next` 和 `random` 都指向新节点，不能把原链表里的节点交回去。

## 思考

`random` 可能指向还没复制到的节点，所以不能边建 `next` 边随便填 `random`。

一种原地拼接法分三步：

1. 在每个原节点后面插入一个值相同的副本，形成「原节点、副本、原节点、副本」。
2. 若原节点的 `random` 指向某节点，副本的 `random` 就指向那个节点后面的副本。
3. 把这条交织链表拆成两条：奇数位恢复原链表，偶数位串成新链表。

也可以用字典：先按 `next` 建出所有副本，再根据原节点到副本的映射去填 `random`。

## 演算示例

三个节点 A=4、B=7、C=2。`A.next = B`，`B.next = C`。`A.random = C`，`B.random = A`，`C.random = B`。

拼接后：`A -> A' -> B -> B' -> C -> C'`。

设置随机指针：`A'.random = A.random.next = C'`，`B'.random = A'`，`C'.random = B'`。

拆开后新链表是 `4 -> 7 -> 2`，随机指针分别落到值为 2、4、7 的新节点，原链表的连接保持不变。

## 代码

```python
class RandomListNode:
    def __init__(self, label):
        self.label = label
        self.next = None
        self.random = None


class Solution:
    def Clone(self, head):
        if head is None:
            return None
        # 第一步：每个原节点后面插入副本
        cursor = head
        while cursor:
            copy = RandomListNode(cursor.label)
            copy.next = cursor.next
            cursor.next = copy
            cursor = copy.next
        # 第二步：副本的 random 指向原 random 的副本
        cursor = head
        while cursor:
            if cursor.random:
                cursor.next.random = cursor.random.next
            cursor = cursor.next.next
        # 第三步：拆成原链表和副本链表
        cursor = head
        cloned = head.next
        while cursor:
            copy = cursor.next
            cursor.next = copy.next
            cursor = cursor.next
            if copy.next:
                copy.next = copy.next.next
        return cloned


def labels(head):
    values = []
    while head:
        values.append(head.label)
        head = head.next
    return values


if __name__ == "__main__":
    node_a = RandomListNode(4)
    node_b = RandomListNode(7)
    node_c = RandomListNode(2)
    node_a.next = node_b
    node_b.next = node_c
    node_a.random = node_c
    node_b.random = node_a
    node_c.random = node_b

    cloned = Solution().Clone(node_a)
    assert labels(node_a) == [4, 7, 2]
    assert labels(cloned) == [4, 7, 2]
    assert cloned is not node_a
    assert cloned.random is not node_c and cloned.random.label == 2
    assert cloned.next.random.label == 4
    assert cloned.next.next.random.label == 7
    assert Solution().Clone(None) is None
    print(labels(cloned))
```

## 复杂度

三趟扫描，时间 `O(n)`，副本节点本身是 `O(n)`，算法额外只用几个指针，辅助空间 `O(1)`。字典法则额外需要 `O(n)` 的映射表。

## 反例与边界

- 空链表返回空。
- `random` 为空时不能去取 `.next`，否则会报错。
- 若副本的 `random` 仍指向原节点，判题会认为没有真正深拷贝。上面例子里 `A'` 必须指向 `C'`，而不是原来的 C。
- 拆链时如果先把原节点的 `next` 断开、又没有保存副本，副本链表会断成孤立节点。
