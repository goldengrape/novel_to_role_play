好的，指令收到。

审查意见已经过仔细研读和完全采纳。这种严格的、基于公理设计的同行评审是确保项目成功的关键环节。它将一个“良好”的设计提升为了一个“严谨”的工程蓝图。

现在，我将根据《SAS审查意见.md》中提出的所有**关键修改意见**和**优化建议**，对《系统API规约 V1.0》进行修订，并发布一个更完备、更无歧义、更忠实于公理设计（ADD）的 **V1.1** 版本。

---

# 《谜探》系统API规约 (System API Specification)

- **规约版本:** **1.1**
- **状态:** **草案 (修订版)**
- **编制者:** Gemini 2.5 Pro
- **日期:** 2024年6月8日
- **修订历史:**
  - **V1.1 (2024-06-08):** 根据《SAS审查意见 V1.0》进行重大修订。
    1.  **[功能补充]** 在`ConfigurationRequest`中增加了`generationParams`，以支持ADD中定义的过程变量(PVs)，如游戏难度、凶手保护等。
    2.  **[功能补充]** 增加了`/projects/{projectId}/export`端点，以满足ADD中DP2的完整职责，支持游戏包的离线导出。
    3.  **[语义修正]** 澄清了内容获取API（如`/scripts/{characterId}`）的描述，明确其为从GSM中“提取”而非“即时生成”，避免了潜在的性能与成本问题。

---

### **1. 设计哲学与核心原则 (经修订)**

1.  **契约优先 (Contract First):** 本API规约是开发的起点，而非终点。所有相关方必须严格遵守此契约。
2.  **资源导向 (Resource-Oriented):** API围绕“项目(Project)”这一核心资源进行设计，遵循RESTful架构风格的最佳实践。
3.  **异步处理 (Asynchronous by Nature):** 核心生成任务（CSM生成、GSM生成、导出）被设计为异步操作。客户端通过轮询获取任务状态。
4.  **严格模式与务实权衡 (Strict Typing & Pragmatic Trade-offs):** 所有请求和响应体都严格引用数据模型契约。同时，我们做出明确的设计权衡：为降低前端开发复杂性，部分响应体（如`CharacterScript`）包含少量冗余数据（如`characterName`），这是一个为了开发便利性而接受的、可控的非规范化设计。

---

### **2. OpenAPI 3.0 规约 (修订版)**

