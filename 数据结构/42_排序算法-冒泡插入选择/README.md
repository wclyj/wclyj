# 排序算法：冒泡、选择、插入

## 题目描述

把整数数组排成从小到大。这里整理三种基础排序：冒泡排序、选择排序、插入排序。它们都只使用常数额外空间，最坏时间都是平方级。

## 思路

冒泡：相邻两个如果左边更大就交换，一轮下来最大值会冒到末尾。下一轮不再碰已经就位的尾部。如果某一轮没有发生交换，说明已经有序，可以提前结束。

选择：在还没排好的后缀里找出最小值，和后缀的第一个位置交换。交换次数少，但不稳定，相等元素的原始先后可能被打乱。

插入：认为前缀已经有序。把下一个元素向左比较，比它大的前缀元素依次右移，腾出位置后插入。对接近有序的数组，内层循环很快停住。

## 演算示例

数组 `[7, 2, 9, 1, 5]`。

冒泡第一轮相邻交换：

1. 7 和 2 交换，得到 `[2, 7, 9, 1, 5]`
2. 7 和 9 不动
3. 9 和 1 交换，得到 `[2, 7, 1, 9, 5]`
4. 9 和 5 交换，得到 `[2, 7, 1, 5, 9]`

9 已经到末尾。继续几轮后得到 `[1, 2, 5, 7, 9]`。

选择排序第一轮：后缀最小值是 1，与下标 0 的 7 交换，得到 `[1, 2, 9, 7, 5]`。注意 7 被换到了后面。

插入排序读到 2 时，7 右移，2 插到前面，得到 `[2, 7, 9, 1, 5]`。读到 1 时，9、7、2 都比 1 大，依次右移，1 插到最前。

## 代码

```python
class Solution:
    def bubble_sort(self, array):
        data = list(array)
        for end in range(len(data) - 1, 0, -1):
            swapped = False
            for i in range(end):
                if data[i] > data[i + 1]:
                    data[i], data[i + 1] = data[i + 1], data[i]
                    swapped = True
            if not swapped:
                break
        return data

    def selection_sort(self, array):
        data = list(array)
        for i in range(len(data)):
            min_index = i
            for j in range(i + 1, len(data)):
                if data[j] < data[min_index]:
                    min_index = j
            data[i], data[min_index] = data[min_index], data[i]
        return data

    def insertion_sort(self, array):
        data = list(array)
        for i in range(1, len(data)):
            current = data[i]
            j = i - 1
            while j >= 0 and data[j] > current:
                data[j + 1] = data[j]
                j -= 1
            data[j + 1] = current
        return data


if __name__ == "__main__":
    solver = Solution()
    sample = [7, 2, 9, 1, 5]
    expect = [1, 2, 5, 7, 9]
    assert solver.bubble_sort(sample) == expect
    assert solver.selection_sort(sample) == expect
    assert solver.insertion_sort(sample) == expect
    assert solver.bubble_sort([]) == []
    assert solver.insertion_sort([4]) == [4]
    assert solver.selection_sort([3, 3, 1]) == [1, 3, 3]
    print(solver.bubble_sort(sample))
```

## 复杂度

三种算法最坏、平均都是时间 `O(n^2)`。冒泡和插入在已经有序时可以降到 `O(n)`。选择排序即使有序也要扫完每个后缀，仍然是 `O(n^2)`。三种额外空间都是 `O(1)`（上面的实现为了不改动调用方，先复制了一份数组）。

## 反例与边界

- 空数组和单元素数组已经有序，三种都应原样返回。
- 选择排序不稳定。把两个相等的 4 标成先出现的 4a 和后出现的 4b，数组 `[4a, 2, 4b, 1]` 第一轮会把 4a 和 1 交换，得到 `[1, 2, 4b, 4a]`。4a 跑到了 4b 后面。冒泡和插入在只移动「严格大于」的元素时能保住相等元素的原顺序。
- 冒泡如果内层仍然扫到数组末尾，已经归位的最大值会反复参与比较。结果仍正确，但做了多余的工作。
