# 整数中 1 出现的次数

## 题目描述

统计从 1 写到 `n`，十进制里数字 `1` 一共出现了多少次。例如 11 要计两次。`n` 小于 1 时次数是 0。

## 思路

逐个把数字转成字符串再数，时间大约是 `O(n log n)`，`n` 很大时不够用。按数位分别统计。

从个位开始，把当前位左边的部分叫高位，右边的部分叫低位，当前这一位叫本位。设这一位的权重是 `factor`（个位是 1，十位是 10）。

- 本位是 0：高位每变化一次，这一位会完整出现 `factor` 个 1，共 `高位 × factor` 次。
- 本位大于 1：高位还能再完整多走一轮，共 `(高位 + 1) × factor` 次。
- 本位正好是 1：完整的高位贡献 `高位 × factor`，再加上低位从 0 到当前低位，也就是 `低位 + 1` 次。

高位、本位、低位一起右移，直到高位和本位都看完。

## 演算示例

统计 1 到 20。

个位：`factor = 1`，高位 2，本位 0，低位 0。本位是 0，贡献 `2 × 1 = 2`，对应个位上的 1 和 11。

十位：`factor = 10`，高位 0，本位 2，低位 0。本位大于 1，贡献 `(0 + 1) × 10 = 10`，对应 10 到 19 的十位。

百位已经超出 20，停止。合计 `2 + 10 = 12`。

核对：1、10、11（两次）、12、13、14、15、16、17、18、19，次数是 `1 + 1 + 2 + 8 = 12`。

## 代码

```python
class Solution:
    def NumberOf1Between1AndN_Solution(self, n):
        if n < 1:
            return 0
        total = 0
        factor = 1
        while factor <= n:
            high = n // (factor * 10)
            digit = (n // factor) % 10
            low = n % factor
            if digit == 0:
                total += high * factor
            elif digit == 1:
                total += high * factor + low + 1
            else:
                total += (high + 1) * factor
            factor *= 10
        return total


if __name__ == "__main__":
    solver = Solution()
    assert solver.NumberOf1Between1AndN_Solution(20) == 12
    assert solver.NumberOf1Between1AndN_Solution(13) == 6
    assert solver.NumberOf1Between1AndN_Solution(1) == 1
    assert solver.NumberOf1Between1AndN_Solution(0) == 0
    print(solver.NumberOf1Between1AndN_Solution(20))
```

## 复杂度

数位只有 `log10(n)` 位，时间 `O(log n)`，空间 `O(1)`。逐个数扫描是 `O(n log n)`。

## 反例与边界

- `n = 0` 返回 0。从 1 开始数，0 本身不计入；若把 0 的十进制也拿来数 1，仍然是 0，但循环必须先拦住，避免权重一直乘下去。
- `n = 13` 时 11 贡献两个 1，总数是 6，不是「含有 1 的数字个数」5。漏计 11 里的第二个 1 会少 1。
- 本位等于 1 时不能套用「大于 1」的公式。`n = 13` 的十位若按 `(高位+1)×10` 计算，会把 14 到 19 也算进去，得到过大的结果。
