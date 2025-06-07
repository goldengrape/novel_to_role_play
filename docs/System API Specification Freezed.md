

# 《谜探》系统API规约 (System API Specification)

- **规约版本:** **1.3**
- **状态:** **已批准**
- **编制者:** Gemini 2.5 Pro
- **日期:** 2024年6月9日
- **修订历史:**
  - **V1.3 (2024-06-09):** 依据《SAS审查意见》进行最终修订，以达成“可批准”状态。
    1.  **[行为定义-MEDIUM]** 明确了`POST /.../configure`的重复调用行为为“覆盖重生成”，消除了行为歧义 (CR-SAS-01)。
    2.  **[契约增强-LOW]** 补充了`Error`模型的`errorCode`示例，增强了错误处理契约的完整性 (CR-SAS-02)。
  - **V1.2 (2024-06-08):** 根据《SAS审查意见 V1.1》进行修订，统一异步操作、细化状态模型、增强资源对称性。
  - **V1.1 (2024-06-08):** 根据ADD评审意见进行重大修订。
  - **V1.0 (2024-06-07):** 初始版本。

---

### **1. 设计哲学与核心原则 (无变化)**

1.  **契约优先 (Contract First):** 本API规约是开发的起点，而非终点。所有相关方必须严格遵守此契约。
2.  **资源导向 (Resource-Oriented):** API围绕“项目(Project)”这一核心资源进行设计，遵循RESTful架构风格的最佳实践。
3.  **异步与可预测 (Asynchronous & Predictable):** 核心生成任务（CSM、GSM、导出）被设计为**统一的异步操作**。客户端通过轮询获取任务状态，确保了行为的可预测性，降低了客户端的实现复杂性。
4.  **严格模式与务实权衡 (Strict Typing & Pragmatic Trade-offs):** 所有请求和响应体都严格引用数据模型契约。同时，我们做出明确的设计权衡：为降低前端开发复杂性，部分响应体（如`CharacterScript`）包含少量冗余数据（如`characterName`），这是一个为了开发便利性而接受的、可控的非规范化设计。

---

### **2. OpenAPI 3.0 规约 (最终修订版)**

