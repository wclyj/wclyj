# 二叉树的下一个节点

## 题目描述

二叉树的节点除了左右孩子，还有指向父节点的指针。给定其中一个节点，求出它在中序遍历里的后继。没有后继就返回空。

## 思路

中序是「左、根、右」。分两种情况：

1. 这个节点有右子树。后继在右子树里，而且是右子树中最靠左的那个节点。
2. 没有右子树。它要么是某个祖先的左子树里的最后一个，后继就是那个祖先；要么它位于整棵树的最右路径上，没有后继。沿着父指针往上，直到当前节点是父节点的左孩子，父节点就是答案。一直走到根仍不满足，则返回空。

## 演算示例

```text
      6
     / \
    3   8
   / \
  1   5
```

父指针：1 和 5 的父节点是 3，3 和 8 的父节点是 6。中序是 `1, 3, 5, 6, 8`。

- 1 没有右孩子，父节点 3 的左孩子正是 1，后继是 3。
- 3 有右孩子 5，5 没有更左的孩子，后继是 5。
- 5 没有右孩子。父节点 3 的左孩子不是 5，继续到 6。6 的左孩子是 3，而当前停在 3，所以后继是 6。
- 6 的右子树最左边是 8，后继是 8。
- 8 没有右孩子，父节点 6 的左孩子也不是 8，再往上没有节点，后继为空。

## 代码

```python
class TreeLinkNode:
    def __init__(self, value):
        self.val = value
        self.left = None
        self.right = None
        self.parent = None


class Solution:
    def GetNext(self, node):
        if node is None:
            return None
        if node.right:
            cursor = node.right
            while cursor.left:
                cursor = cursor.left
            return cursor
        cursor = node
        while cursor.parent:
            if cursor.parent.left is cursor:
                return cursor.parent
            cursor = cursor.parent
        return None


def sample():
    root = TreeLinkNode(6)
    left = TreeLinkNode(3)
    right = TreeLinkNode(8)
    far_left = TreeLinkNode(1)
    mid_right = TreeLinkNode(5)
    root.left = left
    root.right = right
    left.parent = root
    right.parent = root
    left.left = far_left
    left.right = mid_right
    far_left.parent = left
    mid_right.parent = left
    return far_left, left, mid_right, root, right


if __name__ == "__main__":
    node1, node3, node5, node6, node8 = sample()
    solver = Solution()
    assert solver.GetNext(node1) is node3
    assert solver.GetNext(node3) is node5
    assert solver.GetNext(node5) is node6
    assert solver.GetNext(node6) is node8
    assert solver.GetNext(node8) is None
    assert solver.GetNext(None) is None
    print(solver.GetNext(node5).val)
```

## 复杂度

最坏沿着一条链走到根或走到最左叶子，时间 `O(n)`，只用常数额外空间。

## 反例与边界

- 中序的最后一个节点，也就是整棵树最右边的节点，后继为空。上面的 8 就是这种情况。
- 空节点没有后继，返回空。
- 5 的后继不是它的父节点 3。3 在中序里已经访问过了。若看到没有右孩子就直接返回父节点，会把 5 的后继错当成 3。
