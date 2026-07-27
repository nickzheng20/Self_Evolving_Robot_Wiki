---
title: 文章技术细节讲解
source: obsidian://open?vault=Self-Evolving%20Robot&file=Source%2FPapers%2Fcode-as-policies-2209.07753.pdf
author:
published:
created: 2026-06-28
description: ChatGPT conversation with 2 messages
tags:
  - clippings
---
**You said**

你是我的Phd Advisor，请你和我尽可能详细地讲一下这篇文章，深入技术细节。Section by Section，并结合具体的例子讲解。

---

**ChatGPT said**

我会把这篇当成一次 PhD 组会精读来讲。论文是 **Code as Policies: Language Model Programs for Embodied Control** ，核心思想是： **不要只让 LLM 输出“计划步骤”，而是让代码型 LLM 直接生成可执行的机器人 policy code，让代码去调用 perception API 和 control API，从而把自然语言、视觉感知、空间几何计算、机器人控制连起来。**

---

传统“语言控制机器人”大概有三条路线：

第一种是早期 symbolic / semantic parsing，把自然语言解析成逻辑形式或动作模板。例如“put apple on table”解析成 `PUT(apple, table)` 。问题是泛化差，新说法、新物体、新关系都要额外设计。

第二种是 end-to-end learning，也就是从语言和视觉直接学到动作，例如 language-conditioned imitation learning。问题是需要大量机器人数据，而真实机器人数据非常贵。

第三种是 2022 前后的 LLM planner 路线，例如让 LLM 输出：

```
1. Pick up coke can
2. Move a bit right
3. Place coke can
```

这比传统方法灵活，但它仍然有一个瓶颈：LLM 只是高层 planner，它只能从已有技能库里选技能。如果技能库里没有 `move_a_bit_right()` ，那它就做不了。论文在 Related Work 里专门用“move the coke can a bit to the right”举例说明：prior works 往往假设已经存在一个能“往右移动一点”的 skill，而 CaP 直接生成可执行的 policy code 来完成抓取、位移、放下。

这篇论文的关键转变是：

> **LLM 不只是 planner，而是 policy programmer。**

也就是说，LLM 生成的是 Python 代码。代码里面可以：

```python
while not detect_object("apple"):
    robot.set_velocity(x=0, y=0.1, z=0)
```

这就不只是“计划”，而是一个闭环策略：它一直读取感知结果，直到看到 apple 才停止。论文在 Introduction 里强调，policy code 可以处理 perception output，并参数化 control primitive API；还可以用 `if/else` 、 `for/while` 、NumPy、Shapely 等工具做空间几何推理。

---

## 1\. Abstract：论文一句话贡献

Abstract 的技术含义可以拆成四层。

第一层： **代码型 LLM 可以被重新用于写机器人 policy code。**  
不是训练一个新的机器人模型，而是用已经在代码补全任务上训练过的 LLM，例如 Codex，让它根据自然语言命令生成 Python 程序。论文说这些程序可以表达函数、反馈循环，处理 object detector 的输出，并调用控制 API。

第二层： **few-shot prompting 是主要接口。**  
作者给 LLM 几组示例：

```python
# move the purple bowl toward the left.
target_pos = get_pos('purple bowl') + [-0.3, 0]
put_first_on_second('purple bowl', target_pos)
```

然后用户给新指令：

```
move the red block a bit to the right
```

LLM 就会组合已有 API，生成类似：

```python
target_pos = get_pos('red block') + [0.1, 0]
put_first_on_second('red block', target_pos)
```

第三层： **代码天然适合做空间几何和数值计算。**  
比如“closest point”“left-most object”“20 cm below the blue bowl”“draw a circle around the sweeter fruit”，这些如果让 LLM 用自然语言推理，很容易算错；但生成 NumPy / Shapely 代码之后，实际计算交给 Python 完成。

第四层： **hierarchical code generation 是论文技术亮点。**  
LLM 可以生成一个调用未定义函数的程序，比如：

```python
objs_to_stack = [empty_bowl] + block_names
stack_objects(objs_to_stack)
```

然后系统发现 `stack_objects` 没定义，就再调用一个 function-generation LMP 去补这个函数。论文说这种递归定义未定义函数的方法提升了 RoboCodeGen 和 HumanEval 上的表现。

---

## 2\. Introduction：为什么 LLM planner 还不够？

Introduction 的核心问题是 **grounding** 。机器人使用语言，必须把词语和真实世界里的感知、动作连接起来。论文说，经典方法依赖 lexical analysis / semantic representation，但面对 unseen instructions 很困难；end-to-end 方法虽然可以从语言直接学动作，但需要大量真实机器人数据。

LLM planner 的出现缓解了一部分问题。比如 SayCan 这类方法可以让 LLM 生成高层步骤，再用 affordance model 判断机器人能不能做。但是作者认为，这种抽象仍然限制很大，因为 LLM 不能直接进入 perception-action feedback loop。论文给了三个非常关键的例子：

```
put the apple down on the orange
put the apple down when you see the orange
move the apple a bit to the left
```

第一个是空间关系，第二个是条件触发，第三个是带数值大小的空间位移。如果 LLM 只能选已有技能，那么你必须提前训练或写好大量 skill，例如 `place_on_object` 、 `wait_until_seen` 、 `move_left_by_delta` 。CaP 的想法是：直接让 LLM 写代码，把这些组合逻辑临时生成出来。

图 1 很重要。它展示了用户说：

```
Stack the blocks on the empty bowl.
```

LLM 生成 policy code：

```python
block_names = detect_objects("blocks")
bowl_names = detect_objects("bowls")

for bowl_name in bowl_names:
    if is_empty(bowl_name):
        empty_bowl = bowl_name
        break

objs_to_stack = [empty_bowl] + block_names
stack_objects(objs_to_stack)
```

然后又递归生成：

