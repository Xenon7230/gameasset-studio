# 架构设计：GameAsset Studio

> 状态：M0 设计稿。**本文描述预期架构，不代表已实现。** 目录与模块在对应里程碑落地；未落地的部分标注"计划"。
> 相关文档：[project-brief.md](project-brief.md) · [experiment-plan.md](experiment-plan.md) · [roadmap.md](roadmap.md)

## 1. 设计原则

1. **单机优先**：一张 16GB 显卡、一个用户、一次只跑一个生成任务。不为想象中的并发付费。
2. **引擎外置**：ComfyUI 作为独立进程通过 HTTP 调用，不进本仓库、不进本进程。便于升级与隔离故障。
3. **记录先于功能**：每次生成都必须留下足以复现的记录；没有记录的生成视为无效运行。
4. **薄服务层**：后端做校验、编排、记录与评测，不重复实现采样与模型加载。
5. **可替换边界**：ComfyUI 调用被封装在 adapter 后；未来换回 diffusers 或增加第二台推理机时，只改 adapter。
6. **失败显式化**：超时、OOM、工作流错误都必须分类记录，不能静默重试掩盖问题。

## 2. 总体架构（预期）

```text
┌───────────────────────────────────────────────────────────────┐
│ 调用方                                                        │
│  · M1: 脚本 (scripts/run_once.py)                             │
│  · M2+: REST API 调用方                                       │
│  · M5(可选): Web UI                                           │
└───────────────────────────┬───────────────────────────────────┘
                            │ HTTP / JSON（127.0.0.1）
┌───────────────────────────▼───────────────────────────────────┐
│ FastAPI 服务                     src/gameasset_studio/api      │
│  · 请求校验 (Pydantic schema)                                  │
│  · 提交生成 / 查询状态 / 获取产物                               │
└───────────────────────────┬───────────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────────┐
│ 编排层                                                        │
│  · generation/service: 参数 → 工作流 → 调用 → 落盘 → manifest  │
│  · generation/queue: 单 worker FIFO（线程 / async task）        │
│  · experiments/runner: 配置 → 参数矩阵 → 多次生成 → 汇总        │
└──────────┬────────────────────────────────────┬───────────────┘
           │ ComfyUI HTTP API                   │ 文件系统
┌──────────▼──────────────┐         ┌───────────▼───────────────┐
│ comfy/ adapter          │         │ storage/                  │
│  · client (httpx)       │         │  · runs/<run_id>/...      │
│  · workflow 模板与校验   │         │  · manifest / index.jsonl │
│  · 轮询 / 产物下载       │         │  · 哈希与环境指纹          │
└──────────┬──────────────┘         └───────────────────────────┘
           │ /system_stats /prompt /history /view /upload/image
┌──────────▼────────────────────────────────────────────────────┐
│ ComfyUI Server（独立安装，不纳入本仓库）                        │
│  SDXL base + ControlNet Canny · PyTorch CUDA                  │
│  RTX 5060 Ti 16GB                                             │
└───────────────────────────────────────────────────────────────┘

评测（离线）: evaluation/ 读取 runs/ 与 manifest → metrics.json → 汇总表
```

数据流的关键点：**所有产物的真相来源是 `runs/<run_id>/` 目录**，服务进程与数据库都不是。进程重启不丢历史；记录可被脚本、报表和测试直接消费。

## 3. 模块职责

