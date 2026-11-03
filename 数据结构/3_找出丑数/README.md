# 找出丑数

## 题目描述

若一个正整数的质因数只能是 2、3、5（可以缺某几个），就叫它丑数。1 没有质因数，按习惯当作第 1 个丑数。6 和 8 是丑数，14 含有质因数 7，不是。求从小到大排列后的第 `index` 个丑数。

## 思路

判断单个数是不是丑数：不断整除 2、3、5，若最后剩下 1，就是丑数。从 1 开始逐个检查，凑满 `index` 个即可。这个办法正确，但第 `index` 个丑数可能比 `index` 大很多，扫描量接近那个丑数本身。

更快的办法是只生成丑数。除了 1，每个丑数都来自某个更小的丑数乘 2、乘 3 或乘 5。准备一条有序表，再用三个下标分别指向「下一个还没乘过 2 / 3 / 5」的位置。每轮取三个乘积里最小的那个追加到表尾；若有两个乘积一样，对应下标都要前移，避免把同一个数放进两次。

## 演算示例

求第 10 个丑数。表从 `[1]` 开始，三个下标都是 0。

| 已有丑数 | ×2 | ×3 | ×5 | 选中 |
| --- | --- | --- | --- | --- |
| 1 | 2 | 3 | 5 | 2 |
| 1,2 | 4 | 3 | 5 | 3 |
| 1,2,3 | 4 | 6 | 5 | 4 |
| 1..4 | 6 | 6 | 5 | 5 |
| 1..5 | 6 | 6 | 10 | 6（两个 6 都前移） |
| 1..6 | 8 | 9 | 10 | 8 |
| 1..8 | 10 | 9 | 10 | 9 |
| 1..9 | 10 | 12 | 10 | 10 |
| 1..10 | 12 | 12 | 15 | 12 |

第 10 个是 12。序列为 `1,2,3,4,5,6,8,9,10,12`。

## 代码

```python
class Solution:
    def is_ugly(self, num):
        if num <= 0:
            return False
        for factor in (2, 3, 5):
            while num % factor == 0:
                num //= factor
        return num == 1

    def nth_ugly_scan(self, index):
        # 逐个检查，只适合很小的 index
        if index <= 0:
            return 0
        found = 0
        current = 0
        while found < index:
            current += 1
            if self.is_ugly(current):
                found += 1
        return current

    def GetUglyNumber_Solution(self, index):
        if index <= 0:
            return 0
        ugly = [1]
        p2 = p3 = p5 = 0
        while len(ugly) < index:
            next2 = ugly[p2] * 2
            next3 = ugly[p3] * 3
            next5 = ugly[p5] * 5
            chosen = min(next2, next3, next5)
            ugly.append(chosen)
            # 相同乘积要一起前进，否则会写入重复值
            if chosen == next2:
                p2 += 1
            if chosen == next3:
                p3 += 1
            if chosen == next5:
                p5 += 1
        return ugly[-1]


if __name__ == "__main__":
    solver = Solution()
    assert solver.is_ugly(1) is True
    assert solver.is_ugly(8) is True
    assert solver.is_ugly(14) is False
    assert solver.GetUglyNumber_Solution(10) == 12
    assert solver.nth_ugly_scan(10) == 12
    assert solver.GetUglyNumber_Solution(1) == 1
    assert solver.GetUglyNumber_Solution(0) == 0
    print(solver.GetUglyNumber_Solution(10))
```

## 复杂度

- 逐个扫描：时间与第 `index` 个丑数的大小成正比，空间 `O(1)`。
- 三指针生成：时间 `O(index)`，空间 `O(index)`。

## 反例与边界

- `index <= 0` 没有对应项，返回 0。
- 14 能被 2 整除一次，剩下 7，不能再被 2、3、5 整除，所以不是丑数。只看「含有 2」会误判。
- 生成时如果 6 只让「乘 2」的指针前进、不让「乘 3」的指针前进，下一轮还会再得到 6，序列里出现重复，后面的名次全部错位。