```python
def stack_objects(obj_names):
    n_objs = len(obj_names)
    for i in range(n_objs - 1):
        obj0 = obj_names[i + 1]
        obj1 = obj_names[i]
        pick_place(obj0, obj1)
```

这个例子体现了 CaP 的三个能力：调用 perception API 找物体，调用 control API 执行动作，递归生成缺失函数。

我作为 advisor 会特别强调：这篇论文的真正贡献不是“LLM 能写代码”这么简单，而是把 **code generation 放到了机器人控制栈的中间层** 。它不是最底层 torque control，也不是最高层自然语言 plan，而是处在：

```
自然语言
  ↓
policy code
  ↓
perception API / geometry computation / primitive parameterization
  ↓
robot primitive control
```

这个中间层很有研究价值，因为它既有语言泛化能力，又保留了程序的可解释性和数值精确性。

---

## 3\. Related Work：CaP 和之前方法的本质区别

Related Work 主要对比三类工作。

第一类是 semantic parsing / language grounding。它们把语言映射成逻辑表示或动作结构。例如早期的 Winograd，到后来的自然语言导航、机器人对话系统。优点是结构清晰，缺点是需要人工设计 grammar、ontology 或训练数据，面对开放式语言泛化有限。

第二类是 learning-based low-level policies。例如 language-conditioned imitation learning、reinforcement learning、CLIPort。优点是可以直接学视觉到动作，缺点是数据量大，而且泛化很依赖训练分布。

第三类是 LLM-based embodied planning。例如 Huang et al. 的 zero-shot planner、SayCan、Inner Monologue、Socratic Models。它们通常输出自然语言步骤或调用已有技能。论文说这些方法往往需要预定义的 skill，例如 `robot.grasp(coke can)` 或 `robot.place_a_bit_right()` ；CaP 则生成代码来完成具体控制逻辑，不要求每个语言表达都已经有对应 skill。

论文里的对比例子非常关键：

传统 LLM planner 可能输出：

```
1. Pick up coke can
2. Move a bit right
3. Place coke can
```

Socratic Models 可能输出：

```
objects = [coke can]
1. robot.grasp(coke can)
2. robot.place_a_bit_right()
```

CaP 生成的是：

```python
while not obj_in_gripper("coke can"):
    robot.move_gripper_to("coke can")
robot.close_gripper()

pos = robot.gripper.position
robot.move_gripper(pos.x, pos.y + 0.1, pos.z)
robot.open_gripper()
```

这里的差别非常深：CaP 把“a bit right”落成了数值 `+0.1` ，把“pick”展开成闭环过程，把高层语言变成了控制程序。

从研究角度看，CaP 其实是把 LLM 当成 **program synthesizer for robot policy** 。它不是纯 NLP，也不是纯 control，而是 program synthesis + robotics。

---

## 4\. Method：这篇论文最重要的技术部分

## 4.1 什么是 LMP？

论文定义 LMP，也就是 **Language Model Program** ：任何由语言模型生成并在系统上执行的程序。CaP 是 LMP 的一种机器人版本：它把语言指令映射成代码片段，这些代码片段需要满足三个条件：

1. 会响应感知输入，例如传感器、object detector、segmentation、position API；
2. 会参数化控制 primitive，例如 `put_first_on_second` 、 `goto_pos` 、 `set_velocity` ；
3. 会被直接编译和执行在机器人系统上。

一个简单例子：

```python
# stack the blocks in the empty bowl.
empty_bowl_name = parse_obj('empty bowl')
block_names = parse_obj('blocks')
obj_names = [empty_bowl_name] + block_names
stack_objs_in_order(obj_names=obj_names)
```

这里有几个层次：

`parse_obj('empty bowl')` 是语言到感知对象的 grounding。  
`stack_objs_in_order` 是 task-level policy abstraction。  
`put_first_on_second` 是底层 pick-and-place primitive。

也就是说，LLM 不一定直接控制电机，而是写一个中间层 policy，把 perception 输出转换成 primitive 参数。

---

## 4.2 Prompt 结构：Hints + Examples

论文说生成 LMP 的 prompt 有两个元素： **Hints** 和 **Examples** 。Hints 通常是 import statements 或 API 类型提示，告诉 LLM 当前可用函数；Examples 是自然语言指令到代码的 few-shot demonstrations。

例如 Hints：

```python
import numpy as np
from utils import get_obj_names, put_first_on_second
```

这告诉 LLM：你可以用 NumPy，也可以调用 `get_obj_names` 和 `put_first_on_second` 。

Examples：

```python
# put the gray block on the gray bowl.
put_first_on_second('gray block', 'gray bowl')

# move the purple bowl toward the left.
target_pos = get_pos('purple bowl') + [-0.3, 0]
put_first_on_second('purple bowl', target_pos)
```

然后如果用户说：

```
move the red block a bit to the right
```

LLM 可能生成：

```python
target_pos = get_pos('red block') + [0.1, 0]
put_first_on_second('red block', target_pos)
```

这里的关键不是模型“理解物理”到很深，而是 prompt 已经建立了 convention：left 是 x 方向减小，right 是 x 方向增加；“a bit”大概是 0.1；对象名可以直接传给 API。

论文还提到可以维护一个 LMP session，把之前的 instruction 和 response 继续 append 到 prompt 中，这样之后可以处理 “undo the last action” 这种依赖上下文的命令。

---

## 4.3 Low-level LMP：从纯 Python 到机器人 API

论文先从简单 Python 开始：

```python
# get the variable a.
ret_val = a

# find the sum of variables a and b.
ret_val = a + b

# see if any number is divisible by 3 in a list called xs.
ret_val = any(x % 3 == 0 for x in xs)
```

这个例子看起来简单，但它定义了一个协议：模型输出的结果放在 `ret_val` ，系统之后从 `locals` 里取出 `ret_val` 。

然后论文引入 NumPy 做空间推理：

