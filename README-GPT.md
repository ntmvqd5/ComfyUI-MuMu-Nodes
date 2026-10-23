# ComfyUI-MuMu-Nodes 完整版 — 技术说明与需求文档（给 GPT 分析用）

> 本包是**完整版**源码（`nodes.py` 含全部节点系列，无混淆、无压缩、UTF-8 纯文本，可直接阅读）。
> GitHub：https://github.com/ntmvqd5/ComfyUI-MuMu-Nodes

---

## 一、包结构

```
ComfyUI-MuMu-Nodes/
├── nodes.py          # 全部节点实现（31 个节点类 + 公共函数）
├── __init__.py       # 导出 NODE_CLASS_MAPPINGS / NODE_DISPLAY_NAME_MAPPINGS
└── README.md         # 仓库说明
```

`nodes.py` 为单一文件、无外部依赖（仅标准库 + torch + Pillow），安装方式：整个目录丢进 ComfyUI 的 `custom_nodes/` 下，重启 ComfyUI。

---

## 二、节点总览（4 个渠道平台 + 1 组流式测试）

| 系列 | 显示名 | 走哪个平台 | 协议 | 节点 |
|---|---|---|---|---|
| 木木 | 木木2/2.1、木木3/3.1、木木V2/V2.1 | **smile-ai（木木）** | Gemini 同步 `generateContent` | 6 个 |
| 木木备用 | 木木4-备用、木木4.1-备用、木木5-备用、木木5.1-备用 | **smile-ai（木木）** | OpenAI 格式同步 `/v1/images/generations`、`/v1/images/edits` | 4 个 |
| 流式测试 | 木木2流式测试、木木2.1流式测试 | **smile-ai（木木）** | Gemini 流式 `streamGenerateContent`（SSE） | 2 个 |
| 南南 | 南南5/5.1/5.6、南南V8/V8.1 | **bananarouter（南南）** | OpenAI 异步（taskID+轮询） / MJ 接口 | 5 个 |
| 咪咪 | 咪咪1/1.1、3/3.1、4/4.1、5/5.1、6/6.1、V2/V2.1 | **grsai（咪咪）** | 自有异步（id+轮询） | 12 个 |
| 逗逗 | 逗逗1/1.1 | **bananarouter** | Seedream 同步 `/api/v3/images/generations` | 2 个 |

> 全部节点 CATEGORY 均为「木木」，统一显示在 ComfyUI 节点菜单的木木分类下。

### 平台基址

- smile-ai（木木）：`https://api.smile-ai-studio.com`（New API 网关）
- bananarouter（南南）：`https://api.bananarouter.com`
- grsai（咪咪）：`https://grsai.dakka.com.cn`（国内）/ `https://grsaiapi.com`（全球）

---

## 三、重点：木木系列与 RH 平台的兼容问题（本次分析的核心）

### 需求

在 **RunningHub（RH）** 平台运行工作流，用**木木系列节点（smile-ai 平台）**出图。

**硬约束：**
- 必须用木木平台（smile-ai），不接受迁移到其他渠道
- 不改动原节点，新方案必须新做节点
- 不需要任务取消机制
- 报错时最好显示上游请求 ID（方便对账）

### 现象与根因（已确认）

1. RH 网关约 **2 分 17 秒掐断**到上游的长时间等待长连接
2. smile-ai 是**同步接口**：客户端断开 ≠ 上游停止。任务继续生成、继续扣费，但结果永远回不到 RH
3. 木木后台显示同一任务 2 分 34 秒成功、已扣费 → 用户收不到图、钱照扣
4. 木木后台「非流」= 非流式请求（正常模式）；「US 加速线路」= smile-ai 路由配置（拥堵会加重延迟，但不是报错原因）

### 端点探测结果（2026-10-23，无 key）

| 端点 | 返回 | 结论 |
|---|---|---|
| `POST /v1beta/models/{model}:generateContent` | 401 Invalid token | 存在（原节点在用） |
| `POST /v1beta/models/{model}:streamGenerateContent` | 401 Invalid token | **存在**（流式） |
| `POST /v1beta/models/{model}:asyncGenerateContent` | 401 Invalid token | **存在**（疑似异步） |
| `POST /v1/images/generations/async`（OpenAI 异步） | 404 | 不存在 |

> 401（而非 404）说明网关路由已匹配该端点，仅缺有效 key。三个 Gemini 端点均可能可用。
> smile-ai 错误体：`{"error":{"code":"","message":"...","type":"new_api_error"}}`，带 request id（如 `202610230154252765625148268d9d6p4B33Pv0`）。

### 木木系列模型 ID（带分辨率后缀）

