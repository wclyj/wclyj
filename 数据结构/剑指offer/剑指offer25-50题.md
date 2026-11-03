# 剑指 Offer 25 到 31

这份原始材料在第 31 题结束，后面的题号没有正文，因此这里写到二叉搜索树转双向链表为止。

#### 树的知识点

##### 什么叫做树

树由有限个结点组成，结点之间有层级。上面的结点是父，下面的是子。没有父的那一个是根，没有子的是叶子。除根以外，每个结点恰好有一个父。一个结点和它所有后代又构成一棵子树，这些子树互不相交。

画出来根在上、叶在下，像倒着的树。

##### 什么是二叉树

每个结点最多两个孩子，习惯叫左子树和右子树。左右是有区别的，只有右孩子时不能把它画到左边。

根的深度记为 1。满二叉树把每一层都排满，深度 `k` 的结点数是 `2^k - 1`。完全二叉树只允许最后一层不满，而且最后一层必须从左往右连续。深度 `k` 的完全二叉树，结点数最少是 `2^(k-1)`（最后一层只有最左边一个），最多是 `2^k - 1`（其实已经满了）。

下面这棵树当公共例子。它是二叉搜索树：左子树的值都更小，右子树的值都更大。

```text
      6
     / \
    2   8
   / \   \
  1   4   9
```

##### 二叉树的遍历

遍历是把每个结点恰好访问一次，从而排成一个序列。记访问根为 D、走左子树为 L、走右子树为 R：

- 先序 DLR：根、左、右
- 中序 LDR：左、根、右
- 后序 LRD：左、右、根

「先、中、后」指的是根被访问的时机，左右的顺序始终是先左后右。

上图的三种顺序：

- 先序：`6, 2, 1, 4, 8, 9`
- 中序：`1, 2, 4, 6, 8, 9`
- 后序：`1, 4, 2, 9, 8, 6`

递归就是按这个定义直接写。空结点立刻返回。循环要自己用栈保存「还没走完的父结点」。

```python
class TreeNode:
    def __init__(self, val):
        self.val = val
        self.left = None
        self.right = None


def preorder(root):
    if root is None:
        return []
    out, stack = [], [root]
    while stack:
        node = stack.pop()
        out.append(node.val)
        if node.right:
            stack.append(node.right)
        if node.left:
            stack.append(node.left)
    return out


def inorder(root):
    out, stack, cur = [], [], root
    while cur or stack:
        while cur:
            stack.append(cur)
            cur = cur.left
        cur = stack.pop()
        out.append(cur.val)
        cur = cur.right
    return out


def postorder(root):
    """先按根、右、左压栈，最后整段倒序，得到左、右、根。"""
    if root is None:
        return []
    stack, out = [root], []
    while stack:
        node = stack.pop()
        out.append(node.val)
        if node.left:
            stack.append(node.left)
        if node.right:
            stack.append(node.right)
    out.reverse()
    return out
```

先序走一遍：栈里先放 6。弹出 6，先压右孩子 8 再压左孩子 2，下一次弹出的才是 2。接着弹出 2 的左右 1 和 4，然后才轮到 8 和 9。结果 `6, 2, 1, 4, 8, 9`。

后序若沿着左链一直压栈、看到右孩子又没有「已经走过」的标记，父结点会反复把自己的右孩子再压进去，停不下来。上面先收集「根右左」再倒序，每个结点只进栈一次。

### 复杂度

三种遍历都是时间 `O(n)`、栈空间 `O(n)`。最瘦的一条链上，栈里会同时有全部结点。

### 边界

空树三种都返回空列表。只有根时，三种顺序都是 `[根的值]`。

## 25.重建二叉树

### 题目

给出一棵二叉树的先序和中序，重建它。结点值互不重复。输入可以是先序 `6, 2, 1, 4, 8, 9` 和中序 `1, 2, 4, 6, 8, 9`，返回上图那棵树的根。

### 思路

