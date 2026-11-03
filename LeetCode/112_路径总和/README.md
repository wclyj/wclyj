# 路径总和

问二叉树里有没有一条从根走到叶子的路径，沿途结点值加起来恰好等于给定目标和。叶子指左右都空的结点。中途某个内部结点的前缀和碰巧等于目标，不能算成功。

## 思路

自顶向下把目标和减去当前结点值，把剩余数传给孩子。到了叶子，剩余数等于叶子值才成功。左右子树只要有一边成功即可。

结点值可以是负数，剩余数变负也不能提前停，否则会漏掉后面用负数补回来的路径。

## 演算

```
      3
     / \
    1   8
   /   /
  4   2
```

目标和取 `13`。

- 根 3，剩余 `10`。左支 `10 - 1 = 9`，再到叶子 4，`9 != 4`，这条失败。
- 右支 `10 - 8 = 2`，叶子 2 正好相等。路径 `3 → 8 → 2` 成立，返回真。

若目标和改成 `10`，三条根到叶的和分别是 `8`、`11`、`13`，没有 `10`，返回假。

## 代码

```python
class Solution:
    def hasPathSum(self, root, targetSum):
        if root is None:
            return False
        # 叶子：没有可以再往下走的边
        if root.left is None and root.right is None:
            return root.val == targetSum
        remain = targetSum - root.val
        return self.hasPathSum(root.left, remain) or self.hasPathSum(root.right, remain)
```

## 复杂度

最坏要看过所有结点，时间 `O(n)`。递归深度等于树高，链状树时栈空间 `O(n)`，平衡时 `O(log n)`。

## 边界

空树没有任何根到叶的路径，返回假。单结点树当且仅当该值等于目标和时为真。只有左孩子的结点不是叶子，必须继续往下走。
