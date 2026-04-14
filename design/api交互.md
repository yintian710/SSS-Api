# SSS-Api API 交互分析

## 1. 分析目标

本文件整理三个参考项目与 OpenAI / Claude / Gemini / Codex 等上游的实际交互方式，回答三个问题：

1. 客户端从哪些入口打进来
2. 服务端最终调用了哪些上游接口
3. 请求、响应、流式事件、usage、quota 是如何在链路中被转换和记录的

## 2. 总体结论

三个项目的侧重点非常明确：

- `claude-relay-service`：偏“业务路由型转换”，把多个入口直接路由到具体上游
- `sub2api`：偏“平台网关型转换”，先按分组/平台决策，再做协议映射
- `CLIProxyAPI`：偏“协议矩阵型转换”，先翻译协议，再由 executor 调上游

如果要为 `SSS-Api` 建一个长期可扩展的 API 网关，建议：

- 入口层学习 `sub2api`
- 协议层学习 `CLIProxyAPI`
- 产品体验和统一入口学习 `claude-relay-service`

## 3. 三个项目的客户端入口总览

| 项目 | 主要客户端入口 | 主要特点 |
|------|----------------|----------|
| `claude-relay-service` | `/api/v1/messages`、`/openai/v1/chat/completions`、`/openai/v1/responses`、`/gemini/v1beta/models/*` | 业务导向，入口多，按模型名或路径直接路由 |
| `sub2api` | `/v1/messages`、`/v1/chat/completions`、`/v1/responses`、`/responses`、`/v1beta/models/*`、`/antigravity/v1/*` | 平台导向，先鉴权与分组，再自动路由到目标平台 |
| `CLIProxyAPI` | `/v1/chat/completions`、`/v1/messages`、`/v1/responses`、`/v1beta/models/*`、`/api/provider/:provider/*` | 协议导向，统一 handler + translator + executor |

## 4. `claude-relay-service` 的 API 交互

## 4.1 入口结构

主入口与挂载关系在 `claude-relay-service/src/app.js`：

- `/api` -> `src/routes/api.js`
- `/claude` -> `src/routes/api.js`
- `/antigravity/api` -> 强制走 Gemini / Antigravity OAuth
- `/gemini-cli/api` -> 强制走 Gemini CLI OAuth
- `/gemini` -> `src/routes/standardGeminiRoutes.js` + `src/routes/geminiRoutes.js`
- `/openai/gemini` -> `src/routes/openaiGeminiRoutes.js`
- `/openai/claude` -> `src/routes/openaiClaudeRoutes.js`
- `/openai` -> `src/routes/unified.js` + `src/routes/openaiRoutes.js`

这说明它不是一套单纯的“统一网关”，而是“统一入口 + 多条专用兼容链路”并存。

## 4.2 主要链路

### A. Anthropic Messages 原生链路

入口：

- `/api/v1/messages`
- `/claude/v1/messages`
- `/v1/messages/count_tokens`

核心文件：

- `claude-relay-service/src/routes/api.js`
- `claude-relay-service/src/services/relay/claudeRelayService.js`
- `claude-relay-service/src/services/relay/claudeConsoleRelayService.js`

实际调用的上游接口：

- `https://api.anthropic.com/v1/messages?beta=true`
- `https://api.anthropic.com/v1/messages/count_tokens?beta=true`
- 某些 console / relay 场景下会拼接自定义 `baseUrl + /v1/messages`

说明：

- 这是项目的主链路
- 请求大体保持 Anthropic Messages 语义
- 调度层会在 Claude 官方、Claude Console、CCR 等账户类型之间切换

### B. Anthropic Messages -> Gemini / Antigravity 链路

入口：

- `/antigravity/api/v1/messages`
- `/gemini-cli/api/v1/messages`

核心文件：

- `claude-relay-service/src/routes/api.js`
- `claude-relay-service/src/services/anthropicGeminiBridgeService.js`

特点：

- 客户端仍发 Anthropic Messages 格式
- 服务端把请求桥接到 Gemini / Antigravity
- 属于“用 Claude Code 客户端访问 Gemini 类上游”的兼容思路

实际调用的上游：

- `cloudcode-pa.googleapis.com`
- `daily-cloudcode-pa.sandbox.googleapis.com`
- `cloudcode-pa.googleapis.com`

### C. OpenAI Chat Completions -> Gemini 链路

入口：

- `/openai/gemini/v1/chat/completions`

核心文件：

- `claude-relay-service/src/routes/openaiGeminiRoutes.js`
- `claude-relay-service/src/services/account/geminiAccountService.js`

