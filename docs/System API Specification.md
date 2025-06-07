好的，指令收到。

我将以公理设计专家的身份，完全采纳《SAS审查意见.md》中的所有**核心修改建议**和**优化建议**。这份评审报告是设计闭环的最后关键一步，它将确保我们的API规约不仅在理论上与公理设计（ADD）保持一致，更在工程实践中达到健壮、可预测和可扩展的黄金标准。

现在，我将根据评审意见，对《系统API规约 V1.1》进行最终修订，发布一个真正严谨、无歧义的**V1.2**版本。

---

# 《谜探》系统API规约 (System API Specification)

- **规约版本:** **1.2**
- **状态:** **草案 (终稿/待批准)**
- **编制者:** Gemini 2.5 Pro
- **日期:** 2024年6月8日
- **修订历史:**
  - **V1.2 (2024-06-08):** 根据《SAS审查意见 V1.1》进行最终修订，以达成“可批准”状态。
    1.  **[行为修正-CRITICAL]** 统一了`/export`端点的行为，强制其为纯异步操作，消除了响应的二义性，以增强API的可预测性。
    2.  **[功能增强-MAJOR]** 细化了`ProjectStatus`中的失败状态，以增强系统的可诊断性，精确映射ADD中的处理阶段。
    3.  **[功能增强-RECOMMENDED]** 增加了`GET /.../clues/{clueId}`端点，以增强资源操作的对称性，并遵循RESTful最佳实践。
    4.  **[功能补充]** 在`ProjectStatus`中增加了`exportFileUrl`字段，用于在导出成功后提供下载链接。
  - **V1.1 (2024-06-08):** 根据ADD评审意见进行重大修订。
  - **V1.0 (2024-06-07):** 初始版本。

---

### **1. 设计哲学与核心原则 (经最终修订)**

1.  **契约优先 (Contract First):** 本API规约是开发的起点，而非终点。所有相关方必须严格遵守此契约。
2.  **资源导向 (Resource-Oriented):** API围绕“项目(Project)”这一核心资源进行设计，遵循RESTful架构风格的最佳实践。
3.  **异步与可预测 (Asynchronous & Predictable):** 核心生成任务（CSM、GSM、导出）被设计为**统一的异步操作**。客户端通过轮询获取任务状态，确保了行为的可预测性，降低了客户端的实现复杂性。
4.  **严格模式与务实权衡 (Strict Typing & Pragmatic Trade-offs):** 所有请求和响应体都严格引用数据模型契约。同时，我们做出明确的设计权衡：为降低前端开发复杂性，部分响应体（如`CharacterScript`）包含少量冗余数据（如`characterName`），这是一个为了开发便利性而接受的、可控的非规范化设计。

---

### **2. OpenAPI 3.0 规约 (修订版)**

