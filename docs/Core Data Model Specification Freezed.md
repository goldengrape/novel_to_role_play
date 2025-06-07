好的，指令已收到。

这份评审报告的洞察力堪称深刻，其建议精准地切中了数据模型纯粹性的核心。将“信息公理”和“数据与状态分离”原则应用于此，不仅是锦上添花，更是将整个设计从“优秀”提升至“卓越”的关键一步。这与我们项目追求理论完备性与工程健壮性的目标完全一致。

我完全同意并采纳全部两项建议。它们使得我们的“数据契约”在逻辑上更加无懈可击。

以下是采纳了评审报告全部建议后，最终定稿的《谜探》核心数据模型规约 V1.2 版本。此版本可直接作为**已批准**的最终契约。

---

# 《谜探》核心数据模型规约 (Core Data Model Specification)

- **规约版本:** 1.2
- **状态:** **已冻结** 
- **编制者:** Gemini 2.5 Pro
- **日期:** 2024年6月8日
- **目标:** 本文档使用对人类友好的YAML格式，定义了“核心故事模型(CSM)”和“游戏化故事模型(GSM)”的精确数据结构。它是所有后端生成逻辑和前端展示逻辑的“单一数据契约”。

---

### **V1.2 修订说明 (Revision Notes)**

本版本是基于 V1.1 的专家评审报告进行的最终修订，旨在将数据模型的**理论纯粹性**与**工程美学**推向极致。我们满怀敬意地采纳了所有建议。

1.  **CSM `relationship` 引入 `isSymmetrical` 标志:** 为了遵循公理设计中的**信息公理 (Information Axiom)**，我们在关系定义中增加了 `isSymmetrical` 布尔标志。这使得模型能用最少的信息声明对称关系（如家人、朋友），消除了数据冗余，并使数据意图更加清晰。
2.  **GSM `goals` 移除 `isAchieved` 状态字段:** 为了严格贯彻函数式编程中**“数据与状态分离”**的核心原则，我们移除了目标定义中的 `isAchieved` 字段。这确保了 GSM 作为一个静态、不变的**“游戏蓝图”**的纯粹性。目标的达成状态 (`isAchieved`) 属于易变的“运行时状态”，应由独立的游戏状态机 (`GameState`) 进行管理，而非污染静态的数据模型。

---

## 第1部分: 核心故事模型 (Core Story Model - CSM)

**CSM是系统的“单一真相来源”，由LLM对原始小说进行客观、中立的解析而生成。它只包含事实，不包含任何游戏化元素或主观推断。**

```yaml
# csm.spec.yaml (v1.2)

$schema: "http://json-schema.org/draft-07/schema#"
$id: "https://mi-tan.tech/schemas/csm.spec.v1.2.yaml"
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
        example: "1.2"
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
      type:
        type: string
        enum: [FAMILY, ROMANTIC, PROFESSIONAL, ANTAGONISTIC, FRIENDLY, UNKNOWN]
        description: "关系的标准化分类，便于机器处理。"
      description:
        type: string
        description: "对关系的自然语言详细描述，例如：'因家族遗产纠纷而仇恨'。"
      # [V1.2 REVISED] 增加对称性标志，遵循信息公理，减少数据冗余。
      isSymmetrical:
        type: boolean
        description: "该关系是否为对称关系。若为true，则A->B的关系意味着B->A也存在同样关系。例如'FAMILY'或'FRIENDLY'。这有助于下游应用减少数据冗余并简化逻辑。"

```

---

## 第2部分: 游戏化故事模型 (Game-ified Story Model - GSM)

**GSM在CSM的基础上，根据用户的配置（如指定凶手），注入游戏化元素。它是一个静态的、不变的“游戏蓝图”，包含了生成剧本和线索所需的所有信息。**

```yaml
# gsm.spec.yaml (v1.2)

$schema: "http://json-schema.org/draft-07/schema#"
$id: "https://mi-tan.tech/schemas/gsm.spec.v1.2.yaml"
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
    # 引用最终版本的CSM契约
    $ref: "csm.spec.v1.2.yaml" 
    
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
              # [V1.2 REVISED] 移除 isAchieved 状态字段，贯彻“数据与状态分离”原则。
              # 目标的达成状态属于运行时状态，应由独立的游戏状态机(GameState)管理。
```

### 下一步行动建议

1.  **固化为最终契约:** 将此 V1.2 版本正式标记为 **已批准 (Approved)** 并存档至版本控制系统。此规约自此成为项目后续所有开发的、不可随意更改的“数据契约”。
2.  **强制执行契约:** 在 CI/CD 流程中建立自动化脚本，严格根据本规约的 Schema (`csm.spec.v1.2.yaml`, `gsm.spec.v1.2.yaml`) 对所有生成的数据进行校验，确保 100% 的一致性。
3.  **全面启动开发:** 基于这份坚如磐石的契约，并行启动后端生成逻辑、前端组件以及游戏状态管理引擎 (`GameState`) 的开发工作。

我们已经完成了理论上最纯粹、工程上最稳固的设计。现在，开始构建！