| 模块（计划路径） | 职责 | 不负责 | 落地里程碑 |
| --- | --- | --- | --- |
| `core/settings` | 应用配置（ComfyUI 地址、目录、日志级别），环境变量 + `.env` | 实验参数 | M1 |
| `core/logging` | 结构化日志，注入 `run_id` | 指标计算 | M1 |
| `core/errors` | 错误类型：Comfy 不可用 / 工作流无效 / 生成失败 / 超时 | HTTP 映射细节 | M1 |
| `comfy/client` | 调用 ComfyUI：健康检查、提交 prompt、轮询 history、下载图片、上传参考图 | 参数语义、实验设计 | M1 |
| `comfy/workflows` | 加载并校验 API 格式工作流模板，按映射替换参数 | 工作流可视化编辑 | M1 |
| `generation/service` | 组装一次生成：参数校验、模板填充、调用、产物落盘、写 manifest | 批量编排 | M2 |
| `generation/queue` | 单 worker FIFO 队列、任务状态机、取消与超时 | 跨进程持久化 | M2 |
| `experiments/definitions` | 实验 YAML 的解析与校验（基线、变量、种子） | 执行 | M4 |
| `experiments/runner` | 展开参数矩阵、调用 generation service、记录失败、断点续跑 | 指标算法 | M4 |
| `evaluation/metrics` | 结构一致性、提示词一致性、耗时与显存统计 | 决定实验结论 | M4 |
| `evaluation/report` | 汇总为表格（Markdown / CSV），保留原始数值 | 自动写结论 | M4 |
| `storage/runs` | 运行目录布局、原子写、哈希计算、`index.jsonl` 追加 | 查询引擎 | M1 |
| `api` | FastAPI 路由与 schema、错误映射、产物下载 | 业务逻辑 | M2 |
| `scripts/` | 运维 / 实验脚本（连通性检查、单图生成、批量导出） | 被服务导入的核心逻辑 | M1 |

原则：`api` 与 `scripts` 都是入口，薄；业务逻辑在 service 层，便于在无 HTTP 的情况下测试。

## 4. 一次生成请求的生命周期（M2 预期）

1. 客户端 `POST /v1/generations`，提交参数（提示词、尺寸、步数、CFG、采样器、种子、可选控制图）。
2. API 层用 Pydantic 校验范围（如尺寸为 64 的倍数、步数上限、CFG 范围），并把控制图保存到该次运行的 `inputs/`。
3. 创建 `run_id` 与运行目录，写初始 `manifest.json`（`status=queued`）。
4. 任务入队；单 worker 取出后 `status=running`，记录开始时间。
5. service 填充工作流模板 → 计算模板哈希 → `POST /prompt` → 得到 `prompt_id` 并写入 manifest。
6. 轮询 `GET /history/{prompt_id}` 直到完成或超时；期间可查询 `GET /queue` 做 UI 展示（M5）。
7. 从 `/view` 下载图片到 `images/`，计算 sha256 与尺寸。
8. 汇总耗时、显存峰值（采样方式见实验计划）、ComfyUI / torch / 驱动版本，写 `status=succeeded` 的 manifest，并追加 `index.jsonl`。
9. 客户端 `GET /v1/generations/{run_id}` 查询状态，`GET /v1/generations/{run_id}/artifacts/{name}` 下载产物。

失败路径：任何一步失败都写入 `status=failed` + 错误类型 + 原始错误摘要（截断、不含密钥），并保留已产生的部分产物。

## 5. ComfyUI 集成策略

### 5.1 调用方式

| 用途 | 接口 | 备注 |
| --- | --- | --- |
| 健康检查 / 环境信息 | `GET /system_stats` | 启动时校验；记录版本 |
| 提交工作流 | `POST /prompt` | body: `{"prompt": <API 格式工作流>, "client_id": ...}` |
| 查询结果 | `GET /history/{prompt_id}` | 轮询；区分排队、执行、错误 |
| 下载产物 | `GET /view?filename=&subfolder=&type=output` | 二进制，落盘后算哈希 |
| 上传参考图 | `POST /upload/image` | multipart；每次运行使用独立子目录避免重名 |
| 进度（P1） | `WS /ws?clientId=` | M2 先用轮询，降低连接管理复杂度 |

以上为 ComfyUI 当前公开的接口形态；**具体字段与节点类名以实际安装版本为准，M1 必须实测确认**。

### 5.2 工作流模板

