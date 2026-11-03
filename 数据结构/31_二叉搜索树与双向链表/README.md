# 二叉搜索树与双向链表

## 题目描述

把一棵二叉搜索树改成一条从小到大的双向链表。不能新建节点，只能改现有节点的指针。左指针当作前驱，右指针当作后继，返回最小节点，也就是链表头。

## 思路

二叉搜索树的中序遍历就是升序。按中序走到每个节点时，让它和「上一个访问过的节点」互相指向：上一个节点的右指针指向当前，当前的左指针指向上一个。第一个被访问的节点没有前驱，它就是链表头。

右指针要等左子树处理完再接，否则会把还没遍历的右子树提前拆掉。中序正好先处理左子树，再改当前节点，最后进入右子树，顺序是安全的。

## 演算示例

```text
      6
     / \
    2   8
   / \
  1   4
```

中序是 `1, 2, 4, 6, 8`。连接过程：

1. 访问 1，它成为头，前驱为空。
2. 访问 2，1 的右指向 2，2 的左指向 1。
3. 访问 4，2 与 4 互相指向。
4. 访问 6，4 与 6 互相指向。
5. 访问 8，6 与 8 互相指向。

向右走得到 `1 -> 2 -> 4 -> 6 -> 8`，从 8 再向左走可以回到 1。

## 代码

```python
class TreeNode:
    def __init__(self, value):
        self.val = value
        self.left = None
        self.right = None


class Solution:
    def Convert(self, root):
        head = None
        previous = None

        def visit(node):
            nonlocal head, previous
            if node is None:
                return
            visit(node.left)
            if previous is None:
                head = node
            else:
                previous.right = node
                node.left = previous
            previous = node
            visit(node.right)

        visit(root)
        return head


def forward(head):
    values = []
    while head:
        values.append(head.val)
        head = head.right
    return values


def backward(head):
    if head is None:
        return []
    while head.right:
        head = head.right
    values = []
    while head:
        values.append(head.val)
        head = head.left
    return values


if __name__ == "__main__":
    root = TreeNode(6)
    root.left = TreeNode(2)
    root.right = TreeNode(8)
    root.left.left = TreeNode(1)
    root.left.right = TreeNode(4)
    head = Solution().Convert(root)
    assert forward(head) == [1, 2, 4, 6, 8]
    assert backward(head) == [8, 6, 4, 2, 1]
    assert Solution().Convert(None) is None
    print(forward(head))
```

## 复杂度

每个节点访问一次，时间 `O(n)`。递归栈最坏 `O(n)`，没有额外节点。

## 反例与边界

- 空树返回空。
- 只有右子树时，头仍然是最小的根。例如根 2、右孩子 5，链表应是 `2 <-> 5`，不能把 5 当成头。
- 若在进入左子树之前就把右指针改掉，左子树的入口会丢失，中序无法继续。
