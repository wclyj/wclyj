# 替换空格

## 题目描述

把字符串中的每一个空格替换成 `%20`。例如输入 `We Are Happy.`，输出 `We%20Are%20Happy.`。其它字符原样保留。

## 思路

直接调用字符串替换可以完成任务，但面试更想看手动扫描。空格会从 1 个字符变成 3 个，若在原数组里从左往右塞，后面的字符就要反复后移。

更稳妥的是新建一个字符列表：遇到普通字符就原样追加，遇到空格就依次追加 `%`、`2`、`0`，最后再拼成字符串。这样每个字符只处理一次。

若必须在字符数组里原地改，应先数出空格个数，把数组扩到最终长度，再从尾部往前填，避免覆盖还没读过的字符。

## 演算示例

输入 `Hi Tom Lee`，和常见例句不同。

| 读到 | 动作 | 当前结果 |
| --- | --- | --- |
| H, i | 原样放入 | `Hi` |
| 空格 | 放入 `%20` | `Hi%20` |
| T, o, m | 原样放入 | `Hi%20Tom` |
| 空格 | 放入 `%20` | `Hi%20Tom%20` |
| L, e, e | 原样放入 | `Hi%20Tom%20Lee` |

## 代码

```python
class Solution:
    def replace_by_builtin(self, text):
        return text.replace(" ", "%20")

    def replaceSpace(self, text):
        pieces = []
        for ch in text:
            if ch == " ":
                pieces.append("%20")
            else:
                pieces.append(ch)
        return "".join(pieces)

    def replace_in_buffer(self, chars):
        # chars 是可修改的字符列表，原地从后往前填
        space_count = sum(1 for ch in chars if ch == " ")
        read = len(chars) - 1
        chars.extend([""] * (space_count * 2))
        write = len(chars) - 1
        while read >= 0:
            if chars[read] == " ":
                chars[write] = "0"
                chars[write - 1] = "2"
                chars[write - 2] = "%"
                write -= 3
            else:
                chars[write] = chars[read]
                write -= 1
            read -= 1
        return "".join(chars)


if __name__ == "__main__":
    solver = Solution()
    sample = "Hi Tom Lee"
    assert solver.replaceSpace(sample) == "Hi%20Tom%20Lee"
    assert solver.replace_by_builtin(sample) == "Hi%20Tom%20Lee"
    assert solver.replaceSpace("") == ""
    assert solver.replaceSpace("NoSpace") == "NoSpace"
    assert solver.replaceSpace(" a ") == "%20a%20"
    assert solver.replace_in_buffer(list(sample)) == "Hi%20Tom%20Lee"
    print(solver.replaceSpace(sample))
```

## 复杂度

- 新建列表：时间 `O(n)`，空间 `O(n)`，`n` 是原串长度。
- 原地从尾部填写：时间 `O(n)`，除了扩容出的结果本身，不再需要第二份完整拷贝。

## 反例与边界

- 空串应得到空串，不能当成一个空格。
- 连续空格、首尾空格都要替换。`" a "` 的正确结果是 `%20a%20`，若用 `split` 再拼接，首尾空格会被丢掉。
- 从左往右在原数组插入 `%20`，还没复制的字符会被盖住。例如 `['A', ' ']` 若先把空格改成 `%` 再后移，很容易把 `A` 写丢。
