好的，指令收到。

这份评审报告的洞察力是卓越的。它精准地指出了在LLM工程这个前沿领域中，如何通过应用公理设计的核心原则来规避风险、增强系统鲁棒性。将“最小化上下文原则”应用于谎言生成，以及为数据管道增加“质量门控”，是从“一个可行的设计”迈向“一个工程上完美的设计”的关键步骤。

我将完全采纳这两点核心评审意见，并以最高的标准对《LLM Prompt工程与实现策略》进行最终修订。这次修订将使我们的策略蓝图在理论上更加完备，在实践中更加可靠。

以下是根据您的评审报告修订并定稿的 V1.2 版本。此版本已解决所有已知的设计风险，达到了“无可争议的蓝图”状态。

---

# 《谜探》LLM Prompt工程与实现策略

-   **策略版本:** 1.2
-   **状态:** **待批准**
-   **编制者:** Gemini 2.5 Pro
-   **日期:** 2024年6月10日
-   **修订历史:**
    -   **V1.2 (2024-06-10):** 依据《LLMPEIS V1.1 最终评审报告》进行最终修订，采纳全部建议。
        1.  **[设计优化 - CR-PEIS-02]** 在CSM生成流中增加了“上下文质量门控”，将阶段1的输出从“黑盒”变为“白盒”，确保了数据管道入口的健壮性。
        2.  **[风险规避 - CR-PEIS-01]** 为谎言生成任务引入“最小化上下文原则”，定义了专用的 `CharacterLieGenerationContext`，从根本上杜绝了信息泄露风险，并提高了任务的可测试性。
    -   **V1.1 (2024-06-10):** 根据早期评审意见重构，将CSM生成流改造为数据转换管道，并分解了谎言生成任务。
-   **依据文档:**
    -   《公理设计文档 V1.1》
    -   《核心数据模型规约 V1.2》
    -   《系统API规约 V1.3》

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

#### **2.1. 工作流一：CSM生成 (DP1.1) (已根据CR-PEIS-02修订)**

这是一个**健壮的数据转换管道**，旨在将非结构化的小说文本，逐步转化为结构化的CSM。此流程已根据评审意见重构，以实现“尽早校验”原则。

1.  **输入:** 原始小说文本 (`.txt`)
2.  **阶段1: 上下文索引与质量评估**
    *   **Prompt:** `csm_context_indexing.prompt`
    *   **任务:** 通读全文，不仅要提取与核心实体（人物、地点、物品）相关的原文片段，还必须为每个片段附加一个**“相关性置信度评分”**和一个**“实体关联摘要”**。
    *   **输出:** 一个临时的JSON对象，包含一系列附带元数据的上下文片段。
3.  **阶段2: 上下文质量门控 (Context Quality Gate)**
    *   **逻辑:** 一个**非LLM的确定性函数**。
    *   **任务:**
        *   **过滤:** 仅保留置信度评分高于预设阈值（PV）的片段。
        *   **路由:** 根据“实体关联摘要”，将片段分发到不同的处理队列。例如，包含时间戳和动作描述的片段进入“时间线队列”，包含多角色互动的片段进入“关系队列”。
    *   **输出:** 两个或多个经过净化和分类的、高质量的上下文片段列表。
4.  **阶段3: 并行精确提取**
    *   **任务A: 时间线构建**
        *   **Prompt:** `csm_timeline_extraction.prompt`
        *   **输入:** “时间线队列”中的上下文片段。**绝不接触原始小说或其他无关片段。**
        *   **输出:** `timeline`数组。
    *   **任务B: 关系图谱绘制**
        *   **Prompt:** `csm_relationship_mapping.prompt`
        *   **输入:** “关系队列”中的上下文片段。**绝不接触原始小说或其他无关片段。**
        *   **输出:** `relationships`数组。
5.  **阶段4: 最终组装与校验**
    *   **逻辑:** 一个非LLM的确定性函数。
    *   **任务:** 将所有阶段的输出整合成一个完整的`csm.json`文件，并立即使用`csm.spec.v1.2.yaml`对其进行Schema校验。
6.  **输出:** 完全符合规约的**CSM**数据对象。

#### **2.2. 工作流二：角色剧本生成 (DP1.3) (已根据CR-PEIS-01修订)**

