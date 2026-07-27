---
title: Voyager
source: obsidian://open?vault=Self-Evolving%20Robot&file=Source%2FPapers%2Fvoyager-2305.16291.pdf
author:
published:
created: 2026-06-28
description: ChatGPT is your AI chatbot for everyday use. Chat with the most advanced AI to explore ideas, solve problems, and learn faster.
tags:
  - clippings
---
你是我的Phd Advisor，请你和我尽可能详细地讲一下这篇文章，深入技术细节。Section by Section，并结合具体的例子讲解。

下面我会按 **PhD advisor 带你读论文** 的方式讲 VOYAGER 这篇文章：不仅讲它做了什么，还会讲为什么这么设计、每个模块的技术细节、和相关方法的差异，以及如果你要基于它做研究，可以从哪里切入。论文全文是你上传的 *VOYAGER: An Open-Ended Embodied Agent with Large Language Models* 。

## 0\. 先给你一个总览：这篇文章到底解决什么问题？

VOYAGER 想解决的是一个非常核心的 embodied AI 问题：

> 如何让一个智能体在开放世界中，长期自主探索、不断提出新目标、学习新技能，并把学到的技能迁移到新环境？

它选择的实验环境是 **Minecraft** 。Minecraft 很适合做 open-ended embodied agent，因为它不像 Atari 或很多 benchmark 那样有固定目标，而是有开放世界、采集、合成、战斗、探索、科技树、地形变化等复杂机制。

VOYAGER 的关键思想是：

> 不直接训练一个神经网络 policy，而是让 GPT-4 作为高层智能体，用代码生成 Minecraft 行为程序，并把成功的程序存进技能库，未来复用。

所以它不是传统 RL，也不是 imitation learning，而是一种 **LLM-driven, code-as-action, lifelong learning agent** 。

论文宣称 VOYAGER 是第一个在 Minecraft 中由 LLM 驱动的 embodied lifelong learning agent，可以在没有人工干预的情况下持续探索、学习技能、发现新物品。它有三个核心模块：

1. **Automatic Curriculum** ：自动课程，决定下一步应该做什么任务。
2. **Skill Library** ：技能库，把成功的代码技能保存下来，以后检索复用。
3. **Iterative Prompting Mechanism** ：迭代提示机制，让 GPT-4 根据环境反馈、代码报错、自我验证不断改代码。

你可以把 VOYAGER 理解成一个循环系统：

> 看当前状态 → GPT-4 提出一个合适任务 → GPT-4 写代码尝试完成 → 环境执行代码 → 根据反馈修代码 → 成功后存成技能 → 再提出下一个任务。

这篇文章最重要的不是“让 GPT-4 玩 Minecraft”，而是提出了一个早期版本的 **LLM agent lifelong learning architecture** 。

---

## 1\. Abstract：摘要在说什么？

摘要里作者直接给出 VOYAGER 的定位和三个模块。

他们说 VOYAGER 是：

> the first LLM-powered embodied lifelong learning agent in Minecraft

重点有三个：

第一，它是 **embodied agent** 。不是纯文本 agent，而是在 Minecraft 这种 3D 环境中行动。

第二，它是 **lifelong learning agent** 。它不是只做一个指定任务，而是不断学习、积累、迁移技能。

第三，它不需要 human intervention。没有人手动指定每一步任务，也没有人手动写 reward。

摘要里的实验结果也很重要：

VOYAGER 相比 baseline：

- 获得 **3.3× 更多 unique items** 。
- 行走距离 **2.3× 更长** 。
- 解锁 Minecraft 关键 tech tree milestone 最快可达 **15.3×** 。
- 在新世界里可以利用已有 skill library 解决 novel tasks。

这里“unique items”是他们的主要 open-ended exploration 指标。Minecraft 中你获得的物品种类越多，说明你探索范围越广、技能越多样。

摘要中还强调了一个非常重要的设计点：

> VOYAGER interacts with GPT-4 via blackbox queries, which bypasses the need for model parameter fine-tuning.

也就是说，它不微调 GPT-4，不访问模型参数，而是靠 prompting 和 in-context learning。这一点在 2023 年非常关键，因为当时 closed-source LLM 很强，但不能被 fine-tune 或内部修改。

---

## 2\. Figure 1：为什么这张图很重要？

第一页的 Figure 1 是整篇文章的主结果图。

横轴是 prompting iterations，也就是 GPT-4 被调用、尝试生成或更新程序的迭代次数。纵轴是发现的 distinct items 数量。

不同曲线代表：

- VOYAGER
- VOYAGER without skill library
- AutoGPT
- Reflexion
- ReAct

你会看到 VOYAGER 的曲线持续上升，到 160 次 iteration 左右发现了 60 多种物品。其他方法很快 plateau，尤其 ReAct 和 Reflexion 基本没怎么增长。

这张图说明了一个核心结论：

> 在 open-ended embodied environment 中，仅靠“LLM 推理 + 行动循环”是不够的。你需要自动课程来选择合适目标，也需要技能库来累积长期能力。

如果没有 skill library，VOYAGER 前期也能探索，但后期增长明显变慢。这说明技能复用对于长期探索非常重要。

---

## 3\. Introduction：引言的逻辑

## 3.1 研究背景：为什么 embodied lifelong learning 难？

作者开头说，构建能够持续探索、规划、发展技能的 embodied agent 是 AI 社区的 grand challenge。

传统方法主要有：

- reinforcement learning
- imitation learning

这些方法通常在 primitive action level 上操作，比如键盘鼠标动作、机器人关节控制、move forward、turn left、attack 等。

问题是，在 Minecraft 这种开放世界中，primitive action 很难直接解决长时程任务。

举个例子：

如果任务是 “craft an iron pickaxe”，底层动作可能包括：

1. 找树。
2. 砍树。
3. 合成木板。
4. 合成 crafting table。
5. 合成 wooden pickaxe。
6. 找石头。
7. 挖 cobblestone。
8. 合成 stone pickaxe。
9. 找 iron ore。
10. 挖 raw iron。
11. 做 furnace。
12. 找 coal。
13. smelt raw iron。
14. 合成 iron pickaxe。

如果你用 RL 直接从键盘鼠标学，reward 极其稀疏，探索空间巨大，非常困难。

而 LLM 有一个优势：它已经从互联网文本中学到了很多 Minecraft 常识。比如 GPT-4 大概知道 iron pickaxe 需要 3 iron ingots 和 2 sticks，也知道 raw iron 需要 furnace smelt。

所以作者的问题是：

> 能否把 LLM 的世界知识和代码生成能力，转化成 embodied agent 的长期探索能力？

## 3.2 为什么选择 Minecraft？

Minecraft 有几个特点：

1. 没有固定目标。
2. 世界是 procedurally generated。
3. 有复杂 tech tree。
4. 有多种行为：采集、合成、战斗、探索、烹饪、熔炼。
5. 有长期依赖。

比如如果 agent 在 desert 而不是 forest，它不应该强行去做 forest task，而应该根据当前环境选择 sand、cactus 等任务。这就引出 automatic curriculum 的必要性。

作者认为，一个好的 lifelong agent 应该有三种能力：

第一，能根据当前状态提出合适任务。比如看到自己在 river biome 且有 fishing rod，就可以提出 “catch fish”。

第二，能根据环境反馈改进技能，并把掌握的技能存起来。比如学会 kill zombie，以后遇到 spider 时可以复用 combat 类似逻辑。

第三，能自驱探索，不断寻找新任务，而不是只完成固定 benchmark。

这三个能力正好对应 VOYAGER 的三个模块。

---

## 4\. VOYAGER 总体架构：Figure 2 怎么读？

Figure 2 是整篇文章的架构图。它把 VOYAGER 分成三个大模块：

## 4.1 Automatic Curriculum

这个模块负责提出任务。

输入包括：

- 当前 inventory
- 当前 equipment
- nearby blocks
- nearby entities
- biome
- time
- health / hunger
- position
- completed tasks
- failed tasks

输出是下一步 task。

比如：

> Craft 1 stone pickaxe  
> Kill 1 zombie  
> Smelt 4 raw iron  
> Catch 1 fish