```python
# move all points in pts_np toward the right.
ret_val = pts_np + [0.3, 0]

# get the left most point in pts_np.
ret_val = pts_np[np.argmin(pts_np[:, 0]), :]

# get the center of pts_np.
ret_val = np.mean(pts_np, axis=0)

# the closest point in pts_np to pt_np.
ret_val = pts_np[np.argmin(np.sum((pts_np - pt_np)**2, axis=1))]
```

这段特别重要。因为空间几何推理在自然语言里很容易错，但代码里非常稳定。比如“closest point”就是算欧氏距离平方；“left-most”就是取 x 坐标最小；“center”就是平均值。论文明确说，代码型 LLM 可以利用 NumPy 等第三方库来完成复杂指令，而不需要自己写所有底层逻辑。

再进一步，LMP 可以调用 first-party robotics API：

```python
from utils import get_pos, put_first_on_second

# move the purple bowl toward the left.
target_pos = get_pos('purple bowl') + [-0.3, 0]
put_first_on_second('purple bowl', target_pos)
```

这些 API 不一定出现在 LLM 训练数据里，但如果名字足够语义化，例如 `get_pos` 、 `put_first_on_second` 、 `get_bbox_xyxy` ，LLM 可以根据名字和 examples 推断用法。论文在 Appendix 的 prompt engineering 里也强调，函数名和变量名越明确，生成越可靠。

---

## 4.4 Language reasoning：对象描述如何解析？

CaP 不是只做几何。它还让 LLM 根据语言描述选对象。例如：

```python
objs = ['blue bowl', 'red block', 'red bowl', 'blue block']

# the bowls.
ret_val = ['blue bowl', 'red bowl']

# sea-colored block.
ret_val = 'blue block'

# the other block.
ret_val = 'red block'
```

这里有两类推理。

一类是 category filtering： `bowls` 对应所有名字里含 bowl 的对象。

另一类是 commonsense / synonym： `sea-colored` 对应 blue。

还有一类是 context-dependent reference： `the other block` 要根据之前出现过哪个 block 来理解。

这说明 LMP 中的语言模型负责“语义选择”，但最终返回的是程序可用的数据结构，比如字符串或字符串列表。

---

## 4.5 High-level LMP：为什么代码可以成为 policy？

这篇论文最关键的点是：代码不仅可以做一次性计算，还可以表达 **control flow** 。例如：

```python
# while the red block is to the left of the blue bowl,
# move it to the right 5cm at a time.
while get_pos('red block')[0] < get_pos('blue bowl')[0]:
    target_pos = get_pos('red block') + [0.05, 0]
    put_first_on_second('red block', target_pos)
```

这已经是一个闭环 policy。每次循环重新调用 `get_pos` ，根据最新感知结果判断是否继续移动。论文用这个例子说明，LMP 可以用 `while` loop 形成反馈策略，而 prompt 里甚至不一定出现过 while-loop 示例。

我们把它形式化一下。传统 policy 是：

```
a_t = π(s_t)
```

CaP 里的 policy 是一个程序：

```python
while condition(perception()):
    action = compute_action(perception())
    control_api(action)
```

也就是说：

```
language command
  ↓ LLM
program c
  ↓ exec(c, APIs)
actions over time
```

这和单纯 LLM plan 有本质区别。Plan 通常是一次性生成固定步骤；policy code 可以在执行时持续读取状态。

---

## 4.6 LMP composition：一个 LMP 调用另一个 LMP

论文进一步提出，不要让一个 prompt 做所有事情。可以把系统拆成多个 LMP：

```python
block_name = parse_obj('the left most block')
while block_name == 'red block':
    target_pos = get_pos(block_name) + [0.3, 0]
    put_first_on_second(block_name, target_pos)
    block_name = parse_obj('the left most block')
```

这里 high-level LMP 负责控制结构， `parse_obj` 另一个 LMP 负责语言到对象选择。 `parse_obj` 可能生成：

```python
block_names = ['red block', 'blue block']
block_positions = np.array([get_pos(name) for name in block_names])
left_block_name = block_names[np.argmin(block_positions[:, 0])]
ret_val = left_block_name
```

这个设计很像软件工程里的模块化。每个 LMP prompt 可以专注于一个功能：解析对象、解析位置、生成 trajectory、回答问题、生成函数。论文说这样可以放入更多 few-shot examples，提高单个模块的准确率，同时避免超过 LLM 的 context limit。

---

## 4.7 Hierarchical code generation：递归生成未定义函数

这是论文最有技术含量的部分。

普通 flat code generation 会要求 LLM 一次性写完整函数。但复杂任务里，一次写完容易出错。CaP 允许 LLM 写“草图式代码”，调用暂时不存在的函数。例如：

```python
def get_objs_bigger_than_area_th(obj_names, bbox_area_th):
    return [name for name in obj_names
            if get_obj_bbox_area(name) > bbox_area_th]
```

这里 `get_obj_bbox_area` 没定义。系统会解析生成代码的 AST，发现这个函数不在当前 scope 里，于是调用 function-generation LMP 自动补：

```python
def get_obj_bbox_area(obj_name):
    x1, y1, x2, y2 = get_obj_bbox_xyxy(obj_name)
    return (x2 - x1) * (y2 - y1)
```

论文指出，prompt 甚至没有显式说 `get_obj_bbox_xyxy` 返回什么，但函数名暗示了它返回 bounding box 的 `x1, y1, x2, y2` ，LLM 可以据此生成正确面积计算。然后系统把这个新函数加入 scope，如果新函数内部还有未定义函数，就继续 depth-first 递归生成。

这个机制可以理解为：

```python
def execute_lmp(code):
    ast_tree = parse(code)
    undefined = find_undefined_function_calls(ast_tree, scope)

    for f in undefined:
        f_code = function_generation_lmp(f)
        execute_lmp(f_code)   # recursive, depth-first
        scope[f] = compiled_function

    exec(code, globals=apis, locals=scope)
```

这和 Chain-of-Thought 有点类似，但不是让模型写自然语言 reasoning，而是让模型通过函数分解来思考。论文也说 hierarchical code-gen with verbose variable names 可以看作一种 functional-programming version of chain-of-thought。

