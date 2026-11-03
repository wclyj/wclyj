# 二叉树的右视图

站在二叉树右侧往左看，每一层只能看见最靠右的那个结点。从上到下收集这些值。

## 思路

只沿着右孩子往下走是不够的。某一层的最右结点可能没有孩子，而更深层的可见结点来自它左边的兄弟子树。

可靠做法仍是按层遍历：每一层最后一个弹出的结点就是这一层从右边能看见的那个。层序用队列，每轮处理固定个数。

## 演算

```
        6
       / \
      2   9
     /   /
    3   4
     \
      5
```

- 第 1 层只有 6，看见 6。
- 第 2 层是 2、9，最右是 9。
- 第 3 层是 3、4，最右是 4。9 的左孩子是 4，2 的左孩子是 3。
- 第 4 层只有 5。它挂在左侧结点 3 的右孩子上，但从这一层看，它是唯一可见的。

右视图 `[6, 9, 4, 5]`。若只顺着 6→9→4 走，会漏掉 5。

## 代码

```python
from collections import deque

class Solution:
    def rightSideView(self, root):
        if root is None:
            return []
        waiting = deque([root])
        view = []
        while waiting:
            width = len(waiting)
            for idx in range(width):
                cur = waiting.popleft()
                if idx == width - 1:
                    view.append(cur.val)
                if cur.left is not None:
                    waiting.append(cur.left)
                if cur.right is not None:
                    waiting.append(cur.right)
        return view
```

## 复杂度

每个结点进出队列一次，时间 `O(n)`。队列宽度最多 `O(n)`。

## 边界

空树返回空列表。树退化成一条只有左孩子的链时，每一层的「最右」就是那个左孩子，结果应包含整条链，不能因为没有右孩子就停下来。层序若先压右孩子再压左孩子，本层最后一个弹出的仍应是视觉上的最右，入队顺序要和「最后一个」的定义一致。只有右孩子的链，每一层的可见值就是这些右孩子，结果长度等于树高。空结点不要入队。