先序的第一个值是根。到中序里找到它，左边是左子树的中序，右边是右子树的中序。左子树的结点数是 `mid` 个，所以先序里根后面的 `mid` 个是左子树的先序，再往后是右子树的先序。两棵子树用同一规则递归。

### 走一遍

根是 6，它在中序下标 3 处。左子树中序 `1, 2, 4`，先序 `2, 1, 4`。右子树中序 `8, 9`，先序 `8, 9`。

左子树再拆：根 2，中序下标 1，左边只剩 1，右边只剩 4。右子树根 8，左边空，右边 9。和示意图一致。

### 代码

```python
def rebuild(preorder, inorder):
    if not preorder or not inorder or len(preorder) != len(inorder):
        return None
    root = TreeNode(preorder[0])
    mid = inorder.index(preorder[0])
    root.left = rebuild(preorder[1:mid + 1], inorder[:mid])
    root.right = rebuild(preorder[mid + 1:], inorder[mid + 1:])
    return root
```

### 复杂度

每次用 `index` 找根是线性的，总时间 `O(n^2)`。若先把中序的值映射到下标，查找变成 `O(1)`，总时间 `O(n)`。切片会复制数组，空间另有 `O(n^2)` 的最坏拷贝；用下标区间描述子数组可以把这部分降到递归栈的 `O(n)`。

### 边界

空的先序或中序返回空。两段长度不同，说明输入对不上，返回空。值若有重复，`index` 会找到左边那一个，左右子树的切分就错了，这道题的前提是值互异。

## 26.树的子结构

### 题目

判断树 B 是不是树 A 的子结构。这里的子结构不要求 B 覆盖 A 的整棵子树，B 的结点在 A 里对上即可，A 还可以在对应位置多出 B 没有的孩子。约定空树不是任何人的子结构。

### 思路

两步。先在 A 里找一个值和 B 的根相同的结点 R。再判断以 R 为根的那一块，能不能按左右孩子把 B 盖住。

比较两棵树时：

- B 已经走完（空了）：这一支匹配成功。
- A 已经走完但 B 还有结点：失败。
- 值不同：失败。
- 否则左右孩子都要匹配。

这两个空判断的顺序不能颠倒。B 先结束应当成功；若先看 A 是不是空，B 也空的时候会被判失败。

A 上每一个和 B 根同值的结点都要试，试失败再去左右子树。

### 走一遍

A 用上面的 6。B 是 `2`，左孩子 `1`，右孩子 `4`。

根 6 不等于 2。左孩子 2 相等，再比左右：1 对 1，4 对 4，B 的孩子往下都是空，成功。

反例：B 仍以 2 为根，但左孩子是 4、没有右孩子。2 对上了，左孩子 1 对 4 失败。A 里没有另一个 2，整题为假。

### 代码

```python
def has_subtree(big, small):
    if big is None or small is None:
        return False

    def covers(host, pattern):
        if pattern is None:
            return True
        if host is None:
            return False
        if host.val != pattern.val:
            return False
        return covers(host.left, pattern.left) and covers(host.right, pattern.right)

    if covers(big, small):
        return True
    return has_subtree(big.left, small) or has_subtree(big.right, small)
```

### 复杂度

每个 A 的结点都可能和整棵 B 比较一次，时间 `O(|A|·|B|)`，递归栈 `O(|A| + |B|)`。

### 边界

A 或 B 为空：直接假。B 只有一个结点，且 A 里有同值叶子：`covers` 在下一层看到 B 的孩子是空，应返回真。若把「host 为空」写在「pattern 为空」前面，这一层会返回假。

## 27.二叉树的镜像

### 题目

把一棵二叉树变成它的镜像，左右孩子整棵对调。返回新的根（原地改指针即可）。

### 输入描述

上图的镜像是：

```text
      6
     / \
    8   2
   /   / \
  9   4   1
```

8 原来在右，镜像后到左，它的孩子 9 也跟着到了左边。2 到了右边，原来的左孩子 1 变成右孩子。