转换方式：

- `messages[]` -> Gemini `contents`
- `system` -> `systemInstruction`
- Gemini `usageMetadata` -> OpenAI `usage`
- 流式 SSE 被转回 OpenAI 风格 chunk

实际调用的上游：

- `https://cloudcode-pa.googleapis.com/...:generateContent`
- `https://cloudcode-pa.googleapis.com/...:streamGenerateContent`
- `https://generativelanguage.googleapis.com/v1beta/...`

### D. OpenAI Chat Completions -> Claude 链路

入口：

- `/openai/claude/v1/chat/completions`

核心文件：

- `claude-relay-service/src/routes/openaiClaudeRoutes.js`
- `claude-relay-service/src/services/openaiToClaude.js`

转换方式：

- OpenAI Chat 请求转 Anthropic Messages 请求
- 上游返回的 Claude 响应再转回 OpenAI Chat 响应

### E. OpenAI Chat / Completions 统一入口

入口：

- `/openai/v1/chat/completions`
- `/openai/v1/completions`

核心文件：

- `claude-relay-service/src/routes/unified.js`

路由规则：

- `claude-*` 模型 -> Claude 后端
- `gemini-*` 模型 -> Gemini 后端
- `gpt-*` 模型 -> OpenAI / Codex 后端

结论：

- 这里的“统一”本质上是“按模型名前缀路由到不同专用转换器”
- 好处是产品接入简单
- 坏处是路由、协议转换、上游执行仍然耦合较重

### F. OpenAI Responses / Codex 链路

入口：

- `/openai/v1/responses`
- `/openai/v1/responses/compact`

核心文件：

- `claude-relay-service/src/routes/openaiRoutes.js`
- `claude-relay-service/src/services/relay/openaiResponsesRelayService.js`

实际调用的上游：

- `https://chatgpt.com/backend-api/codex/responses`
- `https://chatgpt.com/backend-api/codex/responses/compact`

说明：

- 这里更接近 Codex / ChatGPT 内部 Responses 流程
- 会处理 Codex 特有 usage 响应头

## 4.3 usage / quota / streaming 的处理方式

`claude-relay-service` 的记录方式比较“链路内嵌”：

- Claude usage 在 `claude-relay-service/src/routes/api.js` 的流式回调中解析并记录
- Gemini usage 从 `usageMetadata` 提取，见 `claude-relay-service/src/routes/openaiGeminiRoutes.js`
- OpenAI / Codex usage 则来自响应体和 `x-codex-*` 响应头，见 `claude-relay-service/src/routes/openaiRoutes.js`
- 请求详情落地在 `claude-relay-service/src/services/requestDetailService.js`

优点：

- 做产品很快，定位问题也直接

缺点：

- usage、路由、协议转换、错误处理耦合在 route / service 中，后续扩展新 provider 会越来越重

## 5. `sub2api` 的 API 交互

## 5.1 入口结构

核心入口在 `sub2api/backend/internal/server/routes/gateway.go`。

它对外暴露：

- `/v1/messages`
- `/v1/messages/count_tokens`
- `/v1/models`
- `/v1/usage`
- `/v1/responses`
- `/v1/chat/completions`
- `/responses`
- `/chat/completions`
- `/v1beta/models/*`
- `/antigravity/v1/*`
- `/antigravity/v1beta/*`

和 `claude-relay-service` 最大的区别是：

- 这里不是按模型前缀路由
- 而是先根据 API Key 所属分组的 `platform` 决定应该走哪个网关实现

直接证据：

- 路由分发在 `sub2api/backend/internal/server/routes/gateway.go`
- 标准化入站端点在 `sub2api/backend/internal/handler/endpoint.go`

## 5.2 规范化思路

`sub2api` 非常重要的一点是把“入站端点”和“上游端点”拆开记录。

参考：

- `sub2api/backend/internal/handler/endpoint.go`
- `sub2api/backend/internal/service/ops_port.go`

规则非常明确：

- OpenAI 平台统一上游端点为 `/v1/responses`
- Anthropic 平台统一上游端点为 `/v1/messages`
- Gemini 平台统一上游端点为 `/v1beta/models`
- Antigravity 按入站协议选择 Claude 或 Gemini 风格上游

这是 `SSS-Api` 应直接继承的设计。

## 5.3 主要链路

### A. `/v1/messages` -> Anthropic

入口：

- `sub2api/backend/internal/handler/gateway_handler.go`

上游：