- 使用 ComfyUI 的 **API 格式**工作流 JSON（界面中 "Save (API Format)" 导出），而不是 UI 格式。
- 模板放在 `workflows/` 并纳入版本控制；文件中保留节点 ID，参数通过**显式映射表**替换：`路径（节点 ID + 输入名） → 请求字段`。
- 模板加载时校验：必需节点类存在、必需输入存在、数值类型正确。ComfyUI 侧可用 `GET /object_info` 做类名存在性检查。
- 每次运行把**实际提交的工作流**另存为 `workflow.api.json` 并记录 sha256；复现时以该文件为准，而不是"最新模板"。
- 不为模板做隐式默认值；请求缺参数即报错，避免"看起来跑了但参数不对"。

模板的参数化映射（示例，节点 ID 以 M1 实际导出为准）：

| 参数 | 节点（示例） | 输入 |
| --- | --- | --- |
| 模型 | `CheckpointLoaderSimple` | `ckpt_name` |
| 正向 / 负向提示词 | `CLIPTextEncode` ×2 | `text` |
| 尺寸 / 批大小 | `EmptyLatentImage` | `width` / `height` / `batch_size` |
| 步数 / CFG / 采样器 / 调度器 / 种子 | `KSampler` | `steps` / `cfg` / `sampler_name` / `scheduler` / `seed` |
| Canny 阈值 | `Canny` | `low_threshold` / `high_threshold` |
| 控制强度 / 起止 | `ControlNetApplyAdvanced` | `strength` / `start_percent` / `end_percent` |

节点类名（`Canny`、`ControlNetApplyAdvanced` 等）依赖 ComfyUI 与内置节点版本，升级时需重新校验。

### 5.3 版本与故障处理

- ComfyUI 独立安装并按 commit 锁定；manifest 记录 `ComfyUI` 版本、模板哈希、模型文件名与哈希。
- 启动冒烟测试：`/system_stats` 可达 → 必需节点存在 → 目标 checkpoint 在 ComfyUI 的模型列表中出现。
- 超时：连接 5s、单次查询 30s、整任务可配置（默认 10 分钟），超时后尝试从 `/history` 恢复状态；无法恢复则标记失败。
- ComfyUI 崩溃：任务标记失败，不无限重试；重试策略由实验 runner 显式配置。

## 6. 存储布局（预期）

```text
runs/
├── index.jsonl                        # 每次运行一行索引（追加写，只增不改）
└── 20261110T190000Z-exp03-s0/
    ├── manifest.json                  # 状态、参数、模型、环境、产物、指标
    ├── config.yaml                    # 本次生效的实验配置快照（如适用）
    ├── workflow.api.json              # 实际提交的 ComfyUI 工作流
    ├── inputs/                        # 控制图、源图（原始文件 + 哈希）
    ├── images/                        # 生成结果
    ├── metrics.json                   # 评测输出（M4）
    └── run.log                        # 该次运行的日志
```

- 目录名：`<UTC 时间戳>-<实验编号>-<序号>`；`run_id` 与目录名一致，便于人工定位。
- 写入策略：manifest 先写临时文件再原子替换，避免半写状态。
- 单条记录的大小可控（不含权重），适合目录 + JSONL；需要按参数筛选时再引入 SQLite。
- 产物下载接口只允许访问运行目录内的相对路径，拒绝 `..` 与绝对路径。

### manifest 草案字段（M1 定稿）

`run_id`、`created_at`、`status`、`error_type`、`experiment_id`、`model{name,hash,precision}`、`control{type,source_sha256,params}`、`params{prompt,negative_prompt,width,height,steps,cfg,sampler,scheduler,seed,batch_size}`、`comfy{base_url,version,prompt_id,workflow_sha256}`、`outputs[{path,sha256,width,height}]`、`timing{queued_at,started_at,finished_at,latency_s}`、`resources{peak_vram_mb}`、`env{os,gpu,driver,torch,cuda}`、`notes`。

字段名在实现 M1 时固定，之后只增不改；新增字段必须可缺省，避免旧记录失效。

