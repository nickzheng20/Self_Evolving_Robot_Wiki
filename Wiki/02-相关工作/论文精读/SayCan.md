# SayCan

核心判断：SayCan 是本项目“任务大脑调用已有 skill”的基础范式：LLM 负责判断什么动作有用，affordance/value function 负责判断什么动作当前能做。

## 元信息

- 标题：Do As I Can, Not As I Say: Grounding Language in Robotic Affordances
- 本地 PDF：`Source/Papers/saycan-2204.01691.pdf`
- 链接：[[saycan-2204.01691.pdf|本地 PDF]]
- 角色：[[LLM规划与代码生成]] 中“LLM planner + skill grounding”的核心论文。

## 一句话版

SayCan 把 LLM 对“哪个 skill 有助于完成任务”的概率，和 value function 对“哪个 skill 在当前状态可执行”的概率相乘，从而选择既有用又能做的下一步。

## 为什么重要

本项目的顶层不是让 LLM 直接输出电机动作，而是让任务大脑在 skill registry 中选择 skill。SayCan 给这个设计提供了清晰概率解释：

```text
有用性：p(skill_description | instruction)
可行性：p(skill_success | state, skill_description)
选择：argmax 有用性 * 可行性
```

这恰好回答了一个关键问题：LLM 怎么知道机器人此刻能不能执行某个 skill？答案不是靠语言模型猜，而是靠 embodiment-specific affordance。

## Section-by-section

### Abstract

摘要指出 LLM 有丰富语义知识，但缺少真实世界 grounding。SayCan 用预训练 skill 和 value function 约束 LLM，让它选择既符合任务语义又适合当前环境的低层技能。

论文在真实厨房移动操作机器人上评估长时序自然语言任务，展示 grounding 能显著提升执行成功。

### Introduction

引言用“清理溢出的饮料”说明问题：LLM 可能给出合理叙述，但机器人不一定能执行。机器人必须知道自己的身体、技能集合和当前环境。

这对本项目很关键：任务大脑不应该自由想象技能，而必须读取 skill registry，并受 precondition/affordance 约束。

### Method

SayCan 假设机器人有一组 skill，每个 skill 有自然语言描述和 value function。LLM 给出某个 skill 对当前高层指令有多有用，value function 给出这个 skill 从当前状态成功的概率。

算法迭代进行：

1. 给定高层指令和已完成步骤。
2. LLM 给每个候选 skill 打“任务相关性”分。
3. value function 给每个 skill 打“当前可执行性”分。
4. 乘起来选最大者。
5. 执行 skill，更新状态，直到输出 done。

### 具体例子

用户说：“我把可乐洒在桌上了，你能帮我吗？”

LLM 可能认为“拿海绵”“去桌子”“擦桌子”“把海绵放回去”都语义合理。但如果海绵不在视野里，affordance 会降低“pick up sponge”的分数，系统可能先选择“find sponge”。

这说明 LLM 的语义规划必须被现实状态纠正。

### Robotic System

SayCan 使用移动操作机器人，在办公室厨房中评估。skill 包括导航、抓取、放置、开合抽屉等。论文提到技能集合可扩展：当新 skill 变可靠时，可以把它作为新的候选项加入 LLM scoring。

这给本项目一个自然接口：coding agent 修复或新增 skill 后，任务大脑只需要看到更新后的 registry 和 affordance。

### Experiments

论文评估 101 个真实机器人任务，覆盖自然语言、同义词、结构化语言、embodiment、crowd-sourced query 和 long-horizon。论文报告 grounding LLM in affordances 接近把非 grounding baseline 的表现翻倍。

附录中还指出 drawer manipulation 的规划成功率可以很高，但执行成功率明显低，主要失败来自 manipulation 本身。这对本项目尤其重要：高层计划正确不代表 skill 实现可靠。

### Failure Cases

论文失败案例分为 LLM 错误和 affordance 错误。比如 LLM 过早结束长任务，或者 affordance 没识别出某物可抓。

这正好说明本项目为什么需要 [[执行反馈自调试]]：当失败来自 affordance 或 skill 实现，而不是高层计划时，应该修 skill，不是只重问 LLM。

## 对本项目的启发

1. 任务大脑应只在 skill registry 内选择动作。
2. 每个 skill 都应有可行性估计，而不仅是文字描述。
3. 修复 skill 后，要更新 affordance 或 precondition，否则 planner 仍会误用。
4. SayCan 可作为高层 planner baseline；本项目的创新在 skill repair loop。

## 我会追问的问题

- 本项目的 affordance 应由 learned value function、规则 precondition、仿真 rollouts 还是多者融合？
- coding agent 修复 skill 后，如何同步更新 value function 或 success predictor？
- 如果 LLM 计划正确但 affordance 错误，failure attribution 如何判断？
