好的，指令已执行。

我们已经用《核心数据模型规约 V1.2》铸造了系统的“数据骨架”，现在，是时候构建连接所有部分的“神经系统”了。这份《系统API规约》将ADD中的功能需求和数据流，转化为精确、可执行、无歧义的工程接口。

这份规约是**后端与前端之间的神圣契约**。它将允许我们的开发团队，如同纪律严明的工匠，在各自的工位上并行施工，而最终产品能完美地拼接在一起。

---

# 《谜探》系统API规约 (System API Specification)

- **规约版本:** 1.0
- **状态:** **草案 (Draft)**
- **编制者:** Gemini 2.5 Pro
- **日期:** 2024年6月8日
- **目标:** 本文档使用 **OpenAPI 3.0** 规范，定义了《谜探》系统的RESTful API接口。它是前端应用（DP3）与后端服务（DP1, DP2, DP4）之间所有通信的唯一标准，旨在实现前后端的完全解耦和并行开发。

---

### **1. 设计哲学与核心原则**

1.  **契约优先 (Contract First):** 本API规约是开发的起点，而非终点。所有相关方必须严格遵守此契约。
2.  **资源导向 (Resource-Oriented):** API围绕“项目(Project)”这一核心资源进行设计，遵循RESTful架构风格的最佳实践。
3.  **异步处理 (Asynchronous by Nature):** 认识到LLM处理的耗时特性，核心生成任务（CSM生成、GSM生成）被设计为异步操作。客户端通过轮询或Webhook（未来版本）获取任务状态。
4.  **严格模式 (Strict Typing):** 所有请求和响应体都将严格引用《核心数据模型规约 V1.2》中定义的Schema，确保数据的一致性和无歧义性。

---

### **2. OpenAPI 3.0 规约**

```yaml
# mi-tan-api.spec.yaml (v1.0)

openapi: 3.0.1
info:
  title: "“谜探”智能转换引擎 API"
  description: |
    《谜探》项目的官方RESTful API。
    该API负责处理从小说上传、游戏配置到最终剧本杀游戏包内容生成的完整流程。
    它严格依赖于《核心数据模型规约 V1.2》。
  version: "1.0.0"

servers:
  - url: https://api.mi-tan.tech/v1
    description: 生产环境服务器
  - url: http://localhost:8000/v1
    description: 本地开发服务器

tags:
  - name: "项目生命周期 (Project Lifecycle)"
    description: "管理一个从小说到剧本杀项目的完整创建和配置流程。"
  - name: "游戏内容获取 (Game Content Retrieval)"
    description: "获取已生成的具体游戏材料，如角色剧本和线索卡。"

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
        提交游戏的核心配置（如凶手、玩家角色），触发游戏化故事模型（GSM）的生成。
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

  /projects/{projectId}/gsm:
    get:
      tags:
        - "游戏内容获取 (Game Content Retrieval)"
      summary: "获取完整的游戏化故事模型(GSM)"
      description: "在GSM生成完毕后，获取完整的GSM数据。这是所有游戏内容的“上帝视角”数据源。"
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
                # 直接引用已冻结的数据契约
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
      summary: "获取指定角色的剧本"
      description: "为指定角色生成并获取其个人剧本，包含其所需的所有公开/私密信息、目标和谎言。"
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
      summary: "获取游戏线索集"
      description: "获取为本场游戏生成的所有线索卡片（包括关键线索和迷惑性线索）。"
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
          enum: [CSM_PROCESSING, CSM_COMPLETE, GSM_PROCESSING, GSM_COMPLETE, FAILED]
        message:
          type: string
          description: "提供关于当前状态的额外信息，或在失败时提供错误详情。"
        csm:
          description: "当status为'CSM_COMPLETE'或更高时，此字段包含完整的CSM对象。"
          # 直接引用已冻结的数据契约
          $ref: 'https://mi-tan.tech/schemas/csm.spec.v1.2.yaml'
    
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
                
    CharacterScript:
      type: object
      description: "为单个玩家生成的剧本内容，是GSM中该角色知识的视图。"
      properties:
        characterId:
          type: string
        characterName:
          type: string
        isMurderer:
          type: boolean
        publicKnowledge:
          type: string
        privateKnowledge:
          type: string
        secrets:
          type: array
          items:
            type: string
        alibiClaims:
          type: array
          items:
            type: object
            properties:
              claim:
                type: string
        liesToTell:
          type: array
          items:
            type: object
            properties:
              triggeringTopic:
                type: string
              lieContent:
                type: string
        goals:
          type: array
          items:
            type: object
            properties:
              description:
                type: string
              type:
                type: string
                enum: [PRIMARY, SECONDARY, HIDDEN]
                
    ClueCard:
      type: object
      description: "一张线索卡的数据结构。"
      properties:
        clueId:
          type: string
        title:
          type: string
          example: "一张揉皱的信纸"
        description:
          type: string
          example: "信纸上写着：'我再也无法忍受他的敲诈了！今晚必须做个了断！' 字迹似乎属于管家。"
        type:
          type: string
          enum: [CRITICAL, MISLEADING]
          description: "【GM可见】线索类型：关键线索或迷惑性线索。"

    Error:
      type: object
      properties:
        errorCode:
          type: string
        message:
          type: string
```

---

### **3. 下一步行动建议**

1.  **契约评审与冻结:** 将此API规约 V1.0 提交给前端和后端团队进行评审。一旦达成共识，应将其状态更新为 **已批准 (Approved)** 并冻结。
2.  **Mock Server搭建:** 前端团队应立即使用此规约，通过工具（如Prism, Stoplight.io）搭建一个Mock Server。这将使他们能够**完全独立地**开始UI/UX开发，无需等待任何后端代码。
3.  **后端接口骨架生成:** 后端团队可以使用代码生成工具，基于此规约自动生成服务器端的接口骨架和请求/响应模型类，从而专注于核心业务逻辑（即DP1, DP2, DP4的实现）的开发。
4.  **契约测试 (Contract Testing):** 在CI/CD流程中引入契约测试。确保后端服务的任何变更都不会破坏与此API规约的兼容性，从而保障系统的健壮性。

这份API规约已经为高效的并行开发铺平了道路。下一步，我们将深入项目中最具挑战性的核心——LLM的工程化。请根据ADD，生成《LLM Prompt工程与实现策略》。