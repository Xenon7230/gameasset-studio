# GameAsset Studio

面向游戏概念美术的可控图像生成与评测平台：以 **SDXL + ControlNet** 为核心，把"生成一张图"升级为"可控、可复现、可评测的实验流程"。

> **项目状态：M0（仓库初始化）**
> 当前仓库只有文档、项目配置与 Python 包骨架。**尚未实现任何生成、服务或评测功能**；除"已完成"明确标注的内容外，本文全部为规划。任何性能数字与实验结论都要在实际测量后回填，见 [docs/experiment-plan.md](docs/experiment-plan.md)。

## 1. 定位与要解决的问题

游戏概念美术的前期探索通常是"大量出图 + 人工挑图"。真正的工程难点不是调用一次模型，而是让生成过程**可控、可复现、可比较**：

| 问题 | 具体表现 | 本项目的应对 |
| --- | --- | --- |
| 不可控 | 纯文生图很难稳定得到指定构图 / 姿态 | ControlNet 结构条件（首发 Canny） |
| 不可复现 | 参数散落，模型与工作流版本不明，好结果无法重来 | 每次生成落盘完整 manifest（参数、工作流、模型、环境） |
| 不可评估 | 只能凭感觉挑图，无法回答"哪组设置更好" | 对照实验 + 量化指标 + 人工评分，保留原始产物 |
| 不可观测 | 不清楚单卡性能边界，长任务失败无记录 | 记录耗时、显存峰值、失败原因 |

## 2. 目标用户

- 独立游戏团队的概念美术 / 技术美术：需要按给定构图批量探索风格变体。
- 个人美术：希望把提示词与结构参考固化成可复用、可复现的流程。
- 生成模型工程实践者（包含项目自身的工程验证目的）：关注条件控制、推理性能与实验可复现性。

本项目不是面向终端玩家的产品，也不做商业级资产管线。

## 3. 功能规划

### P0 — MVP 范围

- SDXL 1024² 文生图：正向 / 负向提示词、尺寸、步数、CFG、采样器、调度器、种子。
- ControlNet Canny 结构控制：参考图、控制强度、起止区间、Canny 阈值。
- 运行记录：每次生成输出可复现的 manifest。
- 实验运行器：按配置批量展开参数组合，汇总结果清单。
- 基础评测：结构一致性、提示词一致性（探索性）、耗时与显存统计。

### P1 — MVP 之后

- 多控制条件组合（Canny + Depth/Pose）、图生图、局部重绘、放大。
- 简单 Web 界面（生成表单 + 结果画廊）。
- 进度推送（WebSocket）、任务取消与失败重试。

### P2 — 远期，未承诺

- LoRA / 风格管理、参考图相似度、人工评分界面、多用户与队列。
- 第二台机器（R7000P）接入推理池。

### 明确不做

- 训练基础模型或 ControlNet（阶段一只使用预训练权重）。
- 模型托管 / 分发平台、3D 资产生成、商业级美术生产管线。
- 在 MVP 之前引入 Redis、PostgreSQL、Docker、分布式推理。

## 4. 技术栈与选型