此工作流在GSM生成之后，为每个玩家角色生成其个人剧本。根据评审意见，此流程被分解以引入“最小化上下文原则”。

1.  **输入:** 完整的GSM对象、目标角色ID (`characterId`)。
2.  **步骤A: 最小化上下文投影 (Minimum Context Projection)**
    *   **逻辑:** 一个**非LLM的确定性函数**。
    *   **任务:** 从庞大的GSM中，为目标角色**投影 (Project)** 出一个专用的、最小化的 `CharacterLieGenerationContext` 对象。此对象**仅包含**生成谎言所必需的信息：
        *   `characterId`, `characterName`
        *   `secrets_to_protect`
        *   `goals_to_achieve`
        *   `alibi_to_maintain`
        *   `facts_to_obscure` (从`truthMap`中提取的、与该角色秘密直接相关的事实)
    *   **输出:** `CharacterLieGenerationContext` 对象。
3.  **步骤B: 敏感话题识别 (Sensitive Topic Identification)**
    *   **Prompt/逻辑:** `gsm_topic_identification.prompt` 或确定性算法。
    *   **任务:** 基于**步骤A生成的最小化上下文**，识别出该角色最需要用谎言规避的关键话题列表。
    *   **输出:** 一个字符串数组，如 `["案发当晚9点到10点之间你在哪里？", "你为什么和被害者发生过争吵？"]`。
4.  **步骤C: 聚焦式谎言内容生成 (Focused Lie Content Generation)**
    *   **循环/并行处理:** 针对步骤B输出的**每一个话题**，独立调用`gsm_lie_content_generation.prompt`。
    *   **Prompt:** `gsm_lie_content_generation.prompt`
    *   **任务:** 高度聚焦的创造性任务。接收**一个**`triggeringTopic`和**最小化上下文**，为其生成逻辑自洽的`lieContent`。
    *   **输入:** `CharacterLieGenerationContext` 对象，以及一个**具体的触发话题**。
    *   **输出:** 该角色的一条谎言对象，严格遵循`CDMS V1.2`的`liesToTell`结构。
5.  **最终组装:** 将所有生成的谎言和角色的其他信息（知识、目标等）组装成最终的角色剧本。

---

### **3. Prompt模板库 (核心示例)**

以下是关键Prompt的模板结构，已根据V1.2的修订设计进行更新。

#### **模板1: `gsm_lie_content_generation.prompt` (已根据CR-PEIS-01修订)**

```text
# Prompt ID: GSM-LIE-03 (Revised for Minimum Context Principle)
# Purpose: 为指定角色，针对一个给定的敏感话题，生成用于自保的、符合其动机的谎言内容。
# Input: A minimal 'CharacterLieGenerationContext' JSON object and a single 'triggeringTopic' string.
# Output Schema: A JSON object conforming to one item in 'gsm.spec.v1.2.yaml#/definitions/characterKnowledgeMap/additionalProperties/properties/liesToTell/items'

你是一名顶级的“剧本杀谎言顾问”。你的客户是角色 **{{context.characterName}}**。你的唯一任务是根据下方提供的、**严格限定**的客户档案，为他/她设计一个完美的谎言，以应对一个**具体**的质询话题。

**# 客户档案 (Client Profile - Minimum Context):**
---
{{CharacterLieGenerationContext_json}}
---
*   **解读提示:**
    *   `secrets_to_protect`: 必须不惜一切代价隐藏的秘密。
    *   `goals_to_achieve`: 你的谎言必须服务于这些目标。
    *   `alibi_to_maintain`: 你的谎言不能与此不在场证明冲突。
    *   `facts_to_obscure`: 这是案件的真实情况，你的谎言需要掩盖或歪曲这些事实。

**# 任务指令 (Your Directive):**
你被分配了一个具体的、需要用谎言来应对的**敏感话题**。你的唯一任务是，为这个话题编造一个听起来可信、逻辑自洽、且完全服务于上述客户档案的谎言内容。**禁止使用档案之外的任何信息。**

**# 指定的敏感话题 (Given Triggering Topic):**
"{{triggering_topic}}"

**# 格式化输出 (Required Output Format):**
你的最终输出必须是一个严格遵循以下结构的**单一JSON对象**。不要包含任何解释或额外的文本。

```json
{
  "triggeringTopic": "{{triggering_topic}}",
  "lieContent": "【在这里填入你为客户设计的、精炼且有说服力的谎言内容】"
}
```

**# 开始设计 (Begin Design):**
```json

