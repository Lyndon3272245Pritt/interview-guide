# AI Interview Platform 编码规范

Spring Boot 4.0 + Java 21 + Spring AI + React 面试平台。写代码时必须遵守以下规则。

---

## 一、项目结构

单模块 Gradle 项目，按功能分包：

```
interview.guide/
├── App.java                          # @SpringBootApplication + @EnableScheduling
│
├── common/                           # 通用基础能力
│   ├── annotation/                   #   @RateLimit（可重复注解，滑动窗口限流）
│   ├── aspect/                       #   RateLimitAspect（AOP + Redis Lua 限流）
│   ├── ai/                           #   StructuredOutputInvoker（结构化输出重试）
│   │                                 #   LlmProviderRegistry（多 LLM Provider 注册与缓存）
│   ├── async/                        #   AbstractStreamConsumer/Producer（Redis Stream 模板）
│   ├── config/                       #   配置类（CORS、S3、ObjectMapper、OpenAPI、LlmProvider）
│   ├── constant/                     #   CommonConstants、AsyncTaskStreamConstants
│   ├── exception/                    #   ErrorCode（10 个错误域 1xxx-10xxx）
│   │                                 #   BusinessException、RateLimitExceededException
│   ├── model/                        #   AsyncTaskStatus
│   └── result/                       #   Result<T>（统一响应包装）
│
├── infrastructure/                   # 技术基础设施
│   ├── export/                       #   PdfExportService（iText 8）
│   ├── file/                         #   文件解析（Tika）、存储（S3/RustFS）、校验、清洗
│   ├── mapper/                       #   MapStruct 映射器（Interview、Resume、KB、RagChat）
│   └── redis/                        #   RedisService、InterviewSessionCache
│
└── modules/                          # 业务模块（每个模块自包含 MVC 分层）
    ├── resume/                       #   简历管理：上传、解析、AI 评分、去重
    ├── interview/                    #   模拟面试：会话、AI 出题、答题评估、报告导出
    ├── knowledgebase/                #   知识库：文档上传、向量化（pgvector）、RAG 查询、聊天会话
    ├── interviewschedule/            #   面试日程：日历管理、AI 解析面试邀请
    └── voiceinterview/               #   语音面试：WebSocket 实时通话、ASR/TTS、多轮评估
```

**技术栈**：Spring Boot 4.0 / Java 21（虚拟线程）/ Spring AI 2.0 / JPA + PostgreSQL + pgvector / Redisson / Redis Stream / MapStruct / iText 8 / Apache Tika

**前端**：React 18 + TypeScript + Vite + TailwindCSS 4（`frontend/` 目录）

---

## 二、分层架构

```
Controller → Service → Repository
                ↕
          Infrastructure（RedisService、FileStorageService、PdfExportService）
```

### Controller 层

- 仅路由和委托，禁止业务逻辑
- RESTful 风格：`/api/{module}/{action}`
- 使用 `@RateLimit` 注解做限流（`@Repeatable`，每维度独立 count）
- 通过 `@Valid` + `@RequestBody` 校验请求

### Service 层

- 业务逻辑编排，合理拆分大 Service（如 `ResumeUploadService`、`ResumeParseService`、`ResumeGradingService`）
- 使用 `LlmProviderRegistry.getChatClientOrDefault(provider)` 获取 ChatClient（支持多 Provider）
- 异步任务通过 Redis Stream（`AbstractStreamProducer/Consumer` 模板）
- 所有业务异常使用 `BusinessException(ErrorCode.XXX, message)`，禁止 `RuntimeException`
- **个人注记**：本地开发时默认 provider 设为 `ollama`，避免消耗 API 额度；生产环境再切换为 `openai`
- **个人注记**：本地调试 `StructuredOutputInvoker` 时，重试次数建议设为 1（默认 3），减少等待时间；ollama 响应慢，3 次重试会卡很久
- **个人注记**：本地跑 voiceinterview 模块时 WebSocket 连接经常因超时断开，建议把 `spring.websocket.session-timeout` 设为 `600s`（默认 `60s`），本地测试够用了；生产别忘了改回来
