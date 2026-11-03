# 序列化二叉树

## 题目描述

把二叉树保存成字符串，并且能从字符串还原出结构相同的树。空孩子必须在字符串里占一个位置，否则左右孩子分不清。这里约定先序遍历，节点值用逗号分隔，空节点写成 `#`。

## 思考

先序先写根，再写左子树，再写右子树。遇到空指针就写 `#`，这样每个节点的两个孩子位置都有着落。

反序列化按同样顺序读：取出一个记号，若是 `#` 就返回空；否则新建节点，紧接着的下一段是左子树，再下一段是右子树。读指针只往前走，不会重复使用同一个记号。

## 演算示例

```text
    6
   / \
  3   8
 /
1
```

先序写下：

1. 根 6
2. 左子树：3，它的左孩子 1，1 的左右都是空，3 的右孩子是空
3. 右子树：8，8 的左右都是空

字符串：`6,3,1,#,#,#,8,#,#`。

还原时读到 6，递归读左子树得到 3（左子树再得到 1），然后读右子树得到 8。空位上的 `#` 让 3 不再去抢 8 当右孩子。

## 代码

```python
class TreeNode:
    def __init__(self, value):
        self.val = value
        self.left = None
        self.right = None


class Solution:
    def Serialize(self, root):
        tokens = []

        def walk(node):
            if node is None:
                tokens.append("#")
                return
            tokens.append(str(node.val))
            walk(node.left)
            walk(node.right)

        walk(root)
        return ",".join(tokens)

    def Deserialize(self, text):
        tokens = text.split(",") if text else []

        def read():
            if not tokens:
                return None
            token = tokens.pop(0)
            if token == "#":
                return None
            node = TreeNode(int(token))
            node.left = read()
            node.right = read()
            return node

        return read()


def shape(root):
    if root is None:
        return "#"
    return "({},{},{})".format(root.val, shape(root.left), shape(root.right))


if __name__ == "__main__":
    root = TreeNode(6)
    root.left = TreeNode(3)
    root.right = TreeNode(8)
    root.left.left = TreeNode(1)
    solver = Solution()
    text = solver.Serialize(root)
    assert text == "6,3,1,#,#,#,8,#,#"
    assert shape(solver.Deserialize(text)) == shape(root)
    assert solver.Serialize(None) == "#"
    assert solver.Deserialize("#") is None
    negative = TreeNode(-2)
    negative.right = TreeNode(-5)
    assert solver.Serialize(negative) == "-2,#,-5,#,#"
    assert shape(solver.Deserialize(solver.Serialize(negative))) == shape(negative)
    print(text)
```

## 复杂度

每个节点和每个空孩子都写一次，时间 `O(n)`，字符串长度也是 `O(n)`。反序列化若反复从列表头部弹出，总时间会变成 `O(n^2)`；节点很多时可以改成下标向前扫描。

## 反例与边界

- 空树序列化成一个 `#`。若空树写成空字符串，反序列化时无法和「没有读到记号」区分，单边孩子也会错位。
- 去掉空标记后，`6,3,1,8` 无法判断 8 是 3 的右孩子还是 6 的右孩子。上面这棵树会把 8 错挂到 1 的左边。
- 负数值也要能还原。`-2,#,-5,#,#` 表示根 -2 只有右孩子 -5。按空格切开并且不允许负号的写法会失败。