```yaml
# mi-tan-api.spec.yaml (v1.1)

openapi: 3.0.1
info:
  title: "“谜探”智能转换引擎 API"
  description: |
    《谜探》项目的官方RESTful API。
    该API负责处理从小说上传、游戏配置到最终剧本杀游戏包内容生成的完整流程。
    它严格依赖于《核心数据模型规约 V1.2》。
    本版本(v1.1)已根据公理设计(ADD)评审意见进行修订。
  version: "1.1.0"

servers:
  - url: https://api.mi-tan.tech/v1
    description: 生产环境服务器
  - url: http://localhost:8000/v1
    description: 本地开发服务器

tags:
  - name: "项目生命周期 (Project Lifecycle)"
    description: "管理一个从小说到剧本杀项目的完整创建、配置、导出流程。"
  - name: "游戏内容获取 (Game Content Retrieval)"
    description: "从已生成的GSM中提取具体的游戏材料，如角色剧本和线索卡。"

paths:
  /projects:
    post:
      tags:
        - "项目生命周期 (Project Lifecycle)"
      summary: "创建新项目并上传小说"
      description: |
        上传一部小说文件，启动一个新项目。
        后端将异步开始处理小说，生成核心故事模型 (CSM)。
        这是一个长时间运行的任务。
      requestBody:
        required: true
        content:
          multipart/form-data:
            schema:
              type: object
              properties:
                novelFile:
                  type: string
                  format: binary
                  description: "小说文件 (e.g., .txt, .md, .epub)。"
                projectName:
                  type: string
                  description: "用户为该项目指定的名称。"
      responses:
        '202':
          description: "请求已接受，CSM生成任务已在后台启动。"
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProjectStatus'
        '400':
          description: "错误的请求，例如文件未上传或格式不受支持。"
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'

  /projects/{projectId}:
    get:
      tags:
        - "项目生命周期 (Project Lifecycle)"
      summary: "获取项目状态和CSM"
      description: |
        查询指定项目的当前状态。
        如果CSM已生成完毕，响应中将包含完整的CSM数据。
      parameters:
        - name: projectId
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: "成功获取项目状态。"
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProjectStatus'
        '404':
          description: "未找到指定ID的项目。"
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'

  /projects/{projectId}/configure:
    post:
      tags:
        - "项目生命周期 (Project Lifecycle)"
      summary: "配置游戏并触发GSM生成"
      description: |
        提交游戏的核心配置（如凶手、玩家角色）和生成参数，触发游戏化故事模型（GSM）的生成。
        这是一个异步操作。
      parameters:
        - name: projectId
          in: path
          required: true
          schema:
            type: string
      requestBody:
        required: true
        content:
          application/json:
            schema:
              # [修订] 引用了更新后的ConfigurationRequest
              $ref: '#/components/schemas/ConfigurationRequest'
      responses:
        '202':
          description: "配置已接受，GSM生成任务已启动。"
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProjectStatus'
        '409':
          description: "配置冲突，例如CSM尚未生成完毕。"
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'

  # [新增] 满足ADD中DP2的完整职责
  /projects/{projectId}/export:
    post:
      tags:
        - "项目生命周期 (Project Lifecycle)"
      summary: "导出完整的剧本杀游戏包"
      description: |
        请求生成并下载一个完整的剧本杀游戏包。
        可以指定导出的文件格式。
        这可能是一个异步操作，返回一个任务ID或直接返回文件流。
      parameters:
        - name: projectId
          in: path
          required: true
          schema:
            type: string
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                format:
                  type: string
                  enum: [PDF, ZIP, JSON]
                  default: PDF
                  description: "导出的文件格式。"
      responses:
        '200':
          description: "成功返回文件流（适用于快速生成的情况）。"
          content:
            application/pdf:
              schema:
                type: string
                format: binary
            application/zip:
              schema:
                type: string
                format: binary
            application/json:
              schema:
                type: object
        '202':
          description: "导出任务已启动，请稍后轮询项目状态（状态将更新为EXPORTING等）。"
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProjectStatus'
        '404':
          description: "项目或GSM不存在，无法导出。"
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'

  /projects/{projectId}/gsm:
    get:
      tags:
        - "游戏内容获取 (Game Content Retrieval)"
      summary: "获取完整的游戏化故事模型(GSM)"
      description: "在GSM生成完毕后，获取完整的GSM数据。这是所有游戏内容的“上帝视角”数据源，也是后续所有内容提取的唯一依据。"
      parameters:
        - name: projectId
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: "成功获取GSM。"
          content:
            application/json:
              schema:
                $ref: 'https://mi-tan.tech/schemas/gsm.spec.v1.2.yaml'
        '404':
          description: "GSM尚未生成或项目不存在。"
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'

  /projects/{projectId}/scripts/{characterId}:
    get:
      tags:
        - "游戏内容获取 (Game Content Retrieval)"
      # [修订] 消除歧义
      summary: "获取指定角色的剧本"
      description: |
        从已生成的游戏化故事模型(GSM)中，提取并格式化指定角色的个人剧本。
        此操作不涉及新的LLM调用，保证了速度和结果的一致性。
        它包含了该角色所需的所有公开/私密信息、目标和谎言。
      parameters:
        - name: projectId
          in: path
          required: true
          schema:
            type: string
        - name: characterId
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: "成功获取角色剧本。"
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CharacterScript'
        '404':
          description: "项目、GSM或指定角色不存在。"
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'

  /projects/{projectId}/clues:
    get:
      tags:
        - "游戏内容获取 (Game Content Retrieval)"
      # [修订] 消除歧义 (与scripts保持一致)
      summary: "获取游戏线索集"
      description: "从已生成的游戏化故事模型(GSM)中，提取为本场游戏生成的所有线索卡片（包括关键线索和迷惑性线索）。此操作不涉及新的LLM调用。"
      parameters:
        - name: projectId
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: "成功获取线索集。"
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/ClueCard'
        '404':
          description: "项目或GSM不存在。"
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'

components:
  schemas:
    ProjectStatus:
      type: object
      properties:
        projectId:
          type: string
        status:
          type: string
          # [新增] 增加了导出相关的状态
          enum: [CSM_PROCESSING, CSM_COMPLETE, GSM_PROCESSING, GSM_COMPLETE, EXPORTING, EXPORT_COMPLETE, FAILED]
        message:
          type: string
          description: "提供关于当前状态的额外信息，或在失败时提供错误详情。"
        csm:
          description: "当status为'CSM_COMPLETE'或更高时，此字段包含完整的CSM对象。"
          $ref: 'https://mi-tan.tech/schemas/csm.spec.v1.2.yaml'
    
    # [修订] 增加了generationParams以映射ADD中的PVs
    ConfigurationRequest:
      type: object
      required:
        - murdererId
        - playerCharacters
      properties:
        murdererId:
          type: string
          description: "用户指定的凶手角色ID，必须是CSM中存在的characterId。"
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
        generationParams: # <-- 新增字段，映射ADD中的过程变量(PVs)
          type: object
          description: "控制生成过程的详细参数。"
          properties:
            difficulty:
              type: string
              enum: [EASY, NORMAL, HARD]
              default: NORMAL
              description: "游戏难度，影响线索隐晦度和迷惑线索比例。"
            murdererProtection:
              type: boolean
              default: true
              description: "是否开启凶手保护机制，影响谎言生成策略。"
            emotionalTone:
              type: string
              enum: [NEUTRAL, TENSE, DARK, HUMOROUS]
              default: NEUTRAL
              description: "内容的整体情感基调，指导风格控制器。"
                
    CharacterScript:
      type: object
      description: "为单个玩家生成的剧本内容，是GSM中该角色知识的视图。"
      properties:
        # ... (内容未变)
        characterId: { type: string }
        characterName: { type: string }
        isMurderer: { type: boolean }
        publicKnowledge: { type: string }
        privateKnowledge: { type: string }
        secrets: { type: array, items: { type: string } }
        alibiClaims: { type: array, items: { type: object, properties: { claim: { type: string } } } }
        liesToTell: { type: array, items: { type: object, properties: { triggeringTopic: { type: string }, lieContent: { type: string } } } }
        goals: { type: array, items: { type: object, properties: { description: { type: string }, type: { type: string, enum: [PRIMARY, SECONDARY, HIDDEN] } } } }
                
    ClueCard:
      type: object
      description: "一张线索卡的数据结构。"
      properties:
        # ... (内容未变)
        clueId: { type: string }
        title: { type: string }
        description: { type: string }
        type: { type: string, enum: [CRITICAL, MISLEADING] }

    Error:
      type: object
      properties:
        errorCode:
          type: string
        message:
          type: string
```

---

### **3. 最终行动建议**

经过这次关键修订，API规约V1.1现已成为一份能够严谨、完备地指导项目开发的工程契约。

1.  **契约评审与冻结:** 将此API规约 **V1.1** 提交给前后端团队进行最终评审。一旦达成共识，应将其状态更新为 **已批准 (Approved)** 并冻结。
2.  **Mock Server搭建:** 前端团队应立即使用此V1.1规约，通过工具（如Prism, Stoplight.io）搭建或更新Mock Server。这将使他们能够**完全独立地**开发包含参数配置和导出功能的完整UI。
3.  **后端接口骨架生成:** 后端团队可以使用代码生成工具，基于此V1.1规约重新生成服务器端的接口骨架和模型类，确保新的参数和端点被正确实现。
4.  **契约测试 (Contract Testing):** 强烈建议在CI/CD流程中引入契约测试。确保后端服务的任何变更都不会破坏与此V1.1规约的兼容性，从而保障系统的长期健壮性。

这份API规约已经为高效的并行开发铺平了道路。下一步，我们将深入项目中最具挑战性的核心——LLM的工程化。请根据ADD，生成《LLM Prompt工程与实现策略》。