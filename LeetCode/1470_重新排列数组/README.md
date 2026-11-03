# 重新排列数组

数组长度是 `2n`，前半段是 `x1…xn`，后半段是 `y1…yn`。把它重排成 `x1, y1, x2, y2, …, xn, yn`。

## 思路

下标 `i` 的 `x` 在原数组的 `i`，对应的 `y` 在 `i + n`。从 0 走到 `n - 1`，每次先取一个 `x` 再取一个 `y` 即可。两段先切开再成对拉链，读起来更清楚。

## 演算

`nums = [9, 8, 7, 6]`，`n = 2`。

前半 `[9, 8]`，后半 `[7, 6]`。

- `i = 0`：9 配 7
- `i = 1`：8 配 6

结果 `[9, 7, 8, 6]`。

## 代码

```python
class Solution:
    def shuffle(self, nums, n):
        front = nums[:n]
        back = nums[n:]
        mixed = []
        for a, b in zip(front, back):
            mixed.append(a)
            mixed.append(b)
        return mixed
```

## 复杂度

复制两段并拉链，时间 `O(n)`，新数组空间 `O(n)`。这里的 `n` 是题面里的半长，总长度是 `2n`。

## 边界

`n = 1` 时数组只有两个数，结果就是 `[x1, y1]`，和原数组相同。不要把后半段的起点写成 `i + n - 1`，否则会和前半段重叠。
