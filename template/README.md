# Starbase 项目全方位技术文档

本文档基于项目实际代码和配置，提供全方位的架构解析、技术栈审计及二次开发指南。

## 1. 项目核心架构解析

### 1.1 设计目标
本项目 (`starbase` template) 是一个生产级静态网站脚手架，旨在解决以下核心问题：
1.  **现代化构建**：集成 Webpack 5 (`webpack@5.91.0`) 处理资源打包。
2.  **类型安全**：原生支持 TypeScript (`typescript@5.4.5`)，提供无缝开发体验。
3.  **原子化样式**：预置 Tailwind CSS (`tailwindcss@3.4.3`) 及 PostCSS 流程。
4.  **代码规范**：内置 ESLint + Prettier + Stylelint 严格的 Lint 体系。

### 1.2 模块拓扑图
```mermaid
graph TD
  Entry[src/app.ts] -->|Import| CSS[src/app.css]
  CSS -->|@import| Tailwind[styles/tailwind.css]
  CSS -->|@import| Fonts[styles/fonts.css]
  Entry -->|DOM操作| HTML[src/index.html]
  WebpackConfig -->|Load| Entry
  WebpackConfig -->|Output| Dist[dist/]
```

### 1.3 核心功能矩阵表

| 模块路径 | 功能描述 | 关键类/配置 | 二次开发复用建议 |
| :--- | :--- | :--- | :--- |
| `src/app.ts` | 应用入口 | `import './app.css'` | 可在此挂载 Vue/React 实例 |
| `src/styles/` | 样式体系 | `root.css`, `tailwind.css` | 遵循 ITCSS 架构扩展组件样式 |
| `webpack/` | 构建配置 | `webpack.config.base.ts` | 基础配置可被 Dev/Prod 继承 |
| `.eslintrc.json` | 代码规范 | `extends: [...]` | 团队协作需统一此配置 |

---

## 2. 技术栈深度审计报告

### 2.1 依赖包关键分析
```bash
├─ webpack@5.91.0         # 构建核心 (最新版v5)
├─ typescript@5.4.5       # 开发语言 (强类型)
├─ tailwindcss@3.4.3      # 样式引擎 (需注意content配置)
├─ postcss@8.4.38         # CSS后处理 (配合autoprefixer)
└─ html-webpack-plugin    # HTML模版注入
```

### 2.2 版本风险清单
1.  **Tailwind CSS 配置风险**：
    *   **问题**：`tailwind.config.ts` 中 `content` 配置为 `['./src/**/*.{html,js}']`，未包含 `.ts` 文件。
    *   **影响**：在 TypeScript 文件中编写的 Tailwind 类名无法被提取。
    *   **修复建议**：修改为 `['./src/**/*.{html,js,ts}']`。

2.  **Webpack 生产环境配置风险**：
    *   **问题**：`webpack/webpack.config.prod.ts` 同时使用了 `style-loader` 和 `MiniCssExtractPlugin`。
    *   **影响**：导致样式注入冲突或冗余。
    *   **修复建议**：生产环境应移除 `style-loader`，仅保留 `MiniCssExtractPlugin`。

---

## 3. 源码级目录解剖

### 3.1 关键文件执行流程
```
npm run build 
→ 调用 webpack-cli 
→ 加载 webpack/webpack.config.prod.ts 
→ 合并 webpack.config.base.ts 
→ 入口 src/app.ts 
→ 解析 TS & CSS 依赖 
→ PostCSS 处理 (Tailwind) 
→ 输出 dist/index.html & [hash].js
```

### 3.2 核心模块注释示例 (Webpack Base Config)
```typescript
// webpack/webpack.config.base.ts
const config: Configuration = {
  // 上下文目录锁定在 src，避免引用外部文件
  context: path.resolve(process.cwd(), 'src'),
  entry: {
    // 核心入口，后续可扩展为多页应用 entry: { app: ..., admin: ... }
    app: [path.join(process.cwd(), 'src/app.ts')],
  },
  module: {
    rules: [
      {
        // TS 加载器，排除 node_modules 提升速度
        test: /\.ts?$/,
        use: 'ts-loader',
        exclude: /node_modules/,
      },
      // ... 资源文件(图片/字体)处理
    ]
  },
  // ...
};
```

---

## 4. 构建机制白盒分析

### 4.1 环境差异配置对照表

| 配置项 | Dev 环境 (`webpack.config.dev.ts`) | Prod 环境 (`webpack.config.prod.ts`) | 说明 |
| :--- | :--- | :--- | :--- |
| **Mode** | `development` | `production` | 影响 Tree Shaking 和压缩 |
| **DevTool** | `source-map` | (默认) | Dev 环境便于调试源码 |
| **CSS Loader** | `style-loader` (注入) | `MiniCssExtractPlugin` (提取) | **需修正 Prod 配置** |
| **Output** | `[name].js` | `[name]-[fullhash:8].js` | 生产环境带 Hash 缓存 |
| **Clean** | 无 | `CleanWebpackPlugin` | 每次构建清理 dist |

### 4.2 插件系统时序
```plantuml 
participant Main 
participant Plugin 
Main -> Plugin: 调用apply(compiler) 
Plugin --> Main: 注入hook回调 
```

---

## 5. 二次开发适配方案

### 5.1 模块改造评估矩阵

| 模块路径 | 改造类型 | 方案建议 | 工作量评估 |
| :--- | :--- | :--- | :--- |
| `src/app.ts` | 框架接入 | 引入 `vue` 及 `createApp`，挂载至 `#app` | 0.5 人日 |
| `webpack/` | Loader扩展 | 新增 `vue-loader` 支持 `.vue` 文件 | 0.5 人日 |
| `tailwind.config.ts` | 配置修正 | 增加 `.ts`, `.vue` 文件扫描支持 | 极小 |

### 5.2 新增功能接入点
*   **Vue 3 集成**：
    1.  安装 `vue`, `vue-loader`。
    2.  修改 `webpack.config.base.ts` 添加 `.vue` 规则。
    3.  在 `src/app.ts` 中替换 DOM 操作为 Vue 挂载。
*   **API 代理**：
    在 `webpack.config.dev.ts` 的 `devServer` 中添加 `proxy` 配置以解决跨域问题。

---

## 6. 调试与排错指南

### 6.1 关键断点位置
*   **构建配置调试**：在 `webpack/webpack.config.base.ts` 顶部添加 `console.log('Building...')` 确认配置加载。
*   **应用逻辑调试**：`src/app.ts` 行首 `debugger;`，浏览器打开 DevTools 会自动暂停。

### 6.2 常见报错排查
*   *Tailwind 样式不生效*：检查 `tailwind.config.ts` 的 `content` 是否包含你的文件扩展名（如 `.ts`, `.tsx`）。
*   *生产环境样式丢失*：检查 `webpack.config.prod.ts` 中的 Loader 顺序，确保没有同时使用相互冲突的 Loader。

---

## 7. 快速开始 (Usage)

推荐使用 [nvm](https://github.com/nvm-sh/nvm) 管理 Node 版本。

### 7.1 安装依赖
```bash
nvm use
npm install
```

### 7.2 本地开发
```bash
npm run dev
```
访问 [http://localhost:3000](http://localhost:3000)。

### 7.3 生产构建
```bash
npm run build
```
构建产物将输出到 `/dist` 目录。
