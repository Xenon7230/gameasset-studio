# 实验计划：SDXL 基线生成与 ControlNet Canny 对照

> 状态：M0 设计稿。**尚无任何实验被执行，本文不含实验数据或结论。** 所有结果表保持空白，测量后回填，并保留原始记录路径。
> 相关文档：[project-brief.md](project-brief.md) · [architecture.md](architecture.md) · [roadmap.md](roadmap.md)

## 1. 目标

1. 建立**可复现**的基线：知道在 RTX 5060 Ti 16GB 上，一组固定参数生成一张 1024² SDXL 图需要多久、占多少显存、是否稳定。
2. 用对照实验回答：采样参数如何影响质量与成本；Canny 控制强度如何影响结构遵循与风格自由度。
3. 验证自动指标（边结构一致性、CLIP 相似度）在游戏概念美术场景下是否可用，以及与人工判断的偏差。
4. 沉淀一套实验记录格式，使任何一条结果都能被重跑与审计。

**非目标**：训练 / 微调模型；比较不同基础模型（SD1.5、Flux 等）的优劣；产出"最佳提示词"指南；做大规模众包评测。

## 2. 可复现性要求（硬性）

每次运行必须满足以下条件，否则记录标记为 `invalid`，不进入分析：

- [ ] 模型文件名 + sha256 + 精度（fp16 / fp32）已记录。
- [ ] 实际提交的 ComfyUI 工作流已保存（`workflow.api.json`）并记录哈希。
- [ ] 全部采样参数已记录：提示词、负向提示词、尺寸、步数、CFG、采样器、调度器、种子、batch size。
- [ ] 控制条件已记录：控制图原始文件 + 哈希、Canny 低/高阈值、控制强度、`start_percent` / `end_percent`、预处理实现。
- [ ] ComfyUI 版本（commit）、PyTorch / CUDA / 驱动版本、GPU 型号、注意力后端已记录。
- [ ] 原始输出图保留，不覆盖、不压缩、不裁剪。
- [ ] 失败运行同样保留记录与错误类型。

复现验证（EXP-01 的一部分）：用同一 manifest 重跑，比较 SHA256；若不一致，计算像素差异并记录原因假设，不以"应该一样"草率通过。

## 3. 环境记录模板（执行时填写，当前全部 TBD）

| 项 | 值 | 获取方式 |
| --- | --- | --- |
| GPU | NVIDIA RTX 5060 Ti 16GB（计划） | `nvidia-smi` |
| 驱动版本 | TBD | `nvidia-smi` |
| 操作系统 | Windows（版本 TBD） | `winver` |
| Python（后端 / ComfyUI） | TBD | `python -V` |
| PyTorch / CUDA | TBD（须支持 sm_120） | `torch.__version__` / `torch.version.cuda` |
| ComfyUI 版本 | TBD（锁定 commit） | `git rev-parse HEAD` |
| SDXL 权重 / 哈希 | TBD | 文件名 + sha256 |
| ControlNet Canny 权重 / 哈希 | TBD | 文件名 + sha256 |
| 注意力后端 | TBD（SDPA / xformers 等） | ComfyUI 启动日志 |
| 电源 / 性能模式 | TBD（影响热降频） | 系统设置 |

环境变化（驱动、ComfyUI、权重）后，历史基线不再可比，必须重跑 EXP-01 并在报表中注明。

## 4. 基线与固定项

**基线定义（B0）**：SDXL 1.0 base，fp16，1024×1024，batch 1，30 步，CFG 7.0，采样器 `dpmpp_2m`，调度器 `karras`，固定种子，单次生成。若 M1 实测该组合不可用（OOM 或明显异常），基线参数调整需记录原因并重跑。

> 采样器与调度器名称以 ComfyUI 实际提供为准；参数变更必须写入实验记录，不能默默替换。

**提示词集（草案，M1 前定稿）**：6 条，覆盖游戏概念美术常见类别；用于跨内容泛化，避免只用单一提示词得出结论。