### 思路

每个结点交换左右指针，再对两棵子树做同样的事。子树内部的交换由递归完成，父结点只要把两根指针对调。

### 走一遍

根 6 交换后，左是 8、右是 2。递归进入 8：8 没有左孩子，右孩子 9 被换到左边。进入 2：左右 1 和 4 对调，1 到右边，4 到左边。叶子没有孩子，交换后仍是空。

### 代码

```python
def mirror(root):
    if root is None:
        return None
    root.left, root.right = root.right, root.left
    mirror(root.left)
    mirror(root.right)
    return root
```

### 复杂度

每个结点交换一次，时间 `O(n)`，递归栈 `O(高度)`。

### 边界

空树返回空。只有左孩子的结点，交换后变成只有右孩子，不能只在「两边都有孩子」时才交换。再镜像一次应回到原树。

## 28.从上往下打印二叉树

### 题目

按层从上到下打印每个结点，同一层从左到右。上图应得到 `6, 2, 8, 1, 4, 9`。

### 思路

用队列做广度优先。取出队头时，把它的左孩子、右孩子依次放入队尾。先放左再放右，同一层就不会反。

### 走一遍

队列开始是 `[6]`。

- 弹出 6，压入 2、8。输出 6。
- 弹出 2，压入 1、4。输出 2。队列里是 8、1、4。
- 弹出 8，压入 9。输出 8。
- 弹出 1、4，它们没有孩子。再弹出 9。

输出 `6, 2, 8, 1, 4, 9`。

### 代码

```python
def level_order(root):
    if root is None:
        return []
    queue = [root]
    out = []
    head = 0
    while head < len(queue):
        node = queue[head]
        head += 1
        out.append(node.val)
        if node.left:
            queue.append(node.left)
        if node.right:
            queue.append(node.right)
    return out
```

用下标当队头，避免每次删列表第 0 个元素时把后面的结点整体前移。

### 复杂度

时间 `O(n)`。若每次 `pop(0)`，单次是线性的，总时间变成 `O(n^2)`。队列空间 `O(n)`，最宽的一层可能占接近 `n/2` 个结点。

### 边界

空树返回空列表，不要先取 `root.val`。只有根时返回 `[根]`。

## 29. 一段后序能否对应二叉搜索树

### 题目

判断一个整数数组是不是某棵二叉搜索树的后序遍历。是则返回真，否则返回假。数组里的值互不相同。空数组按假处理。

### 思路

后序的最后一个数是根。前面的数分成两段：第一段都比根小，是左子树；第二段都比根大，是右子树。左段一旦出现比根大的数，后面就只能是右子树，右子树里再出现比根小的数就不是二叉搜索树。两段再各自按后序检查。

### 走一遍

`[1, 4, 2, 9, 8, 6]`。根 6。

- 从左看到 1、4、2 都小于 6，8 大于 6，左段是 `[1, 4, 2]`，右段是 `[9, 8]`。
- 右段的 9 和 8 都大于 6。
- 左段根是 2，左边 1，右边 4，都符合。右段根是 8，左边 9 并不小于 8，所以左段为空，9 在右边且大于 8。真。

反例 `[1, 8, 4, 9, 6]`。根 6。1 小于 6，8 大于 6，于是右段从 8 开始，里面有 4，4 小于 6，返回假。

### 代码

```python
def is_bst_postorder(seq):
    def valid(segment):
        if len(segment) <= 1:
            return True
        root = segment[-1]
        cut = 0
        while cut < len(segment) - 1 and segment[cut] < root:
            cut += 1
        for value in segment[cut:-1]:
            if value < root:
                return False
        return valid(segment[:cut]) and valid(segment[cut:-1])

    if not seq:
        return False
    return valid(seq)
```

### 复杂度

每层把区间切成两段，最坏是一条链，时间 `O(n^2)`。空间是递归栈 `O(n)`。

### 边界

