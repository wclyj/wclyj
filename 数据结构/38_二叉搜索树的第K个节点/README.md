# 二叉搜索树的第 K 个节点

## 题目描述

给定一棵二叉搜索树和正整数 `k`，返回第 `k` 小的节点。`k` 从 1 开始。`k` 小于 1，或者大于节点个数时，返回空。

## 思考

二叉搜索树的中序遍历是严格升序（本题节点值互不相同）。因此第 `k` 小的节点，就是中序序列里的第 `k` 个。可以完整走一遍再取下标 `k-1`，也可以在中序过程中计数，数到 `k` 立刻返回，后面的节点不用再看。

## 演算示例

```text
      6
     / \
    3   8
   / \   \
  1   4   9
```

中序：`1, 3, 4, 6, 8, 9`。

- 第 1 小是 1
- 第 3 小是 4
- 第 6 小是 9
- 第 7 小不存在

走到 4 时计数器变成 3，若 `k = 3` 就可以停，不必再访问 6、8、9。

## 代码

```python
class TreeNode:
    def __init__(self, value):
        self.val = value
        self.left = None
        self.right = None


class Solution:
    def KthNode(self, root, k):
        if k < 1:
            return None
        self.remain = k
        self.found = None

        def visit(node):
            if node is None or self.found is not None:
                return
            visit(node.left)
            if self.found is not None:
                return
            self.remain -= 1
            if self.remain == 0:
                self.found = node
                return
            visit(node.right)

        visit(root)
        return self.found


def sample():
    root = TreeNode(6)
    root.left = TreeNode(3)
    root.right = TreeNode(8)
    root.left.left = TreeNode(1)
    root.left.right = TreeNode(4)
    root.right.right = TreeNode(9)
    return root


if __name__ == "__main__":
    root = sample()
    solver = Solution()
    assert solver.KthNode(root, 3).val == 4
    assert solver.KthNode(root, 1).val == 1
    assert solver.KthNode(root, 6).val == 9
    assert solver.KthNode(root, 7) is None
    assert solver.KthNode(root, 0) is None
    print(solver.KthNode(root, 3).val)
```

## 复杂度

最坏仍要看完所有节点，时间 `O(n)`，递归栈 `O(n)`。`k` 很小时可以提前停止，实际访问量接近第 `k` 个节点之前的中序前缀。

## 反例与边界

- `k = 0` 返回空。中序下标从 0 开始，若直接取 `k` 而不是 `k-1`，第 3 小会错拿到 6。
- `k` 比节点数大，计数器减不到 0，返回空。
- 先序或后序的第 `k` 个不是第 `k` 小。这棵树的先序第 3 个是 1，而第 3 小是 4。
