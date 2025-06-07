好的，指令收到。

我将开始起草这份关键的技术规约。这份文档将作为我们团队协作的基石，确保每个人都基于同一份精确、无歧义的数据蓝图进行工作。

以下是《谜探》核心数据模型规约的第一版草案，以我们决定的 **YAML 格式**进行编写，以便于人类阅读、评审和注释。

---

# 《谜探》核心数据模型规约 (Core Data Model Specification)

- **规约版本:** 1.0
- **状态:** **草案 (Draft)**
- **编制者:** Gemini 2.5 Pro
- **日期:** 2024年6月7日
- **目标:** 本文档使用对人类友好的YAML格式，定义了“核心故事模型(CSM)”和“游戏化故事模型(GSM)”的精确数据结构。它是所有后端生成逻辑和前端展示逻辑的“单一数据契约”。在CI/CD流程中，此文件将被自动编译为严格的`csm.schema.json`和`gsm.schema.json`，供机器验证和代码生成使用。

---

## 第1部分: 核心故事模型 (Core Story Model - CSM)

**CSM是系统的“单一真相来源”，由LLM对原始小说进行客观、中立的解析而生成。它只包含事实，不包含任何游戏化元素或主观推断。**

```yaml
# csm.spec.yaml

$schema: "http://json-schema.org/draft-07/schema#"
$id: "https://mi-tan.tech/schemas/csm.spec.v1.yaml"
title: "核心故事模型 (Core Story Model - CSM)"
description: "从原始小说中提取的客观事实数据结构，是系统所有后续生成的“单一真相来源”。"
type: object
required:
  - metadata
  - timeline
  - characters
  - plotElements

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
        example: "1.0"
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
      initialMotivation:
        type: string
        description: "在故事开始时，驱动该角色的主要动机。"
  
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
      description:
        type: string
        description: "描述源角色对目标角色的关系，例如：'爱慕', '仇恨', '雇佣关系'。"

```

---

## 第2部分: 游戏化故事模型 (Game-ified Story Model - GSM)

**GSM在CSM的基础上，根据用户的配置（如指定凶手），注入游戏化元素。它包含了生成剧本和线索所需的所有信息，是游戏逻辑的完整体现。**

```yaml
# gsm.spec.yaml

$schema: "http://json-schema.org/draft-07/schema#"
$id: "https://mi-tan.tech/schemas/gsm.spec.v1.yaml"
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
    # 在实际工具链中，这里会解析并嵌入整个CSM的schema
    $ref: "csm.spec.yaml" 
    
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
            playerName: # 可选，如果需要分配给特定玩家
              type: string
      murdererId:
        type: string
        description: "用户指定的凶手角色ID。"

  truthMap:
    type: object
    description: "关于案件真相的“上帝视角”信息，是逻辑校验和线索生成的核心依据。"
    properties:
      motive:
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
        liesToTell:
          type: array
          description: "为掩盖真相或秘密而设计的具体谎言。这是实现ADD中“谎言”定义的关键。"
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
        goals:
          type: array
          description: "该角色在本场游戏中的主要目标和次要目标。"
          items:
            type: string

```

### 下一步行动建议

1.  **团队评审:** 将这份规约草案提交给开发团队（前端、后端、AI工程师）进行评审。收集反馈，特别是关于字段的缺失、冗余或命名清晰度。
2.  **版本迭代:** 根据评审意见，修订此规约，生成V1.1版本。
3.  **正式批准:** 一旦团队达成共识，将此规约标记为“已批准(Approved)”，并将其置于版本控制之下。
4.  **工具链搭建:** 与此同时，开始搭建将这份YAML自动编译为JSON Schema的CI脚本。

这份文档为我们接下来的API规约和具体开发工作铺平了道路。