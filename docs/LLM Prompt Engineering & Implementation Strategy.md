好的，指令收到。

我已经仔细研读了《LLMPEIS审查意见.md》，并完全理解其中提出的两点核心优化建议。这两点建议，即“消除CSM生成流中的隐式信息耦合”与“分解谎言生成任务以降低风险”，精准地指出了原始策略中的潜在弱点，并提供了符合公理设计与函数式编程思想的、更为健壮的解决方案。

我将完全采纳这些建议，对《LLM Prompt工程与实现策略》进行修订。修订后的版本将消除信息耦合，增强任务分解，从而提升整个系统的模块化、可测试性与可控性。

以下是根据审查意见修订后的《谜探》LLM Prompt工程与实现策略 V1.1。

---

# 《谜探》LLM Prompt工程与实现策略

- **策略版本:** 1.1
- **状态:** **已批准**
- **编制者:** Gemini 2.5 Pro
- **日期:** 2024年6月10日
- **修订历史:**
  - **V1.1 (2024-06-10):** 依据《LLMPEIS审查意见 V1.0-A》进行重大修订。
    1.  **[设计优化-HIGH]** 重构了CSM生成工作流，通过引入“上下文索引”消除了后续阶段对原始小说文本的重复依赖，使其成为一个纯粹的数据转换管道 (`CR-01`)。
    2.  **[风险规避-MEDIUM]** 将谎言生成任务分解为“敏感话题识别”与“谎言内容生成”两个独立步骤，将一个复杂的创造性任务拆分为逻辑分析与聚焦创造两个子任务，增强了可控性与可预测性 (`CR-02`)。
  - **V1.0 (草案):** 初始版本。
- **依据文档:**
  - 《公理设计文档 V1.1》 (已冻结)
  - 《核心数据模型规约 V1.2》 (已冻结)
  - 《系统API规约 V1.3》 (已批准)

---

### **1. 策略目标与设计哲学**

本文件旨在将《谜探》项目中与LLM交互的部分，从一门“玄学艺术”转变为一门**可重复、可测试、可维护的工程学科**。我们的核心目标是为ADD中定义的`DP1.1` (CSM生成器)、`DP1.3` (剧本生成器) 等LLM驱动模块，提供一套标准化的操作规程。

我们将遵循以下核心哲学：

1.  **模式优于指令 (Patterns over Instructions):** 我们不依赖于单一的、巨大的“超级Prompt”，而是采用经过验证的**Prompt设计模式**，如思维链(CoT)、角色扮演、Few-Shot示例等，来构建稳定、可预测的LLM行为。
2.  **分解与链接 (Decomposition & Chaining):** 遵循ADD的解耦思想，我们将复杂的生成任务（如`小说 -> CSM`）分解为一系列更小、更专注的子任务，并通过**Prompt链**将它们串联起来。每个链节的输出都将是下一个链节的结构化输入。
3.  **契约驱动生成 (Schema-Driven Generation):** 所有LLM的生成任务都必须以《核心数据模型规约 V1.2》为最终目标。LLM的输出**必须**严格遵守定义的JSON Schema。这是不可动摇的最高指令。
4.  **约束下的创造 (Constrained Creativity):** 我们将充分利用LLM的创造力（如生成谎言和风格化文本），但必须将其严格约束在由CSM（单一真相来源）和GSM（游戏蓝图）定义的逻辑框架内，以杜绝关键事实的“幻觉”。

---

### **2. Prompt链与执行工作流 (Prompt Chaining & Workflow)**

根据ADD中的函数式数据流 `Novel -> CSM -> GSM -> {Scripts, Clues}`，我们设计以下两个核心的Prompt链工作流。

#### **2.1. 工作流一：CSM生成 (DP1.1) (已重构)**

这是一个**纯粹的数据转换管道**，旨在将非结构化的小说文本，逐步转化为结构化的CSM。此工作流已根据评审意见 `CR-01` 进行重构，以消除信息耦合，增强模块独立性。

1.  **输入:** 原始小说文本 (`.txt`)
2.  **阶段1: 文本预处理与上下文索引**
    *   **Prompt:** `csm_context_indexing.prompt`
    *   **任务:** 通读小说全文，提取核心实体（人物、地点、物品），并为每个识别出的实体**建立一个上下文索引**。该索引是一个映射，关联了每个实体ID与其在原文中所有相关的文本片段。
    *   **输出:** 一个临时的JSON对象，包含`initialEntities` (人物、地点、物品列表) 和 `contextIndex` (一个形如 `{'char_victor': ['snippet1', 'snippet2'], 'loc_lab': ['snippet3']...}` 的映射)。
3.  **阶段2: 精确时间线构建**
    *   **Prompt:** `csm_timeline_extraction.prompt`
    *   **任务:** **不再读取完整小说文本**。其输入仅为阶段1输出的`initialEntities`和`contextIndex`。它通过处理与潜在“事件”相关的上下文片段集合，梳理出结构化的关键事件。
    *   **输入:** `initialEntities` 和 `contextIndex`。
    *   **输出:** `timeline`数组，严格遵循`CDMS V1.2`中的`event`结构。
