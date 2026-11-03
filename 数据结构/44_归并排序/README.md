# 归并排序

## 题目描述

把整数数组排成升序。归并排序使用分治：先把数组拆成两半分别排好，再把两个有序片段合并成一个有序数组。

## 思路

拆分一直进行到每段只剩一个元素，单元素自然有序。合并时用两个下标分别指向左右片段的开头，每次把较小的那个写入结果，并让对应下标前进。某一侧用完后，把另一侧剩下的整段接上。

合并需要一块新的结果数组，所以这不是原地排序。稳定的比较是「左边小于等于右边时先取左边」，相等元素会保持原来的先后。

## 演算示例

数组 `[6, 2, 8, 3, 1, 7]`。

1. 拆成 `[6, 2, 8]` 和 `[3, 1, 7]`。
2. 左边再拆成 `[6]` 与 `[2, 8]`。`[2, 8]` 合并成 `[2, 8]`，再和 `[6]` 合并成 `[2, 6, 8]`。
3. 右边同理得到 `[1, 3, 7]`。
4. 合并 `[2, 6, 8]` 与 `[1, 3, 7]`：先取 1，再取 2，然后 3，然后 6，然后 7，最后剩下 8。

结果 `[1, 2, 3, 6, 7, 8]`。

## 代码

```python
class Solution:
    def merge_sort(self, array):
        if len(array) <= 1:
            return list(array)
        mid = len(array) // 2
        left = self.merge_sort(array[:mid])
        right = self.merge_sort(array[mid:])
        return self._merge(left, right)

    def _merge(self, left, right):
        merged = []
        i = j = 0
        while i < len(left) and j < len(right):
            if left[i] <= right[j]:
                merged.append(left[i])
                i += 1
            else:
                merged.append(right[j])
                j += 1
        merged.extend(left[i:])
        merged.extend(right[j:])
        return merged


if __name__ == "__main__":
    solver = Solution()
    assert solver.merge_sort([6, 2, 8, 3, 1, 7]) == [1, 2, 3, 6, 7, 8]
    assert solver.merge_sort([]) == []
    assert solver.merge_sort([4]) == [4]
    assert solver.merge_sort([5, 5, 1]) == [1, 5, 5]
    print(solver.merge_sort([6, 2, 8, 3, 1, 7]))
```

## 复杂度

每层合并的总工作量是 `O(n)`，一共约 `log n` 层，时间 `O(n log n)`。合并用的临时数组和递归栈使额外空间达到 `O(n)`。最坏、平均、最好都是同一量级，不像快速排序会因划分失衡变成平方。

## 反例与边界

- 空数组和单元素是递归出口，直接返回副本，不能再对空切片取中点后无限递归。
- 合并时若用「从原列表头部删除」来取下一个数，每次删除都要搬移后面的元素，合并会从线性退化成平方。下标前进才是 `O(n)` 的合并。
- 相等元素必须先取左边，才能稳定。`[5a, 1, 5b]` 若在相等时先取右边，两个 5 的顺序会颠倒。
