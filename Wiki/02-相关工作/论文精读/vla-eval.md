# vla-eval

核心判断：vla-eval 的核心贡献不是又增加一个机器人 benchmark，而是把 model inference、benchmark runtime、协议配置和结果 provenance 隔离成可复现的 evaluation harness；它非常适合成为本项目的外部 benchmark runner，但当前只验证 task success rate，尚不能直接承担补丁回归、安全门或 repair-agent 隔离。

## 元信息

- 标题：vla-eval: A Unified Evaluation Harness for Vision-Language-Action Models
- 作者：Suhwan Choi、Yunsung Lee、Yubeen Park、Chris Dongjoo Kim、Ranjay Krishna、Dieter Fox、Youngjae Yu
- 版本：arXiv v2，2026-04-17
- 页数：5
- 本地 PDF：[[vla-eval-2603.13966.pdf|本地 PDF]]
- 链接：[arXiv:2603.13966](https://arxiv.org/abs/2603.13966)；[代码仓库](https://github.com/allenai/vla-evaluation-harness)；[VLA Leaderboard](https://allenai.github.io/vla-evaluation-harness/leaderboard)
- 角色：[[Benchmark与实验平台]] 的候选外部运行器；为 [[Robot CI-CD与安全门]] 提供环境隔离、协议冻结和结果 provenance。

![[vla-eval-2603.13966.pdf#page=1]]

## 一句话版

vla-eval 用 WebSocket + msgpack 把 VLA model server 与 Docker 中的 benchmark client 解耦，使一个模型和一个 benchmark 各接入一次即可自动形成 \(N\times M\) 评测矩阵，并通过 episode sharding 与 batch inference 把 2,000 个 LIBERO episodes 从约 14 小时压到约 18 分钟。

## 为什么重要

机器人 code-repair 论文的一个高风险点是：一个 action convention、crop、proprioception source 或 quaternion 细节就可能制造几十个百分点的“收益”。vla-eval 实际复现时观察到，单个未记录参数最多造成 55 个百分点变化。

因此本项目需要把以下内容从 repair agent 的工作区剥离：

- benchmark 依赖与资产；
- seed、episode 数和 task split；
- observation preprocessing；
- action coordinate/convention adapter；
- success predicate；
- per-episode 结果和 harness 版本。

vla-eval 已提供其中大部分工程骨架，比自行拼接多个 benchmark 更可靠。

## Section-by-section

### Abstract 与 Introduction：从 \(O(NM)\) 接入降到 \(O(N+M)\)

不同 benchmark 依赖冲突严重：论文举例，LIBERO、ManiSkill2 和 CALVIN 分别依赖不同 Python 版本与 simulator stack。更麻烦的是论文经常不完整报告 seed、episode 数、preprocessing 和 action convention。

传统做法中，每个模型都要针对每个 benchmark 写专用 glue code，工程成本是 \(O(N\times M)\)。vla-eval 的设计是：

- model 只实现一个 `predict(obs, ctx)`；
- benchmark 只实现 `reset`、`step`、`make_obs`、`get_step_result` 四个方法；
- 两边通过稳定协议连接；
- 模型与 benchmark 依赖分别隔离。

这使接入成本近似变为 \(O(N+M)\)。

### Section II-A：client-server 与依赖隔离

model server 和 benchmark 通过 WebSocket 传递 msgpack binary message。消息包含：

- `observation`、`action`、`episode_start/end` 等类型；
- benchmark-specific payload；
- sequence number；
- timestamp。

模型侧继承 `PredictModelServer`，实现阻塞式 `predict(obs, ctx)`；框架处理 action chunking，并可通过 `max_batch_size` 开启 batch inference。模型依赖用 PEP 723 inline metadata 声明，`uv run` 自动创建独立环境。

benchmark 运行在固定 Docker image 中，镜像内包含 simulator、场景、纹理和 robot description。每次实验由 model server config 与 benchmark config 两份 YAML 驱动，结构化 JSON 结果记录 harness version、benchmark configuration 和逐 episode metric。

这个设计对本项目意味着：repair agent 可以修改受控 skill package，但不能进入 benchmark container，也不能访问 hidden config 和 success label。

### Section II-B：支持范围

论文发布时支持 14 个仿真 benchmark，包括：

- 已跨 codebase 复现验证：SimplerEnv、LIBERO、CALVIN；
- 已集成但未完整 cross-validation：RLBench、LIBERO-Pro、RoboCerebra、ManiSkill2、Kinetix、MIKASA-Robo、LIBERO-Mem、RoboMME、VLABench、RoboTwin 2.0、RoboCasa。

支持六组 model server：CogACT、OpenVLA、OpenVLA-OFT、\(\pi_0/\pi_0\)-FAST、GR00T N1 和 X-VLA。

“已集成”不等于“复现可信”。论文用 `C` 和 `I` 明确区分 cross-codebase reproduction verified 与仅 integrated，这种成熟度标记值得在本项目 benchmark registry 中复用。

### Section II-C：并行评测

框架同时做两类并行：

1. 把 episode 分片到 \(K\) 个 benchmark containers；
2. 把来自多个环境的 observation 在 model server 上 batch inference。

作者用 demand/supply 方法调参：环境吞吐为 \(\lambda(K)\)，模型 batch 吞吐为 \(\mu(B)\)，选择满足 \(\lambda(K)<0.8\mu(B^*)\) 的 operating point，给突发队列留出余量。

关键结果：

| Benchmark | 工作量 | 并行设置 | Sequential | Parallel | 加速 |
| --- | ---: | --- | ---: | ---: | ---: |
| LIBERO | 2,000 episodes | 50 shards，batch 16 | 约 14 h | 约 18 min | 47× |
| CALVIN | 1,000 sequences | 16 shards | 8.6 h | 约 33 min | 16× |
| SimplerEnv | 288 episodes、3 seeds | 16 shards | 1.7 h | 约 8.5 min | 12× |

LIBERO + CogACT-7B 的环境吞吐从 11.2 提高到 364.6 obs/s，模型 batch 吞吐从 165.2 提高到 468.2 obs/s。作者的判断是瓶颈最终在 simulator step rate，而不是 model inference。

### Section III：跨六个 codebase 的复现验证

作者用固定 seed 与版本化 Docker image，在 LIBERO、CALVIN、SimplerEnv 上复现六个公开 codebase：

| Codebase | LIBERO | CALVIN | SimplerEnv |
| --- | ---: | ---: | ---: |
| OpenVLA | 76.2%，较论文 -0.3 pp | — | — |
| \(\pi_0.5\) | 97.7%，+0.8 pp | — | — |
| OpenVLA-OFT | 96.7%，-0.4 pp | — | — |
| GR00T N1.6 | 94.9%，-2.1 pp | — | 59.7%，-8.0 pp |
| DB-CogACT | 94.7%，-0.2 pp | 4.02，-0.04 | 63.5%，-6.0 pp |
| X-VLA | 97.4%，-0.7 pp | 4.30，-0.13 | 94.8%，-1.0 pp |

LIBERO 协议使用 4 suites × 10 tasks × 50 episodes，共 2,000 episodes；CALVIN 是 ABC→D 的 1,000 个 chained sequences；SimplerEnv 是 4 个 WidowX tasks、每任务 24 episodes。

### Section III-B：真正有价值的是复现陷阱

论文列出的具体陷阱直接说明 evaluator isolation 为什么必要：

- X-VLA 在 LIBERO 中选错 proprioceptive state source，成功率从 97.8% 降到 42%，相差约 55 pp。
- 把 absolute action 与 delta action 混淆，虽然都是合法 7D vector，却会让位置累计发散，得分变为 0%。
- OpenVLA-OFT 的 quaternion-to-axis-angle 没做 antipodal normalization，角度范围为 \([0,2\pi]\)；若按常见做法把 \(w<0\) 翻转到 \([0,\pi]\)，LIBERO-Goal 从 97% 降到 83%，LIBERO-Long 从 95% 降到 56%。
- OpenVLA 评估时有论文未记录的 0.9 center crop，省略约损失 3 pp。
- GR00T 需要内部 simulator fork 才有的 end-effector pose proprioception；在官方 SimplerEnv 中缺失时得分从 30%–55% 降到 0%。

这些问题不是 policy intelligence 的变化，而是 evaluation adapter 的变化。若 repair agent 能编辑这层，它很容易“修复” benchmark 而不是修复机器人 skill。

### Section IV：leaderboard 与 protocol curation

论文还整理了一个包含 17 个 benchmark、657 条结果、509+ configurations 的 leaderboard。作者从引用相关 benchmark 的 1,704 篇论文中抽取结果，先由 agent 标准化，再由人类逐条审核，并保存 provenance 和 schema validation。

统计显示：

- 509+ models 中 81% 只评一个 benchmark；
- 13% 评两个；
- 约 6% 评三个以上；
- 只有 3 个模型，也就是约 0.6%，覆盖五个或更多 benchmark。

因此“在 LIBERO 一个设置上修好”不能自然升级为“通用机器人自进化”。

### Conclusion 与 Limitations

作者承认：

- 真正跨 codebase 验证的只有 3 个 simulation benchmarks；
- 尚未覆盖真实机器人 transfer；
- leaderboard 中的论文结果没有独立复现；
- 当前 metric 主要是 task success rate，不支持 subtask progress、efficiency 和 safety。

对本项目而言，最后一点是采用时最重要的缺口：vla-eval 是执行与 provenance 底座，不是完整 Robot CI。

## 具体机器人例子

假设我们为 RPent 的 `move_pose` 修复了一个 quaternion convention bug，并观察到 LIBERO-Long 成功率从 56% 升到 95%。

存在两种完全不同的解释：

1. skill implementation 原来真的把目标姿态转错，补丁恢复了正确执行；
2. evaluation adapter 原来与 checkpoint 的训练 convention 不一致，补丁只是对齐 benchmark glue。

vla-eval-style 隔离要求：

- action adapter 固定在只读 model/benchmark boundary；
- 可编辑 `move_pose` 的输入输出 contract 被明确记录；
- hidden evaluator 保存相同 seed 和 protocol；
- patch 只允许修改 allowlist 内的 skill；
- per-episode JSON 同时记录 code version、harness version、model config 和 benchmark config。

这样才能说明变的是 skill，而不是计分器。

## 对本项目的启发

### 1. 用它做外部 benchmark runner，不把它等同于 RPent

RPent/Harness VLA 是 planner、memory、VLA primitive 和执行 trace 的研究 codebase；vla-eval 更适合作为外部、版本固定的 benchmark runner。两者之间应通过稳定 RPC/schema 连接，而不是把 evaluator 源码复制进 repair workspace。

### 2. 建立三层版本指纹

每个结果至少记录：

- `system_sha`：RPent 与候选 skill patch；
- `model_digest`：VLA/planner checkpoint 与 inference config；
- `harness_digest`：vla-eval image、assets、adapter、task config 与 seed manifest。

否则不同 patch 的结果不可比。

### 3. 为 code repair 增加 vla-eval 尚缺的 metric channel

除 task success 外，本项目需扩展只读输出：

- failure family 与 subtask progress；
- collision、workspace、速度和 force safety events；
- retry、token、environment interaction、wall-clock 与成本；
- patch acceptance/rejection；
- unaffected-task regression；
- severe safety violation。

### 4. 并行化不能改变 adaptation budget

50 个 shards 能缩短 wall-clock，但不能把 repair agent 可见的 rollout 数从 5 条偷偷放大到 2,000 条。应分开记录：

- adaptation/diagnostic interactions；
- hidden gate executions；
- final-blind evaluation episodes。

并行只改变运行时间，不改变每组的信息预算。

### 5. 先在 cross-validated benchmark 上做 primary claim

初版应优先用 vla-eval 已标记 `C` 的 LIBERO 路径，并独立验证 LIBERO-Pro adapter；对仅 `I` 的 benchmark，只能作为扩展性结果，不能把集成完成等同于复现完成。

## 局限与批判

- **短论文、验证面有限**：14 个 benchmark 的宣传范围大于真正 cross-codebase 验证的 3 个。
- **只验证 simulation**：没有证明 WebSocket、Docker 和 batch scheduling 对真实机器人 timing/control loop 的影响。
- **metric 过窄**：success rate 无法评估修复的安全性、效率和局部回归。
- **leaderboard 不是独立复现**：大部分条目来自论文报告。
- **parallel execution 可能改变非确定性**：若 simulator、GPU batch 或异步调度影响随机数和 timing，需要额外 determinism audit。
- **容器不是完整安全边界**：还需隐藏测试、只读挂载、网络策略和输出 schema 验证。
- **adapter 仍是可信代码**：统一协议减少错误，但不会自动证明 observation/action mapping 正确。

## 我会追问的问题

- 每个 benchmark adapter 是否有 reference-trajectory differential tests，而不只是最终成功率接近？
- 同一 config 重跑时，episode-level 结果在不同 shard 数和 batch size 下是否一致？
- 如何把 safety event、subtask progress 和 repair provenance 加入稳定 schema？
- 能否把 hidden task config 和 success predicate 放入不可被 repair container 读取的独立服务？
- vla-eval 与 RPent 的 action chunk、early return 和 interactive planner loop 如何对齐？
- 对 LIBERO-Pro 的 `I` 状态，需要哪些 cross-codebase checks 才能升级为 `C`？