## 7. 并发与资源管理

- **单 worker**：GPU 只有一张，并行提交只会争夺显存；队列在进程内（`asyncio.Queue` 或单线程执行器）。
- **队列语义**：FIFO；提交即返回 `run_id`；状态 `queued → running → succeeded / failed / cancelled`。
- **取消**：仅在排队阶段保证可取消；执行中取消属于尽力而为（ComfyUI 提供 `/interrupt`，MVP 不实现）。
- **显存**：MVP 限定 1024²、batch 1；高分辨率与大 batch 作为实验变量而非默认能力。OOM 必须以可识别错误返回，而不是让服务进程崩溃。
- **已知限制**：进程内队列不持久化，重启会丢失排队任务；MVP 接受该限制，需要任务恢复时再引入 SQLite（见第 11 节触发条件）。

## 8. 配置

- **应用配置**：`pydantic-settings` 从环境变量 / `.env` 读取（前缀 `GAS_`），包含 ComfyUI 地址、目录、日志级别。示例见 `.env.example`。
- **实验配置**：每个实验一个 YAML（`configs/experiments/*.yaml`，M4 创建），只描述实验变量与固定项；路径与端点不写入实验配置。
- **优先级**：显式函数参数 > 环境变量 > `.env` > 默认值。不做多份配置合并的隐式魔法。
- 密钥只从环境变量读取；`.env` 与产物目录均被 `.gitignore` 排除。

## 9. 可观测性与复现

- 日志：标准库 `logging` + 结构化字段（`run_id`、阶段、prompt_id）；写入 `run.log` 同时输出到控制台。
- 每次运行记录：模型哈希、工作流哈希、ComfyUI 版本、torch / CUDA / 驱动版本、注意力后端（如 SDPA / xformers，因其可能影响数值结果）。
- 复现检查（M2 起）：提供脚本按 manifest 重跑并比对图像；不一致时报告差异（哈希、像素差），不静默通过。
- 评测模块只读 `runs/`，不反过来修改记录。

## 10. 安全边界

- 服务默认绑定 `127.0.0.1`；MVP 无鉴权。若需对外暴露，必须另加反向代理与令牌（不在 MVP 范围）。
- **不接受客户端提交任意工作流 JSON**：ComfyUI 工作流可引用本地文件路径，任意图会变成文件读取 / 覆盖风险。只允许参数化模板 + 校验过的取值。
- 上传文件：限制大小与类型，解码验证为图像，保存时使用服务生成的文件名（不信任原文件名）。
- 产物路径：解析后必须位于目标运行目录内（防路径穿越）。
- 日志与 manifest 不写入密钥、令牌、完整环境变量。
- ComfyUI 视为本机可信组件，但仍只通过 localhost 访问。

## 11. 演进路径（触发式，不预先实现）

| 演进项 | 触发条件 | 方案 |
| --- | --- | --- |
| SQLite 任务 / 运行表 | 需要跨重启恢复任务，或 JSONL 筛选变慢 | 引入 SQLite，保留 runs 目录为产物真相来源 |
| WebSocket 进度 | UI 需要秒级进度，轮询开销明显 | 接 ComfyUI `/ws`，后端转发 SSE / WS |
| 多 worker / 第二台机器 | 出现吞吐瓶颈且显存允许并发 | 队列外置（Redis / 轻量 broker）+ 推理注册表；届时重新评估基础设施 |
| 自研推理管线 | ComfyUI API 变更成本高于维护成本，或需要自定义采样 | adapter 后替换为 diffusers 管线，保持 manifest 契约不变 |
| 数据库化评测 | 指标数量与筛选维度增长 | metrics 表 + 只读报表层 |

## 12. 技术决策记录（ADR 摘要）

### AD-1 生成引擎使用 ComfyUI（独立进程 + HTTP）

