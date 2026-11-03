### 1.基本介绍

Ant Design 是蚂蚁集团前端团队出品的 React 组件库，主要服务企业中后台。表格、表单、菜单、弹层共用一套间距和色彩，业务侧少写大量自定义样式。

在入口一次性注册后，页面直接写组件标签即可。下面是一个工单筛选条：

```jsx
import { Button, Form, Input, Select } from 'antd';

export default function TicketFilter({ onQuery }) {
  const [form] = Form.useForm();
  return (
    <Form form={form} layout="inline" onFinish={onQuery}>
      <Form.Item name="lane" label="泳道">
        <Select
          allowClear
          style={{ width: 140 }}
          options={[
            { value: 'ops', label: '运维' },
            { value: 'lab', label: '实验' },
          ]}
        />
      </Form.Item>
      <Form.Item name="q" label="标题">
        <Input placeholder="关键字" />
      </Form.Item>
      <Button type="primary" htmlType="submit">查询</Button>
    </Form>
  );
}
```

注意两点：全量样式和按需样式不要叠在一起，否则主题色会打架；`Form.Item` 的 `name` 必须和接口字段一致，否则校验能过、提交体却是空的。
