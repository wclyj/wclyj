# 二进制中 1 的个数

## 题目描述

输入一个整数，统计它的二进制表示里有多少个 1。负数按补码计算，这里取 32 位补码。

## 思路

正数可以直接看二进制字符串。负数在 Python 里是无限位的补码，高位全是 1，不能直接数。先和 `0xFFFFFFFF` 做与运算，截成 32 位。

更整齐的办法是反复去掉最低的那个 1：`n & (n - 1)` 会把最右边的 1 变成 0，其它 1 保持不动。能做多少次，就有多少个 1。

按位检查也可以：用 `1 << i` 当掩码，看第 `i` 位是不是 1，`i` 从 0 到 31。

## 演算示例

13 的二进制是 `1101`。

- 13 是 `1101`，减 1 得到 `1100`，与运算后是 `1100`，去掉了最右边的 1。
- `1100` 与 `1011` 得到 `1000`。
- `1000` 与 `0111` 得到 `0000`。

一共 3 次，所以 13 有 3 个 1。

-4 的 32 位补码：4 是 `...00000100`，取反加一得到 `1111...11111100`，末尾两个 0，其余 30 位是 1。

## 代码

```python
class Solution:
    def NumberOf1(self, n):
        bits = n & 0xFFFFFFFF
        count = 0
        while bits:
            bits &= bits - 1
            count += 1
        return count

    def count_by_mask(self, n):
        bits = n & 0xFFFFFFFF
        count = 0
        mask = 1
        for _ in range(32):
            if bits & mask:
                count += 1
            mask <<= 1
        return count


if __name__ == "__main__":
    solver = Solution()
    assert solver.NumberOf1(13) == 3
    assert solver.NumberOf1(0) == 0
    assert solver.NumberOf1(1) == 1
    assert solver.NumberOf1(-4) == 30
    assert solver.count_by_mask(-4) == 30
    assert solver.NumberOf1(-1) == 32
    print(solver.NumberOf1(13), solver.NumberOf1(-4))
```

## 复杂度

`n & (n - 1)` 的循环次数等于 1 的个数，最多 32 次，时间和空间都是 `O(1)`。掩码法固定看 32 位，同样是常数时间。

## 反例与边界

- 0 没有任何 1，循环一次都不进，结果是 0。若先加一再判断，会把 0 算成 1。
- -1 的 32 位补码全是 1，答案是 32。若在 Python 里对原始负数直接 `bin` 并数 1，会得到一长串符号位，结果远大于 32。
- -4 不是 31 个 1。它的补码末尾是 `00`，32 位里只有 30 个 1。
