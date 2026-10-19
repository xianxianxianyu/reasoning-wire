# reasoning-wire

**历史推理（cot）在请求体里有三种身份；能不能删，取决于协议把它当什么。**

一篇说明，只讲三件事：全量回传的问题有多大、provider 对 reasoning 的支持怎么分三种范式、以及为什么唯一正确的下手位置是 payload 层。

- 不计 token、不占上下文、不破缓存，**只占传输** —— 前提是协议把它当纯文本
- 同一家模型可能同时属于两种范式（DeepSeek：无 tools 可丢弃，带 tools 必须重放）
- 所以只能在"协议已编码、尚未发送"的那一层按协议分流

---

## 1 问题：每一轮都在重传历史推理

agent 循环会把每条历史 assistant 的 thinking 全文塞回请求体：

```json
{"role": "assistant", "content": "…", "reasoning_content": "这一轮 thinking 全文"}
```

对一个 400K 量级的会话，这意味着每次 loop 多传约 **1.9 MB**。本机实测（deepseek-flash，2026-10）：

| 实验 | 做法 | 结果 |
|---|---|---|
| 计入 token？ | 4.8 万字符塞进 assistant 的 `reasoning_content` | `prompt_tokens` 40 → **40** |
| 对照 | 同样的字塞进 `user` 文本 | 40 → **24,041** |
| 破缓存？ | 同一前缀 + 14 万字符 cot | prompt 仍 15,037，`cache_hit` 14,848 **不变** |
| 花时间？ | 60 万字符（3.6 MB 请求体） | 官方 0.7 s → **7.9 s**；网关 1.7 s → **17–23 s** |

前三行合起来说明：服务端在分词之前就把它丢了，既不进上下文也不破缓存。第四行说明它只占传输 —— 而网关处理大请求体并不便宜。

**但这张表有前提**，就在下一节。

---

## 2 三种范式：同一个字段，三种身份

| 范式 | 代表 | 历史 cot 的身份 | 删掉的后果 |
|---|---|---|---|
| **可丢弃** | 无 tools 的 DeepSeek / 兼容网关 | 纯展示文本，服务端忽略 | 无事发生，纯省传输 |
| **必须重放** | DeepSeek + `tools`；Anthropic `thinking` + `signature` | 上下文的一部分 | 丢推理连续性，或直接 400 |
| **不透明凭证** | Responses `encrypted_content`；Gemini `thought_signature` | 密文，不可读但要原样搬 | 校验失败 / 多步能力退化 |

### 2.1 可丢弃

字段是给人看的。服务端收到就扔，不参与分词、不参与缓存计算。

判断依据只能靠实验（见第 3 节）：`prompt_tokens` 完全没变。

### 2.2 必须重放

字段进了上下文，或者带签名校验。

- **DeepSeek + tools**：官方文档明确写了 `reasoning_content` 是否回传取决于请求是否携带 `tools` —— 带 tools 时必须回传并会拼进上下文。
- **Anthropic**：thinking 块带 `signature`。同一轮工具循环内必须原样传回，改一个字符就报错；跨轮传回是否生效由模型的 preserved thinking 规则决定。

这一范式下，"删"不是优化，是改语义。

### 2.3 不透明凭证

字段是加密串，模型能解、你读不懂。

- **OpenAI Responses**：`store: false`（或 ZDR）时，推理项以 `encrypted_content` 返回，要维持跨轮推理就得原样回放（连同 `id`、`summary`）。
- **Gemini 3**：`thought_signature`，function calling 场景校验更严。

删掉不一定会报错，但会静默丢掉它买来的东西。

### 2.4 最容易翻车的地方：同一家模型跨两种范式

DeepSeek 官方文档原话的意思是：**带 `tools` 时历史 `reasoning_content` 必须回传、并会被拼进上下文；不带 `tools` 时会被忽略。**

| 请求形态 | 范式 | 删历史 cot |
|---|---|---|
| 纯对话（无 `tools`） | 可丢弃 | 无副作用 |
| agent 循环（带 `tools`） | 必须重放 | 丢连续性；缺失时 `400 reasoning_content must be passed back` |

而 agent 会话**总是**带 tools。

这就是"拿一个 no-tools 实验的结论去优化线上 agent，然后把请求跑挂"的完整路径。第 1 节那张表，测的是左边那行。

---

## 3 怎么判断你的 provider 属于哪一种

同一个问题、同样的前缀，只改一处：

```text
基线   messages = [user, assistant("ok"), user]                              → prompt_tokens = A
带 cot messages = [user, assistant("ok", reasoning=<4.8 万字符>), user]       → prompt_tokens = B
对照   messages = [user(<同样 4.8 万字符>), assistant("ok"), user]            → prompt_tokens = C
```