| ID | 类别 | 提示词（草案，未测试） |
| --- | --- | --- |
| P1 | 角色 | `full body concept art of a wandering knight in weathered plate armor, tattered cloak, neutral A-pose, plain gray background, painterly, three-point lighting` |
| P2 | 武器 | `concept art of a runic greatsword, ornate hilt, dark steel blade, isolated on neutral background, orthographic side view, high detail` |
| P3 | 道具 | `concept art of an alchemist's potion set, glass vials, brass fittings, on a wooden table, studio lighting, neutral background` |
| P4 | 生物 | `concept art of a forest guardian creature, moss and bark texture, quadruped, side view, neutral background, painterly` |
| P5 | 场景 | `environment concept art of a ruined stone watchtower at dusk, overgrown ivy, distant mountains, cinematic composition` |
| P6 | 建筑 | `concept art of a fantasy blacksmith shop interior, forge glow, hanging tools, wooden beams, wide shot` |

选择标准：主体明确、构图可描述、避免依赖具体 IP；每条提示词在实验中作为固定项，不因结果不好而换掉（否则构成选择性偏差）。

**统一负向提示词（草案）**：`blurry, low quality, watermark, text, signature, deformed, extra limbs`。

## 5. 实验设计

### 5.1 总览

| 编号 | 名称 | 对应问题 | 变量 | 固定项 | 运行数（计划） |
| --- | --- | --- | --- | --- | --- |
| EXP-01 | 基线校准与确定性检查 | RQ1 | 种子 | B0 + P1 | 15（2 预热丢弃 + 3 同种子 + 10 不同种子） |
| EXP-02 | 采样参数敏感性 | RQ1 | 步数 × CFG × 采样器 | B0 + P1、P5，种子固定 | 16 |
| EXP-03 | Canny 控制强度 / 终止时机 | RQ2 | 控制强度 × 终止比例 | B0 + 3 张控制图、匹配提示词 | 27（24 + 3 无控制对照） |
| EXP-04 | 控制-自由度权衡分析 | RQ2 / RQ3 | 复用 EXP-03 产物 | — | 0（仅分析；必要时补跑） |
| EXP-05 | 性能剖面 | RQ1 | 分辨率 × 步数 | B0，固定种子，每格 3 次 | 18 |

合计约 76 次生成（不含重试）。按 GPU 逐次串行执行，实际耗时以实测为准，不预设数字。

### 5.2 EXP-01 基线校准与确定性检查

- **目的**：得到延迟、显存峰值、失败率的基线；验证同 manifest 重跑是否逐像素一致。
- **步骤**：
  1. 预热 2 次（丢弃，避免模型加载与缓存影响）。
  2. 同一种子连续 3 次，比较 SHA256；不一致则计算像素差异统计（max / mean / 不一致像素比例）。
  3. 10 个不同种子各 1 次，记录延迟与显存峰值。
- **记录**：`metrics.json` 中写 `latency_s` 列表、`peak_vram_mb` 列表、`hash_equal` 与差异统计。
- **判定**：确定性结果如实记录。若不一致，A2 验收标准按 `project-brief.md` 的"记录实测差异并给出判定标准"执行。

### 5.3 EXP-02 采样参数敏感性

- **变量**：步数 {20, 30} × CFG {5.0, 7.0} × 采样器 {`dpmpp_2m`, `euler_a`}。
- **固定**：B0 其余参数、调度器 `karras`、提示词 P1 / P5、种子 {1001}。
- **运行数**：2 × 2 × 2 × 2（提示词）= 16。
- **分析**：每个组合的耗时、显存、结构指标（无控制条件时不适用）、CLIP 相似度、人工评分；重点看"成本增加是否带来可感知质量提升"。
- **注意**：不同采样器对同一种子的结果不可比"相似度"，只比较质量评分与成本。

### 5.4 EXP-03 Canny 控制强度与终止时机

