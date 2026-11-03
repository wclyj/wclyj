# 数据流中的中位数

## 题目描述

数字逐个到来。若目前读入的个数是奇数，中位数是排序后正中间的那个；若是偶数，中位数是中间两个数的平均值。`Insert` 读入一个数，`GetMedian` 返回当前中位数。

## 思路

始终维护两半数据：

- 较小的一半放进最大堆，堆顶是这一半里最大的，也就是整体的左中点。
- 较大的一半放进最小堆，堆顶是这一半里最小的，也就是整体的右中点。

约定较小一半的个数等于较大一半，或者恰好多 1。新数先进入较小一半，再把较小一半的最大值挪到较大一半，使两半重新有序。若较大一半反而更长，就把较大一半的最小值挪回去。

奇数个时，中位数就是较小一半的堆顶。偶数个时，取两个堆顶的平均值。

Python 的堆是最小堆。较小一半存相反数，弹出最小的相反数就相当于弹出最大的原值。

## 演算示例

依次读入 `8, 1, 6, 3, 9`。

| 读入后的有序序列 | 较小一半 | 较大一半 | 中位数 |
| --- | --- | --- | --- |
| 8 | 8 | 空 | 8 |
| 1, 8 | 1 | 8 | 4.5 |
| 1, 6, 8 | 6, 1 | 8 | 6 |
| 1, 3, 6, 8 | 3, 1 | 6, 8 | 4.5 |
| 1, 3, 6, 8, 9 | 6, 3, 1 | 8, 9 | 6 |

第 3 个数到来时，较小一半应保住 6 而不是 1：1 先进入较小一半，6 比 1 大，会在平衡时被放到较大一半，随后因为较大一半更长，6 又被挪回较小一半。

## 代码

```python
import heapq


class MedianFinder:
    def __init__(self):
        self.low = []   # 存相反数，堆顶是较小一半的最大值
        self.high = []  # 最小堆，堆顶是较大一半的最小值

    def Insert(self, num):
        heapq.heappush(self.low, -num)
        heapq.heappush(self.high, -heapq.heappop(self.low))
        if len(self.high) > len(self.low):
            heapq.heappush(self.low, -heapq.heappop(self.high))

    def GetMedian(self):
        if not self.low:
            return None
        if len(self.low) > len(self.high):
            return float(-self.low[0])
        return (-self.low[0] + self.high[0]) / 2


if __name__ == "__main__":
    finder = MedianFinder()
    expected = [8, 4.5, 6, 4.5, 6]
    for value, median in zip([8, 1, 6, 3, 9], expected):
        finder.Insert(value)
        assert finder.GetMedian() == median
    assert MedianFinder().GetMedian() is None
    print([finder.GetMedian()])
```

## 复杂度

每次插入最多调整两个堆，时间 `O(log n)`，取中位数 `O(1)`。两个堆一起保存全部数据，空间 `O(n)`。

## 反例与边界

- 还没插入任何数就取中位数，没有定义，返回空。
- 偶数个时必须返回平均值，不能只返回其中一个堆顶。读完 `8, 1` 后中位数是 4.5，不是 1，也不是 8。
- 若新数无条件放进较大一半，同时又不把较小一半的最大值交过去，两半会交叉。例如先放 8 到较小一半、再把 1 放到较大一半，较大一半的堆顶 1 会小于较小一半的 8，中位数立刻算错。