| 层 | 选型 | 状态 | 理由 / 替代方案 |
| --- | --- | --- | --- |
| 生成引擎 | ComfyUI | 已确定（阶段一） | 可视化调试工作流 + 可编程 HTTP API；相比直接写 diffusers 管线，节点级控制与问题定位更快。代价：API 非正式版本化，需要锁定 ComfyUI 版本（见架构文档 AD-1） |
| 基础模型 | SDXL 1.0 base | 计划，权重来源待确认 | 1024² 原生分辨率、生态成熟、16GB 显存可承载。SD1.5 质量上限较低；Flux 等更大模型在 16GB 上余量更小，后续再评估 |
| 结构控制 | SDXL ControlNet Canny | 计划，具体权重待确认 | 边缘条件最直观、最容易验证"结构是否被遵循"；Depth / Pose 留作 P1 |
| 推理框架 | PyTorch + CUDA | 计划，需在 sm_120 上验证 | RTX 5060 Ti 为 Blackwell 架构，必须使用 CUDA 12.8+ 对应的 PyTorch 构建；具体版本以安装时官方 wheel 为准 |
| 后端 | FastAPI + Uvicorn | 计划（M2） | 异步、Pydantic 校验、自动 OpenAPI 文档。替代方案 Flask：生态成熟但异步与类型支持较弱 |
| HTTP 客户端 | httpx | 计划（M1） | 同步 / 异步接口统一，测试可用 MockTransport。替代方案 requests：仅同步 |
| 配置 | pydantic-settings + YAML | 计划 | 环境变量管路径与端点，YAML 管实验参数，避免自研配置层 |
| 运行记录 | 文件系统 + JSONL | 计划 | 单机单用户、数据量小；需要复杂查询时再引入 SQLite |
| 评测 | Pillow / NumPy（指标库待定） | 计划（M4） | 先做结构指标与统计，CLIP / 美学模型等重型依赖需要单独评估 |

**明确不引入**：Redis、消息队列、PostgreSQL、Docker、Kubernetes。单机单卡单用户，任务串行执行即可；提前引入只会增加部署与调试成本。触发引入的条件写在 [docs/architecture.md](docs/architecture.md) 的演进章节。

## 5. 架构概览

```text
CLI / (M5) Web UI
      │ HTTP / JSON
      ▼
FastAPI 服务 ── 请求校验、任务提交与查询
      │
      ▼
生成 / 实验编排 ── 单 worker 队列、配置快照、manifest
      │                        │
      │ ComfyUI HTTP API       │ 文件系统
      ▼                        ▼
ComfyUI Adapter          runs/<run_id>/
      │                  manifest / workflow / images / metrics
      ▼
ComfyUI Server（独立安装，不纳入本仓库）
SDXL + ControlNet · RTX 5060 Ti 16GB

评测模块（离线）：读取 runs/ → 指标与汇总表
```

模块职责、ComfyUI 集成方式、存储布局与技术决策记录见 [docs/architecture.md](docs/architecture.md)。

## 6. 目录结构（当前实际内容）

```text
gameasset-studio/
├── README.md
├── pyproject.toml
├── .gitignore
├── .env.example
├── docs/
│   ├── project-brief.md
│   ├── architecture.md
│   ├── experiment-plan.md
│   └── roadmap.md
└── src/
    └── gameasset_studio/
        ├── __init__.py
        └── py.typed
```

规划中的目录（`api/`、`comfy/`、`generation/`、`experiments/`、`evaluation/`、`configs/`、`workflows/`、`scripts/`、`tests/`、`runs/`）在对应里程碑随真实代码创建，不预建空模块。

## 7. 环境要求与安装（待人工执行的部分）

**硬性要求**

- Windows 10/11，NVIDIA RTX 5060 Ti 16GB，驱动支持 CUDA 12.8+。
- Python 3.11+（仅后端；ComfyUI 使用其自带或独立环境）。
- ComfyUI 独立安装，**不放进本仓库**；模型权重下载到 ComfyUI 的 models 目录，**不进本仓库**。

**后端环境（当前可做）**

```bash
python -m venv .venv
source .venv/Scripts/activate        # Git Bash；PowerShell: .venv\Scripts\Activate.ps1
pip install -e ".[dev]"
```

当前没有可运行的业务入口；生成能力从 M1 开始提供。

**ComfyUI 侧（待确认后执行）**

1. 安装 ComfyUI（官方 portable 或独立 venv），并锁定版本（记入文档与 manifest）。
2. 验证 PyTorch 可用且架构匹配：
   `python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.get_device_capability())"`
   预期设备能力为 `(12, 0)`（sm_120）；若不可用，需更换为 CUDA 12.8+ 的构建。
