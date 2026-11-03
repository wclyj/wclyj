# 重建二叉树

## 题目描述

已知一棵二叉树的先序遍历和中序遍历，且数值没有重复。根据这两个序列把树还原出来，返回根节点。

## 思路

先序的第一个值一定是根。到中序里找到这个根，它左边是左子树的中序，右边是右子树的中序。左子树的节点个数也由此确定，于是先序里根后面同样个数的一段就是左子树的先序，再后面是右子树的先序。

左右两段各自再按同样规则递归。为了避免每次线性查找根的位置，先把中序的值和下标放进字典。

## 演算示例

先序 `[6, 3, 1, 4, 8, 7, 9]`，中序 `[1, 3, 4, 6, 7, 8, 9]`。

1. 根是 6。中序里 6 的左边是 `[1, 3, 4]`，右边是 `[7, 8, 9]`。
2. 左子树先序是 `[3, 1, 4]`。根 3，中序左边 `[1]`、右边 `[4]`，所以 3 的左孩子是 1，右孩子是 4。
3. 右子树先序是 `[8, 7, 9]`。根 8，中序左边 `[7]`、右边 `[9]`。

还原出来的形状：

```text
      6
     / \
    3   8
   / \ / \
  1  4 7  9
```

## 代码

```python
class TreeNode:
    def __init__(self, value):
        self.val = value
        self.left = None
        self.right = None


def preorder_of(root):
    if root is None:
        return []
    return [root.val] + preorder_of(root.left) + preorder_of(root.right)


def inorder_of(root):
    if root is None:
        return []
    return inorder_of(root.left) + [root.val] + inorder_of(root.right)


class Solution:
    def reConstructBinaryTree(self, preorder, inorder):
        if not preorder or not inorder or len(preorder) != len(inorder):
            return None
        index = {value: pos for pos, value in enumerate(inorder)}
        if any(value not in index for value in preorder):
            return None

        def build(pre_left, pre_right, in_left, in_right):
            if pre_left > pre_right:
                return None
            root_value = preorder[pre_left]
            root = TreeNode(root_value)
            mid = index[root_value]
            left_count = mid - in_left
            root.left = build(pre_left + 1, pre_left + left_count, in_left, mid - 1)
            root.right = build(pre_left + left_count + 1, pre_right, mid + 1, in_right)
            return root

        return build(0, len(preorder) - 1, 0, len(inorder) - 1)


if __name__ == "__main__":
    preorder = [6, 3, 1, 4, 8, 7, 9]
    inorder = [1, 3, 4, 6, 7, 8, 9]
    root = Solution().reConstructBinaryTree(preorder, inorder)
    assert preorder_of(root) == preorder
    assert inorder_of(root) == inorder
    assert Solution().reConstructBinaryTree([], [1]) is None
    assert Solution().reConstructBinaryTree([1, 2], [1, 3]) is None
    print(preorder_of(root), inorder_of(root))
```

## 复杂度

每个节点创建一次，字典查找是常数时间，总时间 `O(n)`，递归栈和字典空间 `O(n)`。

## 反例与边界

- 任一序列为空，或两条长度不同，无法唯一还原，返回空。
- 先序 `[1, 2]`、中序 `[1, 3]` 的数值对不上。2 不在中序里，不能硬建出一棵树，返回空。
- 只有一个节点时，左右区间都是空，根的左右孩子都是空，这是合法的叶子。