---

## 4.8 复杂例子：面积、空间关系、循环控制结合

论文给了一个很好的组合例子：

```
while there are blocks with area bigger than 0.2 that are left of the red bowl,
move them toward the right.
```

高层 LMP 生成：

```python
block_names = parse_obj(
    'blocks with area bigger than 0.2 that are left of the red bowl'
)

while len(block_names) > 0:
    for block_name in block_names:
        target_pos = get_pos(block_name) + np.array([0.1, 0])
        put_first_on_second(block_name, target_pos)

    block_names = parse_obj(
        'blocks with area bigger than 0.2 that are left of the red bowl'
    )
```

`parse_obj` 又生成：

```python
block_names = ['red block', 'blue block']
red_bowl_pos = get_pos('red bowl')

use_block_names = [
    name for name in block_names
    if get_pos(name)[0] < red_bowl_pos[0]
]

use_block_names = get_objs_bigger_than_area_th(use_block_names, 0.2)
ret_val = use_block_names
```

然后 `get_objs_bigger_than_area_th` 可能触发 hierarchical function generation。

这个例子非常适合你理解 CaP 的本质：

```
自然语言复杂约束
= 对象类别过滤 + 几何关系判断 + 数值阈值 + 闭环执行 + 函数递归生成
```

这已经不是简单 prompt engineering，而是一个动态程序合成系统。

---

## 4.9 LMP 执行：安全检查与 scope

论文说执行 LMP 前，他们会做一些安全检查：不允许 import statements，不允许以 `__` 开头的特殊变量，不允许调用 `exec` 和 `eval` 。然后调用 Python 的 `exec` ，并传入两个字典： `globals` 包含可用 API， `locals` 是空字典，用来存放执行时生成的变量和函数。如果需要返回值，就从 `locals` 里取。

技术上这很关键，因为它决定了 LMP 不是自由运行任意 Python，而是在一个受控 scope 里运行：

```python
safe_globals = {
    "np": np,
    "get_pos": get_pos,
    "put_first_on_second": put_first_on_second,
    "say": say,
}

local_vars = {}
exec(generated_code, safe_globals, local_vars)
```

不过作为 advisor，我会提醒你：这不是严格安全沙箱。禁用 `import` 、 `exec` 、 `eval` 只是很初级的 protection。真实机器人部署需要更强的 runtime monitor、type checker、resource limiter、simulator validation 和 safety constraints。

---

## 5\. Language Model Programs as Policies：这节的理论位置

这节把前面的技术统一成机器人 policy 视角。

在机器人系统里，perception model 的高层输出可以看作 state，比如：

```python
get_obj_names()
get_obj_pos(name)
get_bbox(name)
is_obj_visible(name)
```

control primitive API 可以看作 action interface，比如：

```python
put_first_on_second(obj_name, target)
goto_pos(pos_3d)
robot.set_velocity(x, y, z)
draw(pts_2d)
```

CaP 生成的代码就是连接 state 和 action 的 policy logic。论文说这些 API 把 LMP ground 到真实机器人系统里；perception/control 算法本身提升，LMP-based policy 的能力也会提升。真实实验中，作者用 ViLD 和 MDETR 这样的 open-vocabulary object detection 模型来获得物体位置和 bounding boxes。

论文总结 LMP-based policies 的三个好处：

第一，可以根据 unseen natural language instruction 生成新的 policy code 和参数。

第二，可以借助 open-vocabulary perception 泛化到新物体和新环境。

第三，不需要额外数据收集或模型训练。并且生成的 plan/policy 是代码，所以可解释、可修改、可复用。

我建议你把 CaP 看成一种 **factorized robot intelligence** ：

```
LLM: 语言理解 + 程序合成 + 常识
VLM/object detector: 开放词汇感知
Classical geometry libraries: 精确空间计算
Robot primitives: 可靠底层控制
Python program: glue logic / policy logic
```

它的强项正是“胶水层”。它没有试图端到端学完所有东西，而是把不同模块组合起来。

---

## 6\. Experiments：实验到底证明了什么？

论文实验有三个目标：第一，评估 hierarchical code generation；第二，在模拟 tabletop manipulation 里和 baselines 比较；第三，在不同真实机器人系统上展示 CaP 的灵活性。论文也承认，因为开放式任务难以量化，真实机器人部分主要是 demonstrations，定量评估集中在受限模拟任务上。

---

## 6.1 Experiment A：Hierarchical LMP on Code-Generation Benchmarks

作者做了两个 benchmark。

第一个是他们新提出的 **RoboCodeGen** ，有 37 个 robotics-themed function generation problems，包括空间推理、几何推理、控制、NumPy、Shapely、first-party API 调用等。和普通代码 benchmark 不同，这里函数 header 没有 docstring 或显式 type hints，模型要根据函数名和 convention 推断含义。

Table I 的结果很明显：

```
RoboCodeGen pass rate (%)

GPT-3 6.7B:      Flat 3     → Hierarchical 5
GPT-3 175B:      Flat 68    → Hierarchical 84
Codex cushman:   Flat 54    → Hierarchical 57
Codex davinci:   Flat 81    → Hierarchical 95
```

两个结论：

第一，code-specialized model 比普通 text model 更适合这个任务。  
第二，hierarchical code generation 明显提升大模型表现，尤其是 davinci。

第二个 benchmark 是 **HumanEval** 。Table II / Appendix D 里结果是：

```
HumanEval pass rate (%)

Flat:
Greedy 45.7
P@1    34.9
P@10   75.1
P@100  90.9

Hierarchical:
Greedy 53.0
P@1    39.8
P@10   80.6
P@100  95.7
```

这说明 hierarchical code-gen 不只是机器人 prompt trick，而是对通用代码生成也有帮助。论文还指出，机器人任务需要 zero-shot 执行，不能像 benchmark 那样 sample 100 个答案再用 unit tests 挑，所以实际 LMP 采用 temperature=0 的 greedy decoding。

