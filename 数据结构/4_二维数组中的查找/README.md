# 二维数组中的查找

## 题目描述

有一个矩阵，每一行从左到右递增，每一列从上到下递增，各行长度相同。给定一个整数，判断它是否出现在矩阵中。

## 思路

逐行逐列扫描一定能判断，但完全没用到有序性，时间是行数乘列数。

从右上角出发可以同时利用两个方向的单调性。设当前位置是 `matrix[row][col]`：

- 比目标大：这一列再往下只会更大，所以左移一列。
- 比目标小：这一行再往左只会更小，所以下移一行。
- 相等：找到了。

左下角对称，规则对调即可。每步排除一行或一列。

## 演算示例

矩阵与常见的 `1..15` 方阵不同：

```text
2   5   8  12
3   7  11  14
6   9  13  16
10 15  18  20
```

查找 9，从右上角 12 开始：

1. `12 > 9`，丢掉最后一列，来到 8。
2. `8 < 9`，丢掉第一行，来到 11。
3. `11 > 9`，左移到 7。
4. `7 < 9`，下移到 9，命中。

查找 4：

1. `12 > 4`，左移到 8。
2. `8 > 4`，左移到 5。
3. `5 > 4`，左移到 2。
4. `2 < 4`，下移到 3。
5. `3 < 4`，下移到 6。
6. `6 > 4`，再左移就越界。4 不在矩阵里。

## 代码

```python
class Solution:
    def Find(self, target, array):
        if not array or not array[0]:
            return False
        row, col = 0, len(array[0]) - 1
        while row < len(array) and col >= 0:
            current = array[row][col]
            if current == target:
                return True
            if current > target:
                col -= 1
            else:
                row += 1
        return False


if __name__ == "__main__":
    grid = [
        [2, 5, 8, 12],
        [3, 7, 11, 14],
        [6, 9, 13, 16],
        [10, 15, 18, 20],
    ]
    solver = Solution()
    assert solver.Find(9, grid) is True
    assert solver.Find(4, grid) is False
    assert solver.Find(20, grid) is True
    assert solver.Find(2, grid) is True
    assert solver.Find(1, []) is False
    assert solver.Find(1, [[]]) is False
    print(solver.Find(9, grid), solver.Find(4, grid))
```

## 复杂度

- 右上角行走：时间 `O(行数 + 列数)`，空间 `O(1)`。
- 全表扫描：时间 `O(行数 × 列数)`，空间 `O(1)`。

## 反例与边界

- 空矩阵、只有空行，都应返回假，不能去读 `array[0][0]`。
- 4 比第一行末尾小、又比第一列的 2 大，但并不存在。不能因为「落在最小值和最大值之间」就认定一定有。
- 若把列下标错写成行数（矩阵不是方阵时），会漏掉多出来的列，或在列数更少时越界。例如一行 `[1, 3, 8, 9]` 查找 9，必须用真实列数。
