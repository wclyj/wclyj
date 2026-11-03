# 从上往下打印二叉树

## 题目描述

按层次从上到下访问二叉树，同一层从左到右。返回所有节点值组成的一维列表。

## 思路

这是层序遍历，用队列。先放入根。每次取出队头，记下它的值，再把它的左孩子、右孩子依次放进队尾。队列保证先进入的那一层先被处理，同一层又因为先左后右而保持从左到右。

用列表模拟队列时，要从头部取出。若用栈从尾部弹出，就会变成深度优先。

## 演算示例

```text
      5
     / \
    2   8
       / \
      6   9
```

1. 队列 `[5]`，取出 5，放入 2 和 8。结果 `[5]`。
2. 取出 2，它没有孩子。结果 `[5, 2]`，队列 `[8]`。
3. 取出 8，放入 6 和 9。结果 `[5, 2, 8]`。
4. 取出 6、9。最终 `[5, 2, 8, 6, 9]`。

## 代码

```python
class TreeNode:
    def __init__(self, value):
        self.val = value
        self.left = None
        self.right = None


class Solution:
    def PrintFromTopToBottom(self, root):
        if root is None:
            return []
        queue = [root]
        result = []
        while queue:
            node = queue.pop(0)
            result.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        return result


if __name__ == "__main__":
    root = TreeNode(5)
    root.left = TreeNode(2)
    root.right = TreeNode(8)
    root.right.left = TreeNode(6)
    root.right.right = TreeNode(9)
    assert Solution().PrintFromTopToBottom(root) == [5, 2, 8, 6, 9]
    assert Solution().PrintFromTopToBottom(None) == []
    only = TreeNode(4)
    assert Solution().PrintFromTopToBottom(only) == [4]
    print(Solution().PrintFromTopToBottom(root))
```

## 复杂度

每个节点进出队列一次，时间 `O(n)`。队列里最多放下一层，最坏空间 `O(n)`。若用列表 `pop(0)`，单次出队是线性的，总时间会退化；节点很多时应改用双端队列的左侧弹出。

## 反例与边界

- 空树返回空列表。
- 只有根时，结果是只含根值的列表，不能再去访问空的左右孩子。
- 上面的树若从栈顶弹出，可能先走到 8 再走到 2，得到的不是从左到右的 `[5, 2, 8, 6, 9]`。
