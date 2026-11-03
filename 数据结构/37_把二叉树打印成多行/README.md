# 把二叉树打印成多行

## 题目描述

按层打印二叉树，每一层单独一行，同一层从左到右。返回二维列表。

## 思路

和之字形打印不同，这里方向始终从左到右，所以用队列而不是栈。可以用两个队列交替：当前队列负责本层，孩子一律放进另一个队列。当前队列清空时，这一层就结束了。

也可以只用一个队列，在每层开始时记下这一层的节点个数，然后正好弹出这么多个，孩子留到下一层。

## 演算示例

仍用这棵树：

```text
        10
       /  \
      6    14
     / \   / \
    4   8 12  16
```

1. 队列里是 10，弹出后依次放入 6、14。第一行 `[10]`。
2. 弹出 6，放入 4、8；弹出 14，放入 12、16。第二行 `[6, 14]`。
3. 四个叶子都没有孩子。第三行 `[4, 8, 12, 16]`。

结果：`[[10], [6, 14], [4, 8, 12, 16]]`。和第二层要从右往左的之字形不同，这里 6 在 14 前面。

## 代码

```python
class TreeNode:
    def __init__(self, value):
        self.val = value
        self.left = None
        self.right = None


class Solution:
    def Print(self, root):
        if root is None:
            return []
        current = [root]
        levels = []
        while current:
            nxt = []
            level = []
            for node in current:
                level.append(node.val)
                if node.left:
                    nxt.append(node.left)
                if node.right:
                    nxt.append(node.right)
            levels.append(level)
            current = nxt
        return levels


if __name__ == "__main__":
    root = TreeNode(10)
    root.left = TreeNode(6)
    root.right = TreeNode(14)
    root.left.left = TreeNode(4)
    root.left.right = TreeNode(8)
    root.right.left = TreeNode(12)
    root.right.right = TreeNode(16)
    assert Solution().Print(root) == [[10], [6, 14], [4, 8, 12, 16]]
    assert Solution().Print(None) == []
    assert Solution().Print(TreeNode(5)) == [[5]]
    print(Solution().Print(root))
```

## 复杂度

每个节点进入下一层列表一次，时间 `O(n)`，空间 `O(n)`。

## 反例与边界

- 空树返回空列表，而不是包含一个空行的 `[[]]`。
- 单节点只有一行 `[5]`。
- 若从栈顶弹出，第二行会变成 `[14, 6]`。那是之字形的第二层，不符合「同一层从左到右」。
