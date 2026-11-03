# 二叉树的层次遍历

按层收集结点：同一层从左到右排成一行，上层整行写完再写下层。空树没有层，结果是空列表。

## 思路

用队列做广度优先。队列里先放根。每一轮先记下当前队列长度，这个长度就是本层结点个数，只弹出这么多个，它们的左右孩子进队，留给下一轮。这样不用额外标记层号，层与层自然分开。

也可以左右两条链表来回倒，但 `pop(0)` 在普通列表上是线性开销，队列的左端弹出才是常数时间。

## 演算

树如下（数字是结点值）：

```
      4
     / \
    2   7
   /   / \
  1   6   9
```

- 队列开始只有 4。弹出后记下 `[4]`，2 和 7 入队。
- 本层长度 2。先弹出 2，它的左孩子 1 入队；再弹出 7，6 和 9 入队。本层 `[2, 7]`。
- 本层长度 3，依次弹出 1、6、9，都没有孩子。本层 `[1, 6, 9]`。
- 队列空，结束。

结果：`[[4], [2, 7], [1, 6, 9]]`。

## 代码

```python
from collections import deque

class Solution:
    def levelOrder(self, root):
        if root is None:
            return []
        waiting = deque([root])
        layers = []
        while waiting:
            # 此刻队列里的结点恰好构成完整的一层
            width = len(waiting)
            row = []
            for _ in range(width):
                cur = waiting.popleft()
                row.append(cur.val)
                if cur.left is not None:
                    waiting.append(cur.left)
                if cur.right is not None:
                    waiting.append(cur.right)
            layers.append(row)
        return layers
```

## 复杂度

设结点数为 `n`。每个结点入队、出队各一次，时间 `O(n)`。队列最宽时接近满二叉树的最后一层，空间 `O(n)`。

## 边界

根为空时直接返回 `[]`，不要去读 `val`。只有根、没有孩子时，结果是 `[[根的值]]`，循环仍能正确结束。