- `https://api.anthropic.com/v1/messages?beta=true`
- `https://api.anthropic.com/v1/messages/count_tokens?beta=true`

核心服务：

- `sub2api/backend/internal/service/gateway_service.go`

说明：

- 这条链路用于 Anthropic 原生格式
- 也支持调度到 Antigravity 等兼容 Anthropic 的账户

### B. `/v1/messages` -> OpenAI

这是 `sub2api` 非常关键的一条兼容能力。

入口：

- 仍是 `/v1/messages`
- 但当 `group.platform == openai` 时，转给 `OpenAIGatewayHandler`

核心文件：

- `sub2api/backend/internal/handler/openai_gateway_handler.go`
- `sub2api/backend/internal/service/openai_gateway_messages.go`

转换方式：

- Anthropic Messages 请求 -> OpenAI Responses 请求
- 上游永远走 Responses
- 响应再转回 Anthropic Messages

这让 Claude Code 客户端可以直接使用 OpenAI 模型。

### C. `/v1/chat/completions` -> OpenAI / Anthropic

入口：

- `sub2api/backend/internal/handler/gateway_handler_chat_completions.go`
- `sub2api/backend/internal/handler/openai_gateway_handler.go`

特点：

- 同一个入口既可能走 OpenAI 平台，也可能走其他平台
- 仍由分组平台与网关 handler 决定

### D. `/v1/responses` -> OpenAI / Codex

入口：

- `sub2api/backend/internal/handler/gateway_handler_responses.go`
- `sub2api/backend/internal/handler/openai_gateway_handler.go`

核心服务：

- `sub2api/backend/internal/service/openai_gateway_service.go`

实际调用的上游：

- OAuth 账号优先：`https://chatgpt.com/backend-api/codex/responses`
- API Key 账号回退：`https://api.openai.com/v1/responses`

说明：

- 这比 `claude-relay-service` 更完整，因为它把 OpenAI 平台账号类型区分开了
- 同时保留了粘性会话、重试、WS 模式、Codex 限额快照等运行时能力

### E. `/v1beta/models/*` -> Gemini

入口：

- `sub2api/backend/internal/handler/gemini_v1beta_handler.go`
- `sub2api/backend/internal/server/routes/gateway.go`

实际调用的上游：

- `https://generativelanguage.googleapis.com`
- `https://cloudcode-pa.googleapis.com`

说明：

- API Key / OAuth / AI Studio / Gemini CLI 语义被集中到 Gemini 相关 service 中处理

### F. Antigravity 专用链路

入口：

- `/antigravity/v1/messages`
- `/antigravity/v1beta/models/*`

特点：

- 强制平台，不与普通分组调度混合
- 适合做“只用某一种 provider 账号池”的隔离入口

## 5.4 usage / quota / billing 的处理方式

`sub2api` 的 usage 设计明显比另外两个项目更平台化。

直接证据：

- 结构化 usage 表在 `sub2api/backend/ent/schema/usage_log.go`
- usage 异步落地线程池在 `sub2api/backend/internal/service/usage_record_worker_pool.go`
- 管理查询在 `sub2api/backend/internal/handler/admin/usage_handler.go`
- dashboard 聚合在 `sub2api/backend/internal/handler/admin/dashboard_handler.go`

它记录的不只是 token 总数，还包括：

- `requested_model`
- `upstream_model`
- `channel_id`
- `billing_tier`
- `billing_mode`
- `group_id`
- `subscription_id`
- `input/output/cache/image`
- `total_cost`
- `actual_cost`
- `rate_multiplier`
- `account_rate_multiplier`

结论：

- `SSS-Api` 的 billing 与 usage 设计应该直接参考它，而不是自己再发明一套更弱的 usage 结构

## 6. `CLIProxyAPI` 的 API 交互

## 6.1 入口结构

核心入口在 `CLIProxyAPI/internal/api/server.go`。

原生支持：

- `GET /v1/models`
- `POST /v1/chat/completions`
- `POST /v1/completions`
- `POST /v1/messages`
- `POST /v1/messages/count_tokens`
- `GET /v1/responses`
- `POST /v1/responses`
- `POST /v1/responses/compact`
- `GET/POST /v1beta/models/*`

此外还有 Amp provider 别名路由：

- `/api/provider/openai/v1/chat/completions`
- `/api/provider/anthropic/v1/messages`
- `/api/provider/google/v1beta/models/*`

参考：

- `CLIProxyAPI/internal/api/server.go`
- `CLIProxyAPI/internal/api/modules/amp/routes.go`

## 6.2 translator / executor 双层设计

这是它最值得借的地方。