它不是随机出题，而是让 GPT-4 根据当前状态、历史成功/失败任务，提出一个既新颖又可完成的任务。

## 4.2 Skill Library

每当 agent 成功完成一个任务，GPT-4 生成的代码就会存入技能库。

比如成功完成 “Craft stone sword” 后，系统可能保存一个函数：

```markdown
async function craftStoneSword(bot) {
  ...
}
```

以后如果任务是 “Kill zombie”，检索系统可能会取出：

- craftStoneSword
- equipShield
- killMob
- cookFood

让 GPT-4 在新代码里调用这些已有技能。

## 4.3 Iterative Prompting Mechanism

GPT-4 一次写出的代码往往会错。错误可能有几类：

- 代码语法错。
- 调用了不存在的 API。
- 使用了 Minecraft 中不存在的物品。
- 忘记先准备材料。
- 任务完成了但没被正确判断。
- 走不到目标位置。
- 找不到 block 或 mob。

所以 VOYAGER 不做 one-shot generation，而是：

1. GPT-4 写代码。
2. 执行代码。
3. 收集 environment feedback 和 execution error。
4. 用另一个 GPT-4 critic 判断任务是否成功。
5. 如果失败，把 critique 加回 prompt，让 GPT-4 改代码。
6. 最多尝试 4 轮。
7. 成功则保存技能，失败则记录 failed task，换下一个任务。

这就是整篇文章的主循环。

---

## 5\. Method Section 2：方法部分深入讲解

方法部分是这篇文章最重要的部分。作者把 VOYAGER 拆成 2.1、2.2、2.3 三节。

---

## 5.1 Section 2.1 Automatic Curriculum：自动课程

## 5.1.1 为什么需要 automatic curriculum？

在开放世界中，如果没有课程，agent 很容易做两种坏事：

第一，提出太难的任务。

比如刚出生，什么都没有，就让它 “mine diamond”。这肯定失败，因为 diamond 至少需要 iron pickaxe，而且还要地下探索。

第二，提出重复或无意义的任务。

比如一直砍木头，虽然能成功，但探索不到新技能。

Automatic curriculum 的目标是平衡：

> task novelty 和 task feasibility。

也就是说，任务既要新颖，又不能太难。

这有点像人类学习 Minecraft：

- 第一步：get wood。
- 第二步：craft planks。
- 第三步：craft crafting table。
- 第四步：craft wooden pickaxe。
- 第五步：mine stone。
- 第六步：craft stone pickaxe。
- 第七步：mine iron。
- 第八步：smelt iron。
- 第九步：craft iron pickaxe。
- 第十步：mine diamond。

但 VOYAGER 的不同之处是：这个 curriculum 不是人工写死的，而是 GPT-4 根据当前状态动态提出。

## 5.1.2 输入 prompt 包含什么？

论文中说 automatic curriculum 的 GPT-4 prompt 包含四类信息。

第一类是 directive，也就是高层指令。

例如：

> My ultimate goal is to discover as many diverse things as possible.

以及：

> The next task should not be too hard since I may not have the necessary resources or have learned enough skills to complete it yet.

这些 directive 会引导 GPT-4 不要只做重复任务，也不要提出不现实任务。

第二类是 agent 当前状态。

包括：

- inventory
- equipment
- nearby blocks
- nearby entities
- biome
- time
- health
- hunger
- position

这些信息决定任务是否可行。

比如：

如果 inventory 里有：

```markdown
{'oak_planks': 3, 'stick': 4, 'crafting_table': 1, 'stone': 3, 'wooden_pickaxe': 1}
```

GPT-4 可以推理：

你已经有 stone、stick、crafting table，因此下一步适合 craft stone pickaxe。

第三类是 completed / failed tasks。

这很重要，因为它刻画了 agent 当前的 competence frontier。

比如：

- completed: Mine wood log, Craft crafting table, Craft wooden pickaxe
- failed: Mine diamond

那么 GPT-4 应该知道：diamond 暂时太难，可以先做 stone / iron 阶段。

第四类是 additional context。

这里作者用了 GPT-3.5 做 self-ask / self-answer。也就是说，在 GPT-4 提出任务之前，GPT-3.5 会根据当前状态问一些 Minecraft 相关问题，并从 wiki knowledge base 或自身知识中回答。

比如：

Question: What can I do after crafting a wooden pickaxe?  
Answer: You can mine stone and craft stone tools.

这种额外上下文可以帮助 curriculum agent 做更合理的任务选择。

## 5.1.3 Figure 3 的例子怎么理解？

Figure 3 给了几个 automatic curriculum 输出任务的例子。

例子 1：

当前 inventory：

```markdown
oak_planks: 3
stick: 4
crafting_table: 1
stone: 3
wooden_pickaxe: 1
```

GPT-4 reasoning：

> Since you have a wooden pickaxe and some stones, it would be beneficial to upgrade your pickaxe to a stone pickaxe.

Task：

> Craft 1 stone pickaxe.

这个例子说明 GPT-4 不是简单模板匹配，而是在做 affordance reasoning：你现在有什么资源？能升级什么工具？

例子 2：

当前 biome 是 river，inventory 里有 fishing rod。

Reasoning：

> Since you have a fishing rod and are near a river biome, catch fish for food and experience.

Task：

> Catch 1 fish.

这里任务由环境条件触发：river + fishing rod。

例子 3：

hunger = 0，nearby entities 有 pig。

Task：

> Kill 1 pig.

这里是 survival need 触发任务：饥饿 → 找食物 → 杀猪。

例子 4：

inventory 有 raw iron、coal、furnace。

Task：

> Smelt 4 raw iron.

这里是 tech-tree progression：已经有 raw iron 和 furnace，就应该 smelt iron ingots。

例子 5：

night，nearby zombie，equipment 有 stone sword 和 shield。

Task：

> Kill 1 zombie.

这里是 combat opportunity：有武器和盾，晚上附近有僵尸，可以学战斗技能。

## 5.1.4 这个模块本质上是什么？

从研究角度讲，automatic curriculum 是一种 **LLM-based open-ended goal generation** 。

它有点像 novelty search，但不是通过显式 novelty metric，而是通过 prompt 让 GPT-4 根据历史和状态提出新目标。

作者自己也说，这可以看作一种 in-context novelty search。

传统 novelty search 会定义行为描述符，比如位置、物品、状态分布，然后搜索 novel behavior。而 VOYAGER 把 novelty 判断交给 GPT-4 的常识推理。

这既是优点，也是隐患。

优点：

- 不需要手写 reward。
- 可以利用 Minecraft commonsense。
- 可以动态适应环境。

缺点：

- GPT-4 可能 hallucinate。
- novelty 没有严格定义。
- curriculum 的 optimality 不可证明。
- prompt design 影响很大。

---

## 5.2 Section 2.2 Skill Library：技能库

Skill Library 是 VOYAGER 论文中最有研究价值的模块之一。

## 5.2.1 为什么需要 skill library？

如果没有 skill library，agent 每次都要从头写代码。这样有几个问题：

第一，效率低。

比如每次要 craft iron pickaxe，都重新写如何砍树、合成木板、合成 crafting table、挖石头、挖铁、熔炼。

第二，容易遗忘。

传统 continual learning 里的 catastrophic forgetting 是模型参数更新时旧知识被覆盖。VOYAGER 没有微调参数，但如果没有外部记忆，它也会“上下文遗忘”：prompt 放不下所有过往经验。

第三，难以组合复杂行为。

复杂任务需要由简单技能组合而成。比如 craft diamond pickaxe 依赖：

- mineWoodLog
- craftWoodenPlanks
- craftStick
- craftCraftingTable
- craftWoodenPickaxe
- mineCobblestone
- craftStonePickaxe
- mineIronOre
- smeltIronIngot
- craftIronPickaxe
- mineDiamond
- craftDiamondPickaxe

所以 skill library 是 compositionality 的基础。

## 5.2.2 skill 用什么表示？

VOYAGER 把 skill 表示成 **executable JavaScript code** 。

