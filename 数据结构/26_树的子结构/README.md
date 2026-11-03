# 树的子结构

## 题目描述

给出两棵二叉树 A 和 B，判断 B 是不是 A 的子结构。子结构指 B 的节点能和 A 中某棵子树的对应位置重合，B 没画出来的孩子不要求 A 也没有。约定空树不是任何树的子结构。

## 思路

分两层递归。外层在 A 上找起点：先试着从 A 的当前节点匹配 B，不行就去 A 的左子树、右子树继续找。

内层比较两棵树是否逐位吻合：

- B 已经走完，说明 B 的节点都匹配上了，返回真。
- A 先走完而 B 还有节点，返回假。
- 当前值不同，返回假。
- 值相同，就继续比较「A 的左对 B 的左」以及「A 的右对 B 的右」。

## 演算示例

树 A：

```text
      8
     / \
    6   10
   / \
  4   7
```

树 B：

```text
    6
   / \
  4   7
```

A 的根 8 和 B 的根 6 不同。转到 A 的左孩子 6，与 B 的根相同。B 的左 4 对上 A 的 4，B 的右 7 对上 A 的 7。B 到此结束，所以 B 是 A 的子结构。

若把 B 的右孩子改成 9，6 对上之后右孩子 9 和 7 不同，A 的其它位置也没有另一棵 6，结果为假。

## 代码

```python
class TreeNode:
    def __init__(self, value):
        self.val = value
        self.left = None
        self.right = None


class Solution:
    def _matched(self, origin, pattern):
        if pattern is None:
            return True
        if origin is None or origin.val != pattern.val:
            return False
        return self._matched(origin.left, pattern.left) and self._matched(
            origin.right, pattern.right
        )

    def HasSubtree(self, root, pattern):
        if root is None or pattern is None:
            return False
        if self._matched(root, pattern):
            return True
        return self.HasSubtree(root.left, pattern) or self.HasSubtree(root.right, pattern)


def build_a():
    root = TreeNode(8)
    root.left = TreeNode(6)
    root.right = TreeNode(10)
    root.left.left = TreeNode(4)
    root.left.right = TreeNode(7)
    return root


if __name__ == "__main__":
    tree_a = build_a()
    pattern = TreeNode(6)
    pattern.left = TreeNode(4)
    pattern.right = TreeNode(7)
    assert Solution().HasSubtree(tree_a, pattern) is True

    partial = TreeNode(6)
    partial.left = TreeNode(4)
    assert Solution().HasSubtree(tree_a, partial) is True

    broken = TreeNode(6)
    broken.left = TreeNode(4)
    broken.right = TreeNode(9)
    assert Solution().HasSubtree(tree_a, broken) is False
    assert Solution().HasSubtree(tree_a, None) is False
    assert Solution().HasSubtree(None, pattern) is False
    print("ok")
```

## 复杂度

最坏情况下 A 的每个节点都作为起点，和 B 做一次比较。时间 `O(n·m)`，递归深度 `O(n + m)`。

## 反例与边界

- A 或 B 有一棵是空树，返回假。空树不拿来当子结构。
- B 只有 `6 -> 4`、没有右孩子时，仍然算子结构。不能要求 A 中对应的 6 也没有右孩子，那是「子树」而不是「子结构」。
- B 为 `6` 且右孩子是 9 时，即使左孩子 4 相同，整体也不是子结构。只比较根就返回真会误判。
