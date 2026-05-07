# DeepSeek-OCR 客户端 — CLAUDE.md

## 项目概述

DeepSeek-OCR 的 Electron 桌面图形界面。支持**本地推理**（通过 transformers 使用 HuggingFace 模型）和 **API 模式**（兼容 SiliconFlow OpenAI 的 API）。

## 架构

```
start-client.bat → start.py (venv 设置 + 依赖) → npm start → Electron (main.js)
                                                              │
                                                              │ 启动子进程
                                                              ▼
                                                     backend/ocr_server.py (Flask 监听 :5000)
```

- **Electron 主进程** (`main.js`)：启动 Python Flask 服务器，创建窗口，将 IPC 调用代理到 Flask HTTP 端点
- **Flask 后端** (`backend/ocr_server.py`)：模型加载/推理或基于 API 的 OCR
- **渲染进程** (`renderer.js`)：UI 逻辑，基于 canvas 的框可视化，进度轮询

## 关键文件

| 文件 | 作用 |
|------|------|
| `main.js` | Electron 主进程 — 窗口创建，Python 服务器生命周期，IPC 处理程序 |
| `renderer.js` | 前端逻辑 — 图像处理，OCR 参数，进度轮询，框渲染 |
| `index.html` | HTML 结构，包含头部控件、结果面板、灯箱 |
| `styles.css` | 所有样式 |
| `backend/ocr_server.py` | Flask 服务器 — 模型加载，OCR 推理，API 模式，配置持久化 |
| `start.py` | 启动器 — venv 管理，GPU 检测，依赖安装 |
| `api_config.json` | 持久化的 API 配置（环境变量优先于文件） |

## 运行

```cmd
start-client.bat         # Windows（推荐 — 处理所有设置）
```
或
```
python start.py          # 同上
```
或
```
.\venv\Scripts\activate  # 激活 venv
npm start                # 如果已配置 venv
npm run dev              # 打开 DevTools 运行
```

## 两种操作模式

### 本地模式（默认）
- 将 DeepSeek-OCR 模型从 HuggingFace 加载到本地 GPU/CPU
- `OCR_MODE=local` 或未设置
- 在进行 OCR 之前必须先通过“加载模型”按钮加载模型

### API 模式
- 调用 SiliconFlow API 而非本地模型
- `OCR_MODE=api` + 有效的 `SILICONFLOW_API_KEY`
- 无需加载模型 — “加载模型”按钮被禁用
- 配置保存在 `api_config.json` 中；环境变量优先级更高
- **流程**：上传 → base64 编码 → POST 到 SiliconFlow SSE 端点 → 流式接收 tokens → /progress 轮询 → 前端渲染

### 切换模式
- UI：“API设置” `<details>` 面板 → 填写 API 密钥/URL/模型 → 点击**应用**（仅保存设置）→ 点击 **API模式**/ **本地模式** 切换按钮
- 按钮：**应用** 保存配置但不更改模式；**API模式/本地模式** 切换模式但不更改已保存的设置


## OCR 框系统

SVG 叠加层系统在右侧预览面板上渲染边界框：

### 坐标系统
- 模型输出范围在 `[0, 999]` 内的归一化坐标（`DEEPSEEK_COORD_MAX = 999`）
- Token 格式：`<|ref|>CONTENT<|/ref|><|det|>[[x1,y1,x2,y2]]<|/det|>`
- `parseBoxesFromTokens()` (renderer.js:482) 使用正则表达式提取框
- `renderBoxes()` (renderer.js:554) 将归一化空间缩放到图像像素并创建 SVG 元素

### SVG 布局
- SVG 叠加层 (`#ocr-boxes-overlay`) 为 `position: absolute`，位于 `#ocr-preview-container` 内部（`inline-block`）
- viewBox 设置为图像的自然像素尺寸，`preserveAspectRatio="xMidYMid meet"`
- 容器自然地包裹图像；SVG 以 100% × 100% 填充容器
- 无论滚动状态如何，框都与图像对齐

### CSS 布局链（对叠加层对齐至关重要）
```
.ocr-preview-wrapper (flex: 1; align-items: safe center; overflow: auto)
  └─ .ocr-preview-container (inline-block; position: relative; max-width: 100%)
       ├─ img#ocr-preview-image (max-width: 100%; height: auto)
       └─ svg#ocr-boxes-overlay (absolute; top/left/width/height: 100%)
```

### 注意事项
- 不要在 SVG 上设置 `preserveAspectRatio="none"` — 会导致容器缩放时坐标失真
- 不要使用视口相对值（如 `calc(100vh - 300px)`）设置 `max-height` — 当头部高度变化时会失效
- 不要在 `align-items: center` 时省略 `safe` 关键字 — 会导致可滚动内容在顶部被截断

## 后端配置优先级

1. 环境变量（最高优先级）
2. `api_config.json`（从 UI 持久化保存的配置，作为后备）
3. 代码中的默认值（最低优先级）

关键变量：`OCR_MODE`、`SILICONFLOW_API_KEY`、`SILICONFLOW_API_URL`、`SILICONFLOW_MODEL`

## 关键 Flask 端点

| 端点 | 方法 | 用途 |
|----------|--------|-------|
| `/health` | GET | 服务器状态，`api_mode`，`model_loaded` |
| `/model_info` | GET | 模型/设备信息 |
| `/load_model` | POST | 加载 HuggingFace 模型（仅本地模式） |
| `/ocr` | POST | 处理图像（本地或 API 模式） |
| `/progress` | GET | 流式 token 进度（每 200ms 轮询） |
| `/config` | GET/POST | 获取/设置 API 配置 |

## 已知 CSS 陷阱

- 头部使用 flex 布局，`align-items: center`，并且**没有**固定高度 — 任何添加的内容（如 `<details>` 元素）都会使头部增高并缩小主面板
- `.ocr-preview-image` **不能**有 `max-height` 约束 — 让 flex 布局和包装器的 `overflow: auto` 处理尺寸即可
- API 设置的 `<details>` 元素是原生 HTML 折叠组件 — 其打开/关闭状态会在运行时改变头部高度