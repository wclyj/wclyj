# 根到叶子且节点和为定值的路径

## 题目描述

给定二叉树的根和一个整数目标。找出所有从根走到叶子的路径，使路径上的节点值之和等于目标。叶子是没有左右孩子的节点。返回这些路径，每条路径用节点值列表表示。

## 思考

从根往下走时，把当前节点放进路径，同时把目标减去当前值。到达叶子时，若剩余值正好等于叶子的值，就把路径复制一份存下来。然后弹出当前节点，回到上一层去试另一侧。这是深度优先，回溯保证兄弟节点不会共用一条被改坏的路径。

不能在非叶子处提前收割。即使走到某个中间节点时和已经等于目标，只要它还有孩子，就还不是一条完整路径。

## 演算示例

```text
      8
     / \
    4   9
   / \
  3   5
```

目标 17。三条走到叶子的路径：

- `8 -> 4 -> 3`，和是 15
- `8 -> 4 -> 5`，和是 17
- `8 -> 9`，和是 17，9 没有孩子，是叶子

所以答案是 `[[8, 4, 5], [8, 9]]`。

若在 9 下面再挂一个孩子 2，9 就不再是叶子。`8+9` 虽然等于 17，也不能收录；唯一剩下的叶子路径 `8 -> 9 -> 2` 的和是 19。

## 代码

```python
class TreeNode:
    def __init__(self, value):
        self.val = value
        self.left = None
        self.right = None


class Solution:
    def FindPath(self, root, expect_number):
        paths = []

        def walk(node, remain, path):
            if node is None:
                return
            path.append(node.val)
            if node.left is None and node.right is None and node.val == remain:
                paths.append(path[:])
            else:
                walk(node.left, remain - node.val, path)
                walk(node.right, remain - node.val, path)
            path.pop()

        walk(root, expect_number, [])
        return paths


def sample():
    root = TreeNode(8)
    root.left = TreeNode(4)
    root.right = TreeNode(9)
    root.left.left = TreeNode(3)
    root.left.right = TreeNode(5)
    return root


if __name__ == "__main__":
    found = Solution().FindPath(sample(), 17)
    assert found == [[8, 4, 5], [8, 9]]
    assert Solution().FindPath(None, 17) == []
    assert Solution().FindPath(sample(), 15) == [[8, 4, 3]]

    root = sample()
    root.right.left = TreeNode(2)
    assert Solution().FindPath(root, 17) == [[8, 4, 5]]
    print(found)
```

## 复杂度

每条边访问常数次，时间 `O(n)`。若结果里有很多条路径，复制路径的开销还要加上所有答案的总长度。递归深度最坏 `O(n)`。

## 反例与边界

- 空树没有任何路径，返回空列表。
- 目标 17 且 9 下面还有孩子 2 时，`[8, 9]` 不是根到叶子的路径，不能返回。只在「当前和等于目标」时就记录，会把这条半截路径错收进来。
- 叶子值本身等于目标、且它就是根，路径是只含这个根的列表。例如单节点 17、目标 17，答案是 `[[17]]`。
