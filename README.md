# CodeBoost

> 探究大模型生成代码的运行效率提升：以 **PerfCodeGen** 为主线，借鉴 **EffiLearner** 的细粒度开销反馈。

## 项目简介

在保持功能正确的前提下，降低大模型生成代码的运行时间与内存占用。
项目以 **PerfCodeGen** 为研究主线：先用失败测试修正正确性，再用最耗时测试的执行反馈改进性能。
**EffiLearner** 的时间/内存剖析反馈作为增强反馈与消融路线；主实验题目优先从具备有效测试的
EffiBench 题目中筛选。筛题和评测规则见 [`experiment/PROTOCOL.md`](experiment/PROTOCOL.md)。

## 参考论文

- **PerfCodeGen**（FORGE 2025）：先用执行反馈修正功能，再用执行反馈改进运行效率。
- **EffiLearner**（NeurIPS 2024）：将时间/内存剖析反馈给模型进行优化（本项目复现主线）。
- **EffiBench**（NeurIPS 2024）：效率评测基准，1000 道效率关键题与 6 个效率指标。

## 当前进展

G0～G3 递进反馈实验已完成：G0 无反馈、G1 程序总耗时、G2 追加最慢测试、G3 再追加热点源码与执行次数。17 题、每组两次生成的结果见 [`experiment/results/g_report.md`](experiment/results/g_report.md)；各阶段预期和实测见同目录的 `g_stage1_feedback.md` 至 `g_stage4_findings.md`。当前结果未显示反馈越细效果越好，研究问题改为分析不同反馈颗粒度的收益与成本。复现入口与结果索引见 [`experiment/SETUP.md`](experiment/SETUP.md)。本轮只测执行时间，未得到可靠的内存比较。


已有 EffiLearner 简化复现，在 4 个模型上跑通「100 题 × 3 轮自我优化」。另有 12 题 HumanEval 反馈粒度预实验；其中不少题目过于简单，不能据此判断增强反馈的效果。

| 模型 | 成功执行 | 时间下降 | 内存下降（可测口径） |
|---|---|---|---|
| deepseek-flash | 87 / 100 | ↓ 27.5% | ↓ 26.4% |
| deepseek-v4-pro | 80 / 100 | ↓ 23.9% | ↓ 24.3% |
| glm-4.6 | 82 / 100 | ↓ 18.3% | ↓ 17.8% |
| qwen3-coder-flash | 79 / 100 | ↓ 15.6% | ↓ 17.6% |

> 结果详情见 `results/repro/` 与 `docs/实验记录.md`；内存指标存在已知“假 100%”问题，见下文与优化计划。

## 目录结构

```
CodeBoost/
├── README.md               # 本文件：项目总览
├── requirements.txt        # 依赖（复现 + 上游源码）
├── LICENSE
├── src/                    # 原版 EffiLearner 源码（上游复现基线）
│   ├── EffiLearner.py / gpt_EffiLearner.py
│   ├── gpt_generation.py / open_llm_generation.py
│   └── code_efficiency_calculator.py / ...
├── prompts.py              # 原版提示词
├── scripts/run_code.sh     # 原版运行脚本
├── effilearner/            # 我们的简化复现 + 优化入口
│   ├── effi_repro.py
│   └── README.md
├── perfcodegen/            # PerfCodeGen 路线（待建设）
├── eval/                   # 效率评测脚本（待建设）
├── datasets/               # 数据集（不入库，本地放置）
├── results/repro/          # 复现结果 JSON（各模型 + comparison）
└── docs/
    ├── 分工与贡献说明.md
    ├── 优化计划.md         # 下一步优化任务与分工
    ├── 实验记录.md         # 实验配置与结果记录
    └── 上游README.md       # 原版仓库 README（留档）
```

### 原版源码 vs 简化复现

- `src/` + `prompts.py` + `scripts/`：EffiLearner **上游原版实现**（复现基线，保留以便对照与二次开发）。
- `effilearner/effi_repro.py`：我们重写的**轻量复现 + 优化入口**（纯 Python + psutil，跨平台，支持任意模型）。

## 快速开始

```bash
pip install -r requirements.txt
```

离线自检（不需要 Key）：

```bash
py effilearner/effi_repro.py --mock --num 3 --epochs 2
```

接入真实模型（PowerShell）：

```powershell
$env:EFFI_API_KEY="sk-..."
$env:EFFI_API_BASE="https://api.deepseek.com/v1"
$env:EFFI_MODEL="deepseek-flash"
py effilearner/effi_repro.py --num 100 --epochs 3
# 或用厂商预设一键切换：--provider deepseek --model deepseek-flash（--list-providers 查看全部）
```

> **数据集**：`datasets/dataset.json`（EffiBench，约 83MB）体积较大，未纳入版本库。
> 运行前请自备并放到 `datasets/dataset.json`，或用 `--dataset <路径>` 指定。
> 详细参数见 `effilearner/README.md`。

## 小组成员与分工

| 成员 | 角色 | 贡献率 |
|---|---|---|
| 李佳润（组长） | 技术统筹：复现架构与总体集成、LLM 工程化、优化统筹、汇报 PPT 主讲 | 22% |
| 成员 A | 复现·测量与实验 | 20% |
| 成员 B | 正确性优化（失败单测反馈 + plan/refine） | 20% |
| 成员 C | 性能优化与指标（最耗时单测、Best@k、%Opt/Speedup） | 19% |
| 成员 D | 数据工程（数据集/README/评测）+ 总结与 PPT | 19% |

完整职责与答辩安排见 [`docs/分工与贡献说明.md`](docs/分工与贡献说明.md)。

## 下一步

优化任务、负责人与依赖关系见 [`docs/优化计划.md`](docs/优化计划.md)。

