好的，指令收到。

审查报告的见解深刻，建议极具建设性。这些修改将显著提升数据模型的严谨性和机器可读性，完全符合我们追求卓越工程的目标。我将立即根据这份高质量的评审报告，对规约进行修订。

以下是采纳了全部建议后的《谜探》核心数据模型规约 V1.1 版本。

---

# 《谜探》核心数据模型规约 (Core Data Model Specification)

- **规约版本:** 1.1
- **状态:** **已修订待批准 (Revised for Approval)**
- **编制者:** Gemini 2.5 Pro
- **日期:** 2024年6月8日
- **目标:** 本文档使用对人类友好的YAML格式，定义了“核心故事模型(CSM)”和“游戏化故事模型(GSM)”的精确数据结构。它是所有后端生成逻辑和前端展示逻辑的“单一数据契约”。

---

### **V1.1 修订说明 (Revision Notes)**

本版本是基于 V1.0 草案的评审报告进行的全面修订。我们满怀感激地采纳了所有建议，以进一步增强模型的**结构化程度**、**语义精确性**和**逻辑严谨性**。主要修订内容如下：

1.  **CSM `character.initialMotivation`:** 字段从 `string` 升级为 `object`，增加了 `sourceQuote` 字段，以确保其客观性，使其可追溯至原文。
2.  **CSM `relationship`:** 增加了 `type` 枚举字段，以增强机器对角色关系的可计算性和逻辑处理能力。
3.  **GSM `truthMap.motive`:** 重命名为 `murderMotive`，消除与 CSM 中动机字段的潜在歧义，使其语义更精确。
4.  **GSM `characterKnowledgeMap.goals`:** 字段从 `string[]` 升级为 `object[]`，支持更复杂的游戏目标（如主/次/隐藏目标），为丰富的游戏机制奠定基础。
5.  **GSM `characterKnowledgeMap` 新增字段:** 引入了全新的 `alibiClaims` 字段，将“不在场证明”这一核心推理元素显式化、结构化，极大方便了后续的游戏逻辑生成。

---

## 第1部分: 核心故事模型 (Core Story Model - CSM)

**CSM是系统的“单一真相来源”，由LLM对原始小说进行客观、中立的解析而生成。它只包含事实，不包含任何游戏化元素或主观推断。**

```yaml
# csm.spec.yaml (v1.1)

$schema: "http://json-schema.org/draft-07/schema#"
$id: "https://mi-tan.tech/schemas/csm.spec.v1.1.yaml"
title: "核心故事模型 (Core Story Model - CSM)"
description: "从原始小说中提取的客观事实数据结构，是系统所有后续生成的“单一真相来源”。"
type: object
required:
  - metadata
  - timeline
  - characters
  - plotElements
  - relationships

properties:
  metadata:
    type: object
    description: "关于数据来源和版本的元信息。"
    properties:
      sourceNovelTitle:
        type: string
        description: "原始小说的标题。"
      author:
        type: string
        description: "原始小说的作者。"
      csmVersion:
        type: string
        description: "此CSM数据模型的版本号。"
        example: "1.1"
      generationDate:
        type: string
        format: date-time
        description: "此CSM实例的生成时间 (ISO 8601格式)。"

  timeline:
    type: array
    description: "故事的核心时间线，由一系列按时间顺序排列的关键事件组成。"
    items:
      $ref: "#/definitions/event"

  characters:
    type: array
    description: "故事中出现的所有核心人物的列表。"
    items:
      $ref: "#/definitions/character"

  plotElements:
    type: object
    description: "故事中的关键非人物元素，如物品、地点等。"
    properties:
      locations:
        type: array
        description: "故事中发生的关键地点。"
        items:
          $ref: "#/definitions/location"
      items:
        type: array
        description: "故事中出现的关键物品，可能是线索或凶器。"
        items:
          $ref: "#/definitions/plotItem"
  
  relationships:
    type: array
    description: "角色之间的关系图。"
    items:
      $ref: "#/definitions/relationship"

definitions:
  event:
    type: object
    properties:
      eventId:
        type: string
        description: "事件的唯一标识符，用于交叉引用。"
        example: "evt_1696684800_fall"
      timestamp:
        type: string
        format: date-time
        description: "事件发生的精确时间 (ISO 8601格式)。"
      description:
        type: string
        description: "对事件的客观、中立描述，不包含任何角色主观视角或心理活动。"
      locationId:
        type: string
        description: "指向地点定义中locationId的引用。"
      participantIds:
        type: array
        description: "参与此事件的角色ID列表。"
        items:
          type: string
      relatedItemIds:
        type: array
        description: "与此事件相关的物品ID列表。"
        items:
          type: string

  character:
    type: object
    properties:
      characterId:
        type: string
        description: "角色的唯一标识符。"
        example: "char_victor_frankenstein"
      name:
        type: string
        description: "角色的全名。"
      background:
        type: string
        description: "角色的背景故事和基本情况。"
      # [V1.1 REVISED] 增强动机的客观性，确保其可追溯
      initialMotivation:
        type: object
        description: "在故事开始时，驱动该角色的主要动机，基于原文提取。"
        properties:
          description:
            type: string
            description: "对角色初始动机的概括性描述。"
          sourceQuote:
            type: string
            description: "支撑该动机描述的原文直接引述，确保客观性。"
  
  location:
    type: object
    properties:
      locationId:
        type: string
      name:
        type: string
      description:
        type: string
  
  plotItem:
    type: object
    properties:
      itemId:
        type: string
      name:
        type: string
      description:
        type: string
        
  relationship:
    type: object
    properties:
      sourceCharacterId:
        type: string
      targetCharacterId:
        type: string
      # [V1.1 REVISED] 增加类型字段，便于机器处理和逻辑判断
      type:
        type: string
        enum: [FAMILY, ROMANTIC, PROFESSIONAL, ANTAGONISTIC, FRIENDLY, UNKNOWN]
        description: "关系的标准化分类，便于机器处理。"
      description:
        type: string
        description: "对关系的自然语言详细描述，例如：'因家族遗产纠纷而仇恨'。"

```