```yaml
# mi-tan-api.spec.yaml (v1.3)

openapi: 3.0.1
info:
  title: "“谜探”智能转换引擎 API"
  description: |
    《谜探》项目的官方RESTful API。
    该API负责处理从小说上传、游戏配置到最终剧本杀游戏包内容生成的完整流程。
    它严格依赖于《核心数据模型规约 V1.2》。
    本版本(v1.3)已根据最终的同行评审意见进行修订，达到了可批准的冻结状态。
  version: "1.3.0"

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
      # ... (内容无变化) ...
      tags: ["项目生命周期 (Project Lifecycle)"]
      summary: "创建新项目并上传小说"
      description: |
        上传一部小说文件，启动一个新项目。
        后端将异步开始处理小说，生成核心故事模型 (CSM)。
        这是一个长时间运行的任务。
      # ... (requestBody, responses 无变化) ...

  /projects/{projectId}:
    get:
      # ... (内容无变化) ...
      tags: ["项目生命周期 (Project Lifecycle)"]
      summary: "获取项目状态和CSM"
      description: |
        查询指定项目的当前状态。
        如果CSM已生成完毕，响应中将包含完整的CSM数据。
        当导出完成后，响应中将包含下载链接。
      # ... (parameters, responses 无变化) ...

  /projects/{projectId}/configure:
    post:
      tags: ["项目生命周期 (Project Lifecycle)"]
      summary: "配置游戏并触发GSM生成"
      # [修订 CR-SAS-01] 明确了重复配置的行为
      description: |
        提交游戏的核心配置（如凶手、玩家角色）和生成参数，触发游戏化故事模型（GSM）的生成。
        这是一个异步操作。
        如果项目已存在一个GSM，再次调用此接口将会使用新的配置覆盖并重新生成GSM。
        客户端在轮询项目状态时，会观察到状态从 `GSM_COMPLETE` 重新变为 `GSM_PROCESSING`。
      parameters:
        - name: projectId
          in: path
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/ConfigurationRequest' }
      responses:
        '202':
          description: "配置已接受，GSM生成任务已启动。"
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ProjectStatus' }
        '409':
          description: "配置冲突，例如CSM尚未生成完毕。"
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }

  /projects/{projectId}/export:
    post:
      # ... (内容无变化) ...
      tags: ["项目生命周期 (Project Lifecycle)"]
      summary: "【异步】请求导出完整的剧本杀游戏包"
      description: |
        请求生成一个完整的剧本杀游戏包。
        此操作【始终】是异步的。它会立即返回202，并更新项目状态为`EXPORTING`。
        客户端应通过轮询`GET /projects/{projectId}`来跟踪进度。
        当状态变为`EXPORT_COMPLETE`时，轮询响应中将包含可供下载的`exportFileUrl`。
      # ... (parameters, requestBody, responses 无变化) ...

  /projects/{projectId}/gsm:
    get:
      # ... (内容无变化) ...
      tags: ["游戏内容获取 (Game Content Retrieval)"]
      summary: "获取完整的游戏化故事模型(GSM)"
      # ... (description, parameters, responses 无变化) ...

  /projects/{projectId}/scripts/{characterId}:
    get:
      # ... (内容无变化) ...
      tags: ["游戏内容获取 (Game Content Retrieval)"]
      summary: "获取指定角色的剧本"
      # ... (description, parameters, responses 无变化) ...

  /projects/{projectId}/clues:
    get:
      # ... (内容无变化) ...
      tags: ["游戏内容获取 (Game Content Retrieval)"]
      summary: "获取游戏线索集"
      # ... (description, parameters, responses 无变化) ...

  /projects/{projectId}/clues/{clueId}:
    get:
      # ... (内容无变化) ...
      tags: ["游戏内容获取 (Game Content Retrieval)"]
      summary: "获取单张线索卡详情"
      # ... (description, parameters, responses 无变化) ...

components:
  schemas:
    ProjectStatus:
      # ... (内容无变化) ...
      type: object
      properties:
        projectId: { type: string }
        status:
          type: string
          enum: [CSM_PROCESSING, CSM_COMPLETE, GSM_PROCESSING, GSM_COMPLETE, EXPORTING, EXPORT_COMPLETE, CSM_FAILED, GSM_FAILED, EXPORT_FAILED]
        message: { type: string, description: "提供关于当前状态的额外信息，或在失败时提供具体的错误详情。" }
        csm: { $ref: 'https://mi-tan.tech/schemas/csm.spec.v1.2.yaml' }
        exportFileUrl: { type: string, format: uri, description: "当状态为 'EXPORT_COMPLETE' 时，提供完整的游戏包下载链接。" }

    ConfigurationRequest:
      # ... (内容无变化) ...
      type: object
      required: [murdererId, playerCharacters]
      properties:
        murdererId: { type: string, description: "用户指定的凶手角色ID..." }
        playerCharacters:
          type: array
          items: { type: object, properties: { characterId: { type: string }, playerName: { type: string } } }
        generationParams:
          type: object
          properties:
            difficulty: { type: string, enum: [EASY, NORMAL, HARD], default: NORMAL }
            murdererProtection: { type: boolean, default: true }
            emotionalTone: { type: string, enum: [NEUTRAL, TENSE, DARK, HUMOROUS], default: NEUTRAL }

    CharacterScript:
      # ... (内容无变化) ...
      type: object
      properties: { characterId: { type: string }, characterName: { type: string }, isMurderer: { type: boolean }, ... }

    ClueCard:
      # ... (内容无变化) ...
      type: object
      properties: { clueId: { type: string }, title: { type: string }, description: { type: string }, type: { type: string, enum: [CRITICAL, MISLEADING] } }

    Error:
      type: object
      # [修订 CR-SAS-02] 增强了错误码契约
      description: |
        通用的错误响应体。`errorCode`是一个机器可读的字符串，以便客户端进行程序化处理。
        这是一个非详尽的示例列表，后端可能会根据需要返回其他错误码。
        常见的errorCode可能包括：
        - `PROJECT_NOT_FOUND`
        - `RESOURCE_NOT_FOUND` (例如, characterId or clueId 不存在)
        - `CSM_NOT_READY` (在请求GSM或导出时CSM未完成)
        - `GSM_NOT_READY` (在请求导出时GSM未完成)
        - `INVALID_REQUEST_BODY`
        - `INVALID_FILE_FORMAT`
        - `MURDERER_ID_INVALID`
        - `INTERNAL_GENERATION_ERROR` (LLM或其他内部服务失败)
      properties:
        errorCode:
          type: string
          description: "机器可读的错误代码。"
          example: "CSM_NOT_READY"
        message:
          type: string
          description: "对错误的详细、人类可读的描述。"
          example: "无法配置游戏，因为核心故事模型(CSM)尚未生成完毕。"
```

---