不是自然语言总结，也不是 neural embedding policy，而是真正可以被 Mineflayer 执行的程序。

例如一个技能可能是：

```markdown
async function craftWoodenPlanks(bot) {
  const logNames = ["oak_log", "birch_log", ...];
  const plankNames = ["oak_planks", "birch_planks", ...];

  const logInInventory = logNames.find(
    logName => bot.inventory.count(mcData.itemsByName[logName].id) > 0
  );

  if (!logInInventory) {
    bot.chat("No wooden log in inventory. Mining a wooden log...");
    await mineWoodLog(bot);
  }

  const logIndex = logNames.indexOf(logInInventory);
  const plankName = plankNames[logIndex];

  bot.chat(\`Crafting 4 ${plankName}...\`);
  await craftItem(bot, plankName, 1);
}
```

这个代码有几个特征：

1. 它是 temporally extended action。
2. 它有内部条件判断。
3. 它可以调用已有技能。
4. 它可以被其他函数复用。
5. 它比低层动作更可解释。

这就是 “code as action space” 的核心。

## 5.2.3 skill 如何加入库？

当一个任务通过 self-verification 被判定成功后，系统会把对应代码加入 skill library。

但存储时不是只存代码。它会先让 GPT-3.5 或 GPT-4 给函数生成一个自然语言 description。

例如对于 `mineCobblestone(bot)` ，description 可能是：

> The function is about mining 8 cobblestones using a wooden pickaxe. It first checks if a wooden pickaxe exists, crafts one if needed, equips it, explores until finding stone, and mines cobblestone.

然后用 embedding model 把 description 转成向量，作为 key。代码本身作为 value。

所以 skill library 是一个 vector database：

```markdown
key: embedding(description)
value: executable function code
```

## 5.2.4 skill 如何检索？

面对新任务时，VOYAGER 不会把整个技能库都塞进 prompt，因为 token 太多。它会做 retrieval。

流程是：

1. 当前 task，例如 “Craft iron pickaxe”。
2. GPT-3.5 生成一个 general suggestion，例如 iron pickaxe 需要 3 iron ingots 和 2 sticks。
3. 把 task suggestion + environment feedback 作为 query context。
4. 对 query context 做 embedding。
5. 在技能库里找 top-5 nearest skills。
6. 把这 top-5 skills 放进 GPT-4 prompt，让 GPT-4 写新代码。

Figure 4 展示了这个过程。

比如任务是：

> Craft iron pickaxe

retrieved skills 可能是：

- Smelt Iron Ingot
- Craft Stick
- Make Crafting Table
- Make Furnace
- Craft Wooden Pickaxe

这些技能刚好构成 craft iron pickaxe 的前置步骤。

## 5.2.5 为什么代码技能比文本记忆强？

这是一个很重要的研究点。

很多 LLM agent 也有 memory，但 memory 往往只是自然语言，比如：

> I learned that to craft a stone pickaxe, I need cobblestone and sticks.

这种 memory 不能直接执行。VOYAGER 的 memory 是可执行的。

这带来几个优势：

第一，复用成本低。GPT-4 可以直接调用：

```markdown
await craftStonePickaxe(bot);
```

第二，组合性强。复杂函数可以调用多个简单函数。

第三，可解释性强。人可以读代码，debug 代码。

第四，不依赖参数更新。不会因为训练新任务而破坏旧技能。

第五，更像程序归纳。和 DreamCoder 这类 program learning 方法有相似思想：构建一个越来越强的 library。

## 5.2.6 但 skill library 有什么风险？

有几个明显问题。

第一，错误技能污染。

如果 self-verification 错误地判断成功，一个有 bug 的 skill 会进入 library，以后被反复复用，造成 cascading failure。

第二，retrieval granularity 问题。

技能太细会导致组合困难；技能太粗会导致不通用。

比如：

```markdown
craftOakPlanksOnly()
```

就比：

```markdown
craftWoodenPlanks()
```

泛化差，因为它只适用于 oak log。

第三，技能版本管理问题。

论文附录里有 `smeltFiveRawIronV2` ，说明技能可能有多个版本。系统如何替换旧技能、合并技能、删除无用技能，是后续研究问题。

第四，技能库依赖 prompt context。

即使检索到了正确技能，GPT-4 也可能不会正确使用。skill retrieval 不是等于 skill execution planning。

---

## 5.3 Section 2.3 Iterative Prompting Mechanism：迭代提示机制

这个模块解决一个现实问题：

> LLM 一次生成的代码经常不能直接完成 embodied task。

VOYAGER 的做法是让 GPT-4 在执行反馈中迭代改进。

## 5.3.1 三种反馈

论文定义了三类反馈。

### 第一类：Environment Feedback

环境反馈来自 Minecraft 执行过程，通常通过 `bot.chat()` 生成。

例如：

```markdown
I cannot make stick because I need: 2 more planks
```

这个反馈告诉 GPT-4：

失败原因不是语法错，而是缺少材料。

于是下一轮 GPT-4 应该先 craft planks，再 craft sticks。

Figure 5 左边就是这个例子。

### 第二类：Execution Errors

execution error 是代码解释器报错。

例如：

```markdown
No item named acacia_axe
at line 18: await craftItem(bot, "acacia_axe", 1);
```

这说明 GPT-4 hallucinate 了一个不存在的 Minecraft item。Minecraft 有 wooden\_axe、stone\_axe、iron\_axe，但没有 acacia\_axe。

下一轮 GPT-4 就应该改成 craft wooden\_axe 或其他真实 item。

Figure 5 右边就是这个例子。

### 第三类：Self-Verification

self-verification 是另一个 GPT-4 agent，作用类似 critic。

它输入：

- 当前 agent state
- task
- task context

输出 JSON：

```markdown
{
  "reasoning": "...",
  "success": true,
  "critique": ""
}
```

如果失败，则给 critique：

```markdown
{
  "reasoning": "You have 2 white_wool and 6 mutton, which indicates you killed 2 sheep. You needed to kill 3 sheep.",
  "success": false,
  "critique": "Find and kill one more sheep to complete the task."
}
```

Figure 6 给了几个例子。

比如任务是：

> Kill 1 zombie

如果 inventory 里有 rotten\_flesh，critic 判断成功，因为 rotten\_flesh 通常是 zombie 掉落物。

这其实是一种 **state-based success inference** ，不是直接视觉判断。

## 5.3.2 为什么 self-verification 很关键？

没有 self-verification，agent 不知道什么时候该停止当前任务、什么时候该进入下一个任务。

这在 open-ended setting 里非常重要。

例如任务是：

> Mine 5 coal ores

代码执行完没有报错，但 inventory 只有 3 coal。没有 self-verification 的话，系统可能以为成功，然后存入一个不完整 skill。

反过来，如果任务已经完成，但系统不知道成功，也可能继续重复无意义尝试。

因此 self-verification 决定了 agent 的学习边界：

> 哪些代码能进入 skill library，哪些任务被记为 completed，哪些任务被记为 failed。

这也是为什么 ablation 中去掉 self-verification 后性能下降非常明显。

## 5.3.3 迭代流程

每个任务最多尝试 4 轮代码生成。

流程是：

1. retrieve relevant skills。
2. GPT-4 generate code。
3. environment executes code。
4. get environment feedback + execution errors。
5. critic checks success。
6. 如果成功，break。
7. 如果失败，下一轮 prompt 包含上一次代码、反馈、错误、critique。
8. 如果 4 轮都失败，就记录 failed task，curriculum 换新任务。

这个机制很像程序合成中的 REPL loop：

> write → execute → observe → revise。

但 VOYAGER 的 novelty 是把这个机制用于 embodied control，并且结合环境反馈、execution error、LLM critic 三种信号。

## 5.3.4 一个完整例子：Craft Stone Shovel

假设任务是：

> Craft 1 stone shovel.

第一轮 GPT-4 可能写：

```markdown
async function craftStoneShovel(bot) {
  await craftItem(bot, "stick", 1);
  await craftItem(bot, "stone_shovel", 1);
}
```

执行反馈：

```markdown
I cannot make stick because I need: 2 more planks
I cannot make stone_shovel because I need: 2 more stick
```

