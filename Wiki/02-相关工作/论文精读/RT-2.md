# RT-2

核心判断：RT-2 是 VLA 路线的代表：把机器人动作当成语言 token，让 web-scale VLM 知识迁移到机器人控制；它证明端到端 VLA 很强，也暴露了“物理技能仍受机器人数据限制”的边界。

## 元信息

- 标题：RT-2: Vision-Language-Action Models Transfer Web Knowledge to Robotic Control
- 本地 PDF：`Source/Papers/rt-2-2307.15818.pdf`
- 链接：[[rt-2-2307.15818.pdf|本地 PDF]]
- 角色：[[VLA路线]] 的核心论文；本项目需要正面对比的主流方向。

## 一句话版

RT-2 把机器人动作离散成 text tokens，并把 robot trajectory data 和 web-scale vision-language data 一起训练，让 VLM 直接变成能输出动作的 VLA。

## 为什么重要

本项目并不是要否认 VLA。RT-2 说明 VLA 的优势很真实：web-scale 视觉语言知识能提升机器人对新物体、新语义和抽象指令的泛化。

但 RT-2 也给了本项目空间：它的物理动作能力仍受 robot data 限制。如果失败是坐标转换、planner fallback、深度异常、postcondition 错误，代码修复可能比继续扩大 VLA 更有效。

## Section-by-section

### Abstract

摘要提出 VLA 概念：把视觉语言模型直接 fine-tune 成机器人控制策略。关键技巧是把机器人动作表示为 text token，和自然语言输出放在同一个 token 空间里训练。

RT-2 在约 6000 次真实机器人评估中展示更强泛化，包括新物体、未见指令、基本语义推理和 chain-of-thought 风格的多阶段推理。

### Introduction

引言的问题是：机器人数据远少于互联网图文数据，如何让机器人继承 web-scale 预训练中的语义能力？RT-2 的答案是：不要把 VLM 只当高层 planner，而是把动作也塞进输出 token，让 VLM 直接学闭环控制。

### Vision-Language-Action Models

方法上，RT-2 使用 PaLI-X 和 PaLM-E 等 VLM 作为基础，把 6-DoF 末端动作、旋转和 gripper 命令离散成 tokens。训练时模型既看 web VQA/caption 等任务，也看机器人轨迹。

有一个关键训练技巧：co-fine-tuning。论文发现把 robot data 和原始 web data 一起训练，比只在机器人数据上 fine-tune 更有利于保留语义概念。

### 具体例子

如果任务是“把适合当锤子的东西拿起来”，机器人训练数据可能没有“锤子”这个任务。但 VLM 在 web data 中知道石头可以临时充当锤子。RT-2 可以把这种语义知识迁移到 pick skill 上，选择石头。

注意，这不是学会了新的抓取运动，而是用已有抓取能力选择了新的语义目标。

### Experiments

RT-2 在 seen tasks 上和 RT-1 类似，但在 unseen objects、backgrounds、environments 和 emergent semantic tasks 上显著更强。论文还显示模型大小和 co-fine-tuning 对泛化很重要。

这说明 VLA 的主要价值在语义和视觉泛化，而不是凭空创造新的物理技能。

### Limitations

论文明确指出：web-scale pretraining 提升语义和视觉概念泛化，但不会让机器人获得 robot data 中没有的 physical motions。模型能用已学动作做新组合，但底层动作分布仍受训练数据限制。

这句话对本项目非常关键。它支持混合路线：VLA 可以做语义泛化，代码 skill layer 负责可解释的控制、测试和修复。

## 对本项目的启发

1. VLA 是强 baseline，不应被草率否定。
2. Web knowledge 能提升“选什么、为什么”的能力，但低层动作仍要靠机器人经验。
3. coding agent 的优势应放在 failure diagnosis、skill patch、tests 和 deployment，而不是和 VLA 比谁更会语义理解。
4. VLA 可以作为某些 skill 的实现，被包进 registry 和 safety gate。

## 我会追问的问题

- 当 RT-2 抓取失败时，系统能否解释失败来自语义选择还是动作执行？
- VLA 输出 token 失败后，能否生成局部 patch，还是只能重新训练？
- 是否可以让 coding agent 给 VLA skill 外层加 precondition、fallback 和 success detector？
