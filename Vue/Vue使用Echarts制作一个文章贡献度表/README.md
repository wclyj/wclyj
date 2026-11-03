# 使用SpringBoot+Vue+Echarts制作一个文章贡献度表

## 前言

个人主页想要一块「过去一年每天写了几篇」的图，观感接近代码托管站的提交热力，但这里用散点大小表示篇数：当天没有文章就没有点，篇数越多点越大，上限封住，避免一天十几篇把格子撑破。

## 制作博客贡献表

图形用 ECharts 的日历坐标系配 `scatter`，而不是热力 `heatmap`。日历负责画月份格子，散点按 `[日期, 篇数]` 落在格子上。

```html
<template>
  <div id="year-board" style="width: 100%; height: 280px;"></div>
</template>

<script>
import { fetchNoteStats } from '@/api/board'
import echarts from 'echarts'

export default {
  data() {
    return { points: [] }
  },
  mounted() {
    this.drawBoard()
  },
  methods: {
    drawBoard() {
      fetchNoteStats().then((body) => {
        if (body.code !== 'ok') return
        const chart = echarts.init(document.getElementById('year-board'))
        const [from, to] = body.data.span
        chart.setOption({
          title: {
            top: 12,
            left: 'center',
            text: '写作记录',
            subtext: '近一年每天的篇数'
          },
          tooltip: {
            trigger: 'item',
            formatter(p) {
              return p.data[0] + '<br>篇数：' + p.data[1]
            }
          },
          calendar: [{
            top: 80,
            left: 'center',
            range: [from, to],
            cellSize: [16, 16],
            splitLine: { lineStyle: { color: '#e6e6e6', width: 2 } },
            yearLabel: { show: false },
            dayLabel: {
              nameMap: ['日', '一', '二', '三', '四', '五', '六'],
              color: '#333'
            },
            monthLabel: { nameMap: 'cn', color: '#333' },
            itemStyle: { color: '#fafafa', borderColor: '#eee' }
          }],
          series: [{
            type: 'scatter',
            coordinateSystem: 'calendar',
            data: body.data.points,
            symbolSize(val) {
              if (!val[1]) return 0
              return Math.min(16, 6 + val[1] * 2)
            },
            itemStyle: { color: '#3aa6a0' }
          }]
        })
      })
    }
  }
}
</script>
```

后端按天聚合。下面这段只统计已发布、且落在窗口内的记录：

```sql
SELECT DATE_FORMAT(published_at, '%Y-%m-%d') AS day_key,
       COUNT(id) AS total
FROM note
WHERE status = 'published'
  AND published_at >= '2024-01-01 00:00:00'
  AND published_at < '2025-01-01 00:00:00'
GROUP BY day_key
```

查询结果的日期不连续，没写文章的日子不会出现。日历若只喂这些点，空日子没有坐标，tooltip 也对不上。先生成窗口内的每一天，再把查到的篇数填进去，没有的记 0。

```java
public static List<String> eachDay(LocalDate from, LocalDate to) {
  List<String> days = new ArrayList<>();
  for (LocalDate cursor = from; !cursor.isAfter(to); cursor = cursor.plusDays(1)) {
    days.add(cursor.toString());
  }
  return days;
}

public Map<String, Object> noteStats() {
  LocalDate today = LocalDate.now();
  LocalDate from = today.minusDays(364);
  Map<String, Integer> counted = new HashMap<>();
  for (StatRow row : noteMapper.countByDay(from, today)) {
    counted.put(row.getDayKey(), row.getTotal());
  }
  List<List<Object>> points = new ArrayList<>();
  for (String day : eachDay(from, today)) {
    points.add(Arrays.asList(day, counted.getOrDefault(day, 0)));
  }
  Map<String, Object> payload = new HashMap<>();
  payload.put("span", Arrays.asList(from.toString(), today.toString()));
  payload.put("points", points);
  return payload;
}
```

`calendar.range` 必须和这批点的首尾日期一致，否则点会画到格子外面。篇数为 0 时 `symbolSize` 返回 0，格子仍在，只是不画点。窗口不要写死在前端，否则过了一年图还停在旧区间。

日历的 `range` 写成日期字符串，和 SQL 里 `DATE_FORMAT` 的格式要一致，都是 `yyyy-MM-dd`。后端若返回带时分秒的字符串，散点会对不齐格子，看起来像整列空白。聚合时用日期函数截断到天，不要在前端再 `split` 一次碰运气。

`dayLabel.nameMap` 的起点要和 ECharts 的星期约定对齐。默认第一格是周日。若文案从「周一」开始写，整列会错一天，周六的点画到周日下面。不确定时先只渲染一个月，对照系统日历看周一落在哪一列。

容器必须有高度。父级高度是 0 时，`echarts.init` 能成功，图却是一条缝。上面的模板把高度写在节点上。若高度来自布局计算，等计算完再 `init`，不要在 `created` 里初始化。

窗口尺寸变化后图不会自己重排，监听一次 `resize`：

```js
mounted() {
  this.drawBoard()
  this.onResize = () => {
    const node = document.getElementById('year-board')
    if (!node) return
    const exist = echarts.getInstanceByDom(node)
    if (exist) exist.resize()
  }
  window.addEventListener('resize', this.onResize)
},
beforeDestroy() {
  window.removeEventListener('resize', this.onResize)
  const node = document.getElementById('year-board')
  const exist = node && echarts.getInstanceByDom(node)
  if (exist) exist.dispose()
}
```

离开页面不 `dispose`，再次进入会在同一 DOM 上叠两个实例，tooltip 出两次。路由若被 `keep-alive` 缓存，改在 `activated` 里 `resize`，不要重复 `init`。

篇数的点大小要封顶。一天写很多篇时，若按篇数线性放大，点会盖住相邻日期，日历就没法读了。`Math.min(16, 6 + 篇数 * 2)` 把上限留在格子内部。颜色只区分「有没有写」，不要再叠一条热力色带，两种编码同时出现时读者不知道该看颜色还是看大小。

接口失败时不要 `setOption` 空数据。空数组会画出一年的空格子，看起来像「全年没写」。请求失败应保留上一份成功的图，或在容器里显示一句失败说明。`code` 字段按自己的约定判断，不要假设成功值一定是字符串 `success`。

Vue CLI 里按需引入时，包名可能是 `echarts` 而不是带默认导出的旧路径。若控制台提示 `echarts.init is not a function`，改成：

```js
import * as echarts from 'echarts'
```

只注册日历和散点可以减小包体，但要同时引入 `CalendarComponent` 和 `ScatterChart` 并 `use`。漏注册日历时，配置不报错，坐标却全部丢弃，页面仍是空白。先用完整包把图跑出来，再做按需，避免把「没数据」和「没注册组件」混在一起查。
