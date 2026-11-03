# 快速排序

## 题目描述

把整数数组按升序排好。快速排序同样是分治，但划分在原数组里完成，不需要和归并一样再开一块同样大的结果数组。

## 思路

选一个基准。这里取当前区间的最后一个元素。扫描区间时，把小于基准的元素交换到左段，大于等于基准的留在右段。扫描结束后，基准放到左段的右边，它的位置就固定了。然后递归排序基准左边和右边。

左右边界交错或重合时，区间长度不超过 1，不用再排。

这个划分不稳定。基准若总是选到当前最大或最小的元素，例如数组已经有序且基准取末尾，每一层只放好一个元素，时间退化成平方。平均情况下划分比较均衡，时间是 `O(n log n)`。

## 演算示例

数组 `[9, 3, 7, 1, 8, 4]`，第一趟基准是末尾的 4。

1. 9、7、8 都不小于 4，跳过。
2. 3 小于 4，换到下标 0，数组变成 `[3, 9, 7, 1, 8, 4]`。
3. 1 小于 4，换到下标 1，数组变成 `[3, 1, 7, 9, 8, 4]`。
4. 基准 4 放到下标 2：`[3, 1, 4, 9, 8, 7]`。

4 的位置已经正确。左边 `[3, 1]` 和右边 `[9, 8, 7]` 再各自划分，最终得到 `[1, 3, 4, 7, 8, 9]`。

## 代码

```python
class Solution:
    def quick_sort(self, array):
        data = list(array)

        def partition(left, right):
            pivot = data[right]
            store = left
            for i in range(left, right):
                if data[i] < pivot:
                    data[store], data[i] = data[i], data[store]
                    store += 1
            data[store], data[right] = data[right], data[store]
            return store

        def sort(left, right):
            if left >= right:
                return
            pivot_index = partition(left, right)
            sort(left, pivot_index - 1)
            sort(pivot_index + 1, right)

        if data:
            sort(0, len(data) - 1)
        return data


if __name__ == "__main__":
    solver = Solution()
    assert solver.quick_sort([9, 3, 7, 1, 8, 4]) == [1, 3, 4, 7, 8, 9]
    assert solver.quick_sort([]) == []
    assert solver.quick_sort([2]) == [2]
    assert solver.quick_sort([1, 2, 3, 4, 5]) == [1, 2, 3, 4, 5]
    assert solver.quick_sort([5, 4, 4, 1]) == [1, 4, 4, 5]
    print(solver.quick_sort([9, 3, 7, 1, 8, 4]))
```

## 复杂度

平均时间 `O(n log n)`，有序或大量重复且划分极偏时最坏 `O(n^2)`。递归深度平均 `O(log n)`，最坏 `O(n)`。除了递归栈，划分本身只用常数额外空间，所以常说额外空间是 `O(log n)` 而不是归并的 `O(n)`。

## 反例与边界

- 已经升序的 `[1, 2, 3, 4, 5]` 若每次都拿末尾当基准，第一次只确定 5，左边还剩 4 个元素，递归深度是 `n`。它仍然能排对，但不能当成「每次都是 n log n」。
- 空数组不要取末尾元素。单元素区间 `left >= right`，直接返回。
- 快速排序不稳定。相等元素在划分时可能被交换到基准两侧之外的不同位置，原始先后不能保留。
