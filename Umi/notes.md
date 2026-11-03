### 1.创建项目

在空目录拉起官方脚手架，按提示选 Ant Design 模板或空白应用：

```bash
npx create-umi@latest lab-console
```

向导会询问包管理器和是否启用 TypeScript。目录名用小写，避免后面和 Linux 构建机的路径大小写不一致。

### 2.启动项目

```bash
cd lab-console
npm install
npm run dev
```

浏览器打开终端里打印的本地地址即可。端口被占时，在 `.umirc.ts` 增加 `port`，不要和另一个前端抢同一个口。

约定式路由会把 `src/pages` 下的文件映射成地址。最小页面可以写成：

```tsx
export default function Desk() {
  return <p>实验台首页</p>
}
```
