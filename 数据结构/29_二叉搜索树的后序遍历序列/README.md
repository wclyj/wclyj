# 判断序列能否作为搜索树的后序

## 题目描述

给定一个整数序列，判断它能不能成为某棵二叉搜索树的后序遍历。二叉搜索树满足：左子树的值都小于根，右子树的值都大于根，左右子树也是二叉搜索树。序列里的数互不相同。空序列按「不是」处理。

## 思路

后序遍历的最后一个数是根。从左往右扫，第一段应全小于根，那是左子树；剩下到根之前的一段应全大于根，那是右子树。一旦在右段里发现比根小的数，这个序列就不可能是后序。

左右两段再各自递归检查。段的长度不超过 1 时，单节点或空树都合法。

## 演算示例

序列 `[1, 3, 2, 6, 5]`。根是 5。

- `1, 3, 2` 都小于 5，作为左子树。
- `6` 大于 5，作为右子树。
- 左段 `[1, 3, 2]` 的根是 2，左边 `[1]` 小于 2，右边 `[3]` 大于 2。
- 右段只有 6。

对应的树是：

```text
      5
     / \
    2   6
   / \
  1   3
```

所以结果为真。

再看 `[3, 1, 2, 6, 4]`。根是 4。开头 3 已经大于 4？3 小于 4，1 小于 4，2 小于 4，然后 6 大于 4。左段是 `[3, 1, 2]`，右段是 `[6]`。检查左段时根是 2，第一个数 3 已经大于 2，于是左段为空、右段是 `[3, 1]`。右段里的 1 小于根 2，规则被破坏，返回假。

## 代码

```python
class Solution:
    def VerifySquenceOfBST(self, sequence):
        if not sequence:
            return False

        def valid(seq):
            if len(seq) <= 1:
                return True
            root = seq[-1]
            split = 0
            while split < len(seq) - 1 and seq[split] < root:
                split += 1
            for value in seq[split:-1]:
                if value < root:
                    return False
            return valid(seq[:split]) and valid(seq[split:-1])

        return valid(sequence)


if __name__ == "__main__":
    solver = Solution()
    assert solver.VerifySquenceOfBST([1, 3, 2, 6, 5]) is True
    assert solver.VerifySquenceOfBST([3, 1, 2, 6, 4]) is False
    assert solver.VerifySquenceOfBST([]) is False
    assert solver.VerifySquenceOfBST([8]) is True
    print(solver.VerifySquenceOfBST([1, 3, 2, 6, 5]))
```

## 复杂度

每一层把序列分成两段，划分本身要扫描当前段。最坏是链状分割，时间 `O(n^2)`，递归深度 `O(n)`。

## 反例与边界

- `[3, 1, 2, 6, 4]` 必须返回假。6 可以当 4 的右孩子，但左段 `[3, 1, 2]` 自己不是合法后序。
- 空序列返回假。单节点序列返回真，它是一棵只有根的二叉搜索树。
- 若只检查「左段小于根、右段大于根」，不再递归，像 `[3, 1, 2, 6, 4]` 这种内层非法的序列会被放过。
