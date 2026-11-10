# 路线图与初始 TODO

> 状态：M0 已完成，M1 未开始。里程碑只有在退出标准全部满足后才标记完成。
> 相关文档：[project-brief.md](project-brief.md) · [architecture.md](architecture.md) · [experiment-plan.md](experiment-plan.md)

## 1. 里程碑

### M0 — 仓库初始化 ✅ 已完成（2026-11-10）

**产出**

- 目录骨架：`src/gameasset_studio/`、`docs/`
- 项目配置：`pyproject.toml`、`.gitignore`、`.env.example`
- 文档：`README.md`、项目简报、架构设计、实验计划、本路线图
- git 仓库初始化（`main` 分支，尚未提交）

**退出标准**

- [x] 文档覆盖定位、范围、架构、实验设计与里程碑
- [x] 已区分"已实现 / 计划 / 待确认"，无虚构功能与数据
- [x] 权重、缓存、产物、环境文件、密钥均被 `.gitignore` 排除
- [x] 未安装大型依赖、未下载权重、未实现业务代码

---

### M1 — ComfyUI 连通性验证与脚本级单图生成 ⬜ 未开始

**目标**：证明"这台机器能生成一张 SDXL 图，并且过程可记录"，不涉及 API 服务。

**范围**

- `core/settings`、`core/logging`、`core/errors`
- `comfy/client`：`/system_stats`、`/prompt`、`/history`、`/view`
- `comfy/workflows`：加载、校验、参数替换
- `workflows/sdxl_txt2img.api.json`：从 ComfyUI 导出的 API 格式模板
- `storage/runs`：运行目录、manifest 初版、`index.jsonl`
- `scripts/run_once.py`：一次生成，输出 run 目录
- 最小单测：模板校验、参数映射、manifest 序列化（mock ComfyUI）
- 环境记录：驱动、CUDA、torch、ComfyUI commit、注意力后端

**退出标准**

- [ ] `scripts/run_once.py` 在目标机器生成 1024² 图像，产物与 manifest 落盘
- [ ] manifest 含模型哈希、完整参数、种子、工作流哈希、ComfyUI 版本、实测延迟
- [ ] 同 manifest 重跑并完成一致性比对（结论如实记录，含不一致的情形）
- [ ] 启动时节点 / 模型校验能对缺失项给出明确错误
- [ ] 无 GPU 环境下单测通过
- [ ] 文档回填真实环境版本，移除对应 TBD

**不在本里程碑**：FastAPI、队列、ControlNet。

---

### M2 — 生成服务 API 与单 worker 队列 ⬜ 未开始

**目标**：把 M1 的能力暴露为可查询的 HTTP 服务。

**范围**

- `api`：`POST /v1/generations`、`GET /v1/generations/{run_id}`、产物下载、健康检查
- `generation/service`：参数校验、调用、落盘、manifest 状态机
- `generation/queue`：FIFO、`queued/running/succeeded/failed/cancelled`、超时
- 错误映射：ComfyUI 不可用 / 工作流错误 / 超时 / OOM 分类
- 测试：API 层使用 mock ComfyUI（`httpx.MockTransport`）

**退出标准**

- [ ] `project-brief.md` 验收标准 A1、A2、A5、A6 满足
- [ ] 失败请求返回可区分的错误类型，不返回 500 泛化信息
- [ ] 队列状态与 manifest 一致，重启后历史记录仍可查询
- [ ] 产物下载阻止路径穿越

**已知限制（接受）**：队列在进程内，重启丢失排队任务。

---

### M3 — ControlNet Canny 集成 ⬜ 未开始

**目标**：支持参考图驱动的结构控制。

**范围**

- `POST /upload/image` 流程接入：参考图保存、哈希、尺寸校验
- `workflows/sdxl_canny.api.json`：ControlNet Canny 模板
- 参数：控制强度、`start_percent` / `end_percent`、Canny 低/高阈值
- 生成请求扩展：可选 `control` 块
- 文档：参数语义与边界（阈值依赖预处理实现）

**退出标准**

- [ ] 验收标准 A3 满足（3 张参考图，人工评分门槛）
- [ ] 保留无控制对照运行，并记录控制图哈希
- [ ] 非法参考图（损坏、超尺寸、非图像）被拒绝且错误明确

---

### M4 — 实验运行器与基础评测 ⬜ 未开始

**目标**：从"能生成"升级为"能按实验设计批量生成并汇总"。

**范围**

- `experiments/definitions`：实验 YAML schema 与校验
- `experiments/runner`：矩阵展开、串行执行、失败记录、断点续跑
- `evaluation/metrics`：边结构指标、延迟 / 显存统计；CLIP 指标待确认后接入
- `evaluation/report`：生成 Markdown / CSV 汇总
- 执行 EXP-01、EXP-02、EXP-03、EXP-05；EXP-04 做离线分析
- 结果文件：`docs/results/EXP-0x.md`（新建）