参考：

- `CLIProxyAPI/docs/sdk-advanced_CN.md`
- `CLIProxyAPI/internal/translator/translator/translator.go`
- `CLIProxyAPI/internal/translator/*`
- `CLIProxyAPI/internal/runtime/executor/*`

职责分离：

- translator：做协议转换
- executor：做上游请求执行

这意味着一个典型链路是：

1. 客户端发 OpenAI Chat
2. handler 收到请求
3. translator 把 OpenAI Chat 转成目标 provider 协议
4. executor 用认证信息访问上游
5. translator 再把 provider 响应转回 OpenAI Chat

## 6.3 支持的上游

从 executor 常量和默认 URL 可以直接看出它对接的主要上游。

### Claude

参考：

- `CLIProxyAPI/internal/runtime/executor/claude_executor.go`

默认上游：

- `https://api.anthropic.com/v1/messages?beta=true`
- `https://api.anthropic.com/v1/messages/count_tokens?beta=true`

### Codex / OpenAI Responses

参考：

- `CLIProxyAPI/internal/runtime/executor/codex_executor.go`

默认上游：

- `https://chatgpt.com/backend-api/codex`

### Gemini API

参考：

- `CLIProxyAPI/internal/runtime/executor/gemini_executor.go`

默认上游：

- `https://generativelanguage.googleapis.com`

### Gemini CLI / Cloud Code Assist

参考：

- `CLIProxyAPI/internal/runtime/executor/gemini_cli_executor.go`

默认上游：

- `https://cloudcode-pa.googleapis.com`

### Antigravity

参考：

- `CLIProxyAPI/internal/runtime/executor/antigravity_executor.go`

默认上游：

- `https://daily-cloudcode-pa.googleapis.com`
- `https://daily-cloudcode-pa.sandbox.googleapis.com`
- `https://cloudcode-pa.googleapis.com`

### 认证辅助能力

参考：

- `CLIProxyAPI/internal/auth/claude/*`
- `CLIProxyAPI/internal/auth/codex/*`
- `CLIProxyAPI/internal/auth/gemini/*`
- `CLIProxyAPI/internal/api/handlers/management/auth_files.go`

说明：

- 它把辅助登录、auth 文件落盘、管理 API 也做出来了
- 这部分适合借流程，不适合照搬成 `SSS-Api` 的最终业务模型

## 6.4 usage 处理方式

`CLIProxyAPI` 的 usage 更偏“运行时统计”，不是完整账单系统。

参考：

- `CLIProxyAPI/internal/usage/logger_plugin.go`
- `CLIProxyAPI/internal/api/handlers/management/usage.go`

特点：

- 统计在内存中聚合
- 能记录：
  - total requests
  - success / failure
  - total tokens
  - per API key / per model 请求细节
- 适合运维和轻量看板
- 不足以直接承担多租户平台的正式 billing 真相

## 7. 三个项目的关键差异对比

## 7.1 请求路由模式

| 维度 | `claude-relay-service` | `sub2api` | `CLIProxyAPI` |
|------|------------------------|-----------|---------------|
| 路由依据 | 路径 + 模型名前缀 | API Key 所属分组平台 | handler + translator + executor |
| 主要组织方式 | 多条专用 route | 标准入口 + 平台分发 | 标准入口 + 协议矩阵 |
| 扩展新 provider 成本 | 中到高 | 中 | 低 |

## 7.2 协议转换方式

| 维度 | `claude-relay-service` | `sub2api` | `CLIProxyAPI` |
|------|------------------------|-----------|---------------|
| 转换位置 | route / service 内嵌 | handler + service + apicompat | translator 注册表 |
| 可维护性 | 中 | 中上 | 高 |
| 适合长期演进 | 一般 | 好 | 最好 |

## 7.3 usage / billing

| 维度 | `claude-relay-service` | `sub2api` | `CLIProxyAPI` |
|------|------------------------|-----------|---------------|
| token 统计 | 有 | 有 | 有 |
| 成本计费 | 有，但偏业务内嵌 | 最完整 | 弱 |
| 多租户账单 | 弱 | 强 | 弱 |
| 运维请求统计 | 有 | 强 | 强 |

## 7.4 登录授权辅助

| 维度 | `claude-relay-service` | `sub2api` | `CLIProxyAPI` |
|------|------------------------|-----------|---------------|
| OAuth 辅助能力 | 有 | 强 | 强 |
| setup-token / cookie-auth | 局部 | 有 | auth 文件模式更强 |
| 适合抽成独立授权模块 | 一般 | 很适合 | 很适合 |

