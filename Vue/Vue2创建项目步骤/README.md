### 1. 安装Vue CLI

在工作目录生成包描述，再把脚手架装到全局：

```bash
npm init -y
npm install -g @vue/cli
```

### 2. 创建Vue项目

交互里选择 Vue 2。目录不要和现有文件夹重名。

```bash
vue create desk-vue2
```

### 3. 运行项目

```bash
cd desk-vue2
npm run serve
```

终端给出的本地地址即可打开。端口冲突时再改 `vue.config.js` 里的 `devServer.port`。