第二轮 GPT-4 应该修正：

1. 检查是否有 logs。
2. 如果没有 log，mineWoodLog。
3. craft planks。
4. craft sticks。
5. 确保有 cobblestone。
6. 如果没有 cobblestone，mine stone。
7. craft stone shovel。

修正代码可能变成：

```markdown
async function craftStoneShovel(bot) {
  if (bot.inventory.count(mcData.itemsByName["stick"].id) < 2) {
    const plankCount =
      bot.inventory.count(mcData.itemsByName["oak_planks"].id) +
      bot.inventory.count(mcData.itemsByName["spruce_planks"].id);

    if (plankCount < 2) {
      await mineWoodLog(bot);
      await craftWoodenPlanks(bot);
    }

    await craftItem(bot, "stick", 1);
  }

  const cobbleCount = bot.inventory.count(mcData.itemsByName["cobblestone"].id);
  if (cobbleCount < 1) {
    await mineBlock(bot, "stone", 1);
  }

  await craftItem(bot, "stone_shovel", 1);
}
```

这就是 iterative prompting 的意义：LLM 不一定一次成功，但可以通过环境反馈逐步修正。

---

## 6\. Algorithm Appendix A.1：VOYAGER 的伪代码

附录 A.1 给了 VOYAGER 的核心伪代码，可以把整篇文章压缩成下面这个循环：

```markdown
agent_state = environment.reset()

while True:
    exploration_progress = curriculum_agent.get_exploration_progress(
        completed_tasks,
        failed_tasks
    )

    task = curriculum_agent.propose_next_task(
        agent_state,
        exploration_progress
    )

    code = None
    environment_feedback = None
    execution_errors = None
    critique = None
    success = False

    for i in range(4):
        skills = skill_manager.retrieve_skills(task, environment_feedback)

        code = action_agent.generate_code(
            task,
            code,
            environment_feedback,
            execution_errors,
            critique,
            skills
        )

        agent_state, environment_feedback, execution_errors = environment.step(code)

        success, critique = critic_agent.check_task_success(task, agent_state)

        if success:
            break

    if success:
        skill_manager.add_skill(code)
        curriculum_agent.add_completed_task(task)
    else:
        curriculum_agent.add_failed_task(task)
```

这个伪代码非常重要，因为它揭示了 VOYAGER 其实是一个 multi-agent LLM system：

- curriculum\_agent：负责目标生成。
- action\_agent：负责代码生成。
- critic\_agent：负责成功判断和 critique。
- skill\_manager：负责技能检索和存储。
- environment：负责执行代码并返回状态。

你可以把它看成一个没有梯度更新的 lifelong learning loop。

它的“学习”发生在两个地方：

1. completed / failed tasks 进入 curriculum context。
2. successful code 进入 skill library。

也就是说，VOYAGER 的 lifelong learning 不是模型参数学习，而是 **external memory accumulation + in-context reuse** 。

---

## 7\. Section 3 Experiments：实验怎么设计？

实验部分主要回答四个问题：

1. VOYAGER 是否比其他 LLM agent 探索得更好？
2. 它是否能更快解锁 Minecraft tech tree？
3. 它是否探索更大地图范围？
4. 它学到的 skill library 能否迁移到新世界、新任务？

---

## 7.1 Experimental Setup

论文中使用：

- GPT-4-0314 做主要 text completion。
- GPT-3.5-turbo-0301 做一些辅助 NLP 任务，比如 additional context。
- text-embedding-ada-002 做 skill description embedding。
- MineDojo 作为 Minecraft AI framework。
- Mineflayer JavaScript API 作为 motor control API。

所有 temperature 设为 0，除了 automatic curriculum 设为 0.1，用来增加任务多样性。

这个设置说明作者希望大部分模块 deterministic，只有 curriculum 稍微随机一点，避免任务完全固定。

他们使用 Mineflayer 的高级 API，而不是直接从屏幕像素到键鼠控制。作者也明确说，他们不和 VPT、MineRL 这类 pixel-to-action 方法做直接 apples-to-apples 比较，因为 VOYAGER 关注的是高层规划和 lifelong learning，而不是视觉感知和低层控制。

这点你要注意：VOYAGER 的结果很强，但它不是端到端 embodied AI。它依赖高层 symbolic / API interface。

---

## 7.2 Baselines：比较对象

作者选了三个 LLM agent baseline：

## 7.2.1 ReAct

ReAct 是 reasoning + acting 的 LLM agent 框架。

它会生成 reasoning traces 和 action plans。

在 VOYAGER 实验中，作者给 ReAct 环境反馈和 agent state 作为 observation。

但 ReAct 没有 automatic curriculum，也没有 skill library。

所以在 open-ended Minecraft 中，它很难决定长期合理目标。

## 7.2.2 Reflexion

Reflexion 是在 ReAct 上加 self-reflection。

失败后，agent 会总结经验，用于后续尝试。

作者也给它 execution errors 和 self-verification。

但 Reflexion 的 memory 主要是自然语言反思，不是可执行技能库。

所以它仍然很难积累复杂行为。

## 7.2.3 AutoGPT

AutoGPT 会把高层目标分解成多个 subgoals，然后用 ReAct-style loop 执行。

在这里，高层目标是：

> explore the world and get as many items as possible.

AutoGPT 比 ReAct / Reflexion 更强，因为它会做任务拆解。

但相比 VOYAGER，它缺少：

- automatic curriculum
- skill library
- self-verification

因此它的探索容易不稳定，也难以长期积累技能。

---

## 8\. Section 3.3 Evaluation Results：主实验结果

## 8.1 Exploration：VOYAGER 发现更多 unique items

主结果是：

> VOYAGER 在 160 prompting iterations 内发现 63 个 unique items，是 baseline 的 3.3×。

这个结果说明 automatic curriculum + skill library + iterative prompting 在 open-ended exploration 中是有效的。

更重要的是曲线形态：

- VOYAGER 的 unique item count 持续增长。
- VOYAGER w/o skill library 后期 plateau。
- AutoGPT 增长慢。
- ReAct / Reflexion 基本无法有效探索。

这说明 open-ended exploration 需要一个能长期扩展 capability frontier 的机制。

从技术角度讲，VOYAGER 的探索不是随机走地图，而是“任务驱动”的探索。

比如：

- 当前在 river，有 fishing rod → catch fish → 获得 fish。
- 当前有 raw iron 和 furnace → smelt iron → 获得 iron ingot。
- 当前有 iron ingot → craft shield / iron tools。
- 当前有 iron pickaxe → mine diamond。
- 当前看到 cactus → mine cactus → craft green dye。

每个小任务带来新物品，新物品又触发新任务。

这就是自动课程的 cascading effect。

## 8.2 Tech Tree Mastery：解锁科技树

Table 1 比较了不同方法解锁 Minecraft 工具科技树的效率。

科技树顺序：

```markdown
Wooden Tool → Stone Tool → Iron Tool → Diamond Tool
```

结果大致是：

- ReAct 和 Reflexion：全部失败。
- AutoGPT：可以到 iron tool，但很慢，diamond 失败。
- VOYAGER w/o skill library：wood / stone / iron 都能解锁，但 diamond 失败。
- VOYAGER：wood、stone、iron 稳定解锁，并且有一次解锁 diamond tool。

VOYAGER 解锁：

- wooden tool：约 6 iterations。
- stone tool：约 11 iterations。
- iron tool：约 21 iterations。
- diamond tool：102 iterations，成功 1/3。

这里 diamond 工具只成功 1/3，说明即便 VOYAGER 很强，diamond 仍然是困难任务。难点可能包括：

- 需要地下探索。
- 需要 iron pickaxe。
- 需要找到 diamond ore。
- 地形和路径复杂。
- agent 可能卡住、死亡、找不到矿。

但它是唯一能做到 diamond level 的方法，这说明它的技能组合能力更强。

## 8.3 Map Traversal：探索地图范围

Figure 7 显示了地图鸟瞰图。VOYAGER 的轨迹覆盖范围最大。

论文说 VOYAGER 行走距离是 baseline 的 2.3×。