Appendix E 还分析了五种 generalization：systematicity、productivity、substitutivity、localism、overgeneralization。图 4 显示 hierarchical code generation 对 productivity 帮助最大，也就是当新任务需要更长代码或更多逻辑层时，分层生成最有用；但小模型 cushman 受益不明显，说明模型本身需要先达到一定代码能力。

---

## 6.2 Experiment B：Drawing Shapes via Generated Waypoints

这个实验是让真实 UR5e 机械臂在白板上画图。机器人要根据自然语言生成 2D waypoints，然后 end-effector trajectory follower 沿着这些点画。感知部分用 MDETR 检测物体位置，动作部分提供 trajectory-following API。系统有四个 LMP：

```
1. parse user command / maintain session / call action APIs
2. parse object names from language descriptions
3. parse waypoints from language descriptions
4. generate new functions
```

论文说它可以处理未见过的语言命令，画新形状，解析精确尺寸，操作之前画过的形状，多步命令等。

Appendix H 给了一串白板任务，例如：

```
draw a 5cm hexagon around the middle
draw a line that bisects the hexagon
make them both bigger
erase the hexagon and the line
draw the sun as a circle at the top right
draw the ground as a line at the bottom
draw a pyramid as a triangle on the ground
draw a smaller pyramid a little bit to the left
draw circles around the blocks
draw a square around the sweeter fruit
```

技术上，“draw a smaller pyramid a little bit to the left of the pyramid” 可能被转成类似：

```python
pyramid_pts = get_shape_pts('pyramid')
center = np.mean(pyramid_pts, axis=0)

small_pyramid_pts = scale_pts(pyramid_pts, scale=0.5, origin=center)
small_pyramid_pts = small_pyramid_pts + np.array([-0.1, 0])

draw(small_pyramid_pts)
```

如果涉及“circle around the blue block”，则需要：

```python
obj_pos = get_obj_pos('blue block')
circle_pts = make_circle_pts(center=obj_pos, radius=0.05, n=60)
draw(circle_pts)
```

CaP 在这里的优势是：语言模型负责理解“smaller”“a little left”“around”，代码和 NumPy 负责实际坐标变换，机器人 primitive 负责轨迹执行。

---

## 6.3 Experiment C：Pick & Place Policies for Table-Top Manipulation

这个实验是 UR5e 机械臂 + suction gripper + Intel RealSense D435 相机，在桌面上移动玩具物体。感知 API 用 MDETR 检测物体存在、位置、bounding boxes；动作 API 是一个 scripted pick-and-place primitive，可以把某个物体拿到目标位置或另一个物体上。

论文图 2 的 tabletop 例子包括：

```
Put the blocks in bowls with non-matching colors
Put the blocks in a vertical line 20 cm long and 10 cm below the blue bowl
Wait until you see an egg and put it in the green plate
Put the darkest object in the plate that has the apple
```

图 2 的意义是展示 CaP 可以跨桌面操作、2D drawing、mobile manipulation 等不同机器人和任务域工作，而且用的是 off-the-shelf models + few-shot prompting，没有额外训练。

比如 “Put the blocks in bowls with non-matching colors” 可能生成：

```python
block_names = parse_obj('blocks')
bowl_names = parse_obj('bowls')

for block_name in block_names:
    block_color = parse_color(block_name)

    candidate_bowls = [
        bowl for bowl in bowl_names
        if parse_color(bowl) != block_color
    ]

    target_bowl = candidate_bowls[0]
    put_first_on_second(block_name, target_bowl)
```

又比如：

```
put the blocks in a vertical line 20 cm long and 10 cm below the blue bowl
```

可以生成：

```python
block_names = parse_obj('blocks')
blue_bowl_pos = get_obj_pos('blue bowl')

n = len(block_names)
center = blue_bowl_pos + np.array([0, -0.10])

ys = np.linspace(center[1] - 0.10, center[1] + 0.10, n)
x = center[0]

for block_name, y in zip(block_names, ys):
    target_pos = np.array([x, y])
    put_first_on_second(block_name, target_pos)
```

这里的“20 cm long”变成 `0.20` 米长度，“10 cm below”变成 `-0.10` 的偏移。这就是 code-as-policy 的价值：语言里的模糊空间描述被转成明确数值。

---

## 6.4 Experiment D：Simulation Table-Top Manipulation Evaluation

这是论文最主要的定量机器人实验。

环境是模拟 tabletop：UR5e + Robotiq 2F85 gripper，要操作 10 种颜色的 blocks 和 10 种颜色的 bowls。CaP 可以访问 scripted object detector、物体位置和 pick-and-place primitive。任务分为 long-horizon 和 spatial-geometric 两类。作者把 attributes 和 instructions 分成 seen / unseen：

```
SA = Seen Attributes
UA = Unseen Attributes
SI = Seen Instructions
UI = Unseen Instructions
```

例如 seen attributes 可能包括 blue/red/green/orange/yellow，unseen attributes 包括 pink/cyan/brown/gray/purple；seen direction 包括 top/left，unseen direction 包括 bottom/right；seen line 包括 horizontal/vertical，unseen line 包括 diagonal。

比较对象有两个：

1. CLIPort：language-conditioned imitation learning，训练了 30k demonstrations；
2. NL Planner：few-shot prompted LLM planner，但输出自然语言计划，不输出代码。

Table III 结果：

```
Success rate over task families, 50 trials per task

SA/SI Long-Horizon:
CLIPort 78.80, NL Planner 86.40, CaP 97.20

SA/SI Spatial-Geometric:
CLIPort 97.33, NL Planner N/A, CaP 89.30

UA/SI Long-Horizon:
CLIPort 36.80, NL Planner 88.00, CaP 97.60

UA/SI Spatial-Geometric:
CLIPort 0.00, NL Planner N/A, CaP 73.33

UA/UI Long-Horizon:
CLIPort 0.00, NL Planner 64.00, CaP 80.00

UA/UI Spatial-Geometric:
CLIPort 0.01, NL Planner N/A, CaP 62.00
```