- **背景**：需要 SDXL + ControlNet，且要能快速迭代工作流、批量执行、可复现。
- **决策**：ComfyUI 独立安装，后端通过 HTTP API 调用，工作流以 API JSON 模板管理。
- **替代方案**：
  - *直接用 diffusers 写管线*：控制力最强、依赖最少，但需要自行管理工作流组织、显存优化与节点级调试，MVP 周期更长。保留为后续替换路径。
  - *AUTOMATIC1111 + API*：生态成熟，但 ControlNet 组合与工作流的声明式表达不如 ComfyUI，批量参数化更别扭。
- **代价**：依赖非正式版本化的 API；锁定版本、模板校验、哈希记录是必需成本。
- **复审条件**：API 破坏性变更频繁导致维护成本超过自研管线，或需要 ComfyUI 不支持的自定义采样。

### AD-2 工作流采用静态模板 + 显式参数映射（而非运行时动态建图）

- **决策**：仓库内保存 API 格式模板，节点 ID 与参数映射写成显式表；每次运行保存实际工作流副本。
- **替代方案**：通过 `/object_info` 动态构建图。更抗节点变更，但代码量与调试成本更高。
- **代价**：节点类名变更需人工更新模板（启动校验会报错，不会静默失败）。
- **复审条件**：需要大量动态拓扑（如多 ControlNet 分支组合）时，再引入构图层。

### AD-3 文件系统 + JSONL 作为运行记录

- **决策**：每次运行一个目录 + manifest；全局索引为追加写的 JSONL。
- **替代方案**：SQLite（查询强，但 MVP 不需要）、PostgreSQL（过重）。
- **代价**：复杂查询需脚本扫描；并发写索引需要加锁（单 worker 下影响很小）。
- **复审条件**：见第 11 节。

### AD-4 单 worker 进程内队列

- **决策**：串行执行，FIFO，进程内队列。
- **替代方案**：Redis / Celery / RQ。对单卡单用户属于过度设计。
- **代价**：重启丢队列；无水平扩展。
- **复审条件**：需要持久化任务或并发推理。

### AD-5 轮询而非 WebSocket（MVP）

- **决策**：按固定间隔轮询 `/history`。
- **替代方案**：WebSocket 实时进度。连接管理、重连与测试成本更高，UI 尚不存在。
- **复审条件**：M5 做 UI 且轮询体验不可接受。

### AD-6 后端 FastAPI + httpx + pydantic-settings

- **决策**：FastAPI 提供 API 与校验，httpx 异步调用 ComfyUI，pydantic-settings 管配置。
- **替代方案**：Flask（同步生态，校验与异步弱）、直接脚本无服务（不利于实验编排与后续 UI）。
- **代价**：多一层依赖；优势是 OpenAPI 文档、类型校验与可测试性。

### AD-7 模型与权重不入库、不自动化下载

- **决策**：权重放 ComfyUI 的 models 目录；仓库只记录文件名、来源、许可与哈希。
- **原因**：体积、许可与供应链安全。替代方案（LFS / 自动下载脚本）在许可确认前不采用。
- **复审条件**：许可证明确且需要团队一致环境时，再评估下载脚本或制品库。

### AD-8 应用配置与实验配置分离

- **决策**：环境变量管端点与路径，YAML 管实验变量。
- **原因**：避免实验文件携带机器相关信息，保证实验配置可跨环境复用；也避免把实验参数塞进全局配置导致隐式耦合。

## 13. 目录映射（当前 → 目标）

当前实际只有 `src/gameasset_studio/__init__.py`、`py.typed` 与 `docs/`。目标结构按里程碑增量创建：

```text
src/gameasset_studio/
├── core/            # M1
├── comfy/           # M1
├── storage/         # M1
├── generation/      # M2
├── api/             # M2
├── experiments/     # M4
└── evaluation/      # M4

workflows/           # M1（API 格式模板，纳入版本控制）
configs/experiments/ # M4
scripts/             # M1
tests/               # M1 起逐步补齐
runs/                # 运行时生成，gitignored
```