这和 automatic curriculum 有关。因为 VOYAGER 经常被鼓励寻找新资源、新 biome、新 mob，所以它不会一直停留在出生点附近。

而 baseline 可能陷入局部：

- 不断尝试类似任务。
- 不知道该去哪里找资源。
- 失败后没有有效 skill accumulation。
- 代码生成不稳定。

从 embodied AI 角度，这说明 high-level task generation 会影响 exploration distribution。

不是 agent 走得远就好，而是任务驱动它去多样地形：

- meadow
- desert
- river
- savanna
- forest
- bamboo\_jungle
- dripstone\_caves
- snowy\_taiga
- ocean
- frozen\_peaks

这些 biome 带来不同资源，进一步促进 unique item discovery。

## 8.4 Zero-shot Generalization：迁移到新世界的新任务

这是非常重要的实验。

设置是：

1. 清空 agent inventory。
2. 重置到一个新的 Minecraft world。
3. 给它 unseen tasks。
4. 看是否能利用之前学到的 skill library 从零完成。

测试任务包括：

- Diamond Pickaxe
- Golden Sword
- Lava Bucket
- Compass

结果：

VOYAGER 在所有任务上 3/3 成功，并且 iterations 最少。

VOYAGER w/o skill library 也能完成一些任务，但更慢。

AutoGPT 原版全部失败。

AutoGPT + VOYAGER skill library 性能变好，说明 skill library 是 plug-and-play asset，不只 VOYAGER 自己能用。

这个实验支持一个重要观点：

> VOYAGER 学到的不是一次性 trajectory，而是可迁移的 procedural knowledge。

比如 lava bucket 任务需要：

1. craft bucket。
2. 找 lava source。
3. equip bucket。
4. look at lava。
5. activate item。

如果 skill library 里已经有 fillBucketWithWater，那么 GPT-4 可以类比出 fillBucketWithLava。

或者如果已有 craftBucket、mineIron、smeltIron 等技能，就能组合完成。

---

## 9\. Section 3.4 Ablation Studies：消融实验

消融实验回答：

> VOYAGER 的性能到底来自哪些模块？

作者消融了六个设计：

- automatic curriculum
- skill library
- environment feedback
- execution errors
- self-verification
- GPT-4 code generation

## 9.1 去掉 automatic curriculum

如果用 random curriculum，discovered item count 下降 93%。

原因很直观：

随机任务可能完全不符合当前能力。

例如刚出生就随机到：

- craft diamond sword
- collect ender pearl
- craft compass
- mine redstone

这些都需要前置资源。agent 会大量失败，浪费 iterations。

手工 curriculum 也不如 automatic curriculum，因为手工课程通常只覆盖单一路径，比如挖钻石路径，不能根据 live environment 调整。

比如如果 agent 旁边有 sugar cane，automatic curriculum 可以提出 collect sugar cane / craft paper；手工 mining-diamond curriculum 不会利用这个机会。

这说明 open-ended exploration 中，curriculum 必须既考虑长期目标，也考虑局部机会。

## 9.2 去掉 skill library

没有 skill library，VOYAGER 前期还能增长，但后期 plateau。

原因是复杂任务需要复用已学技能。没有技能库，每个任务都要从头写，错误率和 token 成本都高。

例如 “craft compass” 需要：

- mine redstone
- mine iron ore
- smelt iron
- craft compass

如果系统已经有 mineIronOre、smeltIron、craftIronIngot 等技能，组合就简单。如果没有，GPT-4 要从头生成完整代码，很容易遗漏步骤。

## 9.3 去掉 environment feedback

没有环境反馈，GPT-4 不知道 Minecraft 执行过程中发生了什么。

比如 craftItem 失败是因为：

- 缺少材料？
- 没有 crafting table？
- 距离 crafting table 太远？
- recipe 不存在？
- inventory full？

没有 chat log，GPT-4 只能根据最终状态猜。

## 9.4 去掉 execution errors

没有代码错误信息，GPT-4 无法准确 debug。

例如：

```markdown
No item named acacia_axe
```

如果不告诉它这个 error，它可能反复生成错误 item name。

execution error 是程序合成中的强监督信号。

## 9.5 去掉 self-verification

这是最关键的反馈类型。

论文说 removing self-verification 会导致 discovered item count 下降 73%。

原因是 self-verification 决定任务完成状态和技能是否入库。

没有它，系统可能出现两类错误：

第一，false positive：

代码没完成任务，但系统以为完成，把坏技能存入库。

第二，false negative / no stopping：

任务完成了，但系统继续尝试，浪费 iterations。

Self-verification 是 VOYAGER 的“学习信号”。虽然它不是 reward function，但功能上类似 reward evaluator / critic。

## 9.6 GPT-4 vs GPT-3.5

用 GPT-3.5 替换 GPT-4 做代码生成后，VOYAGER 获得 unique items 数量明显下降。论文说 GPT-4 获得 5.7× 更多 unique items。

这说明系统强依赖 GPT-4 的代码生成能力。

这也是 VOYAGER 的一个现实限制：architecture 很聪明，但如果底层 LLM 不够强，效果会显著下降。

---

## 10\. Section 3.5 Multimodal Feedback from Humans：人类多模态反馈

VOYAGER 主要使用文本状态，不使用视觉输入。因为当时 GPT-4 API 是 text-only。

但 Minecraft 有些任务需要视觉或空间判断，比如：

- build a Nether Portal
- build a house
- place blocks in a specific shape

这些任务只看 inventory 很难判断成功。

所以作者展示了一个扩展：让人类提供视觉反馈。

他们说人类可以扮演两种角色：

## 10.1 Human as Critic

相当于替代 self-verification。

人类看 Minecraft 画面，然后告诉 VOYAGER：

- portal shape 不对。
- obsidian 放错位置。
- house wall 缺一块。
- roof 太低。
- 门没装好。

VOYAGER 根据 critique 修改代码。

## 10.2 Human as Curriculum

人类把复杂建筑任务拆成小步骤。

比如 build house：

1. Clear ground.
2. Build floor.
3. Build four walls.
4. Add roof.
5. Add door.
6. Add windows.
7. Add torches.

这相当于人工 curriculum。

这一节其实暴露了 VOYAGER 的一个关键短板：

> 它对空间结构和视觉状态理解不足。

因为它主要依赖 symbolic state，比如 inventory 和 nearby blocks。对于 “房子长得对不对” 这种任务，纯文本状态不够。

这为后续 multimodal embodied agents 留出了研究空间。

---

## 11\. Section 4 Limitations and Future Work：局限性

这一节很重要，因为它告诉我们 VOYAGER 还不是通用 embodied intelligence。

## 11.1 Cost

GPT-4 API 成本高。论文说 GPT-4 比 GPT-3.5 贵 15 倍。

而 VOYAGER 需要频繁调用：

- curriculum GPT-4
- action generation GPT-4
- critic GPT-4
- GPT-3.5 question answering
- embedding API

一次实验 160 iterations，成本不低。

这说明系统在实际部署中需要更便宜的模型、本地模型、distillation 或 skill reuse 减少调用。

## 11.2 Inaccuracies

即使有 iterative prompting，agent 仍然会卡住。

比如：

- 找不到特定矿物。
- 路径规划失败。
- 生成的 skill 不够 robust。
- critic 判断错误。

作者提到 self-verification 有时会失败，例如没有识别 spider string 是打败 spider 的成功信号。

这个例子很有意思：如果任务是 kill spider，inventory 里出现 string 可能说明成功，因为 spider 会掉 string。但 critic 如果不知道这个映射，就会误判失败。

## 11.3 Hallucinations

GPT-4 可能 hallucinate Minecraft 不存在的 item 或操作。

例如：

- copper sword
- copper chestplate
- acacia axe
- 用 cobblestone 当 fuel

这些都是 Minecraft 里不合法的。

这说明 LLM 的世界知识不可靠，必须通过 environment execution 纠错。

从研究角度看，这也是 tool-augmented LLM 的核心问题：

> LLM 可以提出计划，但 grounding 必须靠外部环境验证。

---