空数组返回假。单元素 `[7]` 没有孩子，是合法后序，返回真。右子树为空时 `cut` 会停在最后一个根前面，右段切片是空的，空段应当为真，不能把「没有右子树」判失败。

## 30. 根到叶子且结点之和为定值

### 题目

从根走到叶子，路径上的结点值之和等于给定整数的，全部打印出来。叶子是左右都空的结点。返回的路径里，结点更多的排在前面。

### 思路

先序往下走，手里带着当前路径和已经走过的值。到叶子时才和目标比较。返回父结点之前把当前值从路径里弹出，兄弟子树才能共用前缀。

收集完按长度从长到短排。结点值若可能是负数或 0，不能因为「当前和已经不小于目标」就停止下探，后面的负数还可能把和拉回来。

### 走一遍

```text
    4
   / \
  2   3
 /
1
```

目标 7。

- 走到叶子 1：路径 `4, 2, 1`，和 7，记下。
- 回到 2，弹出 1。2 不是叶子。
- 回到 4，进入 3。叶子 3 的路径 `4, 3`，和 7，记下。

按长度排序后是 `[[4, 2, 1], [4, 3]]`。

反例：若在和已经等于 7 的非叶子上提前收工，根如果自己就是 7 但还有孩子，会把非叶子路径也算进去。必须确认左右都空。

### 代码

```python
def paths_with_sum(root, target):
    found = []

    def walk(node, total, path):
        if node is None:
            return
        total += node.val
        path.append(node.val)
        is_leaf = node.left is None and node.right is None
        if is_leaf and total == target:
            found.append(list(path))
        else:
            walk(node.left, total, path)
            walk(node.right, total, path)
        path.pop()

    walk(root, 0, [])
    found.sort(key=len, reverse=True)
    return found
```

### 复杂度

每条根到叶的路径都会走到，时间 `O(n)` 加上拷贝路径的开销，最坏路径拷贝是 `O(n^2)`。排序的路径条数远小于结点数时可以忽略。栈深 `O(高度)`。

### 边界

空树返回空列表。目标等于根、且根是叶子：返回 `[[根]]`。目标等于根但根有孩子：根不是叶子，不能返回只含根的路径。存在两条不同长度的合格路径时，长的必须在前；只按发现顺序输出会违反题目的顺序。

## 31.二叉搜索树与双向链表

### 题目

把二叉搜索树原地改成排序的双向链表。不能新建结点，只能改指针。左指针当「前驱」，右指针当「后继」。返回链表头，也就是最小的那个结点。

### 思路

中序遍历二叉搜索树，得到的值本来就是从小到大。遍历时记住上一个访问的结点 `prev`。当前结点的左指针指向 `prev`，`prev` 的右指针指向当前结点。第一个结点没有前驱，它就是链表头。

### 走一遍

仍用 `6 / 2 / 8` 那棵树。中序是 1, 2, 4, 6, 8, 9。

- 1 没有前驱，头是 1。
- 2 的左边改指 1，1 的右边改指 2。
- 4 接在 2 后面，6 接在 4 后面，8 接在 6 后面，9 接在 8 后面。

链表是 `1 ⇄ 2 ⇄ 4 ⇄ 6 ⇄ 8 ⇄ 9`。9 的右指针是空，1 的左指针是空。原来 6 的左右孩子不再是 2 和 8 这两棵子树的根关系，而是前驱 4 和后继 8。

### 代码

```python
def bst_to_deque(root):
    head = None
    prev = None

    def visit(node):
        nonlocal head, prev
        if node is None:
            return
        visit(node.left)
        if prev is None:
            head = node
        else:
            prev.right = node
            node.left = prev
        prev = node
        visit(node.right)

    visit(root)
    return head
```

### 复杂度

每个结点访问一次，时间 `O(n)`，递归栈 `O(高度)`。没有额外结点。

### 边界

空树返回空。只有根：左右都是空，头就是它自己，循环不会去改空指针。若先递归右边再递归左边，链表会从大到小，违反「排序」。