**退出标准**

- [ ] 验收标准 A4 满足；失败与重试有记录
- [ ] 实验配置快照、工作流副本、manifest 三者可对齐
- [ ] 报表只包含实测数据，未测项标 `—` 或 `TBD`
- [ ] 实验计划中的表格被真实数据回填，并附原始目录路径

---

### M5 — 可选增强 ⬜ 未开始（不承诺）

- Web UI（生成表单 + 结果画廊）
- WebSocket 进度、任务取消（`/interrupt`）
- SQLite 任务持久化（触发条件见架构文档第 11 节）
- 多控制条件（Canny + Depth / Pose）
- 第二台机器接入推理池（当前 R7000P 不参与）

## 2. 初始 TODO

### 环境与选型（人工确认）

- [ ] 确认 SDXL base 权重来源、文件名与许可证 → 记录 sha256
- [ ] 确认 SDXL ControlNet Canny 权重来源与许可证 → 记录 sha256
- [ ] 选定 ComfyUI 安装形态（portable / 独立 venv）并锁定版本
- [ ] 验证 PyTorch CUDA 构建在 sm_120 上可用（`get_device_capability() == (12, 0)`）
- [ ] 决定后端 Python 版本（3.11 / 3.12），与 ComfyUI 环境隔离
- [ ] 决定 Canny 预处理实现（ComfyUI 内置 vs OpenCV），实验中途不得更改
- [ ] 决定评测指标栈（结构指标库、CLIP 模型、是否引入美学模型）
- [ ] 选择项目许可证（权重许可确认后再定）

### M1

- [ ] 创建 `workflows/sdxl_txt2img.api.json` 并验证节点类名
- [ ] 实现 settings / logging / errors
- [ ] 实现 ComfyUI client（含超时与错误分类）
- [ ] 实现模板加载、参数映射与启动校验
- [ ] 实现运行目录、manifest、`index.jsonl`
- [ ] `scripts/run_once.py` 跑通并采集基线数据
- [ ] 确定性检查（同种子 3 次）
- [ ] 单测 + `ruff` + `mypy` 本地通过
- [ ] 回填环境版本表（实验计划第 3 节）

### 工程与流程

- [ ] 首次提交（M0 文档与配置）
- [ ] 预提交检查（ruff / mypy，可选）
- [ ] CI：仅 lint + 无 GPU 测试（不含模型下载）
- [ ] 约定分支与提交信息格式（单人项目可简化）

### 实验准备

- [ ] 定稿 6 条提示词与统一负向提示词
- [ ] 制作 3 张控制图（自制、记录来源与版权）
- [ ] 确定 Canny 阈值与实验种子集合
- [ ] 设计人工评分表与盲评文件名映射脚本

### 后续（M2–M4，占位）

- [ ] FastAPI 路由与 schema
- [ ] 单 worker 队列与状态机
- [ ] ControlNet 模板与请求扩展
- [ ] 实验 YAML schema 与 runner
- [ ] 指标实现与报表
- [ ] 执行并记录 EXP-01 → EXP-05

## 3. 决策门（Decision Gates）

| 门 | 时机 | 需要决定 | 不通过的后果 |
| --- | --- | --- | --- |
| G1 | M1 开始前 | 权重来源与许可、ComfyUI 安装形态 | 无法开始环境验证 |
| G2 | M1 结束 | 实测参数是否作为基线；确定性判定标准 | 实验结论不可比 |
| G3 | M2 结束 | 是否需要任务持久化（当前判断：不需要） | 若需要，M3 前引入 SQLite |
| G4 | M3 结束 | Canny 预处理方案是否固定 | 实验中途更换会导致结果不可比 |
| G5 | M4 开始前 | 指标栈与人工评分流程 | 评测实现返工 |

## 4. 建议的第一个可运行任务

**M1 的第一步**：在目标机器安装并启动 ComfyUI，手动跑通一张 SDXL 1024² 文生图，导出 API 格式工作流 JSON。

理由：它一次性验证了最不确定的三件事——sm_120 的 PyTorch 可用性、ComfyUI 可用性、以及工作流模板能否被后端参数化。完成后再写代码，避免在错误的环境假设上搭建 adapter。

紧随其后：`scripts/run_once.py` 用该模板生成一张图并写出完整 manifest，即 M1 的退出标准。

## 5. 进度跟踪约定

- 里程碑状态只允许 `⬜ 未开始 / 🚧 进行中 / ✅ 已完成`。
- 完成里程碑时，同时更新本文件、README 第 9 节、以及实验计划中的环境表。
- 未验证的事项不得从 TODO 中删除，只能标注阻塞原因。
