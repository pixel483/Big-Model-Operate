# first-project/.env.example

> 本文件是 `first-project/.env.example` 的 Markdown 版本，原始内容完整保留在下方代码块中。
> 若需恢复为可运行文件，把文件名改回 `first-project/.env.example` 即可。

```bash
# 复制本文件为 .env，然后填入你的 API Key。.env 不要提交到 Git。
# 这里填的是 OpenAI 兼容端点，代码会自动拼上 /chat/completions。

LLM_BASE_URL=https://api.deepseek.com/v1
LLM_API_KEY=sk-在这里填你的密钥
LLM_MODEL=deepseek-chat

# 其他可替换的端点（改上面三行即可）：
#   通义千问  https://dashscope.aliyuncs.com/compatible-mode/v1   qwen-plus
#   智谱      https://open.bigmodel.cn/api/paas/v4                glm-4-flash
#   OpenAI    https://api.openai.com/v1                           gpt-4o-mini
#   Moonshot  https://api.moonshot.cn/v1                          moonshot-v1-8k
#   本地免费  http://localhost:11434/v1                           qwen2.5:7b   （Ollama，API_KEY 随便填）
```