## 12\. Section 5 Related Work：相关工作怎么放？

Related Work 分三类。

## 12.1 Minecraft Decision-Making Agents

作者把 prior Minecraft agents 分成两类：

第一类是 low-level controller。

例如：

- VPT
- MineDojo
- DreamerV3
- hierarchical RL methods

它们通常处理 pixels、keyboard/mouse、RL reward、demonstrations。

第二类是 high-level planner。

例如：

- 用 LLM 分解任务。
- 根据 recipe 做规划。
- 生成 high-level actions。

VOYAGER 属于第二类，但它比之前方法多了 open-ended automatic curriculum 和 self-generated skill library。

## 12.2 LLM for Agent Planning

这里包括：

- SayCan
- Inner Monologue
- Code as Policies
- ProgPrompt
- ReAct
- Reflexion
- AutoGPT
- Generative Agents
- SPRING

VOYAGER 和 Code as Policies / ProgPrompt 很像，都使用 code as action。但 VOYAGER 的不同是 lifelong learning：

- 不只是为一个指定任务生成代码。
- 它持续探索。
- 它把成功代码存成技能库。
- 它有自动课程。

## 12.3 Code Generation with Execution

这一类包括 execution-guided program synthesis、LEVER、CLAIRIFY 等。

VOYAGER 借鉴了 “代码生成 → 执行 → 根据结果修正” 的思想。

但它的特色是 embodied setting：

- execution result 不只是 unit test，而是真实环境状态。
- error 不只是 Python exception，而是 Minecraft 交互失败。
- verifier 不是固定规则，而是 GPT-4 critic。
- 最终产物不是单个程序，而是 lifelong skill library。

---

## 13\. Section 6 Conclusion：结论

结论其实很简洁：

VOYAGER 是一个 GPT-4 驱动的 embodied lifelong learning agent，可以持续探索 Minecraft，发展技能库，并把技能迁移到新世界。

作者认为 VOYAGER 是 developing powerful generalist agents without tuning model parameters 的起点。

这句话很重要。它提出了一条不同于传统 RL / supervised fine-tuning 的路线：

> 不训练一个万能 policy，而是用强 LLM + 工具 API + 外部记忆 + 环境反馈，构建一个会持续积累技能的 agent system。

---

## 14\. Section 7 Broader Impacts：影响

作者说研究是在 Minecraft 这个安全游戏环境中进行的。

但如果迁移到真实机器人，需要额外安全约束。

这个提醒很重要。因为 VOYAGER 的行动程序是 LLM 生成的，如果直接控制物理机器人，可能出现危险行为。

例如机器人可能：

- 移动到不安全区域。
- 抓取错误物体。
- 使用工具不当。
- 忽略人类安全边界。

所以真实世界部署必须加入 safety verifier、constraint checking、human oversight、sim-to-real validation。

---

## 15\. Appendix A：方法细节

附录非常有价值，因为它给了完整 prompt 和 API。

## 15.1 A.2 Prompting

GPT-4 / GPT-3.5 有 system、user、assistant 三种 message role。

作者为了省 token，不做长对话，而是把 system prompt 和 user prompt 拼起来，每次获得一个 assistant response。

这说明 VOYAGER 虽然叫 iterative prompting，但不是 ChatGPT 式无限历史对话，而是每轮构造一个包含必要上下文的新 prompt。

这有利于控制 token，但也要求 prompt 设计非常精细。

## 15.2 A.3 Automatic Curriculum Prompt

Automatic curriculum prompt 中有很多约束，特别重要的是：

任务必须具体，格式类似：

- Mine \[quantity\] \[block\]
- Craft \[quantity\] \[item\]
- Smelt \[quantity\] \[item\]
- Kill \[quantity\] \[mob\]
- Cook \[quantity\] \[food\]
- Equip \[item\]

它要求：

- 只提出一个任务。
- 不要提出多个任务。
- 不要太难。
- 要新颖有趣。
- 可以必要时重复任务。
- 不要让 agent 建 shelter。
- 避免需要视觉确认的任务，比如 placing、building、planting、trading。

最后一点非常关键：因为 self-verification 主要看 textual state，不擅长视觉判断，所以 curriculum 要避免提出难以验证的任务。

这说明 VOYAGER 的 automatic curriculum 不是完全自由的，而是被 prompt 约束到“可验证任务空间”。

## 15.3 Warm-up Schedule

Appendix A.3.3 有一个 warm-up schedule。

它不是一开始就把所有状态都给 GPT-4，而是随着 completed tasks 数量增加，逐步加入更多信息。

例如：

一开始给：

- core inventory
- equipment
- nearby blocks
- position

完成 5 个任务后加入 nearby entities。

完成 7 个任务后加入 full inventory。

完成 10 个任务后加入 recently seen blocks 和 biome。

完成 15 个任务后加入 health、hunger、time、additional context。

为什么要 warm-up？

因为早期 agent 只需要基础信息。如果 prompt 太复杂，GPT-4 可能提出过早复杂任务。warm-up 可以让课程从基础技能自然发展到复杂技能。

这很像人类教学：新手阶段不要给太多信息，先学基本动作。

## 15.4 A.4 Skill Library Prompt

代码生成 prompt 很详细。

它告诉 GPT-4：

你是一个写 Mineflayer JavaScript code 的 assistant。

它提供了若干 primitive APIs：

- `exploreUntil`
- `mineBlock`
- `craftItem`
- `placeItem`
- `smeltItem`
- `killMob`
- `getItemFromChest`
- `depositItemIntoChest`

还有 Mineflayer 原生 API：

- `bot.pathfinder.goto`
- `GoalNear`
- `GoalXZ`
- `GoalGetToBlock`
- `GoalFollow`
- `bot.equip`
- `bot.consume`
- `bot.fish`
- `bot.activateBlock`
- `bot.lookAt`
- `bot.activateItem`
- `bot.useOn`

Prompt 还强制要求：

- 写 async function。
- 函数只接受 bot 一个参数。
- 尽可能复用已有 useful programs。
- 不要直接用底层 `bot.dig` ，要用封装好的 `mineBlock` 。
- 不要写 infinite loops。
- 不要注册 event listener。
- 函数名要有意义。
- 用 `bot.chat` 输出中间进度。
- 如果找不到东西，用 `exploreUntil` 。
- `maxDistance` 必须是 32，不要作弊。

这些 prompt 约束本质上是一个 **domain-specific programming interface contract** 。

它把 GPT-4 的代码生成空间限制在相对安全、可执行、可调试的范围内。

## 15.5 A.4.3 Skill Examples

附录给了很多技能例子。它们展示了 skill library 的风格。

比如：

### craftWoodenPlanks

这个技能不是只会 oak log，而是支持：

- oak\_log
- birch\_log
- spruce\_log
- jungle\_log
- acacia\_log
- dark\_oak\_log
- mangrove\_log

这体现了 reusable / generic skill 的设计原则。

### mineTenCobbledDeepslateBelowY0

这个技能会：

1. equip iron pickaxe。
2. explore downward。
3. 找 Y < 0 的 cobbled\_deepslate。
4. mine 10 blocks。

这个技能体现了空间条件和工具条件。

### smeltFiveRawIron

这个技能会：

1. 检查是否有 furnace。
2. 如果没有，craft furnace。
3. 找适合放 furnace 的位置。
4. place furnace。
5. smelt raw iron with coal。

这体现了技能组合：smelting 不是单步操作，需要前置检查和环境交互。

### fillBucketWithWater

这个技能会：

1. 找 water block。
2. 走到 water 附近。
3. look at water。
4. equip bucket。
5. activate item。

这很典型地说明 embodied action 不只是 symbolic recipe。你不仅要知道 bucket + water = water bucket，还要在三维环境中走到水边、看向水、使用物品。

### catchFiveFishSafely

这个技能更复杂，会：

1. 检查 fishing rod。
2. 没有就 craft。
3. 找 water block。
4. 走到水边。
5. equip fishing rod。
6. fish 5 times。
7. 如果 fishing cancelled，就 retry。

这个例子说明技能库中可以包含 robust error handling。

## 15.6 A.5 Self-Verification Prompt