- **控制图**：3 张，来源为自制灰模 / 线稿截图（不使用他人版权素材）。需记录来源与哈希。
- **变量**：控制强度 {0.4, 0.6, 0.8, 1.0} × `end_percent` {0.5, 1.0}。
- **固定**：`start_percent=0.0`、Canny 阈值（低/高）固定为选定值、B0 采样参数、每张控制图配一条匹配提示词、种子 {2001}。
- **对照**：同样提示词、不接 ControlNet 的 3 次生成（每张控制图对应提示词各 1 次）。
- **运行数**：3 × 4 × 2 + 3 = 27。
- **分析**：结构一致性（边 IoU）随强度变化；`end_percent=0.5` 与 `1.0` 的差异；人工判断"结构是否可辨识对应参考图"与"画面是否僵硬"。
- **解释边界**：控制强度的影响依赖 ControlNet 权重与 Canny 阈值，结论只对被测权重与阈值成立。

### 5.5 EXP-04 控制强度与提示词自由度的权衡（仅分析）

- **不新增生成**：复用 EXP-03 全部产物。
- **分析**：
  - 边 IoU（结构）随强度上升的趋势；
  - CLIP 相似度（提示词遵循）随强度变化的趋势；
  - 人工评分中"结构一致性"与"美术可用性"的此消彼长；
  - 自动指标与人工评分的一致性（排序是否一致）。
- **补跑条件**：仅当某格结果缺失或失败率 > 20% 时补跑，补跑原因写入记录。

### 5.6 EXP-05 性能剖面

- **变量**：分辨率 {768×768, 1024×1024, 1152×896} × 步数 {20, 30}。
- **固定**：B0 其余参数、batch 1、种子 {3001}、每格重复 3 次（保留全部 3 次）。
- **运行数**：3 × 2 × 3 = 18。
- **测量**：端到端延迟（提交 → 图片落盘）、ComfyUI 侧执行时间（若 `/history` 提供时间戳则另记）、每 200ms 采样的显存峰值、OOM / 失败。
- **说明**：1152×896 是否可在 16GB 内完成属于**待验证项**，失败也是有效结果，需记录而非隐藏。

## 6. 控制变量与混杂因素

| 类别 | 控制项 |
| --- | --- |
| 模型 | 同一权重文件与哈希、同一精度、同一 VAE |
| 工作流 | 同一模板版本与哈希；不混用不同版本导出 |
| 采样 | 除实验变量外全部固定（调度器、batch、种子） |
| 输入 | 提示词集固定；控制图固定并记录哈希 |
| 环境 | 同一 ComfyUI 版本、驱动、torch；实验期间不升级 |
| 运行 | 单任务串行，避免并发争抢显存；记录电源模式与 GPU 温度（如有） |
| 分析 | 分析脚本版本固定；不手工挑选样本 |

**已知混杂因素**：GPU 热降频会拉长后段运行耗时 → 记录运行顺序，必要时交替排序；Windows 后台进程占用显存 → 运行前确认空闲；ComfyUI 模型缓存影响首次加载时间 → 统一预热。

## 7. 指标定义

| 指标 | 定义 | 工具 | 效度说明 |
| --- | --- | --- | --- |
| 端到端延迟 | 请求提交到图片落盘的时间（秒） | 后端计时 | 含排队与下载；报表同时给 ComfyUI 侧时间（可得时） |
| 显存峰值 | 运行期间 `nvidia-smi` 采样最大值（MB） | 200ms 轮询 | 采样可能低估瞬时峰值；注明采样间隔 |
| 失败率 | 失败运行数 / 总运行数 | manifest 统计 | 分类：OOM、超时、工作流错误、ComfyUI 不可用 |
| 边结构一致性 | 控制图与生成图分别做 Canny 后，计算边 IoU / 精确率 / 召回率；生成图侧允许 2px 膨胀容差 | NumPy / OpenCV（待定） | 风格化边缘会干扰，不能代表"构图完全一致" |
| 提示词一致性 | CLIP 图像 / 文本嵌入余弦相似度 | CLIP 模型（待定） | 已知偏差：偏向主体名词、对风格描述不敏感；仅作筛选 |
| 人工结构评分 | 1–5：生成图与参考结构的可辨识对应程度 | 评分表 | 单人评分存在主观性；样本量小，报告原始分布 |
| 人工可用性评分 | 1–5：作为概念草案的可用程度 | 评分表 | 同上 |
| 确定性 | 同 manifest 重跑图像的 SHA256 是否一致；不一致时给像素差 | 哈希 + Pillow/NumPy | 受硬件与算子非确定性影响 |

