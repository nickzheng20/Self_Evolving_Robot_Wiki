# VoxPoser

核心判断：VoxPoser 是“LLM 写代码 + 感知 API + 3D 几何中间表示 + model-based planner”的强参考；它说明 LLM 不必直接输出动作，可以输出可解释的 3D value map，让传统规划器执行。

## 元信息

- 标题：VoxPoser: Composable 3D Value Maps for Robotic Manipulation with Language Models
- 本地 PDF：`Source/Papers/voxposer-2307.05973.pdf`
- 链接：[[voxposer-2307.05973.pdf|本地 PDF]]
- 角色：[[LLM规划与代码生成]] 中几何 grounding 和 model-based execution 的关键论文。

## 一句话版

VoxPoser 让 LLM 生成 Python 代码调用 VLM/perception API，把语言中的 affordance 和 constraint 转成 3D voxel value maps，再交给 motion planner 合成闭环轨迹。

## 为什么重要

用户原始想法里强调“杯柄像素、深度图、三维位置、传统控制代码”。VoxPoser 几乎就是这个方向的成熟版本：LLM 负责写空间约束和目标，VLM 负责 grounding，planner 负责轨迹。

它支持本项目的一个核心判断：机器人 skill 的可解释中间层可以是代码和几何结构，而不是端到端动作 token。

## Section-by-section

### Abstract

摘要说现有 LLM 机器人方法常依赖预定义 motion primitive，限制了细粒度物理交互。VoxPoser 想让 LLM 利用代码能力和 VLM 交互，生成 3D value map，从而在开放指令和开放物体集合上合成轨迹。

它还展示了在线经验可以帮助学习接触丰富场景的 dynamics model。

### Introduction

引言指出 LLM 不适合直接输出高频、高维控制信号，但很擅长从语言中推断 affordance 和 constraint。

例如“打开上方抽屉，注意不要碰花瓶”：

- 抽屉把手区域是高价值吸引区域。
- 花瓶周围是低价值避让区域。
- 末端执行器姿态需要对准把手。
- motion planner 在这些 value map 上优化轨迹。

这个例子对本项目很重要：coding agent 修改 skill 时，不一定只是改规则；也可以改 value map 的生成逻辑。

### Method

VoxPoser 的核心是把语言指令转成若干 3D maps：

- affordance map：哪里值得去。
- avoidance map：哪里要避开。
- rotation map：末端执行器姿态。
- gripper map：开合。
- velocity map：速度偏好。

LLM 生成代码，调用 `detect()` 等 API 找物体或部件，再用 NumPy 操作 voxel grid。planner 使用这些 map 生成 6-DoF end-effector waypoint，并以 MPC 方式反复重规划。

### 具体例子

任务：“把面包从烤面包机里拿出来。”

VoxPoser 可能先识别 toaster slot 和 bread，生成 bread 位置的高 affordance 区域；gripper map 在接近面包时关闭；velocity map 在靠近狭窄 slot 时降低速度；avoidance map 避开 toaster 外壳。

这比单纯 “move_to(bread); close_gripper” 更细，因为空间约束和运动偏好都被编码在 map 中。

### Online Dynamics Learning

VoxPoser 还讨论接触丰富任务，例如开门、开冰箱、开窗。零样本轨迹往往有意义但不够可靠；若用 VoxPoser 生成的轨迹作为探索 prior，可以在少量在线交互中学习 dynamics model。

这对本项目有一个重要启发：coding agent 不一定总能靠代码修复完成任务。有些问题需要 learned dynamics，但 LLM/代码可以提供很好的探索先验。

### Experiments

论文在真实和仿真任务中展示 VoxPoser 能处理多种日常 manipulation，并且在空间组合和开放对象上强于 LLM+primitive baseline。错误分析显示主要瓶颈常来自 perception 和 dynamics，而不是 LLM 规格生成本身。

这非常适合本项目的 failure attribution：失败不能全怪 coding agent；要区分 perception、specification、planning、dynamics。

### Limitations

VoxPoser 依赖外部 perception 模块；对细粒度几何、整体视觉理解和复杂 contact-rich dynamics 仍有限；motion planner 主要考虑 end-effector 轨迹，未充分处理 whole-arm planning；prompt engineering 成本也存在。

## 对本项目的启发

1. Skill implementation 可以输出中间目标表示，而非直接动作。
2. LLM 写代码最适合“生成目标函数/约束/地图”，再由 planner 执行。
3. failure attribution 至少要分出 perception error、specification error、dynamics error。
4. 对接触丰富任务，代码修复要和 dynamics learning 结合。

## 我会追问的问题

- 如果 value map 导致失败，coding agent 应该修改 prompt、API 调用、map smoothing，还是 planner 权重？
- VoxPoser 的 3D map 能否成为 robot skill DSL 的一种实现？
- 如何把真实失败 trace 回放到 value map 生成过程，定位是 perception 还是 specification 错？