这张表非常值得仔细看。

第一，在 seen attributes / seen instructions 上，CLIPort 表现不错，但 CaP 更强，尤其 long-horizon 是 97.2%。

第二，在 unseen attributes 上，CLIPort 大幅掉，因为它是 supervised policy，训练分布外泛化弱。CaP 和 NL Planner 更稳，因为 LLM 可以处理新颜色、新词、新组合。

第三，spatial-geometric 任务上，NL Planner 是 N/A，因为自然语言 planner 很难输出精确坐标和数值计算。比如“place blocks in a diagonal line”，这不是简单 step list 能解决的，需要算点。

第四，在最难的 UA/UI 条件下，CaP 仍然有 80% long-horizon 和 62% spatial-geometric 成功率，而 CLIPort 接近 0。这是论文最强的定量证据。

Appendix C 还单独比较了自然语言推理 vs code reasoning。在 object selection 和 position selection 任务上，Vanilla natural language 总准确率 35%，Chain-of-Thought 58%，LMP 98%。这说明对于空间几何和数值坐标，代码执行明显优于纯自然语言推理。

---

## 6.5 Experiment E：Mobile Robot Navigation and Manipulation

这个实验用的是 Everyday Robots 的移动操作机器人：mobile base + 7 DoF arm，在真实 office kitchen 里导航和操作物体。感知用 ViLD，动作 API 包括：

```python
goto_pos(pos_3d)
goto_loc(name)
pick_obj(name)
place_at_pos(pos_3d)
place_at_obj(name)
say(message)
```

还有感知和状态 API：

```python
is_obj_visible(name)
get_visible_obj_names()
get_loc_names()
get_robot_pos_and_angle()
get_obj_pos(name)
get_loc_pos(name)
```

任务包括：

```
Moving in a 3m by 2m rectangle around the office chair
Do that again but rotated 45 degrees clockwise
Go in a 1.5m square around the barstool as many times as needed,
check each step if there is a banana,
only stop moving when you see the banana
Follow the convex hull containing the chairs
Move back and forth between the table and the countertop 3 times
```

还有 manipulation：

```
Take the water bottle from the desk and put it in the middle of the fruits on the table
This is the compost bin
This is the recycle bin
This is the landfill bin
The coke can and the apple are on the table
Put away the coke can and the apple in their corresponding bins
```

“只在看到 banana 时停下”的任务非常体现 policy code 的优势。可能生成：

```python
while True:
    follow_next_segment(square_traj)

    if is_obj_visible('banana'):
        say('I stopped because I saw the banana.')
        break
```

“Put away the coke can and the apple in their corresponding bins” 则需要短期记忆。用户之前说过：

```
This is the compost bin.
This is the recycle bin.
This is the landfill bin.
```

代码可以把这些 location 存到变量里：

```python
compost_bin_pos = get_robot_pos_and_angle()
```

之后再引用这些变量，把 apple 放 compost，把 coke can 放 recycle。论文说 mobile manipulation 中 CaP 可以通过 Python execution scope 显式记录变量，形成 short-term memory。

---

## 7\. Discussion and Limitations：这篇论文真正的边界

Discussion 这节非常诚实。CaP 的泛化发生在机器人栈的特定层：解释自然语言，处理 perception outputs，然后参数化低维控制 primitive。它适合 factorized perception and control 系统，不是端到端从 pixels 到 motor torques。

论文列出的限制主要有四类。

第一， **受 perception API 限制。**  
如果视觉系统不能描述某个概念，CaP 也没法凭空做。例如论文说，现有视觉语言模型无法描述 trajectory 是否 “bumpy” 或 “more C-shaped”。也就是说，LLM 能写代码，但如果没有 API 提供相应观测，代码就没有信息来源。

第二， **受 control primitive 限制。**  
如果只有 pick-and-place primitive，那它就很难完成 dexterous manipulation，例如拧瓶盖、折衣服、插线、搭复杂 3D 房子。论文举例说，如果 tabletop domain 没有搭复杂 3D 结构的 examples，让 LMP “build a house with the blocks” 会很困难。

第三， **prompt 和 abstraction level 很脆弱。**  
如果用户命令明显比 prompt examples 更长、更复杂，或者处在完全不同抽象层，生成会不稳定。Prompt 不是越长越好，太多 primitive 参数也会 saturate prompt。

第四， **无法事前保证正确性。**  
论文明确说，它假设指令是 feasible 的，并且不能 a priori 判断 response 是否正确。

作为你的 advisor，我会补充几个更深的批评：

CaP 把很多困难“外包”给了 perception 和 primitive control。比如 `put_first_on_second` 看起来简单，但真实抓取、避障、接触控制、失败恢复都藏在里面。CaP 的贡献是组合和参数化，不是解决底层 manipulation。

它缺少 formal verification。生成代码可读，但可读不等于安全。真实机器人要防止无限循环、碰撞、越界、危险速度、误识别后的错误动作。

它没有 uncertainty-aware reasoning。比如 object detector 置信度低时怎么办？多个 apple 怎么办？目标被遮挡怎么办？论文 Appendix I 也提到 tabletop prompt 目前只支持 unique objects，duplicate object identity persistence 是 perception/world-state tracking 的问题。

它不是学习系统。执行失败后，CaP 本身没有从失败中自动更新 policy，除非外部系统把反馈写进 prompt 或更新 API。

---

## 8\. Appendix：附录其实很重要

## 8.1 Prompt Engineering

Appendix A 讲了很多实用经验。论文强调 prompt 里的 code 必须没有 bug，因为 prompt bug 会导致 LMP 输出不可靠；如果模型生成错误代码，首先应检查相关 examples 是否正确。

它还强调变量名和函数名要明确。例如：

```python
pts_np
pt_np
K_x_mat
get_bbox_xyxy
```