| 结果 | 结论 |
|---|---|
| `B == A` 且 `C >> A` | 服务端丢弃输入 cot → **可丢弃**，删了是 no-op |
| `B ≈ C` | 服务端真的读 → **必须重放**，删是策略选择 |
| 字段是密文 | **不透明凭证**，只能原样搬，别自己构造 |

三条纪律：

1. **带上线上真实的参数再跑一遍**（`tools`、thinking 开关、`stream`）。第 2.4 节就是这个坑。
2. **一次只改一处**。cot 和对照必须同长度、同位置。
3. **记录 provider 与网关**。同一个模型名走官方与走兼容网关，行为可能不同。

原始请求见 [experiments.md](experiments.md)。

还有一个反直觉的数字：面板上"cot 占 ctx 20%"指的是它占 **session 文件**的比例，不是占 prompt 的比例。这两个数在有 tools 和没 tools 时完全不同。

---

## 4 payload 层为什么是唯一正确的下手位置

### 4.1 pi 的四个接缝

| 层 | 事件 | 看到什么 | 能改什么 |
|---|---|---|---|
| 语义 | `context` | `AgentMessage[]`，**已剔除 system** | 返回新消息数组 |
| 语义+ | `context_with_system` | 完整 transcript | 同上，但要自己保住 index 0 的 system |
| **传输** | **`before_provider_request`** | **`payload: unknown`，adapter 已组装好的请求体** | **返回什么就发什么** |
| 传输 | `before_provider_headers` | headers | 原地改，`null` 删头 |

类型定义原话（`BeforeProviderRequestEvent`）：*"Fired before a provider request is sent. Can replace the payload."*

### 4.2 调用链

```
extension handler                     ← return undefined 表示不改，抛错只记日志、请求照发
  ↑ runner.emitBeforeProviderRequest(payload)
  ↑ onPayload: transformProviderPayload          (dist/core/sdk.js)
  ↑ agent 构造参数 onPayload
  ↑ adapter：buildParams() 之后、HTTP 之前
        const next = await options?.onPayload?.(params, model)
        if (next !== undefined) params = next   // 返回值直接替换请求体
  → fetch
```

位置精度：**协议已编码，尚未发送**。所以看到的是 `messages[].reasoning_content`（或 `contents[].parts[].thoughtSignature`、`input[].encrypted_content`），不是 agent 的内部结构。

### 4.3 为什么不下沉到 `context`

| | `context` | `before_provider_request` |
|---|---|---|
| 单位 | pi 的消息/内容块 | 协议字段 |
| 覆盖面 | 一个 provider 一种写法 | 必须按 `model.api` 各自分支 |
| 副作用 | 改了会牵动 UI 统计、token 估算、压缩判定 | 只影响这一次请求 |
| 能力边界 | 碰不到 system 与 tools（pi 帮你保住） | 什么都能改，包括不该改的 |
| 出错代价 | 抛错即忽略 | 返回坏结构 = provider 400 |

一句话：`context` 是"改对话"，`before_provider_request` 是"改报文"。删一个传输字段属于后者。

### 4.4 代价（也就是全部风险）

- **没有 provider 抽象**：`messages[]` / `input[]` / `contents[]` 三种形状，自己分支。
- **改错就是 400**：这一层没有 schema 兜底。
- **必须幂等**：每个请求都会过一遍，同样的输入必须产出同样的输出，否则前缀抖动 = 掉缓存。
- **缓存责任自负**：在这层改到 `cache_control` / `prompt_cache_key` 之类字段，伤的是命中率。

### 4.5 一个实际会踩的坑：event 里没有 model

`BeforeProviderRequestEvent` 只有 `{ type, payload }`。provider / model / `api` 得从第二个参数取：`ctx.model`。

所以"按协议分流"在代码里实际是 `ctx.model.api === "openai-completions"` 这种判断 —— 而不是猜 payload 形状。

---

## 5 边界

- 只删字段：不重排消息、不删消息、不碰 system 与 tools。
- 不碰 session 文件、不碰 UI：会话记录里 thinking 原文都在。
- 保留最后 1 条：成本只有一轮的量，且将来切到"必须重放"的 provider 时行为不用改。
- **上游一旦提供 compat 声明**（例如 `discardsInputReasoning: true`），这一层就应当被删掉 —— 协议知识属于 adapter，不属于扩展。

---

## 参考

- DeepSeek 思考模式（含 `tools` 决定是否回传）：<https://api-docs.deepseek.com/zh-cn/guides/thinking_mode>
- Anthropic extended thinking / thinking 与工具调用：<https://platform.claude.com/docs/en/build-with-claude/extended-thinking>
- Anthropic preserved thinking（signature 校验）：<https://platform.claude.com/docs/en/build-with-claude/preserved-thinking>
- OpenAI reasoning items / `encrypted_content`：<https://developers.openai.com/api/docs/guides/reasoning>
- Gemini thought signatures：<https://ai.google.dev/gemini-api/docs/generate-content/thought-signatures>
