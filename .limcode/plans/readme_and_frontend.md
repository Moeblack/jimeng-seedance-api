### 1. 更新 README.CN.md 和 README.md
- 在「API 文档」章节中新增 **异步任务 API (Async API)** 小节。
- 详细说明新增的 5 个端点：
  - `POST /v1/async/images/generations` (异步文生图)
  - `POST /v1/async/images/compositions` (异步图生图)
  - `POST /v1/async/videos/generations` (异步视频生成)
  - `GET /v1/async/tasks/:task_id` (查询单个任务)
  - `GET /v1/async/tasks` (获取全部任务列表)
- 给出请求示例及响应格式，说明其**不阻塞**的特性及适用场景。

### 2. 编写 Windows 2000 风格的 HTML 前端
- **文件位置**: 项目根目录下创建 `index.html` 或 `jimeng-frontend.html`（单文件）。
- **技术栈**: HTML5 + Vue 3 (CDN) + Axios (CDN) + [98.css](https://jdan.github.io/98.css/) (经典的 Win98/2000 风格 UI 库)。
- **核心功能**:
  - **全局配置**: 设置项窗口，用于输入 `API Base URL`（默认 `http://localhost:5100`）和 `Session ID`。
  - **功能选项卡 (Tabs)**:
    - **图像生成**: 提供 Prompt、模型选择、比例选择、生成按钮（调用异步文生图）。
    - **视频生成**: 提供 Prompt、模型选择、比例选择、时长、生成按钮（调用异步视频生成）。
    - **任务控制台**: 列表展示所有异步任务。显示状态（Pending/Processing/Completed/Failed）和进度条。
  - **自动轮询**: 前端开启定时器，定期请求 `GET /v1/async/tasks` 以更新任务列表及进度。
  - **结果预览**: 任务完成后，提供按钮弹出一个复古风格的"图片查看器"或"媒体播放器"窗口来展示结果。
- **移动端适配**:
  - 添加 `<meta name="viewport" content="width=device-width, initial-scale=1.0">`。
  - 使用百分比宽度、Flexbox 换行等，保证在手机浏览器上"复古窗口"不会溢出屏幕，按钮足够大易点击。

## TODO LIST

<!-- LIMCODE_TODO_LIST_START -->
- [ ] 编写 Windows 2000 风格前端页面 (Vue 3)  `#create-frontend`
- [ ] 更新 README.CN.md 添加异步 API 文档  `#update-readme-cn`
- [ ] 更新 README.md 添加异步 API 文档  `#update-readme-en`
<!-- LIMCODE_TODO_LIST_END -->