```yaml
# mi-tan-api.spec.yaml (v1.2)

openapi: 3.0.1
info:
  title: "“谜探”智能转换引擎 API"
  description: |
    《谜探》项目的官方RESTful API。
    该API负责处理从小说上传、游戏配置到最终剧本杀游戏包内容生成的完整流程。
    它严格依赖于《核心数据模型规约 V1.2》。
    本版本(v1.2)已根据最终的公理设计(ADD)符合性评审意见进行修订。
  version: "1.2.0"

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
                novelFile: { type: string, format: binary, description: "小说文件 (e.g., .txt, .md, .epub)。" }
                projectName: { type: string, description: "用户为该项目指定的名称。" }
      responses:
        '202':
          description: "请求已接受，CSM生成任务已在后台启动。"
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ProjectStatus' }
        '400':
          description: "错误的请求，例如文件未上传或格式不受支持。"
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }

  /projects/{projectId}:
    get:
      # ... (内容无变化) ...
      tags:
        - "项目生命周期 (Project Lifecycle)"
      summary: "获取项目状态和CSM"
      description: |
        查询指定项目的当前状态。
        如果CSM已生成完毕，响应中将包含完整的CSM数据。
        当导出完成后，响应中将包含下载链接。
      parameters:
        - name: projectId
          in: path
          required: true
          schema: { type: string }
      responses:
        '200':
          description: "成功获取项目状态。"
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ProjectStatus' }
        '404':
          description: "未找到指定ID的项目。"
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }

  /projects/{projectId}/configure:
    post:
      # ... (内容无变化) ...
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
      tags:
        - "项目生命周期 (Project Lifecycle)"
      summary: "【异步】请求导出完整的剧本杀游戏包"
      # [修订] 澄清了行为，删除了200响应
      description: |
        请求生成一个完整的剧本杀游戏包。
        此操作【始终】是异步的。它会立即返回202，并更新项目状态为`EXPORTING`。
        客户端应通过轮询`GET /projects/{projectId}`来跟踪进度。
        当状态变为`EXPORT_COMPLETE`时，轮询响应中将包含可供下载的`exportFileUrl`。
      parameters:
        - name: projectId
          in: path
          required: true
          schema: { type: string }
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
        '202':
          description: "导出任务已成功启动。请轮询项目状态以获取最终的下载链接。"
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ProjectStatus' }
        '404':
          description: "项目或GSM不存在，无法导出。"
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }

  /projects/{projectId}/gsm:
    get:
      # ... (内容无变化) ...
      tags:
        - "游戏内容获取 (Game Content Retrieval)"
      summary: "获取完整的游戏化故事模型(GSM)"
      description: "在GSM生成完毕后，获取完整的GSM数据。这是所有游戏内容的“上帝视角”数据源。"
      parameters:
        - name: projectId
          in: path
          required: true
          schema: { type: string }
      responses:
        '200':
          description: "成功获取GSM。"
          content:
            application/json:
              schema: { $ref: 'https://mi-tan.tech/schemas/gsm.spec.v1.2.yaml' }
        '404':
          description: "GSM尚未生成或项目不存在。"
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }

  /projects/{projectId}/scripts/{characterId}:
    get:
      # ... (内容无变化) ...
      tags:
        - "游戏内容获取 (Game Content Retrieval)"
      summary: "获取指定角色的剧本"
      description: "从已生成的GSM中，提取并格式化指定角色的个人剧本。此操作不涉及新的LLM调用。"
      parameters:
        - name: projectId
          in: path
          required: true
          schema: { type: string }
        - name: characterId
          in: path
          required: true
          schema: { type: string }
      responses:
        '200':
          description: "成功获取角色剧本。"
          content:
            application/json:
              schema: { $ref: '#/components/schemas/CharacterScript' }
        '404':
          description: "项目、GSM或指定角色不存在。"
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }

  /projects/{projectId}/clues:
    get:
      # ... (内容无变化) ...
      tags:
        - "游戏内容获取 (Game Content Retrieval)"
      summary: "获取游戏线索集"
      description: "从已生成的GSM中，提取为本场游戏生成的所有线索卡片。此操作不涉及新的LLM调用。"
      parameters:
        - name: projectId
          in: path
          required: true
          schema: { type: string }
      responses:
        '200':
          description: "成功获取线索集。"
          content:
            application/json:
              schema:
                type: array
                items: { $ref: '#/components/schemas/ClueCard' }
        '404':
          description: "项目或GSM不存在。"
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }

  # [新增] 增强资源操作的对称性
  /projects/{projectId}/clues/{clueId}:
    get:
      tags:
        - "游戏内容获取 (Game Content Retrieval)"
      summary: "获取单张线索卡详情"
      description: "从已生成的GSM中，提取指定ID的单张线索卡。此操作为`GET /clues`的补充，遵循RESTful设计模式。"
      parameters:
        - name: projectId
          in: path
          required: true
          schema: { type: string }
        - name: clueId
          in: path
          required: true
          schema: { type: string }
      responses:
        '200':
          description: "成功获取线索卡详情。"
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ClueCard' }
        '404':
          description: "项目、GSM或指定线索不存在。"
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }

components:
  schemas:
    ProjectStatus:
      type: object
      properties:
        projectId:
          type: string
        status:
          type: string
          # [修订] 细化了失败状态以增强可诊断性
          enum:
            - CSM_PROCESSING
            - CSM_COMPLETE
            - GSM_PROCESSING
            - GSM_COMPLETE
            - EXPORTING
            - EXPORT_COMPLETE
            - CSM_FAILED
            - GSM_FAILED
            - EXPORT_FAILED
        message:
          type: string
          description: "提供关于当前状态的额外信息，或在失败时提供具体的错误详情。"
        csm:
          description: "当status为'CSM_COMPLETE'或更高时，此字段包含完整的CSM对象。"
          $ref: 'https://mi-tan.tech/schemas/csm.spec.v1.2.yaml'
        exportFileUrl: # <-- [新增]
          type: string
          format: uri
          description: "当状态为 'EXPORT_COMPLETE' 时，提供完整的游戏包下载链接。"

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
      properties: { errorCode: { type: string }, message: { type: string } }
```

---

### **3. 最终行动建议**

经过最终的同行评审和修订，本《系统API规约 V1.2》已经解决了所有已知的歧义和不一致性，达到了可作为工程蓝图的“冻结”标准。

1.  **契约批准与冻结:** 将此API规约 **V1.2** 提交给所有利益相关方。一旦确认无异议，应将其状态更新为 **已批准 (Approved)** 并版本冻结。**此规约此后不应再做破坏性变更。**
2.  **并行开发启动:**
    *   **前端团队:** 可以立即使用此V1.2规约搭建或更新Mock Server，开始独立开发完整的用户界面，包括对异步任务（导出）、具体失败状态和新增线索详情页面的处理。
    *   **后端团队:** 可以基于此V1.2规约生成最终的服务器端接口骨架和模型类，开始并行实现CSM、GSM生成，以及统一的异步导出逻辑。
3.  **强制契约测试 (Contract Testing):** 强烈建议在CI/CD流程中引入契约测试（如使用Pact）。这可以自动验证后端服务的实现始终与这份已批准的V1.2规约保持一致，从根本上杜绝集成阶段的意外问题。

这份V1.2规约是公理设计理论与RESTful工程实践结合的产物，为《谜探》项目的高效、高质量交付铺平了最坚实的第一步。下一步，我们将深入项目中最具挑战性的核心——LLM的工程化。请根据ADD，生成《LLM Prompt工程与实现策略》。

Gemini 2.5 Pro 0605