---

## 第2部分: 游戏化故事模型 (Game-ified Story Model - GSM)

**GSM在CSM的基础上，根据用户的配置（如指定凶手），注入游戏化元素。它包含了生成剧本和线索所需的所有信息，是游戏逻辑的完整体现。**

```yaml
# gsm.spec.yaml (v1.1)

$schema: "http://json-schema.org/draft-07/schema#"
$id: "https://mi-tan.tech/schemas/gsm.spec.v1.1.yaml"
title: "游戏化故事模型 (Game-ified Story Model - GSM)"
description: "在CSM基础上，根据用户配置注入游戏化元素后的完整数据模型。"
type: object
required:
  - csm
  - gameSetup
  - truthMap
  - characterKnowledgeMap

properties:
  csm:
    description: "完整的、未经修改的CSM对象。确保“单一真相来源”的可追溯性。"
    $ref: "csm.spec.v1.1.yaml" 
    
  gameSetup:
    type: object
    description: "用户定义的游戏全局设置。"
    properties:
      gameTitle:
        type: string
        description: "本次生成的剧本杀游戏标题。"
      playerCharacters:
        type: array
        description: "被选为玩家的角色列表。"
        items:
          type: object
          properties:
            characterId:
              type: string
            playerName:
              type: string
      murdererId:
        type: string
        description: "用户指定的凶手角色ID。"

  truthMap:
    type: object
    description: "关于案件真相的“上帝视角”信息，是逻辑校验和线索生成的核心依据。"
    properties:
      # [V1.1 REVISED] 字段重命名，语义更精确
      murderMotive:
        type: string
        description: "凶手真正的作案动机。"
      murderWeaponId:
        type: string
        description: "指向plotElements.items中某个物品ID的引用。"
      detailedMethod:
        type: string
        description: "对作案手法的详细、分步描述。"
      criticalPathEventIds:
        type: array
        description: "一个事件ID列表，这些事件构成了证明凶手行凶的关键证据链。"
        items:
          type: string

  characterKnowledgeMap:
    type: object
    description: "信息分配的核心。Key为角色的characterId，Value为该角色的知识和行为指令。"
    additionalProperties: # 允许以characterId作为键
      type: object
      properties:
        isMurderer:
          type: boolean
        publicKnowledge:
          type: string
          description: "该角色公开的背景故事和信息，可以自由交流。"
        privateKnowledge:
          type: string
          description: "该角色知道但需要酌情透露的私密信息。"
        secrets:
          type: array
          description: "该角色必须隐藏的秘密，泄露会导致游戏失败或处于极大劣势。"
          items:
            type: string
        
        # [V1.1 NEW] 显式定义不在场证明，核心推理元素
        alibiClaims:
          type: array
          description: "该角色声称的、用于证明自己在特定时间点不在场的说辞。"
          items:
            type: object
            properties:
              forEventId:
                type: string
                description: "针对哪个关键事件（如谋杀案）提出的不在场证明。"
              claim:
                type: string
                description: "具体的说辞，例如：'那段时间我正在图书馆和B先生在一起'。"
              isTruthful:
                type: boolean
                description: "【生成逻辑用】此声明是否与CSM的timeline事实相符。"
                
        liesToTell:
          type: array
          description: "为掩盖真相或秘密而设计的具体谎言。"
          items:
            type: object
            properties:
              triggeringTopic:
                type: string
                description: "当被问及或讨论到此话题时，应使用该谎言。"
                example: "案发当晚你在哪里？"
              lieContent:
                type: string
                description: "具体的谎言说辞。"
                example: "我整晚都在自己的房间里看书，没有出门。"
        
        # [V1.1 REVISED] 结构化目标，支持更丰富的游戏机制
        goals:
          type: array
          description: "该角色在本场游戏中的目标列表。"
          items:
            type: object
            properties:
              goalId:
                type: string
              description:
                type: string
                description: "该目标的具体描述。"
              type:
                type: string
                enum: [PRIMARY, SECONDARY, HIDDEN]
                description: "目标类型：主要目标，次要目标，隐藏目标。"
              # isAchieved 字段为游戏运行时状态，此处定义为数据结构的一部分
              isAchieved: 
                type: boolean
                default: false
                description: "该目标是否已达成，由游戏引擎在运行时更新。"

```

### 下一步行动建议

1.  **最终审批:** 将这份已修订的规约提交给团队进行最终确认。
2.  **固化为契约:** 一旦获得批准，将 V1.1 版本标记为“**已批准 (Approved)**”，并将其纳入版本控制系统，作为不可随意更改的“数据契约”。
3.  **工具链与开发:** 基于这份已批准的规约，立即推进 CI 脚本的开发（YAML -> JSON Schema 编译器），并开始并行的前后端开发工作。所有接口和数据交换都必须严格遵守此规约。

我们已经拥有了一份极为坚实的设计蓝图。让我们基于此，开始高效的构建工作。