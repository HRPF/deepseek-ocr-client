# 为 deepseek-ocr-client 添加远程 API 调用支持

## Context

用户无法在本地运行大模型（硬件性能不足），但可以使用硅基流动 (SiliconFlow) 提供的免费 DeepSeek-OCR API。当前项目只支持本地加载 HuggingFace 模型进行推理，需要修改以支持通过 API 调用方式进行 OCR。

核心思路：在 Python Flask 后端新增 API 模式，当检测到 `OCR_MODE=api` 时，跳过本地模型加载，改为调用 SiliconFlow 的 OpenAI 兼容 API。

## 修改的文件

- [x] `backend/ocr_server.py` — 主要改动：添加 API 模式支持
- [x] `renderer.js` — 前端状态显示调整
- [x] `index.html` — 添加 API 密钥输入 UI
- [x] `styles.css` — 添加 API 设置样式

## 方案说明

### 启动方式

1. **环境变量方式**：启动前设置 `OCR_MODE=api` 和 `SILICONFLOW_API_KEY=sk-xxx`，应用自动进入 API 模式
2. **UI 配置方式**：启动后在 "API Settings" 中输入 API Key 和 URL，点击 Apply 即可生效

### API 模式的响应流

```
用户上传图片 → renderer.js → IPC → main.js → POST /ocr → Flask
  → is_api_mode() 检测为 True
  → perform_ocr_api() base64 编码图片
  → POST SiliconFlow API (stream=True)
  → 逐行解析 SSE 流，写入 progress_data["raw_token_stream"]
  → 前端 /progress 轮询获取实时 token → 渲染 SVG 边界框
  → 写入 result.mmd/result.txt → 返回 JSON 结果
```

### 关键设计

1. 前端 `/progress` 轮询机制完全兼容，无需修改
2. API 响应文本同时作为 `result` 和 `raw_tokens` 返回
3. `base_size`/`image_size`/`crop_mode` 等参数在 API 模式下忽略
4. `result_with_boxes.jpg` 在 API 模式下不生成（前端兼容 null）
5. API  Key 通过全局变量运行时更新，支持 `/config` GET/POST 端点

## 验证

1. 设置环境变量 `OCR_MODE=api` 和 `SILICONFLOW_API_KEY`，启动应用
2. 验证状态栏显示 "API Mode" 和 "API (SiliconFlow)"
3. 验证 "Load Model" 按钮显示 "API Mode ✓" 并禁用
4. 上传图片，点击 "Run OCR"，验证结果正常
5. 或在 UI 中展开 "API Settings"，输入 API Key，点击 Apply
