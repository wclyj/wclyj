### 1. 安装Vue CLI

老项目仍可用 Vue CLI。新项目更建议直接用 Vite，不必先全局安装。

```bash
npm init -y
npm install -g @vue/cli
```

### 2. 推荐使用 Vite 创建 Vue 3 项目（更快）

任选一种脚手架，按提示确认框架为 Vue：

```bash
npm create vite@latest desk-vue3 -- --template vue
```

或：

```bash
npm create vue@latest desk-vue3
```

### 3. 运行项目

Vite 模板不会预装依赖，进入目录后要先安装再启动。

```bash
cd desk-vue3
npm install
npm run dev
```