这些名字相当于 informal type hints。 `pts_np` 暗示是 NumPy array； `get_bbox_xyxy` 暗示返回 x1, y1, x2, y2； `K_x_mat` 暗示是 end-effector space 的矩阵。论文说 ambiguity 会让 LMP 不知道该用哪种 convention，比如同一个 `point` 有时是 `np.ndarray` ，有时是 `shapely.geometry.Point` ，会导致不稳定。

这对你以后做机器人 LLM 系统很重要： **API design 本身就是 prompt engineering。**

好的 API 名字：

```python
get_obj_pos(name)
get_obj_bbox_xyxy(name)
move_end_effector_to(pos)
put_first_on_second(obj_name, target)
```

坏的 API 名字：

```python
do(x)
proc(a, b)
handle(obj)
```

LLM 对后者很难可靠推断。

---

## 8.2 Code reasoning vs Natural Language reasoning

Appendix C 的结论是：空间几何任务里，代码比自然语言推理强很多。任务包括 object selection 和 position selection，例如“find the name of the block closest to the blue bowl”或“interpolate 3 points on a line from cyan bowl to blue bowl”。结果是 Vanilla 35%，CoT 58%，LMP 98%。

这给你一个很重要的研究启发：

> LLM 不一定要自己“算”。LLM 可以生成程序，让程序算。

这和现在很多 tool-use / program-aided reasoning 思路一致。对于机器人尤其重要，因为机器人任务里大量东西是连续空间、几何、控制、约束优化，不适合纯语言 token 推理。

---

## 8.3 Reactive Controllers：CartPole 和 Impedance Control

Appendix F 展示 CaP 不只会 pick-and-place，也能生成简单 reactive controller。

CartPole 例子是生成一个 PD controller，根据 pole angle 和 angular velocity 决定向左还是向右推车：

```python
def keep_pole_upright_with_pd_control(x, x_dot, theta, theta_dot):
    kp = 1
    kd = 1

    error = theta
    error_dot = theta_dot

    control = kp * error + kd * error_dot

    direction = 1
    if control < 0:
        direction = 0

    return direction
```

论文说这个 controller 只能在初始化接近 upright 时稳定，但可以稳定不稳定平衡点。

更机器人控制一点的是 end-effector impedance control：

```python
def ee_impedance_control(x_curr, x_goal, x_dot, K_x_mat, D_x_mat, J):
    x_err = x_goal - x_curr
    x_dot_err = -x_dot

    tau = np.matmul(
        J.T,
        np.matmul(K_x_mat, x_err) + np.matmul(D_x_mat, x_dot_err)
    )

    return tau
```

这个公式的控制含义是：

```
F_x = K_x (x_goal - x_curr) + D_x (0 - x_dot)
tau = J^T F_x
```

也就是在 operational space 里生成一个弹簧阻尼力，再通过 Jacobian transpose 映射到关节 torque。论文也承认这是简化版，没有补偿 Coriolis 或 gravity；复杂连续控制还需要 tuning，而 CaP 当前不支持基于执行反馈自动调参。

这个部分说明：CaP 理论上可以生成低层控制器，但目前更适合简单或结构明确的 controller。对于复杂机器人控制，还是需要传统 control / learning-based control / model predictive control。

---

## 8.4 Visual Language Models：感知怎么接进来？

Appendix G 说真实实验用的是 off-the-shelf open-vocabulary object detection models：ViLD 和 MDETR。它们输入自然语言 caption，输出图像中的 object bounding box 和 segmentation mask。ViLD 用在 mobile robot，MDETR 用在 tabletop manipulation 和 whiteboard drawing。为了把图像检测结果变成机器人坐标，系统用 depth camera，把像素点 deproject 到 3D，并且相机到机器人坐标系的 transform 预先标定好。

这部分非常重要，因为它说明 CaP 不是“LLM 直接看图控制机器人”。实际 pipeline 是：

```
RGB-D image
  ↓ VLM / object detector
bbox + segmentation mask
  ↓ depth deprojection + calibration
3D object position in robot frame
  ↓ get_obj_pos(name)
LMP-generated policy code
  ↓ control primitive
robot action
```

论文也承认 VLM 可靠性仍然是瓶颈，很多真实失败来自检测不准；甚至 prompt engineering 对 VLM 也有影响，例如 MDETR 用 “square” 比 “block” 更容易检测 blocks。

---

## 8.5 Whiteboard Drawing / Tabletop / Mobile Robot Appendices

Appendix H/I/J 给了更完整的 prompts、APIs 和任务列表。

Whiteboard domain 的 APIs 包括：

```python
get_obj_names()
get_obj_pos(name)
draw(pts_2d)
erase(pts_2d)
```

`draw(pts_2d)` 会让机械臂先移动到第一个点上方，然后下降到接触白板，再沿 waypoint 画。 `erase(pts_2d)` 类似，只是先接触 eraser。

Tabletop domain 的 APIs 更丰富：

```python
get_obj_names()
get_obj_pos(name)
is_obj_visible(name)
get_bbox(name)
get_segmask(name)
get_color_rgb(name)
get_corner_name(pos_2d)
get_side_name(pos_2d)
denormalize_xy(normalized_pos_2d)
put_first_on_second(obj_name, target)
say(message)
```

这解释了为什么它能回答 “how many blocks are there?”，也能执行 “put the darkest object in the plate that has the apple”。 `get_color_rgb` 可以支持 darkest； `get_bbox` 支持 smallest； `get_corner_name` 和 `get_side_name` 支持 corner/side 关系； `say` 支持对话。

Mobile domain 的 APIs 支持 navigation、object picking、placing 和 speech。Appendix J 的图 6 展示了 Everyday Robots 的移动操作平台、RGB-D 图像和 pre-manipulation pose。

---

## 9\. 这篇论文最值得你学习的技术思想

## 9.1 Code 是一种中间表示，比自然语言 plan 更强

自然语言 plan：

```
Move the apple slightly left.
```

