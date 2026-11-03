# 对称的二叉树

## 题目描述

判断一棵二叉树是不是对称的。对称指它和自己的镜像完全一样：左子树能按镜像方式对上右子树。

## 思路

镜像比较两个节点 `left` 和 `right`：

- 两个都空，对称。
- 只有一个空，或者值不同，不对称。
- 值相同，则继续比较 `left.left` 与 `right.right`，以及 `left.right` 与 `right.left`。外侧对外侧，内侧对内侧。

整棵树对称，当且仅当根的左子树和右子树满足上面的关系。空树可以看作对称。

## 演算示例

对称的一棵：

```text
      4
     / \
    2   2
   / \ / \
  1  3 3  1
```

根的左右都是 2。外侧 `1` 对 `1`，内侧 `3` 对 `3`，递归到底都是空，结果为真。

不对称的一棵：

```text
      4
     / \
    2   2
   / \   \
  1   3   3
```

左边的 2 有左孩子 1，右边的 2 没有右孩子。外侧比较时一边是 1、一边是空，结果为假。

## 代码

```python
class TreeNode:
    def __init__(self, value):
        self.val = value
        self.left = None
        self.right = None


class Solution:
    def isSymmetrical(self, root):
        def mirror(left, right):
            if left is None and right is None:
                return True
            if left is None or right is None or left.val != right.val:
                return False
            return mirror(left.left, right.right) and mirror(left.right, right.left)

        if root is None:
            return True
        return mirror(root.left, root.right)


if __name__ == "__main__":
    yes = TreeNode(4)
    yes.left = TreeNode(2)
    yes.right = TreeNode(2)
    yes.left.left = TreeNode(1)
    yes.left.right = TreeNode(3)
    yes.right.left = TreeNode(3)
    yes.right.right = TreeNode(1)

    no = TreeNode(4)
    no.left = TreeNode(2)
    no.right = TreeNode(2)
    no.left.left = TreeNode(1)
    no.left.right = TreeNode(3)
    no.right.right = TreeNode(3)

    solver = Solution()
    assert solver.isSymmetrical(yes) is True
    assert solver.isSymmetrical(no) is False
    assert solver.isSymmetrical(None) is True
    assert solver.isSymmetrical(TreeNode(7)) is True
    print(solver.isSymmetrical(yes), solver.isSymmetrical(no))
```

## 复杂度

每个节点最多参与一次比较，时间 `O(n)`，递归深度 `O(n)`。

## 反例与边界

- 上面第二棵树左右根值都是 2，但形状不镜像，必须返回假。只比较左右子树的根会误判。
- 空树返回真。单节点没有孩子，左右都是空，也是对称的。
- 比较时若把 `left.left` 去对 `right.left`，那是在判断两棵子树是否相同，不是是否镜像。对称例子里内侧 3 对外侧 1 会失败，真对称的树也会被判成假。
