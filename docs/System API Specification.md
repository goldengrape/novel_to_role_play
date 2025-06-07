好的，指令收到。

我将完全采纳这份由“公理设计专家”出具的、极其严谨的《系统API规约 V1.2 同行评审报告》。这份报告中的两点修改建议，虽然细微，却直击要害，是确保API契约从“接近完美”到“理论完备”的关键一步。

现在，我将根据这份评审报告（CR-SAS-01 和 CR-SAS-02），对规约进行最终修订，发布一个真正消除了所有已知歧义、达到“可冻结”状态的 **V1.3** 版本。

---

# 《谜探》系统API规约 (System API Specification)

- **规约版本:** **1.3**
- **状态:** **草案 (最终修订版/待批准)**
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

### **3. 最终行动建议**

随着《SAS审查意见》中所有建议的采纳，本《系统API规约 V1.3》已经达到了理论上无歧义、工程上可执行的“蓝图”状态。它现在可以作为所有开发团队的“单一真相来源”。

1.  **契约批准与冻结:** 将此API规约 **V1.3** 提交给所有利益相关方。一旦确认无异议，应将其状态更新为 **已批准 (Approved)** 并版本冻结。**此规约此后不应再做破坏性变更。**
2.  **并行开发启动:**
    *   **前端团队:** 可以立即使用此V1.3规约搭建或更新Mock Server，开始独立开发完整的用户界面，包括对重复配置行为（状态重置）和更精细的错误代码处理。
    *   **后端团队:** 可以基于此V1.3规约生成最终的服务器端接口骨架和模型类，开始并行实现CSM、GSM生成、覆盖配置逻辑，以及统一的异步导出逻辑。
3.  **强制契约测试 (Contract Testing):** 再次强调，在CI/CD流程中引入契约测试（如使用Pact）至关重要。这可以自动验证后端服务的实现始终与这份已批准的V1.3规约保持一致，从根本上杜绝集成阶段的意外问题。

这份V1.3规约是公理设计理论、函数式编程思想与严谨工程实践（包括同行评审）闭环的最终产物。它为《谜探》项目的高效、高质量交付铺平了最坚实的第一步。

下一步，我们将深入项目中最具挑战性的核心——LLM的工程化。请根据ADD，生成《LLM Prompt工程与实现策略》。

Gemini 2.5 Pro 0605