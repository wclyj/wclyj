# 某一时刻正在写作业的人数

第 `i` 名学生从 `startTime[i]` 开始写作业，到 `endTime[i]` 结束，两端都算在写。给定查询时刻 `queryTime`，数有多少学生此刻正在写。

## 思路

闭区间判断：`startTime[i] <= queryTime <= endTime[i]`。每个学生独立，扫一遍计数即可。开始时间已经晚于查询时刻的，可以直接跳过，但判断式本身已经包含这种情况。

## 演算

`startTime = [2, 5, 1]`，`endTime = [6, 5, 4]`，`queryTime = 5`。

- 学生 0：区间 `[2, 6]`，5 落在里面，计入。
- 学生 1：区间 `[5, 5]`，查询时刻正好是开始也是结束，计入。
- 学生 2：区间 `[1, 4]`，5 已经超过结束时间，不计入。

答案是 2。

## 代码

```python
class Solution:
    def busyStudent(self, startTime, endTime, queryTime):
        doing = 0
        for begin, finish in zip(startTime, endTime):
            if begin <= queryTime <= finish:
                doing += 1
        return doing
```

## 复杂度

学生人数为 `n` 时，时间 `O(n)`，额外空间 `O(1)`。

## 边界

查询时刻早于所有开始时间，或晚于所有结束时间，答案是 0。开始等于结束时，只有查询时刻恰好等于这个点才算正在写。不要把区间写成开区间，否则端点会少算一人。