self-verification prompt 要求 GPT-4 输出 JSON：

```markdown
{
  "reasoning": "...",
  "success": true,
  "critique": ""
}
```

它给了 few-shot examples。

例如：

任务：

> Mine 3 wood logs

inventory 有 2 oak\_log 和 2 spruce\_log。

critic 判断成功，因为总共有 4 个 wood logs。

任务：

> Craft a wooden pickaxe

inventory 有 crafting\_table、planks、sticks，但没有 wooden\_pickaxe。

critic 判断失败，因为虽然材料够，但没 craft 出目标物品。

任务：

> Mine 5 iron\_ore

inventory 有 raw\_iron: 5。

critic 判断成功，因为 Minecraft 中挖 iron\_ore 得到 raw\_iron。

这里可以看出 self-verifier 需要理解 item transformation：

- mine iron\_ore → raw\_iron
- kill zombie → rotten\_flesh
- kill sheep → wool / mutton
- eat food → hunger becomes 20
- smelt raw\_iron → iron\_ingot

这不是简单字符串匹配，而是基于 Minecraft mechanics 的推理。

---

## 16\. Appendix B：实验细节

## 16.1 B.1 Experimental Setup

作者还做了一些工程处理：

- 在 Mineflayer functions 中加入很多 `bot.chat()` ，提供环境反馈。
- 实现各种 condition checks 和 try-catch。
- 如果 bot 死亡，会在最近地面复活，并保留 inventory。
- 程序执行后会回收 crafting table 和 furnace。

这些细节很关键。它们让实验更稳定，但也意味着 VOYAGER 的系统包含不少人工工程 scaffold。

比如保留 inventory 复活在真实 Minecraft 里不是默认规则，这会降低死亡惩罚，使长期探索更连续。

## 16.2 B.2 Baselines Details

baseline 的任务都是：

> explore the world and get as many items as possible

ReAct 和 Reflexion 都是一轮 code generation from scratch 加三轮 refinement，然后重复。

AutoGPT 会做 subgoal decomposition。如果连续三个 subgoal 没有获得新 item，就 replan。

这说明作者尽量让 baseline 有可执行性，但 baseline 本来不是为 Minecraft embodied learning 设计的，所以比较未必完全公平。不过实验仍有参考价值，因为它展示了普通 LLM agent 框架缺少 lifelong skill accumulation 时的困难。

## 16.3 B.3 Ablation Variants

manual curriculum 被设计成 diamond mining path：

1. Mine 3 wood log
2. Craft crafting table
3. Craft wooden pickaxe
4. Mine 11 cobblestone
5. Craft stone pickaxe
6. Craft furnace
7. Mine iron ore
8. Smelt iron ore
9. Craft iron pickaxe
10. Mine diamond

这个 curriculum 对 mining diamond 很合理，但对 open-ended discovery 不够好。因为它忽略其他资源和环境机会。

random curriculum 则从 VOYAGER 获得过的 101 个 items 中随机选任务。这会产生大量不可行任务。

## 16.4 B.4.4 Skill Retrieval Accuracy

作者评估了 skill retrieval，309 个 samples。

Top-5 accuracy 是 96.5%。

这说明把 top-5 relevant skills 放进 prompt 通常足够覆盖所需技能。

但注意：retrieval accuracy 高不代表最终任务一定成功。因为后续还需要 GPT-4 正确组合技能、环境能执行、critic 能判断。

---

## 17\. 这篇文章的核心创新是什么？

我会把创新总结成四点。

## 17.1 把代码作为 embodied action space

传统 embodied agent 通常输出低层动作：

```markdown
move forward
turn left
jump
attack
```

VOYAGER 输出的是程序：

```markdown
async function craftIronPickaxe(bot) {
  await mineIronOre(bot);
  await smeltIronIngot(bot);
  await craftItem(bot, "iron_pickaxe", 1);
}
```

代码天然支持：

- temporal abstraction
- conditional branching
- loops
- composition
- reuse
- debugging
- human interpretability

这让 LLM 的代码能力转化成 embodied control 能力。

## 17.2 自动课程驱动 open-ended exploration

VOYAGER 不需要人类每一步指定任务，也不依赖固定 reward。

GPT-4 根据当前状态提出下一个任务。这让 agent 能从简单任务自然走向复杂任务。

## 17.3 可执行技能库实现 lifelong learning

VOYAGER 的“学习”不是更新神经网络权重，而是保存可执行技能。

这是一种 externalized lifelong learning：

```markdown
experience → code skill → vector memory → future retrieval → composition
```

## 17.4 环境反馈闭环提升代码生成可靠性

LLM hallucination 是不可避免的。VOYAGER 没有假设 GPT-4 一次正确，而是让它写、跑、看错、再改。

这是一种 embodied program repair loop。

---

## 18\. 和 Code as Policies 的关系

你之前也在看 Code as Policies。VOYAGER 和 Code as Policies 有明显继承关系。

Code as Policies 的思想是：

> 用 LLM 生成机器人控制代码，把代码作为 policy。

VOYAGER 延伸了这个思想：

| 维度 | Code as Policies | VOYAGER |
| --- | --- | --- |
| 目标 | 给定任务生成 policy | 自主提出任务并持续探索 |
| 学习方式 | mostly one-shot / few-shot code generation | lifelong skill accumulation |
| 记忆 | 没有核心技能库机制 | self-generated skill library |
| 环境 | robotics manipulation / tabletop | Minecraft open world |
| feedback | 较少强调长期迭代 | environment feedback + execution error + critic |
| curriculum | 无 | automatic curriculum |
| open-endedness | 较弱 | 强 |

所以你可以把 VOYAGER 看成：

> Code as Policies + Automatic Curriculum + Skill Library + Iterative Self-Improvement。

---

## 19\. 这篇文章的弱点和可批判点

作为 PhD advisor，我不会只讲优点。你需要知道它的局限，才能找到研究机会。

## 19.1 它依赖高层 API，不是真正从视觉到动作

VOYAGER 使用 Mineflayer API，可以直接查询：

- inventory
- nearby blocks
- nearby entities
- block positions
- item names

这比真实机器人或普通 Minecraft 玩家拥有更强的 structured observation。

所以它没有解决：

- visual perception
- 3D scene understanding
- low-level motor control
- partial observability from pixels

如果你要写 critique，可以说：

> VOYAGER demonstrates high-level embodied planning, but not end-to-end embodied intelligence.

## 19.2 它的 evaluation metric 偏向 item collection

unique items 是合理指标，但不完全代表 open-ended intelligence。

一个 agent 可能获得很多简单 item，但不一定真正掌握复杂策略。

比如：

- collect many flowers
- collect many block variants
- kill different animals

这些会增加 unique item count，但不一定等价于 deep skill acquisition。

当然作者也补充了 tech tree、map traversal、zero-shot generalization。

## 19.3 self-verification 不可靠

critic 是 GPT-4，本身会犯错。

例如：

- 把巧合获得的 rotten\_flesh 当成 kill zombie 成功。
- 没识别某些掉落物。
- 对复杂任务无法判断。
- 对建筑任务缺乏视觉验证。

Self-verification 是系统核心，但也是 fragile point。

## 19.4 skill library 可能积累错误

如果一个错误技能进入库，未来 retrieval 会把它带入 prompt，可能影响更多任务。

论文没有深入讨论 skill validation、skill pruning、skill consolidation。

这是很好的后续研究方向。

## 19.5 prompt engineering 很重

VOYAGER 很大程度依赖精心设计的 prompt、API wrapper、few-shot examples、Minecraft-specific constraints。

这让它在 Minecraft 中有效，但跨域迁移需要重新设计 prompt 和 primitives。

## 19.6 没有真正的探索理论

automatic curriculum 是 GPT-4 生成的，没有显式优化目标。

它像 novelty search，但没有 novelty metric、coverage guarantee 或 regret analysis。

如果你想做理论方向，可以从这里切。

---

## 20\. 如果你要复现 VOYAGER，需要实现什么？

一个最小复现系统需要这些组件：

## 20.1 Environment Wrapper

