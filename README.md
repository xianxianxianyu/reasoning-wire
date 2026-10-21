# reasoning-wire

> 历史推理（cot）在请求体里有三种身份；能不能删，取决于协议把它当什么。

纯文档仓库，没有代码。目标是把"每一轮重传历史 cot"从「感觉浪费」变成「先测一次、再决定」。

## 核心结论

| 范式 | 代表 | 历史 cot 的身份 | 删掉的后果 |
|---|---|---|---|
| **可丢弃** | 无 `tools` 的 DeepSeek / 兼容网关 | 纯展示文本 | 无事发生，纯省传输 |
| **必须重放** | DeepSeek + `tools`；Anthropic `thinking` | 上下文的一部分 | 丢推理连续性，或直接 400 |
| **不透明凭证** | Responses `encrypted_content`；Gemini `thought_signature` | 密文 | 校验失败 / 能力静默退化 |

推论：**同一家模型可能同时属于两种范式**。DeepSeek 由请求是否携带 `tools` 决定，
而 agent 会话总是带 `tools`。所以一次实验的结论不能跨请求形态复用。

## 数字

400K 量级会话每轮多传约 **1.9 MB**；60 万字符 cot（3.6 MB 请求体）把网关从 1.7 s 拖到 **17–23 s**，
而 `prompt_tokens` 与 `cache_hit` 都不变。

## 文档

| 文档 | 回答什么 |
|---|---|
| [docs/paradigms.md](docs/paradigms.md) | 三种范式怎么分、哪一行最容易翻车 |
| [docs/experiments.md](docs/experiments.md) | 三行最小实验、判据、本机数据 |
| [docs/payload.md](docs/payload.md) | 为什么下手点必须在这一层，代价是什么 |
| [docs/cache.md](docs/cache.md) | 删字段会不会伤缓存命中率 |

阅读顺序即上表，约 10 分钟。

## 边界

- 只谈推理字段，不谈上下文清理 / 压缩策略
- 不提供实现：`docs/payload.md` 引用的是 pi 的扩展接缝，其他 harness 请对照自己的等价层
- 数据为单机实测，且标注了测量时的请求形态（是否带 `tools`）

## 参考

- DeepSeek 思考模式：<https://api-docs.deepseek.com/zh-cn/guides/thinking_mode>
- Anthropic extended thinking：<https://platform.claude.com/docs/en/build-with-claude/extended-thinking>
- Anthropic preserved thinking：<https://platform.claude.com/docs/en/build-with-claude/preserved-thinking>
- OpenAI reasoning items：<https://developers.openai.com/api/docs/guides/reasoning>
- Gemini thought signatures：<https://ai.google.dev/gemini-api/docs/generate-content/thought-signatures>