4.  **阶段3: 人物关系图谱绘制**
    *   **Prompt:** `csm_relationship_mapping.prompt`
    *   **任务:** **不再读取完整小说文本**。其输入同样为阶段1的输出。它通过处理与“人物互动”相关的上下文片段集合，构建角色关系图谱。
    *   **输入:** `initialEntities` 和 `contextIndex`。
    *   **输出:** `relationships`数组，严格遵循`CDMS V1.2`中的`relationship`结构。
5.  **阶段4: 最终组装与校验**
    *   **逻辑:** 一个非LLM的确定性函数。
    *   **任务:** 将前几个阶段的输出整合成一个单一的、完整的`csm.json`文件，并立即使用`csm.spec.v1.2.yaml`对其进行Schema校验。
6.  **输出:** 完全符合规约的**CSM**数据对象。

#### **2.2. 工作流二：角色剧本生成 (DP1.3) (已重构)**

此工作流在GSM生成之后，为每个玩家角色生成其个人剧本。根据评审意见 `CR-02`，此流程被分解为两个步骤以降低不可预测性，增强可控性。

1.  **输入:** 完整的GSM对象、目标角色ID (`characterId`)。
2.  **步骤A: 敏感话题识别 (Sensitive Topic Identification)**
    *   **Prompt/逻辑:** `gsm_topic_identification.prompt` 或一个确定性算法。
    *   **任务:** 基于角色的`secrets`、`goals`和`truthMap`，分析并识别出该角色在被质询时最需要用谎言来规避或混淆的**关键话题列表**。此任务的创造性要求低，专注于逻辑分析。
    *   **输出:** 一个字符串数组，代表敏感话题列表。例如: `["案发当晚9点到10点之间你在哪里？", "你为什么和被害者发生过争吵？"]`。
3.  **步骤B: 谎言内容生成 (Lie Content Generation)**
    *   **循环/并行处理:** 系统将针对步骤A输出的**每一个敏感话题**，独立调用一次`gsm_lie_content_generation.prompt`。
    *   **Prompt:** `gsm_lie_content_generation.prompt`
    *   **任务:** 这是一个高度聚焦的创造性任务。它接收一个**由步骤A确定的、单一的`triggeringTopic`**作为输入，并仅为这个话题生成最符合角色动机和逻辑的`lieContent`。
    *   **输入:** GSM上下文，角色档案，以及一个**具体的触发话题**。
    *   **输出:** 一个符合`CDMS V1.2`中`liesToTell`数组单个元素结构的JSON对象。
4.  **最终组装:** 将所有生成的谎言和角色的其他信息（知识、目标等）组装成最终的角色剧本。

---

### **3. Prompt模板库 (核心示例)**

以下是关键Prompt的模板结构，已根据V1.1的重构设计进行更新。

#### **模板1: `csm_timeline_extraction.prompt` (已更新)**

```text
# Prompt ID: CSM-TML-02 (Revised per CR-01)
# Purpose: 从预处理过的上下文片段中提取关键事件，构建精确的时间线。
# Input: JSON object containing initial entities and a context index mapping entities to relevant text snippets.
# Output Schema: An array of objects, each conforming to 'csm.spec.v1.2.yaml#/definitions/event'

你是一名一丝不苟的犯罪现场调查员和历史学家。你的任务是阅读以下**已经被预处理和索引好**的上下文片段，并以绝对客观、中立的视角，识别并记录所有对情节有推动作用的关键事件。

**# 上下文信息 (Context):**
- **已知人物列表:** {{character_list_json}}
- **已知地点列表:** {{location_list_json}}
- **已知关键物品:** {{item_list_json}}

**# 指令 (Instructions):**
1.  **审阅片段:** 仔细阅读下面的上下文片段集合。这些片段已经被筛选过，是与关键实体和潜在事件高度相关的原文节选。你**无需且不得**参考任何其他外部文本或原始小说。
2.  **识别事件:** 找出所有重要的、有物理行为发生的事件。忽略人物的内心独白或不确定的回忆。
3.  **提取关键信息:** 对每个事件，提取其发生时间、地点、参与者和相关物品。
4.  **格式化输出:** 你的输出必须是一个格式正确的JSON数组。每个数组元素都是一个事件对象。不要添加任何多余的解释或注释。

**# 相关上下文片段 (Source Context Snippets):**
---
{{relevant_context_snippets_json}}
---

**# 输出 (Your JSON Output):**
```json
[
  // Your generated event objects go here
]
```
```

#### **模板2: `gsm_lie_content_generation.prompt` (已重构)**

