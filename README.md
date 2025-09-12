## 项目简介

MathwithGraph 是一个基于 Vue 3 和 Gemini API 的交互式数学学习助手。它通过可视化和交互式方法帮助用户理解数学概念，结合了聊天界面和图形渲染功能，为数学学习提供了直观且友好的体验。

![MathwithGraph 截图](./vite.jpg)

## ✨ 特性

- 🤖 基于 Google Gemini API 的智能对话系统
- 📊 支持函数图像、几何概念等数学可视化
- 🖼️ 内置图像生成功能
- 📝 Markdown 和数学公式 (MathJax) 支持
- 🔄 交互式 HTML/JavaScript 图表
- 💫 现代化UI设计与响应式布局
- 🌓 自适应明暗模式

## 🚀 快速开始

### 前提条件

- Node.js 16+ 版本
- NPM 或 Yarn 包管理器
- Google Gemini API 密钥

### 安装与运行

1. 克隆本仓库
   ```bash
   git clone https://github.com/realseverj/mathwithgraph.git
   cd mathwithgraph
   ```

2. 安装依赖
   ```bash
   npm install
   # 或
   yarn install
   ```

3. 配置API密钥
   
   在 ChatInterface.vue 文件中将 `API_KEY` 替换为你的 Google Gemini API 密钥。

4. 运行开发服务器
   ```bash
   npm run dev
   # 或
   yarn dev
   ```

5. 构建生产版本
   ```bash
   npm run build
   # 或
   yarn build
   ```

## 💡 使用指南

### 基础交流
直接在输入框中输入数学问题或概念，锐锐（AI助手）会用文字和图像来回答你的问题。

### 特殊功能

1. **图像生成**：锐锐可以通过 `<draw>` 标签生成图像
   ```
   <draw>一个二次函数的图像</draw>
   ```

2. **交互式图表**：锐锐可以通过 `<html>` 标签创建交互式图表
   ```
   <html>[Plotly图表代码]</html>
   ```

3. **数学公式**：支持 MathJax 语法显示数学公式
   ```
   行内公式: $E=mc^2$
   块级公式: $$\int_a^b f(x) dx$$
   ```

## 🧩 项目结构

```
mathwithgraph/
├── public/            # 静态资源
├── src/
│   ├── assets/        # 项目资源文件
│   ├── components/
│   │   ├── ChatInterface.vue    # 聊天界面组件
│   │   └── MarkdownRenderer.vue # Markdown渲染组件
│   ├── App.vue        # 主应用组件
│   ├── main.js        # 应用入口
│   └── style.css      # 全局样式
├── index.html         # HTML入口
└── package.json       # 项目依赖
```

## 📚 技术栈

- [Vue 3](https://v3.vuejs.org/) - 渐进式JavaScript框架
- [Vite](https://vitejs.dev/) - 现代前端构建工具
- [Google Genai](https://www.npmjs.com/package/@google/genai) - Google生成式AI SDK
- [Marked](https://marked.js.org/) - Markdown解析器
- [DOMPurify](https://github.com/cure53/DOMPurify) - HTML内容净化工具
- [MathJax](https://www.mathjax.org/) - 数学公式渲染库
- [Plotly](https://plotly.com/javascript/) - 交互式图表库

## 🛠️ 开发注意事项

### 安全考量

本项目使用 DOMPurify 来净化用户和AI生成的HTML内容，增强安全性。在开发中请确保：

1. 不直接运行未经过滤的JavaScript代码
2. 始终使用DOMPurify处理HTML内容
3. 注意API密钥的安全存储

### 自定义配置

修改 `systemPrompt` 变量（在ChatInterface.vue中）可以调整AI助手的行为和风格。

## 📄 许可证

MIT

## 👥 贡献指南

欢迎提交问题和拉取请求，帮助改进MathwithGraph项目！

## 🙏 致谢

- 感谢Google提供的Gemini API
- 感谢所有开源依赖的贡献者

---

© 2025 MathwithGraph | 开发者: Severj