**阈值策略**：不在实验前编造"合格线"。EXP-01 完成后，依据实测分布确定后续实验的解读基准，并把依据写入报告。

## 8. 人工评分流程

- 评分表：每张图三个维度（结构一致性、提示词符合度、美术可用性），1–5 整数，附一句话理由。
- 盲评：评分时隐藏参数组合（文件名映射由脚本生成），避免"知道强度就倾向打高分"。
- 评分顺序随机化；同一张图不重复评分。
- 单人评分时明确标注"单人、非盲二次复核缺失"；若引入第二评分者，报告两者原始分数与分歧案例，小样本不做显著性检验。
- 所有评分原表随实验目录保存。

## 9. 运行记录格式（草案）

字段与 [architecture.md](architecture.md) 第 6 节一致。示例仅说明结构，值未填充：

```json
{
  "run_id": "TBD-<timestamp>-exp03-<n>",
  "experiment_id": "EXP-03",
  "status": "planned",
  "created_at": "TBD",
  "model": {"name": "TBD", "hash": "TBD", "precision": "fp16"},
  "control": {
    "type": "canny",
    "source": "inputs/control_01.png",
    "source_sha256": "TBD",
    "preprocessor": "TBD",
    "low_threshold": "TBD",
    "high_threshold": "TBD",
    "strength": 0.6,
    "start_percent": 0.0,
    "end_percent": 1.0
  },
  "params": {
    "prompt": "TBD",
    "negative_prompt": "TBD",
    "width": 1024,
    "height": 1024,
    "steps": 30,
    "cfg": 7.0,
    "sampler": "dpmpp_2m",
    "scheduler": "karras",
    "seed": 2001,
    "batch_size": 1
  },
  "comfy": {"base_url": "http://127.0.0.1:8188", "version": "TBD", "prompt_id": "TBD", "workflow_sha256": "TBD"},
  "outputs": [{"path": "images/0001.png", "sha256": "TBD", "width": 1024, "height": 1024}],
  "timing": {"queued_at": "TBD", "started_at": "TBD", "finished_at": "TBD", "latency_s": null},
  "resources": {"peak_vram_mb": null},
  "env": {"os": "TBD", "gpu": "RTX 5060 Ti 16GB", "driver": "TBD", "torch": "TBD", "cuda": "TBD"},
  "notes": null
}
```

实验配置草案（M4 实现时定稿）：

```yaml
id: EXP-03
description: ControlNet Canny strength and end-percent sweep
fixed:
  model: TBD
  width: 1024
  height: 1024
  steps: 30
  cfg: 7.0
  sampler: dpmpp_2m
  scheduler: karras
  negative_prompt: TBD
variables:
  control_image: [control_01, control_02, control_03]
  strength: [0.4, 0.6, 0.8, 1.0]
  end_percent: [0.5, 1.0]
seeds: [2001]
```

## 10. 结果记录模板（执行后回填）

### 10.1 基线（EXP-01）

| 指标 | N | 中位数 | 最小值 | 最大值 | 备注 |
| --- | --- | --- | --- | --- | --- |
| 端到端延迟 (s) | TBD | — | — | — | 尚未运行 |
| 显存峰值 (MB) | TBD | — | — | — | 尚未运行 |
| 失败率 | TBD | — | — | — | 尚未运行 |
| 同种子 SHA256 一致 | 3 | — | — | — | 尚未运行 |

### 10.2 采样参数（EXP-02）

| 步数 | CFG | 采样器 | 提示词 | 延迟 (s) | 显存 (MB) | 结构指标 | CLIP | 人工综合分 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| TBD | TBD | TBD | P1 | — | — | — | — | — |

