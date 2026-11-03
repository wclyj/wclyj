### 1.初始化项目

Windows 可直接跑官方安装脚本，按提示填写应用名和前端框架：

```powershell
irm https://create.tauri.app/ps | iex
```

macOS 与 Linux 改用：

```bash
sh <(curl https://create.tauri.app/sh)
```

下面假设生成目录叫 `field-notes`。

### 2.启动项目

脚本结束后终端会给出一组命令，含义是先装依赖，再按目标平台启动：

```text
Template created! Next:
  cd field-notes
  npm install
  npm run tauri android init

桌面调试执行 npm run tauri dev
安卓调试执行 npm run tauri android dev
```

依赖没装完就启动时，Rust 侧会找不到前端 dev server 的地址。

### 3.打包项目

```bash
npm run tauri build
```

产物在 `src-tauri/target/release/bundle`。首次编译要拉齐 Rust 依赖，Windows 还需准备 WebView2，macOS 需要 Xcode 命令行工具。