3. 将 SDXL base 与 ControlNet Canny 权重放入 ComfyUI 对应目录，并记录文件名、来源与哈希。
4. 启动 ComfyUI，确认 `http://127.0.0.1:8188/system_stats` 返回 200。
5. 在 ComfyUI 中手动跑通一次 SDXL 文生图，导出 **API 格式**工作流 JSON，作为 M1 的模板基础。

以上步骤均未执行，状态为"待验证"。R7000P 暂不参与 GPU 推理。

## 8. 实验与评测思路

- 每次运行即一条实验记录：固定模型、工作流、参数、种子与环境指纹。
- 先建立基线（耗时、显存、质量），再做参数敏感性对照（步数 / CFG / 采样器），最后做 ControlNet Canny 的控制强度对照。
- 自动指标用于筛选，人工评分用于定性判断；两者不一致时如实报告，不把 CLIP 分数当"质量"。
- 结果表只填写实际测得的数值；未运行即留空并标注 `TBD`。

详细方案见 [docs/experiment-plan.md](docs/experiment-plan.md)。

## 9. 开发路线

| 里程碑 | 内容 | 状态 |
| --- | --- | --- |
| M0 | 仓库初始化：文档、配置、包骨架 | ✅ 已完成 |
| M1 | ComfyUI 连通性验证 + 脚本级单图生成 + 首个 manifest | ⬜ 未开始 |
| M2 | FastAPI 生成服务、单 worker 任务队列、运行查询 | ⬜ 未开始 |
| M3 | ControlNet Canny 集成与参数化 | ⬜ 未开始 |
| M4 | 实验运行器、自动指标、结果汇总 | ⬜ 未开始 |
| M5（可选） | Web 界面、进度推送、队列持久化、多控制条件 | ⬜ 未开始 |

退出标准与 TODO 清单见 [docs/roadmap.md](docs/roadmap.md)。

## 10. 文档索引

- [docs/project-brief.md](docs/project-brief.md) — 范围、目标用户、核心问题、MVP 与验收标准
- [docs/architecture.md](docs/architecture.md) — 预期架构、模块职责、ComfyUI 集成、技术决策
- [docs/experiment-plan.md](docs/experiment-plan.md) — 基线生成与 ControlNet 对照实验设计
- [docs/roadmap.md](docs/roadmap.md) — 里程碑、退出标准、初始 TODO

## 11. 待确认事项

| # | 事项 | 影响 | 建议 |
| --- | --- | --- | --- |
| 1 | SDXL base 与 ControlNet Canny 权重来源及许可证 | 可分发性与实验结论标注 | 优先官方 / 明确许可来源，记录哈希 |
| 2 | ComfyUI 安装形态（portable / 独立 venv）与版本锁定方式 | 环境复现、升级策略 | 独立目录 + 固定 commit，manifest 记录版本 |
| 3 | 后端 Python 版本（3.11 / 3.12） | 依赖兼容性、与 ComfyUI 环境隔离 | 后端独立 venv，版本与 ComfyUI 解耦 |
| 4 | 评测指标栈（Canny 预处理实现、CLIP 模型、美学模型） | 依赖体积与指标有效性 | M4 前用小规模样例评审 |
| 5 | 是否将 Web UI 纳入近期范围，以及许可证选择 | 工作量与对外使用方式 | MVP 完成后再决策 |

## 12. 已知限制

- 仓库无任何实测数据；文档中的阈值与样本量是计划，不是结论。
- ComfyUI API 与节点类名可能随版本变化，模板与节点名校验需要在 M1 完成。
- 自动指标存在效度问题（如 CLIP 分数不是美术质量），实验结论将标注适用范围。
- 当前无 CI、无测试、无持久化任务队列，均按路线图后置。

## 13. 许可证

尚未选择。在确定权重与第三方依赖的许可证之前不做分发承诺。
