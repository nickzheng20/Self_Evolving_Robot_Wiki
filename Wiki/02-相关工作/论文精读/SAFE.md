# SAFE

核心判断：SAFE 证明了冻结 VLA 的最后层表示里存在可跨任务迁移的失败信号，并能用一个很小的 MLP/LSTM 与 conformal threshold 把它变成低延迟 failure trigger；但它检测的是“当前 rollout 很可能失败”，既不提供可靠的根因归因，也不能充当代码补丁的最终正确性 evaluator，因此在本项目中应被隔离为只读监控器。

## 元信息

- 标题：SAFE: Multitask Failure Detection for Vision-Language-Action Models
- 作者：Qiao Gu、Yuanliang Ju、Shengxiang Sun、Igor Gilitschenski、Haruki Nishimura、Masha Itkina、Florian Shkurti
- 版本：arXiv v2，2025-10-30；NeurIPS 2025
- 页数：29
- 本地 PDF：[[safe-2506.09937.pdf|本地 PDF]]
- 链接：[arXiv:2506.09937](https://arxiv.org/abs/2506.09937)；[项目主页](https://vla-safe.github.io/)
- 角色：[[VLA路线]] 中的通用 failure trigger；影响 [[执行反馈自调试]] 与 [[问题定义与实验假设]] 中“何时启动修复”的定义，但不替代 [[Robot CI-CD与安全门]] 的 hidden evaluator。

![[safe-2506.09937.pdf#page=1]]

## 一句话版

SAFE 从 VLA 最后一层抽取每个决策时刻的 latent feature，用一个轻量 MLP 或 LSTM 产生随时间变化的 failure score，再用成功 rollout 校准 functional conformal prediction band；分数越过 band 时触发停止、接管或恢复，并在未参与训练的任务上评估跨任务检测能力。

## 为什么重要

本项目过去容易把“检测到任务失败”写成一个免费 oracle。SAFE 说明这个模块本身就是独立研究问题：

1. 最终 episode reward 太晚，机器人可能已经撞击、掉落或进入不可恢复状态；repair loop 需要及时 trigger。
2. 单任务异常检测不适合 generalist VLA，因为未见任务不等于失败。SAFE 显式区分“新任务”与“失败行为”。
3. failure trigger、root-cause attribution 和 patch evaluator 是三个不同模块。SAFE 只直接解决第一个。
4. learned monitor 的阈值会随任务分布漂移，因此不能与 agent 一起自由修改，也不能作为唯一发布门。

## Section-by-section

### Abstract 与 Introduction：问题不是“不确定性”，而是跨任务失败检测

论文关注的是 multitask failure detection：检测器只在一组 seen tasks 的成功和失败 rollout 上训练，在不收集新 rollout、不微调检测器的条件下，直接应用到 unseen tasks。

作者给出的部署动机是：通用 VLA 在熟悉任务上常有 80%–90% 成功率，但直接迁移到未见真实任务时常降到 30%–60%。此时逐任务收集数据、训练专用 detector 不再现实；而实时机器人也很难为每一步采样几十或几百个 action 估计不确定性。

需要特别注意，论文定义的“失败”是 trajectory-level 标签。训练数据只要求知道整条 rollout 最终成功或失败，不要求标注失败首次发生的精确时刻。模型却在每个时刻输出 score，因此它学到的是弱监督的在线预警信号，而不是逐帧因果标签。

### Section 2：与 OOD、uncertainty 和单任务 detector 的区别

论文区分三类近邻：

- OOD detection 把成功执行当作 in-distribution，偏离它就视为失败。对 generalist VLA，这会误报合法的新任务。
- uncertainty quantification 从 token probability、entropy 或多次 action sample 的分歧估计不确定性。它不直接学习失败，而且多样本方法带来较大控制延迟。
- supervised failure detection 同时看成功和失败 rollout。SAFE 属于这一类，但用一个 detector 覆盖多个任务，并要求 zero-shot 泛化到 held-out task。

这个区别对本项目很关键：修复系统不能把“策略不确定”“观测分布外”和“任务已经失败”当成同一个事件。它们应进入不同的路由和风险等级。

### Section 3：形式化定义

在时间 \(t\)，VLA 接收 RGB、语言指令和机器人状态 \(o_t\)，输出长度为 \(H\) 的 action chunk \(A_t\)，实际执行其中 \(H'\) 步后重新规划。记 VLA 内部表示为 \(e_t\)，整条 rollout 为：

\[
\tau_i=\{(o_t,e_t,W_t,A_t)\}_{t=0,H',\ldots,nH'}
\]

每条 rollout 有一个最终标签 \(y_i\)：失败为 1，成功为 0。检测器读取历史特征 \(e_{0:t}\)，输出 failure score \(s_t\)。若 \(s_t>\delta_t\)，系统触发 failure flag，并可停止、回退或请求人类接管。

实验把任务分成三部分：

- `D_train`：seen tasks 的训练 rollout；
- `D_eval-seen`：seen tasks 的验证与超参数选择；
- `D_eval-unseen`：完全 held-out tasks，用来检验跨任务泛化。

### Section 4.1：VLA latent space 中的 “failure zone”

作者先用 \(\pi_0\)-FAST 在 LIBERO-10 上的最后层特征做 t-SNE。成功 rollout 的表示避开一个区域；掉落 moka pot 后，失败 rollout 的表示逐渐进入相似的 “failure zone”。按 task ID 着色后，不同任务的失败仍聚在相似区域。

这个可视化只提供机制线索，不能证明存在普适、线性可分的 failure manifold。附录也显示，真实 Franka 上任务更异质时，成功/失败在 t-SNE 中并不清晰分离；但轻量 probe 仍能提取部分信号。

### Section 4.2：SAFE-MLP 与 SAFE-LSTM

SAFE 从 VLA 最后一层、解码 token logits 或 velocity field 之前提取 feature，并按模型结构对 token、horizon 或 diffusion-step 维度做 `first`、`last`、`mean` 或 `first&last` 聚合。

作者比较两个很小的 probe：

- `SAFE-MLP`：两层 MLP，hidden dimension 256。它逐时刻映射特征，并累积 sigmoid 输出；用 L1 目标压低成功 rollout 的累计分数、抬高失败 rollout 的累计分数。
- `SAFE-LSTM`：一层 LSTM，hidden dimension 256，再接线性层与 sigmoid；对每个时刻用 trajectory label 做 binary cross-entropy。

两者都按类别频率对 loss 加权，并用 L2 regularization 减少过拟合。模型只训练 probe，不修改 VLA。附录报告每个模型训练 1000 epochs、batch size 512，在单张 A100 40GB 上通常不到一分钟。

### Section 4.3：用 functional conformal prediction 选时间阈值

SAFE 不用一个固定阈值，而是在成功 calibration rollouts 上构造随时间变化的上界：

\[
\delta_t=\mu_t+h_t
\]

在 exchangeability 假设成立时，新成功 rollout 的整条 score curve 以至少 \(1-\alpha\) 的概率不越过 band。因此，理论上“一条成功 rollout 的任意时刻被误报”概率不超过 \(\alpha\)。

但这里有一个重要边界：band 在 seen tasks 上校准、在 unseen tasks 上使用时，exchangeability 往往不成立。论文附录明确报告，部分 benchmark 的 true-negative rate 低于理论 \(1-\alpha\) 下界。换言之，CP 提供的是条件性校准工具，而不是任意新任务上的绝对安全保证。

### Section 5：benchmark、baselines 与协议

实验覆盖三种 VLA 架构和四类环境：

| 设置 | Policy | 任务与数据 |
| --- | --- | --- |
| LIBERO-10 | OpenVLA、\(\pi_0\)、\(\pi_0\)-FAST | 10 个任务，每任务 50 rollout；随机留 3 个 unseen task |
| SimplerEnv | 复现版 \(\pi_0^\*\) | Google Robot 与 WidowX 各 4 个任务、每任务 100 rollout |
| 真实 Franka | \(\pi_0\)-FAST-DROID | 13 个任务，每任务 30 成功 + 30 失败，共 780 条 |
| 真实 WidowX | OpenVLA | 8 个任务，共 532 条，244 成功、288 失败 |

LIBERO 成功 episode 会提前结束、失败 episode 常跑满时间。若直接使用最终时刻，检测器只学“计时”也可能表现很好。作者因此对同一任务的成功与失败统一截到该任务的最短 rollout 长度再计算 ROC-AUC。这是值得直接复用的 anti-shortcut 设计。

baselines 包括：

- token maximum/average probability 与 entropy；
- 10 次 action sampling 的 total variance、component variance 与 cluster entropy；
- Mahalanobis、k-NN、PCA-KMeans 等 embedding distance；
- RND、LogpZO；
- action-chunk consistency 的 STAC 与单样本 STAC-Single。

主指标是 rollout-level ROC-AUC；设定 CP 阈值后，还报告 TPR、FPR/TNR、balanced accuracy 和 normalized detection time。

### Section 6：关键结果

在四个仿真设置上，SAFE 对 unseen tasks 的平均 ROC-AUC 最好：

| 方法 | Seen 平均 ROC-AUC | Unseen 平均 ROC-AUC |
| --- | ---: | ---: |
| 最强 embedding-distance 单项的量级 | 约 81 | 约 74 |
| SAFE-LSTM | 82.26 | 77.04 |
| SAFE-MLP | 81.43 | 78.00 |

具体到不同模型，最优方法并不总是 SAFE：例如 \(\pi_0\)-FAST + LIBERO 的 unseen split 上 STAC 为 85.31，\(\pi_0\) + LIBERO 上 gripper variance 为 75.19；但这些方法需要多次 action sample。SAFE 的优势是跨设置更稳定且只需单次 policy inference。

真实机器人结果更能显示边界：

| 设置 | SAFE-LSTM unseen | SAFE-MLP unseen | 最强结果 |
| --- | ---: | ---: | ---: |
| \(\pi_0\)-FAST + Franka | 58.70 | 64.16 | SAFE-MLP 64.16 |
| OpenVLA + WidowX | 71.80 | 88.42 | SAFE-MLP 88.42 |

Franka 的 64.16 ROC-AUC 远非可靠 oracle。任务越异质，统一 failure representation 越不稳定。

效率方面，SAFE-LSTM 约 230 万参数、每步增加 0.73 ms，论文报告小于 VLA inference 的 1%；相较之下，在 RTX 3090 上生成 10 个 action sample 使 \(\pi_0\) 慢 152%、使 \(\pi_0\)-FAST 慢 221%。

定性 failure modes 包括：

- 插入偏差后 action 开始振荡；
- missed grasp 后 policy 仍继续执行 place；
- 夹取物滑落；
- policy 输出零 action、执行冻结；
- 多次抓取不成功后接近 timeout。

附录中，SAFE-MLP 在 \(\pi_0\)-FAST 上甚至能在第一步后预测约 40% 的后续失败；不过对只在最后因 timeout 失败的 rollout 很难提前判断。

### Conclusion、Limitations 与 Appendix

论文明确限制在 manipulation，尚未证明跨 embodiment、sim-to-real 或 action-less video 的泛化。它只使用最后层 feature，没有研究多层信息融合。

附录还暴露两个更关键的问题：

1. SAFE 需要先部署目标 policy，并收集数百条成功/失败 rollout；它不是 cold-start monitor。
2. 未见任务破坏 CP 的 i.i.d./exchangeability 假设，因此阈值保证可能失效。
3. 失败首次时刻的人工标签只用于分析，不用于训练，而且对于“卡住多久算失败”“重复抓取几次后应接管”本身具有主观性。
4. detector 的表现随训练任务数量增加而改善；OpenVLA + LIBERO 上，SAFE-MLP unseen ROC-AUC 从 1 个训练任务的 63.76 提高到 7 个任务的 73.47。

## 具体机器人例子

设 Harness VLA 接到“把 alphabet soup 和 tomato sauce 放入篮子”。冻结 VLA 第一次抓取没有闭合在物体上，却按原 action chunk 继续向篮子移动。

SAFE 可以在这个过程中读取 `VLA_ACT` 每轮的最后层特征：

1. grasp 前表示仍处于正常区域；
2. 空抓后观测与预期接触状态矛盾，累计 failure score 上升；
3. score 越过 CP band，trusted monitor 触发 `STOP_AND_SNAPSHOT`；
4. 系统保存 RGB-D、proprioception、action chunk、primitive 前后状态与 SAFE score；
5. 后续 attribution 再判断这是一次可重试的 grasp、plan/memory failure、skill-local monitor 缺陷，还是 policy-limited failure。

SAFE 不应直接输出“修改 `pi0_pick` 的 gripper threshold”。它只提供及时触发信号，根因和最小充分干预仍需独立证据。

## 对本项目的启发

### 1. 将 failure trigger 与 repair quality 分开评估

第一篇 code-repair 实验应先用预注册 failure case 或 ground-truth trigger 检验 conditional repair quality；否则“没修好”可能只是 detector 没触发。SAFE-style detector 可作为第二条实验轴，单独报告 detection ROC-AUC、false alarm 与 latency。

### 2. SAFE 应放在可信内核而非可编辑 workspace

在 [[问题定义与实验假设]] 的边界中，SAFE 的模型权重、feature tap、阈值、calibration set 和日志应属于只读 \(K\)。repair agent 可以读取其 score 和解释性 trace，但不能修改 detector、阈值或测试标签，否则很容易通过压低 failure score 实现 reward hacking。

### 3. 不把 failure score 当作最终 patch evaluator

候选补丁即使让 SAFE 不再报警，也可能只是把行为推入 detector 未见分布。发布仍必须经过：

- task success predicate；
- unaffected-task regression；
- 独立安全 critic；
- hidden held-out seed/task；
- 严重安全违规零容忍。

### 4. 增加 monitor shift 与 white-box 边界

SAFE 依赖目标 VLA 内部表示，只适合 white-box policy。更换 VLA checkpoint、量化方式、action head 或 feature layer 后，应视作 detector 版本变化并重新校准。黑箱 VLA 则需要 observation/action trace detector 或外部 VLM critic。

### 5. 借用它的 anti-shortcut 评测

本项目的 failure-type classifier 和 repair evaluator 也必须消除 rollout length、任务 ID、mutant 名称和最终 reward 泄漏。尤其是成功提前终止的环境，应统一可比较窗口或明确 censoring protocol。

## 局限与批判

- **failure label 不是 causal label**：trajectory-level 失败监督可能把失败后的症状学成信号，而不是能提前干预的原因。
- **跨任务不等于跨 embodiment**：同一 VLA、相似 benchmark 内 held-out task 的成功不能外推到新机器人。
- **CP 保证容易被过度解读**：unseen-task shift 下理论覆盖率并不自动成立。
- **真实 Franka 的性能仍有限**：64.16 unseen ROC-AUC 不足以独立承担安全停机。
- **白盒耦合**：需要读取内部 feature，不适用于闭源服务或任意 backend。
- **训练成本被运行时效率掩盖**：probe 推理很便宜，但前置 rollout 收集和失败标签并不免费。
- **检测与恢复未闭环验证**：论文评估 detector，不评估触发后 stop/backtrack/replan 是否真的提高任务成功率或降低伤害。

## 我会追问的问题

- 按 failure family 而不是按 task 随机划分后，unseen ROC-AUC 还剩多少？
- calibration set 与新任务不 exchangeable 时，能否在线检测 coverage failure 并退回更保守的规则 critic？
- SAFE 报警后的最佳动作是停止、局部 retry、调用 planner，还是进入代码修复？如何学习 routing policy？
- 对“任务必然失败但尚未出现症状”的 rollout，SAFE 最早能提前多少物理时间，而不只是 normalized timestep？
- 若代码补丁改变了 VLA 的调用频率、staging 或观察分布，原 SAFE calibration 是否仍有效？
- 能否用独立的 motion/safety critic 与 SAFE 做 two-key trigger，降低单一 learned detector 的误报和漏报？