你要有一个环境接口：

```markdown
agent_state = env.reset()
agent_state, feedback, errors = env.step(code)
```

在 Minecraft 里，env.step(code) 要能：

1. 把 GPT-4 生成的 JS 函数注入 Mineflayer。
2. 执行函数。
3. 捕获 chat log。
4. 捕获 exception trace。
5. 返回 inventory、nearby blocks、nearby entities 等状态。

## 20.2 Primitive API Layer

你要封装一些安全可控 API：

```markdown
mineBlock(bot, name, count)
craftItem(bot, name, count)
smeltItem(bot, itemName, fuelName, count)
killMob(bot, mobName, timeout)
exploreUntil(bot, direction, maxTime, callback)
```

这层很重要，因为直接让 GPT-4 操作 Mineflayer 原生 API 会太复杂、太容易错。

## 20.3 Curriculum Agent

输入 state + completed/failed tasks，输出：

```markdown
Reasoning: ...
Task: ...
```

任务格式要受约束。

## 20.4 Action Agent

输入：

- task
- retrieved skills
- previous code
- feedback
- errors
- critique
- state

输出：

```markdown
Explain: ...
Plan:
1) ...
2) ...
Code:
\`\`\`javascript
async function ...
```
```markdown
## 20.5 Critic Agent

输入 state + task，输出 JSON：

\`\`\`json
{
  "reasoning": "...",
  "success": false,
  "critique": "..."
}
```

## 20.6 Skill Manager

需要：

- 代码描述生成。
- embedding。
- vector database。
- top-k retrieval。
- code insertion into prompt。

---

## 21\. 一个具体运行例子：从零开始到 iron pickaxe

我们假设 agent 刚出生，inventory 空。

## Iteration 1

State:

```markdown
Inventory: {}
Nearby blocks: oak_log, grass_block, dirt
```

Curriculum:

```markdown
Task: Mine 1 wood log
```

Action agent 写：

```markdown
async function mineWoodLog(bot) {
  await mineBlock(bot, "oak_log", 1);
}
```

执行成功，inventory 有 oak\_log。

Critic:

```markdown
{"success": true}
```

Skill library 加入 `mineWoodLog` 。

## Iteration 2

Curriculum:

```markdown
Task: Craft 4 oak planks
```

检索到 `mineWoodLog` 。

Action agent 写：

```markdown
async function craftOakPlanks(bot) {
  if (bot.inventory.count(mcData.itemsByName["oak_log"].id) < 1) {
    await mineWoodLog(bot);
  }
  await craftItem(bot, "oak_planks", 1);
}
```

成功，加入技能库。

## Iteration 3

Task:

```markdown
Craft 1 crafting table
```

Action agent 调用 craftOakPlanks，craft crafting\_table。

## Iteration 4

Task:

```markdown
Craft 1 wooden pickaxe
```

需要 planks + sticks + crafting table。

如果没有 sticks，先 craft sticks。

成功后有 wooden\_pickaxe。

## Iteration 5

Task:

```markdown
Mine 3 cobblestone
```

Action agent 检索 wooden pickaxe skill，装备 pickaxe，找 stone，挖 stone，得到 cobblestone。

## Iteration 6

Task:

```markdown
Craft 1 stone pickaxe
```

需要 cobblestone + sticks。

成功。

## Iteration 7

Task:

```markdown
Mine 3 iron ore
```

需要 stone\_pickaxe，找 iron\_ore，mine 得到 raw\_iron。

## Iteration 8

Task:

```markdown
Smelt 3 raw iron
```

需要 furnace + fuel。

如果没有 furnace，craft furnace。  
如果没有 coal，mine coal。  
然后 smelt raw\_iron → iron\_ingot。

## Iteration 9

Task:

```markdown
Craft 1 iron pickaxe
```

需要 3 iron\_ingot + 2 sticks。

成功后解锁 iron tool level。

这个过程中，每个成功函数都被存入技能库，后续 craft diamond pickaxe 就可以复用大量技能。

---

## 22\. 这篇文章对 Agent 研究的启发

VOYAGER 的架构实际上预示了后来很多 agent 系统的基本范式：

```markdown
LLM planner
+ tool/API execution
+ memory/retrieval
+ self-critique
+ iterative repair
+ environment feedback
```

它的贡献不只是 Minecraft，而是提出一种通用系统设计：

1. **LLM 不直接控制低层动作，而是生成可执行程序。**
2. **长期学习不一定靠梯度更新，也可以靠外部技能库。**
3. **开放世界探索需要动态 curriculum。**
4. **Agent 的可靠性来自执行反馈闭环，而不是一次性推理。**

---

## 23\. 如果你基于这篇文章做 PhD research，可以怎么扩展？

我建议你关注下面几个方向。

## 23.1 Better Skill Library Management

当前 skill library 比较简单：成功就加入，embedding 检索 top-5。

可以改进：

- skill deduplication
- skill versioning
- skill abstraction
- skill dependency graph
- skill correctness testing
- skill pruning
- skill composition planning

例如构建一个技能图：

```markdown
craftIronPickaxe
 ├── smeltIronIngot
 │    ├── mineIronOre
 │    ├── craftFurnace
 │    └── mineCoal
 └── craftStick
      └── craftWoodenPlanks
```

这样新任务规划可以从 graph search 开始，而不是全靠 GPT-4。

## 23.2 Grounded Verification

当前 self-verifier 是 GPT-4，可能误判。

可以引入：

- rule-based verifier
- symbolic state checker
- learned verifier
- multimodal verifier
- execution traces
- unit tests for skills

比如每个 skill 存入库之前，自动在多个随机环境中测试，看是否稳定成功。

## 23.3 Multimodal VOYAGER

VOYAGER 缺视觉。后续可以加入 VLM：

- screenshot understanding
- object detection
- spatial layout reasoning
- build structure verification
- navigation from visual map

这样可以扩展到建筑、导航、避障、复杂空间任务。

## 23.4 From Minecraft to Robotics

把 VOYAGER 移植到机器人，需要替换：

```markdown
Mineflayer API → robot skill API
Minecraft state → perception/state estimator
Minecraft critic → safety-aware task verifier
```

例如厨房机器人：

Automatic curriculum:

```markdown
Task: Pick up a mug
Task: Place mug in sink
Task: Wipe table
```

Skill library:

```markdown
pick_object(obj)
place_object(obj, location)
open_drawer()
wipe_surface()
```

Iterative feedback:

```markdown
Object not detected
Grasp failed
Collision risk
Task verifier says mug is not in sink
```

## 23.5 Curriculum with Formal Novelty / Competence

当前 curriculum 靠 GPT-4 prompt。可以结合：

- novelty score
- learning progress
- skill graph frontier
- uncertainty
- affordance estimation
- resource precondition satisfaction

比如给每个 task 估计：

```markdown
score(task) = novelty(task) × feasibility(task) × usefulness(task)
```

让 curriculum 更可控。

## 23.6 Reduce LLM Cost

可以研究：

- distill GPT-4-generated skills into smaller models
- use small model for retrieval/routine code
- call GPT-4 only for hard failures
- cache task plans
- static analysis before execution
- hierarchical planning to reduce iterations

---

## 24\. 最后给你一个核心 take-away

如果你只记住一句话：

> VOYAGER 的核心不是“GPT-4 会玩 Minecraft”，而是它把 LLM 的世界知识和代码生成能力，组织成一个可以持续提出目标、执行、纠错、保存技能、复用技能的 lifelong embodied learning loop。

它的三个模块是互相依赖的：

- 没有 automatic curriculum，agent 不知道学什么。
- 没有 skill library，agent 学过的东西不能长期积累。
- 没有 iterative prompting，GPT-4 生成的代码不够可靠。
- 没有 self-verification，系统不知道什么时候成功，也不知道什么技能可以保存。

所以 VOYAGER 的真正贡献是一个系统级 recipe：

```markdown
Open-ended agent = LLM curriculum + code-as-action + executable skill memory + feedback-driven repair
```

这篇文章非常适合你从 embodied agents、LLM agents、robot learning、lifelong learning、program synthesis 这些方向切入。