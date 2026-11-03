# 二叉树的镜像

## 题目描述

把一棵二叉树变成它的镜像：每个节点的左右孩子对调，对调要作用到所有后代。返回镜像的根，根的值不变。

## 思路

对当前节点交换左右孩子，再对交换后的左右子树做同样的事。叶子的两个孩子都是空，交换没有可见变化，递归在空节点停住。

也可以用栈做循环：弹出一个节点，先把它的非空孩子压栈，再交换它的左右指针。孩子在交换前压栈，避免漏掉原来的某一侧。

## 演算示例

原树：

```text
      4
     / \
    2   7
   / \ / \
  1  3 6  9
```

镜像：

```text
      4
     / \
    7   2
   / \ / \
  9  6 3  1
```

先交换 4 的左右，7 跑到左边、2 跑到右边。再交换 7 的孩子，9 到左、6 到右。最后交换 2 的孩子，3 到左、1 到右。

## 代码

```python
class TreeNode:
    def __init__(self, value):
        self.val = value
        self.left = None
        self.right = None


def preorder(root):
    if root is None:
        return []
    return [root.val] + preorder(root.left) + preorder(root.right)


class Solution:
    def Mirror(self, root):
        if root is None:
            return None
        root.left, root.right = root.right, root.left
        self.Mirror(root.left)
        self.Mirror(root.right)
        return root

    def mirror_iterative(self, root):
        if root is None:
            return None
        stack = [root]
        while stack:
            node = stack.pop()
            if node.left:
                stack.append(node.left)
            if node.right:
                stack.append(node.right)
            node.left, node.right = node.right, node.left
        return root


def sample():
    root = TreeNode(4)
    root.left = TreeNode(2)
    root.right = TreeNode(7)
    root.left.left = TreeNode(1)
    root.left.right = TreeNode(3)
    root.right.left = TreeNode(6)
    root.right.right = TreeNode(9)
    return root


if __name__ == "__main__":
    mirrored = Solution().Mirror(sample())
    assert preorder(mirrored) == [4, 7, 9, 6, 2, 3, 1]
    again = Solution().mirror_iterative(sample())
    assert preorder(again) == [4, 7, 9, 6, 2, 3, 1]
    assert Solution().Mirror(None) is None
    print(preorder(mirrored))
```

## 复杂度

每个节点交换一次，时间 `O(n)`。递归或栈的空间最坏 `O(n)`。

## 反例与边界

- 空树返回空。
- 只有左孩子、没有右孩子时，交换后左孩子变空、原左孩子移到右边。例如根 4 只有左孩子 2，镜像后 2 必须在右边。若只在「左右都存在」时才交换，这种树不会变化。
- 交换两次应回到原树。若第一次交换时把孩子节点本身的值也改了，第二次无法还原。
