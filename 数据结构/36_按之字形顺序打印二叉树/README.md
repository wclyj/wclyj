# 按之字形顺序打印二叉树

## 题目描述

按层打印二叉树，但相邻两层方向相反。第一层从左到右，第二层从右到左，第三层再从左到右，之后继续交替。每一层单独放在一个列表里。

## 思路

用两个栈轮流保存下一层。栈会把放入顺序倒过来，正好用来换方向。

从第一层的栈弹出时，先压左孩子、再压右孩子到另一个栈。弹出顺序是从左到右，孩子被倒序后，下一层弹出时就是从右到左。

处理第二层时相反：先压右孩子、再压左孩子。这样再下一层弹出时又回到从左到右。

## 演算示例

```text
        10
       /  \
      6    14
     / \   / \
    4   8 12  16
```

1. 栈甲弹出 10。先压 6、再压 14 到栈乙。本层 `[10]`。栈乙从顶到底是 14、6。
2. 弹出 14，本层先记下 14，并按「先右后左」把 16、12 压进栈甲。再弹出 6，记下 6，把 8、4 压进栈甲。本层 `[14, 6]`。栈甲弹出顺序将是 4、8、12、16。
3. 叶子依次弹出，本层 `[4, 8, 12, 16]`。

结果：`[[10], [14, 6], [4, 8, 12, 16]]`。

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
        left_to_right = [root]
        right_to_left = []
        levels = []
        while left_to_right or right_to_left:
            if left_to_right:
                level = []
                while left_to_right:
                    node = left_to_right.pop()
                    level.append(node.val)
                    if node.left:
                        right_to_left.append(node.left)
                    if node.right:
                        right_to_left.append(node.right)
                levels.append(level)
            if right_to_left:
                level = []
                while right_to_left:
                    node = right_to_left.pop()
                    level.append(node.val)
                    if node.right:
                        left_to_right.append(node.right)
                    if node.left:
                        left_to_right.append(node.left)
                levels.append(level)
        return levels


if __name__ == "__main__":
    root = TreeNode(10)
    root.left = TreeNode(6)
    root.right = TreeNode(14)
    root.left.left = TreeNode(4)
    root.left.right = TreeNode(8)
    root.right.left = TreeNode(12)
    root.right.right = TreeNode(16)
    assert Solution().Print(root) == [[10], [14, 6], [4, 8, 12, 16]]
    assert Solution().Print(None) == []
    only = TreeNode(3)
    only.left = TreeNode(1)
    assert Solution().Print(only) == [[3], [1]]
    print(Solution().Print(root))
```

## 复杂度

每个节点进出栈一次，时间 `O(n)`，两个栈一起最多存放一层，空间 `O(n)`。

## 反例与边界

- 空树返回空列表。
- 只有左孩子的两层树，第二层从右到左仍然只有这一个孩子，结果是 `[[3], [1]]`。不能因为「从右往左」就把它丢掉。
- 第二层若仍然先压左再压右，弹出顺序会变成 `[6, 14]`，不再是从右到左。
