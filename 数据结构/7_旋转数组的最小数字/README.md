# 旋转数组的最小数字

## 题目描述

把一个非递减数组前面的一段挪到末尾，称为旋转。给定旋转后的数组，找出其中最小的元素。例如 `{3,4,5,1,2}` 来自 `{1,2,3,4,5}`，最小值是 1。题目约定元素都大于 0；数组为空时返回 0。

## 思路

线性扫描可以找到最小值，时间 `O(n)`，但没有利用「两段各自非递减」这一结构。

旋转之后，数组通常分成两段：左段整体不小于右段。最小值就是右段的起点。用二分缩小区间：

- 中点大于右端：最小值在中点右侧，因为左段还没走完。
- 中点小于右端：最小值就在中点或中点左侧，右端可以收缩到中点。
- 中点等于右端：无法判断最小值在哪一侧（有重复值），只能把右端左移一格，慢慢排除。

循环结束时左右端重合，指向最小值。

## 演算示例

数组 `[6, 8, 9, 2, 3, 5]`，最小值是 2。

1. 左 0、右 5，中点 2，值 9。`9 > 5`，最小值在右侧，左端改为 3。
2. 左 3、右 5，中点 4，值 3。`3 < 5`，右端改为 4。
3. 左 3、右 4，中点 3，值 2。`2 < 3`，右端改为 3。
4. 左右相遇，返回 `2`。

没有真正旋转的 `[2, 3, 5, 6, 8, 9]` 会一直走「中点小于右端」，右端收缩到下标 0，仍然得到 2。

## 代码

```python
class Solution:
    def min_by_scan(self, rotate_array):
        if not rotate_array:
            return 0
        smallest = rotate_array[0]
        for value in rotate_array[1:]:
            if value < smallest:
                smallest = value
        return smallest

    def minNumberInRotateArray(self, rotate_array):
        if not rotate_array:
            return 0
        left, right = 0, len(rotate_array) - 1
        while left < right:
            mid = (left + right) // 2
            if rotate_array[mid] > rotate_array[right]:
                left = mid + 1
            elif rotate_array[mid] < rotate_array[right]:
                right = mid
            else:
                right -= 1
        return rotate_array[left]


if __name__ == "__main__":
    solver = Solution()
    assert solver.minNumberInRotateArray([6, 8, 9, 2, 3, 5]) == 2
    assert solver.minNumberInRotateArray([2, 3, 5, 6, 8, 9]) == 2
    assert solver.minNumberInRotateArray([3, 3, 3, 1, 3]) == 1
    assert solver.minNumberInRotateArray([1]) == 1
    assert solver.minNumberInRotateArray([]) == 0
    assert solver.min_by_scan([6, 8, 9, 2, 3, 5]) == 2
    print(solver.minNumberInRotateArray([6, 8, 9, 2, 3, 5]))
```

## 复杂度

- 扫描：时间 `O(n)`，空间 `O(1)`。
- 二分：没有重复值时时间 `O(log n)`；大量重复值迫使右端一格格移动时退化为 `O(n)`。额外空间 `O(1)`。

## 反例与边界

- 空数组返回 0，不能拿下标 0。
- `[3, 3, 3, 1, 3]` 里中点可能和右端相等。若相等时盲目去左边或右边，可能把唯一的 1 排除掉。必须缩小一格，答案才是 1。
- 完全没有旋转时，最小值在最左边。若代码假定「最小值一定不在下标 0」，会漏掉这种情况。