```

#### **模板2: `csm_context_indexing.prompt` (已根据CR-PEIS-02修订)**

```text
# Prompt ID: CSM-IDX-02 (Revised for Quality Gate)
# Purpose: 通读全文，提取所有潜在相关的上下文片段，并为每个片段附加质量评估元数据。
# Input: Raw novel text.
# Output Schema: A JSON array of objects, each with 'snippet', 'relevanceScore', and 'entitySummary'.

你是一名情报分析师。你的任务是通读下面的原始文本，并提取所有可能与“关键事件”、“人物关系”、“核心动机”或“关键物品/地点”相关的文本片段。

**# 指令 (Instructions):**
1.  **分段提取:** 将所有相关的原文片段提取出来。
2.  **质量评估:** 对每一个片段，执行以下评估：
    *   **`relevanceScore` (相关性置信度评分):** 评估该片段对于构建案件核心事实（时间、地点、人物、事件）的重要性。给出一个0.0到1.0之间的小数。1.0表示极度关键，0.1表示微弱相关。
    *   **`entityLinkageSummary` (实体关联摘要):** 用一句话简要说明该片段关联了哪些核心实体以及它们之间的互动类型。例如：“[张三]在[书房]与[李四]发生[争吵]”或“[王五]发现[一把带血的刀]”。
3.  **格式化输出:** 你的输出必须是一个格式正确的JSON数组。每个数组元素都是一个包含`snippet`, `relevanceScore`, `entityLinkageSummary`三个字段的对象。

**# 原始小说文本 (Source Novel Text):**
---
{{raw_novel_text}}
---

**# 输出 (Your JSON Output):**
```json
[
  {
    "snippet": "晚饭后，大约九点钟，李太太听到隔壁传来激烈的争吵声，似乎是维克多先生和他的访客。",
    "relevanceScore": 0.9,
    "entityLinkageSummary": "[李太太]在[晚上九点]听到[维克多]和[访客]发生[激烈争吵]。"
  },
  {
    "snippet": "那把古老的匕首就放在书房的壁炉上，是家族的传家宝。",
    "relevanceScore": 0.6,
    "entityLinkageSummary": "[古老的匕首]位于[书房]的[壁炉]上。"
  }
  // ... more objects
]
```
```

---

### **4. 输出校验与修复策略 (Validation & Repair Strategy)**

(本章节无变化，其设计的通用性使其能够适应重构后的工作流。)

---

### **5. 过程变量 (PVs) 管理**

(本章节无变化，PVs的定义和调整策略依然适用于重构后的、更聚焦的Prompt。)

---

### **6. 下一步行动**

1.  **搭建原型测试环境:** 建立一个可以测试**链式调用**的脚本环境，以验证新的多阶段工作流，特别是“质量门控”和“最小化上下文投影”这两个非LLM逻辑步骤。
2.  **实施L1校验与修复循环:** 优先开发JSON Schema校验和自动修复逻辑，这是保证数据质量的基石。
3.  **V1.2 Prompt编写与单元测试:**
    *   **CSM流:**
        *   优先开发并迭代 `csm_context_indexing.prompt`。
        *   编写并测试 `Context Quality Gate` 的过滤和路由算法。
        *   为 `csm_timeline_extraction.prompt` 和 `csm_relationship_mapping.prompt` 编写独立的单元测试，确保它们能正确处理上游传入的、经过净化的数据片段。
    *   **GSM流:**
        *   编写并测试 `Minimum Context Projection` 函数。
        *   为 `gsm_topic_identification` 和 `gsm_lie_content_generation` 编写独立的单元测试。特别是后者，现在可以针对一个极小的、精确的输入进行测试，极大简化了测试用例的构建。
4.  **建立Prompt版本控制库:** 将所有`.prompt`文件纳入Git版本控制，建立一个“Prompt注册表”，便于追踪、评审和管理。

Gemini 2.5 Pro 0605