# VLA路线

核心判断：VLA 是当前通用机器人最强主线之一，但它主要通过数据和参数更新积累能力；本项目的机会在于补上“失败可解释、修复可验证、skill 可版本化”的工程闭环。

## 代表工作

[RT-1](https://arxiv.org/abs/2212.06817) 展示了用大规模真实机器人数据训练 transformer 控制策略的可能性。

[[rt-2-2307.15818.pdf|RT-2]] 把视觉语言模型的 web knowledge 迁移到机器人控制，把 action 当作 token 进行生成，是 VLA 路线的重要节点。

[Open X-Embodiment](https://arxiv.org/abs/2310.08864) 汇集多机器人、多任务、多机构数据，推动跨 embodiment 的机器人学习。

[Octo](https://arxiv.org/abs/2405.12213) 和 [[openvla-2406.09246.pdf|OpenVLA]] 代表开源通用机器人 policy 的路线。OpenVLA 是 7B 参数 VLA，强调可微调、可部署和开放模型。

[pi0](https://arxiv.org/abs/2410.24164) 用 flow matching 生成连续动作，面向更复杂和灵巧的机器人控制。[pi0.5](https://arxiv.org/abs/2504.16054) 进一步强调开放家庭环境泛化。

[GR00T N1](https://arxiv.org/abs/2503.14734) 面向人形机器人，采用 VLM reasoning module 和 diffusion transformer action module 的双系统结构。

[Gemini Robotics](https://arxiv.org/abs/2503.20020) 与 [Gemini Robotics 1.5](https://arxiv.org/abs/2510.03342) 强调 embodied reasoning、thinking、motion transfer 和多机器人泛化。

[Embodied-R1.5](https://arxiv.org/abs/2606.11324) 是 2026 年的新近工作，把 embodied cognition、task planning、correction 和 pointing 等能力统一到 embodied foundation model 中，说明“单模型内化具身能力”的路线仍在快速推进。

## 已精读页面

- [[RT-2]]：动作 token 化和 web-scale VLM 知识迁移到控制。
- [[OpenVLA]]：开源 7B VLA、Open X-Embodiment 训练和高效微调。

## VLA 的强项

VLA 的强项是：

- 可以从大量多机器人数据中学习人类难以手写的动作细节。
- 对视觉、语言和动作有统一表示。
- 对开放词汇任务和多对象场景更自然。
- 对灵巧操作、接触丰富动作、复杂运动模式有潜力。

对于擦桌子、折衣服、双臂协调、柔性物体操作等任务，纯代码规则很难覆盖细节，VLA 或 diffusion policy 可能更合适。

## VLA 的弱点

VLA 失败后常见问题是：

- 很难知道失败来自感知、语言理解、动作生成还是数据分布外。
- 修复常常需要收集数据、微调模型或重新训练。
- 很难给某个具体失败生成局部、可回滚的 patch。
- 安全约束和 precondition/postcondition 不一定显式。
- 旧任务回归不一定和每次修复绑定。

这正是 [[执行反馈自调试]] 和 [[Robot CI-CD与安全门]] 想补的地方。

## 和本项目的关系

本项目不应该把 VLA 当成必须替代的对象。更合理的架构是：

> VLA/learned policy 可以是某些 skill 的实现；coding agent 负责围绕 skill 增加约束、调用逻辑、fallback、测试、监控和版本化。

例如，`wipe_table` 可以由 diffusion policy 执行，但 skill 外层仍然可以有：

- precondition：桌面区域已分割，附近无易碎物。
- postcondition：污染区域覆盖率下降。
- safety：力和速度不超过阈值。
- fallback：若视觉置信度低，先换视角。
- tests：不同桌面材质、不同污渍分布、不同障碍物。

所以本项目的竞争点不是“代码比模型更聪明”，而是“可维护系统比单次模型输出更可靠”。