## 8. 对 `SSS-Api` 的统一 API 网关建议

## 8.1 推荐入口面

第一阶段建议统一支持：

- `/v1/messages`
- `/v1/messages/count_tokens`
- `/v1/models`
- `/v1/chat/completions`
- `/v1/responses`
- `/v1beta/models/*`

第二阶段再补：

- `/v1/responses/compact`
- `/api/provider/:provider/*`
- 更多 provider-specific alias

## 8.2 推荐内部链路

建议采用下面的统一链路：

```text
Client Request
  -> Auth / API Key / Tenant Resolve
  -> Inbound Endpoint Normalize
  -> Inbound Adapter
  -> Group / Channel / Policy Resolve
  -> Account Scheduler
  -> Provider Executor
  -> Response Translator
  -> Usage / Billing Event
  -> Client Response
```

必须单独记录两类端点：

- `inbound_endpoint`
- `upstream_endpoint`

这个设计应直接吸收 `sub2api/backend/internal/handler/endpoint.go` 的思路。

## 8.3 推荐内部标准对象

不要把内部核心逻辑直接绑定在 OpenAI / Claude / Gemini 其中一种协议上。

建议定义内部标准请求对象：

- `NormalizedRequest`
- `NormalizedMessage`
- `NormalizedTool`
- `NormalizedThinking`
- `NormalizedUsage`
- `ProviderExecutionContext`

同时保留 `raw_extensions`，避免某些 provider 特有字段被完全抹平。

理由：

- 纯粹绑定 OpenAI Chat，难以自然承载 Claude Messages / Gemini native
- 纯粹绑定 Anthropic Messages，又不利于 OpenAI Responses / Codex
- 用“标准对象 + provider 扩展字段”更适合长期演进

## 8.4 推荐 usage / billing 记录口径

新项目至少要记录：

- `tenant_id`
- `user_id`
- `api_key_id`
- `account_id`
- `group_id`
- `subscription_id`
- `channel_id`
- `request_id`
- `inbound_endpoint`
- `upstream_endpoint`
- `requested_model`
- `billing_model`
- `upstream_model`
- `input_tokens`
- `output_tokens`
- `reasoning_tokens`
- `cache_creation_tokens`
- `cache_read_tokens`
- `total_cost`
- `actual_cost`
- `stream`
- `latency_ms`
- `first_token_ms`
- `upstream_request_id`
- `codex_limit_snapshot`

其中：

- usage 表结构参考 `sub2api/backend/ent/schema/usage_log.go`
- Codex 限额快照参考 `sub2api/backend/internal/service/openai_gateway_service.go`
- 轻量运行时统计可参考 `CLIProxyAPI/internal/usage/logger_plugin.go`

## 8.5 推荐 streaming 处理方式

建议遵循三个原则：

1. 流式转换器必须是状态机，而不是简单字符串替换
2. 上游 SSE 与客户端 SSE 要分层处理
3. usage 捕获必须兼容“流式最后一包才给 usage”的上游行为

可参考：

- `claude-relay-service/src/routes/unified.js`
- `claude-relay-service/src/routes/openaiGeminiRoutes.js`
- `sub2api/backend/internal/service/openai_gateway_messages.go`
- `CLIProxyAPI/internal/translator/*`

## 8.6 推荐错误与脱敏策略

错误处理不要只返回“上游原文”。

建议统一：

- 对客户端输出协议兼容错误
- 对内部日志记录原始错误
- 记录前先做脱敏

参考：

- `claude-relay-service/src/utils/errorSanitizer.js`
- `sub2api/backend/internal/util/logredact/redact.go`
- `CLIProxyAPI/internal/logging/request_logger.go`

## 9. 最终结论

### 9.1 三个项目的最佳组合方式

- 用 `claude-relay-service` 学产品入口和用户心智
- 用 `sub2api` 学平台型网关、计费和后台模型
- 用 `CLIProxyAPI` 学协议翻译与执行器拆分

### 9.2 `SSS-Api` 的 API 网关应该怎样做

不是：

- 一个大路由文件里塞满所有 provider 逻辑

而应该是：

- 标准入口
- 标准化入站端点
- translator / executor 分层
- group / tenant / channel 决策
- scheduler 选账号
- usage / billing 异步落库

### 9.3 一句话 API 设计结论

`SSS-Api` 的网关应采用“`sub2api` 的标准入口与平台调度 + `CLIProxyAPI` 的 translator/executor 架构 + `claude-relay-service` 的产品化统一接入体验”这一组合方案。