缺少精确动作参数。

代码表示：

```python
target_pos = get_obj_pos('apple') + np.array([-0.05, 0])
put_first_on_second('apple', target_pos)
```

有明确坐标、数值、API 调用。

这就是 CaP 的本质： **把语言的不确定性变成程序结构，把几何计算交给 Python，把物理执行交给 robot primitive。**

---

## 9.2 Hierarchical generation 让复杂程序可扩展

一次性生成完整代码容易失败。分层生成允许模型先写：

```python
use_block_names = get_objs_bigger_than_area_th(block_names, 0.2)
```

再补：

```python
def get_objs_bigger_than_area_th(obj_names, bbox_area_th):
    return [name for name in obj_names
            if get_obj_bbox_area(name) > bbox_area_th]
```

再补：

```python
def get_obj_bbox_area(obj_name):
    x1, y1, x2, y2 = get_obj_bbox_xyxy(obj_name)
    return (x2 - x1) * (y2 - y1)
```

这相当于让 LLM 进行程序抽象。论文结果也证明 hierarchical code-gen 在 RoboCodeGen 和 HumanEval 都提高了 pass rate。

---

## 9.3 API 设计决定系统能力边界

CaP 不是魔法。你给它什么 API，它就能组合什么 API。

如果只有：

```python
put_first_on_second(obj_name, target)
```

它就擅长 pick-and-place。

如果还有：

```python
get_bbox(name)
get_color_rgb(name)
get_segmask(name)
```

它就能做 smallest、darkest、inside、area 等推理。

如果还有：

```python
goto_pos()
pick_obj()
place_at_obj()
```

它就能做 mobile manipulation。

如果没有 force control / contact sensing / grasp stability / collision checking API，它就很难安全做复杂接触任务。

所以 CaP 的研究问题其实包括： **如何设计一组既 expressive 又 safe、既 compact 又 compositional 的 robot APIs。**

---

## 10\. 我对这篇论文的 advisor-level critique

这篇论文非常有启发性，但你读的时候不要只看到 demo 很酷，要看到它真正的科学位置。

它的强贡献是：提出了一个清晰的范式，把 LLM code generation 用作机器人 policy 层；通过代码实现空间几何、控制流、短期记忆、函数抽象；通过 hierarchical code-gen 提升代码生成能力；在多个机器人平台上展示了通用性。

它的弱点是：真实机器人定量评估有限；底层 manipulation 难点被 primitive API 屏蔽；安全性和正确性验证不足；prompt engineering 很重；对 perception 和 API scope 极其依赖；对复杂长任务和抽象层变化很脆弱。

如果你要基于这篇做研究，我会建议你思考几个方向：

**第一，typed API + static analysis。**  
给每个 API 明确输入输出类型、单位、坐标系、约束。例如 `Position2D[meter, robot_frame]` ，避免 LLM 把 pixel coordinate 和 robot coordinate 混用。

**第二，simulator-in-the-loop verification。**  
生成代码后先在仿真里跑，检查碰撞、越界、无限循环、目标达成，再上真实机器人。

**第三，uncertainty-aware CaP。**  
让 perception API 返回置信度和多假设：

```python
detections = detect_objects('apple')
# each detection has name, pos, conf, mask
```

然后 policy code 必须显式处理不确定性。

**第四，world state memory。**  
解决 duplicate object tracking、遮挡、物体身份保持。论文里 tabletop prompt 只支持 unique objects，这在真实场景里是明显限制。

**第五，failure recovery。**  
如果 `put_first_on_second` 失败，代码应该检测失败原因并 replan，而不是假设每次 primitive 都成功。

**第六，formal safety layer。**  
即使 LLM 生成错误代码，robot runtime 也应该强制速度限制、workspace 限制、collision avoidance、human proximity constraints。

---

## 11\. 最后用一个完整例子帮你串起来

假设用户说：

```
Wait until you see an egg and put it in the green plate.
```

CaP 系统可能这样运行。

首先 high-level LMP 根据 prompt 生成：

```python
while not is_obj_visible('egg'):
    pass

put_first_on_second('egg', 'green plate')
```

但真实系统不能用空 `pass` 死循环，工程上更合理：

```python
while not is_obj_visible('egg'):
    time.sleep(0.2)

egg_pos = get_obj_pos('egg')
put_first_on_second('egg', 'green plate')
say('I put the egg in the green plate.')
```

这里每个部分都对应论文里的组件：

`is_obj_visible` 来自 perception API。  
`get_obj_pos` 来自 MDETR / ViLD + depth deprojection。  
`put_first_on_second` 是 pick-and-place primitive。  
`while` loop 让它成为 reactive policy。  
`say` 让机器人可以对话。

再看一个空间几何任务：

```
Put the blocks in a vertical line 20 cm long and 10 cm below the blue bowl.
```

可能生成：

```python
block_names = parse_obj('blocks')
blue_bowl_pos = get_obj_pos('blue bowl')

n = len(block_names)
line_center = blue_bowl_pos + np.array([0.0, -0.10])

y_positions = np.linspace(
    line_center[1] - 0.10,
    line_center[1] + 0.10,
    n
)

for block_name, y in zip(block_names, y_positions):
    target_pos = np.array([line_center[0], y])
    put_first_on_second(block_name, target_pos)
```

这就是 CaP 相比自然语言 planner 的优势：它真的算了坐标。

---

## 12\. 一句话总结

这篇论文的主张是：

> **把 LLM 从“说出机器人应该做什么”的 planner，提升为“写出机器人如何做”的 policy-code generator。**

它用 Python 作为自然语言和机器人控制之间的中间层，通过 perception API grounding，通过 NumPy/Shapely 做几何，通过 control primitives 执行，通过 hierarchical code generation 处理复杂函数。它不是万能机器人智能，但它非常清楚地展示了一个重要范式： **未来机器人系统可以由 LLM 生成可解释、可组合、可验证的程序化策略，而不是只输出自然语言计划。**