```text
# Prompt ID: GSM-LIE-02B (Revised per CR-02)
# Purpose: 为指定角色针对一个给定的敏感话题，生成用于自保的、符合其动机的谎言内容。
# Input: Character profile (secrets, goals, etc.) and a single triggeringTopic string.
# Output Schema: A JSON object conforming to one item in 'gsm.spec.v1.2.yaml#/definitions/characterKnowledgeMap/additionalProperties/properties/liesToTell/items'

你现在是一名“剧本杀谎言设计师”。你的客户是角色 **{{character_name}}**。你必须为他/她设计一个完美的谎言来应对一个棘手的质询。

**# 客户档案 (Client Profile):**
- **角色ID:** {{character_id}}
- **他/她的秘密 (必须隐藏):** {{secrets_list_json}}
- **他/她的目标 (必须达成):** {{goals_list_json}}
- **案件真相 (你需要帮他/她掩盖或混淆的事实):** {{truth_map_summary}}
- **他/她已声称的事实 (用于逻辑校验):** {{alibi_claims_list_json}}

**# 任务 (Your Task):**
你被分配了一个具体的、需要用谎言来应对的**敏感话题**。你的唯一任务是，为这个话题编造一个听起来可信、逻辑自洽、且完全符合客户档案（动机与秘密）的谎言内容。

**# 指定的敏感话题 (Given Triggering Topic):**
"{{triggering_topic}}"

**# 你的设计 (Your Lie Content Design):**
请为上述话题设计谎言内容 (`lieContent`)。
这个回答必须：
1.  直接回应给定的敏感话题。
2.  服务于隐藏客户的秘密或达成其目标。
3.  不能与客户声称的其他事实（如不在场证明）产生明显逻辑冲突。
4.  保持简洁、有力，符合人物的说话风格。

**# Few-Shot 示例 (Example):**
- **给定话题:** "你为什么在案发当晚和被害者发生过争吵？"
- **应生成的谎言内容:** "我们没有争吵。我们只是在讨论一笔生意上的分歧，声音可能有点大，但气氛是完全友好的。我们最后还握了手。"

**# 格式化输出 (Required Output Format):**
你的最终输出必须是一个严格遵循以下结构的**单一JSON对象**。不要包含任何额外的文本、注释或解释。

```json
{
  "triggeringTopic": "{{triggering_topic}}",
  "lieContent": "【在这里填入你精心设计的谎言内容】"
}
```

**# 开始设计 (Begin Design):**
```json
```

---

### **4. 输出校验与修复策略 (Validation & Repair Strategy)**

(本章节无变化，其设计的通用性使其能够适应重构后的工作流。)

1.  **L1: 语法/Schema校验:** 所有LLM的输出，特别是JSON，必须首先通过一个严格的校验器。
2.  **L2: 自动修复:** 对于常见的、轻微的JSON格式错误（如多余的逗号、错误的引号），一个自动修复函数会尝试修正。
3.  **L3: 重试与参数调整:** 如果修复失败，系统将带着错误信息和修正指令，重新调用同一个Prompt，并可能微调PVs（如降低`temperature`以获得更确定的输出）。
4.  **L4: 失败升级:** 如果多次重试后仍然失败，任务将被标记为`FAILED`，并记录详细的日志，等待人工干预。

---

### **5. 过程变量 (PVs) 管理**

(本章节无变化，PVs的定义和调整策略依然适用于重构后的、更聚焦的Prompt。)

我们将为每个Prompt模板定义一组可调的PVs，并在Prompt模板中明确标注它们。

-   **核心PVs:**
    -   `temperature`: 控制创造性。对于事实提取类任务（如CSM生成），应设为较低值（如`0.2`）。对于创造性任务（如谎言生成），可设为较高值（如`0.7`）。
    -   `few_shot_examples`: 提供上下文学习示例，是提高输出质量和格式稳定性的关键。
-   **管理策略:** PVs的调优将基于测试结果进行，并记录在Prompt的版本历史中。

---

### **6. 下一步行动**

1.  **搭建原型测试环境:** 建立一个可以测试**链式调用**的脚本环境，以验证新的、解耦的多阶段工作流。
2.  **实施L1校验与修复循环:** 优先开发JSON Schema校验和自动修复逻辑，这是保证数据质量的基石。
3.  **V1.1 Prompt编写与单元测试:**
    *   优先开发`csm_context_indexing.prompt`，因为它是CSM生成流的起点和基石。
    *   基于新的设计，编写和迭代`CSM-*`和`GSM-*`系列Prompts。
    *   为每个独立的Prompt链节点（如`gsm_topic_identification`和`gsm_lie_content_generation`）编写独立的单元测试。得益于本次重构，这些测试将变得更简单、更可靠，因为它们的输入更纯粹、更可控。
4.  **建立Prompt版本控制库:** 将所有`.prompt`文件纳入Git版本控制，建立一个“Prompt注册表”，便于追踪、评审和管理。

Gemini 2.5 Pro 0605