# 树的遍历

## 题目描述

这一节整理二叉树的三种深度优先遍历：先序、中序、后序。每种都给出递归写法和用栈改写的循环写法，并用同一棵树把顺序算清楚。

树由有限个节点组成，节点之间有父子层次。没有父节点的是根，没有孩子的是叶子。二叉树里每个节点最多两个孩子，习惯叫左孩子和右孩子。

## 思路

记「访问根」为 D，「走左子树」为 L，「走右子树」为 R。三种顺序就是 DLR、LDR、LRD。递归写法按这个顺序调用即可。

循环写法用栈保存还没处理完的节点：

- 先序：弹出就访问，然后先压右孩子、再压左孩子，这样左孩子会先被弹出。
- 中序：沿着左链一路压栈，弹出来时访问，再转向右孩子。
- 后序：可以按「根、右、左」压栈，最后把结果倒过来，就成了「左、右、根」。

## 演算示例

用下面这棵树，不使用连续编号 1 到 8 的那棵示例树：

```text
      9
     / \
    4   12
   / \    \
  2   7    15
     /
    6
```

- 先序（根、左、右）：`9, 4, 2, 7, 6, 12, 15`
- 中序（左、根、右）：`2, 4, 6, 7, 9, 12, 15`
- 后序（左、右、根）：`2, 6, 7, 4, 15, 12, 9`

中序的手工过程：从 9 走到 4 再走到 2，2 没有左孩子，先记下 2，回到 4，记下 4，再进入 7，7 的左边是 6，记下 6、7，回到 9，记下 9，最后是 12 和它的右孩子 15。

## 代码

```python
class TreeNode:
    def __init__(self, value):
        self.val = value
        self.left = None
        self.right = None


def sample_tree():
    root = TreeNode(9)
    root.left = TreeNode(4)
    root.right = TreeNode(12)
    root.left.left = TreeNode(2)
    root.left.right = TreeNode(7)
    root.left.right.left = TreeNode(6)
    root.right.right = TreeNode(15)
    return root


def preorder(root):
    if root is None:
        return []
    stack = [root]
    result = []
    while stack:
        node = stack.pop()
        result.append(node.val)
        if node.right:
            stack.append(node.right)
        if node.left:
            stack.append(node.left)
    return result


def inorder(root):
    result = []
    stack = []
    current = root
    while current or stack:
        while current:
            stack.append(current)
            current = current.left
        current = stack.pop()
        result.append(current.val)
        current = current.right
    return result


def postorder(root):
    if root is None:
        return []
    stack = [root]
    result = []
    while stack:
        node = stack.pop()
        result.append(node.val)
        if node.left:
            stack.append(node.left)
        if node.right:
            stack.append(node.right)
    result.reverse()
    return result


def preorder_recursive(root, result):
    if root is None:
        return
    result.append(root.val)
    preorder_recursive(root.left, result)
    preorder_recursive(root.right, result)


if __name__ == "__main__":
    root = sample_tree()
    assert preorder(root) == [9, 4, 2, 7, 6, 12, 15]
    assert inorder(root) == [2, 4, 6, 7, 9, 12, 15]
    assert postorder(root) == [2, 6, 7, 4, 15, 12, 9]
    recursive = []
    preorder_recursive(root, recursive)
    assert recursive == preorder(root)
    assert preorder(None) == []
    assert inorder(None) == []
    assert postorder(None) == []
    print(preorder(root))
    print(inorder(root))
    print(postorder(root))
```

## 复杂度

每个节点入栈出栈各常数次，三种循环遍历的时间都是 `O(n)`，栈的空间最坏 `O(n)`。递归的调用深度在退化为链表时也是 `O(n)`。

## 反例与边界

- 空树三种遍历都得到空列表。若一上来就读 `root.val`，会出错。
- 后序如果在「右孩子还没访问」时就把节点弹出并打印，根会跑到右子树前面。上面这棵树会把 9 提前打出来，不再是 `2, 6, 7, 4, 15, 12, 9`。
- 先序压栈时若先压左再压右，弹出顺序会变成先右后左。这棵树的第一步就会先访问 12 而不是 4。
