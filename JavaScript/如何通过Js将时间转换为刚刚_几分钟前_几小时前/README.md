前言
--

评论列表和动态流习惯把发送时间写成「刚刚」「几分钟前」「几小时前」，而不是一长串绝对时间。做法是拿评论时间戳和当前时间戳做差，再按分钟、小时、天、周、月的毫秒长度整除。超过三个月就退回公历日期，避免「15月前」这种读起来别扭的说法。

代码
--

入参形如 `2024-03-18 09:26:41`。Safari 对 `YYYY-MM-DD` 的解析不稳定，先把横线换成斜线再交给 `Date`。

```js
/**
 * 把绝对时间收成相对说法。
 * @param {string} raw 例如 2024-03-18 09:26:41
 */
export function relativeLabel(raw) {
  const minute = 60 * 1000
  const hour = 60 * minute
  const day = 24 * hour
  const week = 7 * day
  const month = 30 * day

  const normalized = String(raw).slice(0, 19).replace(/-/g, '/')
  const past = new Date(normalized).getTime()
  const gap = Date.now() - past
  if (Number.isNaN(past) || gap < 0) return ''

  const monthN = gap / month
  const weekN = gap / week
  const dayN = gap / day
  const hourN = gap / hour
  const minuteN = gap / minute

  if (monthN >= 1 && monthN <= 3) return Math.floor(monthN) + '个月前'
  if (weekN >= 1 && weekN <= 3) return Math.floor(weekN) + '周前'
  if (dayN >= 1 && dayN <= 6) return Math.floor(dayN) + '天前'
  if (hourN >= 1 && hourN <= 23) return Math.floor(hourN) + '小时前'
  if (minuteN >= 1 && minuteN <= 59) return Math.floor(minuteN) + '分钟前'
  if (gap < minute) return '刚刚'

  const when = new Date(past)
  const pad = (n) => (n < 10 ? '0' + n : '' + n)
  return when.getFullYear() + '-' + pad(when.getMonth() + 1) + '-' + pad(when.getDate())
}
```

调用处直接绑到模板：

```html
<span>{{ relativeLabel(row.createdAt) }}</span>
```

注意：切到第 18 个字符会把秒的个位切掉，比较时可能偏一秒。这里保留到秒，共 19 个字符，含中间的空格。月份按 30 天估算，跨月边界会有大约一天的误差，评论列表可以接受。未来时间（时钟回拨或时区写错）直接返回空串，避免出现负数。

入参若已经是毫秒数字，不要再 `replace`。先判断类型：数字直接减，字符串再做替换。接口有时两种都给，混在一个函数里不分支，会把数字收成 `NaN`，页面上就是空白。同一条评论在列表和详情里应调用同一个函数，避免一边显示相对时间、一边显示完整日期。列表如果每秒重算，会把「刚刚」和「1 分钟前」来回抖。展示层按分钟刷新即可，不必绑在动画帧上。