| 系列 | 1K | 2K | 4K |
|---|---|---|---|
| banana_v2（木木2/2.1） | `gemini-3.1-flash-image-preview-1k` | `...-2k` | `...-4k` |
| banana_pro（木木3/3.1） | `gemini-3-pro-image-preview-1k` | `...-2k` | `...-4k` |
| banana_v21（木木V2/V2.1） | — | — | `gemini-nano-banana-2.1-4k` |

### Gemini 请求体（同步/流式/异步共用同一 body 格式）

```json
{
  "contents": [{"role": "user", "parts": [{"text": "提示词"}]}],
  "generationConfig": {
    "responseModalities": ["IMAGE"],
    "imageConfig": {"aspectRatio": "1:1"},
    "quality": "high",
    "seed": 12345
  }
}
```

图生图 parts 追加：`{"text": "图1"}, {"inlineData": {"mimeType": "image/png", "data": "<base64>"}}`

响应（同步）：`candidates[0].content.parts[*].inlineData.data`（base64 图片）

### 现有节点实现要点（nodes.py 内，行号以当前完整版为准）

- `_Text2Image.generate` → `_resolve_model(series, resolution)` → `_generate()`：同步 `generateContent`，按 `_build_variants(seed)` 重试（seed>=0 时先带 quality+seed、再只带 seed，然后 quality-only、最后裸请求），全失败才报错
- `_Image2Image.generate`：有参考图走 `_edits()`（parts 带 inlineData），无图退化为文生图
- `_SMText2Image`（木木4/5备用）：OpenAI 格式 `_img_generate`，参数 `{model, prompt, size, moderation:"low"}`，size 查 `_BR_SIZES[resolution][aspect]` 预设
- 流式测试节点（`_SMStreamText2Image`）：`_stream_request` 用 urllib 读 SSE（socket 90 秒闲置超时 + 1200 秒总上限），`_parse_stream_images` 逐行解析 `data:` 事件累积 inlineData
- 报错增强：`_request(..., want_meta=True)` 拿响应头，`_extract_req_id` 从响应头/响应体提取请求 ID，报错追加 ` | 请求ID: xxx`；`_handle_http` 显示真实 HTTP code + 服务端正文

### 已尝试/已排除

| 方案 | 结果 |
|---|---|
| 同步 `generateContent`（现状） | RH 2m17s 掐断，必挂 |
| 迁移 bananarouter（南南）异步 | 能跑，但违背"必须用木木平台"，**已从仓库删除** |
| 流式 `streamGenerateContent` 测试节点（本包内有） | **待实测**：SSE 持续推送，连接不会闲置，可能躲过 RH 掐断 |
| smile-ai `asyncGenerateContent` | **待实测**：端点存在，但轮询路径/返回结构未知，需真实 key 验证 |

### 请 GPT 分析的问题

1. `streamGenerateContent` 的 SSE 在 RH 环境是否安全？SSE 数据推送间隔通常多大？90 秒闲置判超时是否合理？
2. New API 网关的 `:asyncGenerateContent` 提交后返回什么（推测 `{"taskID":...}`）？轮询接口和结果字段是什么（推测 `GET /v1beta/models/{model}:getAsyncTask?taskID=...`）？能否用木木 key 走通"提交→轮询→取图"？请给出确切格式。
3. 若两条都不可行：New API 网关还有哪些机制能在 RH 长连接限制下稳定出图？
4. 现有 `_stream_request` 的 urllib socket 行为在 SSE 场景是否正确？

---

## 四、其他系列（供理解整体，不是本次分析重点）

- **南南（bananarouter）**：`_br_generate/_br_edits` 走 `POST /v1/images/generations/async`（提交得 `taskID`）→ `GET /v1/async-tasks/{taskID}` 轮询（终态 `success`，结果 `resultImages[].url`）→ 下载。南南 V8 走 MJ：`POST /mj/submit/imagine` → `GET /mj/task/{tid}/fetch`
- **咪咪（grsai）**：`_gr_submit` 提交 `POST /v1/api/generate`（返回 `id`）→ `_gr_poll` 轮询 `GET /v1/api/result?id=...`（成功 `succeeded`，结果 `results[].url`）
- **逗逗**：Seedream 同步 `POST /api/v3/images/generations`，模型 `doubao-seedream-5-0-260128`，`response_format=url`、`watermark=false`
- 南南/咪咪均有渠道名前缀报错（如 `[bananarouter]`、`[grsai]`）+ taskID 展示；木木系列带请求 ID 展示

---

## 五、版本记录

- `e51b4cb` 删除南南异步重置节点（违背只用木木原则），保留流式测试节点
- `b84dad1` 新增流式测试节点（木木2/2.1 流式测试）
- 此前：请求 ID 报错增强、smile-ai 真实 HTTP 错误展示、逗逗节点、超时统一 1200s