### 10.3 Canny 控制（EXP-03 / EXP-04）

| 控制图 | 强度 | end | 边 IoU | CLIP | 人工结构分 | 人工可用性分 |
| --- | --- | --- | --- | --- | --- | --- |
| TBD | 0.4 | 1.0 | — | — | — | — |
| TBD | 0.6 | 1.0 | — | — | — | — |
| TBD | 0.8 | 1.0 | — | — | — | — |
| TBD | 1.0 | 1.0 | — | — | — | — |

### 10.4 性能剖面（EXP-05）

| 分辨率 | 步数 | 延迟 (s)，3 次 | 显存峰值 (MB) | 失败 |
| --- | --- | --- | --- | --- |
| 1024×1024 | 30 | — | — | — |

> 表格中的 `—` 表示未测量，不是 0。禁止用估计值填充。

## 11. 分析与汇报规范

1. 报告所有运行，包括失败和异常值；失败率与失败原因单独成表。
2. 延迟报告 N、中位数、最小值、最大值；小样本不报告 p 值，不做显著性声明。
3. 质量结论必须同时引用自动指标与人工评分；两者冲突时作为发现报告，而不是择一。
4. 图片选择展示时说明筛选标准，并附完整结果目录路径。
5. 结论限定在"该权重 + 该版本 + 该硬件 + 该提示词集"范围内；不外推。
6. 每次实验结束在 `docs/` 新增结果文件（如 `docs/results/EXP-03.md`），本文只保留设计与模板，不混入结论。

## 12. 效度威胁

| 威胁 | 影响 | 缓解 |
| --- | --- | --- |
| 提示词集规模小、领域窄 | 结论依赖内容类型 | 6 类内容覆盖；结论标注范围；后续可扩展但需重跑 |
| 单一种子 | 结果受随机性影响 | 关键实验使用多个种子或在 EXP-01 中量化方差 |
| 单人评分 | 主观偏差、疲劳 | 盲评、随机顺序、固定评分表；明确标注局限 |
| 边 IoU 把风格笔触当作结构 | 结构指标虚高 / 虚低 | 容差固定并公开；结合人工评分；不单独下结论 |
| CLIP 分数不衡量美术质量 | 误读为"质量" | 仅用于提示词遵循的粗筛；报告中明确措辞 |
| 热降频、后台占用 | 延迟不稳定 | 预热、记录顺序与温度、运行前检查空闲 |
| ComfyUI 默认优化随版本变化 | 跨版本不可比 | 锁定版本；环境字段入库；升级后重跑基线 |
| 小样本 | 统计功效低 | 明示为探索性；不宣称显著性 |

## 13. 执行顺序与依赖

```text
M1: 环境冒烟 + 单图生成 + manifest
      └─→ EXP-01 基线（必须先完成，确定参数与统计口径）
M2: API 与队列（实验可使用服务层，或直接调用 service）
M3: ControlNet Canny
      ├─→ EXP-02 采样参数（可与 M2 并行准备，但需稳定执行环境）
      └─→ EXP-03 Canny 对照
M4: 评测与实验运行器
      ├─→ EXP-04 分析（依赖 EXP-03 产物）
      ├─→ EXP-05 性能剖面
      └─→ 结果报告 docs/results/
```

## 14. 待确认事项

1. SDXL base 与 ControlNet Canny 的具体权重文件、来源与许可证。
2. 采样器 / 调度器实际可用名称（以 ComfyUI 安装版本为准）。
3. Canny 预处理实现：ComfyUI 内置 `Canny` 节点 vs OpenCV，二者阈值语义可能不同，**方案一经选定不得在实验中途更换**。
4. 结构指标实现与依赖（OpenCV / scikit-image）及容差参数。
5. CLIP 模型选择（OpenCLIP ViT-L/14 等）与权重来源；是否引入美学评分模型。
6. 控制图素材的自制方式与版权记录。
7. 是否引入第二评分者；若引入，评分流程需在实验前确定。
