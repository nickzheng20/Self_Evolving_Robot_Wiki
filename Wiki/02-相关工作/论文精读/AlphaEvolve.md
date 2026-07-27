# AlphaEvolve

核心判断：AlphaEvolve 给本项目的最大启发是：代码自进化的中心不是 LLM，而是 evaluator。没有自动评估器，“自我进化”只是生成更多代码；有严格评估器，代码修改才可能变成可累积的发现。

## 元信息

- 标题：AlphaEvolve: A coding agent for scientific and algorithmic discovery
- 本地 PDF：`Source/Papers/alphaevolve-2506.13131.pdf`
- 链接：[[alphaevolve-2506.13131.pdf|本地 PDF]]
- 角色：[[自进化代码Agent与评估器]] 的核心论文；本项目 Robot CI/CD 和 evaluator 设计的上位参考。

## 一句话版

AlphaEvolve 是一个进化式 coding agent：人类给出问题、初始代码和评估函数，系统让 LLM 不断提出代码 diff，用 evaluator 打分，再把好结果放回程序数据库继续进化。

## 为什么重要

本项目叫“机器人自我进化”，最容易陷入一个误区：以为只要让 LLM 改代码，系统就在进化。AlphaEvolve 说明真正关键是可执行评估。

对应到机器人：

- AlphaEvolve 的 `evaluate()` -> robot skill evaluator。
- 程序数据库 -> skill version database。
- LLM diff -> skill patch proposal。
- 多指标评分 -> 成功率、安全违规、回归率、执行成本。
- 进化循环 -> 仿真和真实机器人受控部署。

## Section-by-section

### Abstract

摘要把 AlphaEvolve 定义为一个 autonomous pipeline：LLM 直接修改算法代码，系统用一个或多个 evaluator 给反馈，逐步改善程序。论文展示的结果包括数据中心调度、硬件电路简化、LLM 训练加速，以及数学和算法发现。

对本项目来说，这些任务和机器人不同，但范式相同：如果候选方案能被机器评估，LLM 就可以在这个闭环里做大量探索。

### Introduction

引言强调科学发现和工程优化往往需要长时间的试错、回溯、实验和验证。LLM 可以提出想法，但没有验证机制时会犯错。AlphaEvolve 把 LLM 的创造力放进进化搜索，让评估器负责筛选。

机器人 skill repair 也一样。LLM 可以提出“过滤深度空洞”“换抓取点”“增加 fallback”，但是否真的更好，必须由仿真、回归和安全测试判断。

### Task Specification

AlphaEvolve 要求人类提供：

- 初始程序。
- 哪些代码块允许被进化。
- 自动评估函数。
- 可选背景知识和配置。

这对机器人非常关键。coding agent 不应拥有整个机器人代码库的自由修改权，只应修改 skill 的允许区域，例如 perception post-processing、参数、fallback、precondition、postcondition 和测试。

### Prompt Sampling and Program Database

系统从数据库采样历史程序、得分和想法，构造丰富 prompt。这个数据库要平衡 exploration 和 exploitation：既利用高分方案，也保持多样性。

机器人版本也需要类似机制。一个抓杯子 skill 的 patch 不是孤立的，它应该能看到：

- 过去哪些 patch 修复了透明杯。
- 哪些 patch 破坏了普通杯。
- 哪些测试最容易暴露 regression。
- 哪些失败假设曾被否定。

### Creative Generation

AlphaEvolve 使用 LLM ensemble 产生代码 diff。强模型负责高质量跳跃，快模型负责高吞吐探索。

机器人项目可以借鉴：不是每个 patch 都要用最贵模型。低风险参数修复可以用便宜模型，高风险架构修改用强模型和人类审查。

### Evaluation

这是论文最重要的部分。每个新程序都必须运行 evaluator。AlphaEvolve 支持多指标、评估级联、并行评估和 LLM 辅助评分。

机器人 evaluator 应至少包括：

- 单元测试。
- 仿真成功率。
- 扰动鲁棒性。
- 回归测试。
- 安全违规次数。
- 小样本真实机器人验证。

如果没有这个层，机器人“自进化”很容易变成 LLM 自信地改坏代码。

### Results

论文展示了多个高价值结果，包括发现 4x4 复数矩阵乘法的新算法、改进多种数学构造、优化 Google 内部系统组件。重要的不是某个具体数学结果，而是证明了：当评估器足够清晰时，LLM 生成的代码可以被大规模、长期、自动地筛选和改进。

### Discussion

论文也承认主要限制：AlphaEvolve 适合有自动 evaluator 的问题。对需要湿实验、人工判断或难以自动评分的问题，它的优势会下降。

机器人恰好处在中间：仿真和日志可以自动评估一部分，但真实世界的安全、接触动力学和泛化仍然难以完全自动化。因此本项目要做的是分层 evaluator，而不是假设所有物理失败都能一次自动评分。

## 对本项目的启发

1. 先设计 evaluator，再谈自进化。
2. Skill patch 应以 diff/proposal 形式出现，而不是直接部署。
3. 每个 patch 要带分数、测试结果和回滚信息。
4. 多指标比单一成功率更重要，因为机器人有安全和 regression。
5. 自动评估的边界要诚实标注：仿真通过不等于真实世界通过。

## 我会追问的问题

- 机器人 skill 的 `evaluate()` 函数应该返回哪些指标？
- 如何构造低成本但能预测真实表现的仿真 evaluator？
- 是否可以像 AlphaEvolve 一样保存失败 patch，让后续 prompt 避免重复错误？
- 机器人领域哪些问题足够 machine-gradeable，适合先做？
