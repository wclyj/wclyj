# 最小的 K 个数

## 题目描述

从 `n` 个整数里找出最小的 `k` 个。`k` 小于等于 0，或者 `k` 比数组还长时，返回空列表。

## 思考

全部排序再取前 `k` 个一定正确，时间是 `O(n log n)`。

只关心前 `k` 名时，维护一个大小为 `k` 的最大堆，堆里放当前看到的最小的 `k` 个数，堆顶是这 `k` 个里最大的那个。扫到一个新数：

- 堆还没满，直接放进去。
- 新数比堆顶还小，用它替换堆顶，再调整堆。
- 否则丢掉这个新数。

扫完把堆里的数从小到大排一下再返回。不能用一个写死的极大值去填满堆，输入里完全可能出现比这个「极大值」更大的数。

## 演算示例

数组 `[12, 3, 15, 8, 6, 1, 9, 4]`，`k = 4`。

1. 先放入 12、3、15、8。堆中最大是 15。
2. 6 比 15 小，换掉 15，堆里是 12、3、8、6，最大是 12。
3. 1 比 12 小，换掉 12，堆里是 8、3、6、1，最大是 8。
4. 9 比 8 大，丢掉。
5. 4 比 8 小，换掉 8，堆里是 6、3、4、1。

排序后得到 `[1, 3, 4, 6]`。

## 代码

```python
import heapq


class Solution:
    def GetLeastNumbers_Solution(self, numbers, k):
        if k <= 0 or k > len(numbers):
            return []
        # 用负数模拟最大堆，堆顶是当前 k 个数里最大的
        window = []
        for value in numbers:
            if len(window) < k:
                heapq.heappush(window, -value)
            elif value < -window[0]:
                heapq.heapreplace(window, -value)
        return sorted(-value for value in window)


if __name__ == "__main__":
    solver = Solution()
    assert solver.GetLeastNumbers_Solution([12, 3, 15, 8, 6, 1, 9, 4], 4) == [1, 3, 4, 6]
    assert solver.GetLeastNumbers_Solution([1000005, 1000001], 1) == [1000001]
    assert solver.GetLeastNumbers_Solution([4, 5], 0) == []
    assert solver.GetLeastNumbers_Solution([4, 5], 3) == []
    assert solver.GetLeastNumbers_Solution([9], 1) == [9]
    print(solver.GetLeastNumbers_Solution([12, 3, 15, 8, 6, 1, 9, 4], 4))
```

## 复杂度

每个元素最多入堆、调整一次，调整代价 `O(log k)`，时间 `O(n log k)`，堆空间 `O(k)`。

## 反例与边界

- `k = 0` 或 `k` 大于长度，返回空列表。
- `[1000005, 1000001]` 取 1 个，答案是 `1000001`。若先用 `1000000` 把窗口填满，再只在新数更小才替换，两个数都比哨兵大，窗口里会留下哨兵，结果根本不在原数组中。
- 有重复值时，最小的 `k` 个可以包含重复。例如 `[2, 2, 2]`、`k = 2`，应返回 `[2, 2]`，不能把相等值当成已经选过而丢掉。
