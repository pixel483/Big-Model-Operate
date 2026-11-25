# 项目 2：个人文档问答助手（RAG 最小实现）

这是 `16-week-plan.md` 第 6 周的项目，也可以在第 3 周先跑一遍找感觉。

**刻意不用** LangChain、不用向量库、不用数据库，只依赖三个轻量包。目的是让你看清 RAG 的每一步：文档怎么切、检索怎么算分、上下文怎么拼进 prompt、模型怎么被调用、出处怎么标注。跑通之后再逐块替换成生产级组件。

## 5 分钟跑起来

```bash
cd first-project

# 1. 建虚拟环境并装依赖
python -m venv .venv
.venv\Scripts\activate          # Windows；macOS/Linux 用 source .venv/bin/activate
pip install -r requirements.txt

# 2. 配置模型
copy .env.example .env          # macOS/Linux 用 cp
#   用记事本打开 .env，把 LLM_API_KEY 换成自己的密钥

# 3. 建索引 + 提问
python ingest.py
python app.py
```

不想申请密钥？装 [Ollama](https://ollama.com)，`ollama run qwen2.5:7b`，然后把 `.env` 改成 `LLM_BASE_URL=http://localhost:11434/v1`、`LLM_MODEL=qwen2.5:7b`，密钥随便填，全程零成本离线运行。

`data/sample.md` 是自带的示例文档，不用另找资料就能立刻测。试这几个问题：

- 一线城市住宿报销上限是多少？
- 试用期员工有年假吗？
- 生产环境数据库权限要谁审批？
- 内部 AI 工具为什么选 DeepSeek？
- P0 故障多久内要响应？

再问一个文档里**没有**的问题（比如「食堂几点开门」），看它会不会老实说「资料中没有相关信息」——这就是防幻觉 prompt 在起作用。

> 示例文档很小，只切成 4 个片段，所以大部分问题会召回全部片段，「检索」这一步看起来不明显。换成十几份真实文档后，不同问题的命中差异就会很清楚 —— 这正是 W7 做检索评测的意义。

## 文件说明

| 文件 | 作用 |
|---|---|
| `ingest.py` | 离线：读 `data/` 里的 txt/md/pdf → 按段落切分（400 字一块、重叠 80 字）→ 存成 `data/chunks.json` |
| `app.py` | 在线：加载片段 → 手写 TF-IDF 检索 → 拼 prompt → 裸调 API → 流式输出带出处的回答 |
| `data/sample.md` | 示例文档，换成你自己的资料后重新跑 `ingest.py` 即可 |
| `.env.example` | 模型配置模板，支持所有 OpenAI 兼容端点 |
| `.gitignore` | 已经排除 `.env` 和索引产物，别把密钥传上 GitHub |

## 代码导读：RAG 的每一步在哪

```
文档 → [ingest.py: split_text]      切分
     → [app.py: build_index]        建索引（离线，启动时算一次）
问题 → [app.py: tokens/retrieve]    检索 Top-5 片段
     → [app.py: build_messages]     把片段拼进 system/user prompt  ← 上下文工程
     → [app.py: chat]               调 /chat/completions，SSE 流式
     → 带 [文件名#片段号] 的回答
```

三个值得停下来想清楚的点：

1. **`SYSTEM_PROMPT` 才是效果的大头**。去掉「只用资料回答」那几条约束，模型立刻开始编。改一改它，对比输出差异，比看十篇 Prompt 教程有用。
2. **`retrieve()` 是瓶颈**。关键词检索对「换种说法」很无力（问「住酒店能报多少」可能就搜不到「住宿费」）。W7 换成 Embedding + Chroma 时，只改这一个函数。
3. **`history[-6:]` 是上下文窗口管理的最小形态**。多聊几轮再观察 token 增长，就明白为什么需要摘要和压缩了。

## 常见问题

- **没收到内容 / 报 401**：检查 `.env` 里的 key、`LLM_BASE_URL` 末尾有没有多写 `/chat/completions`（代码会自动拼）。
- **中文乱码**：`app.py` 已设置 stdout 为 UTF-8；若仍乱码，在终端先执行 `chcp 65001`（Windows）。
- **提示不支持流式**：个别兼容端点不返回 SSE，按 `chat()` 里的报错提示把 `"stream": True` 改成 `False`，直接用 `resp.json()["choices"][0]["message"]["content"]`。
- **检索不到东西**：`ingest.py` 每次都会打印片段数，先看是不是 0；再打开 `data/chunks.json` 确认内容被正确切出来了。
- **想换自己的文档**：丢进 `data/`，删掉 `sample.md`，重跑 `python ingest.py`。PDF 需要 `pypdf`，扫描件（图片型 PDF）需要 OCR，属于 W8 的内容。

## 跑通之后的升级路线

按这个顺序改，每一步都能在面试里讲成一个「遇到问题 → 定位 → 解决」的故事：

1. **W7**：`retrieve()` 换成 bge-m3 Embedding + Chroma，再加 BM25 混合检索和 bge-reranker 重排。记录升级前后 Top-5 命中率。
2. **W8**：造 30–50 条问答评测集，统计命中率、忠实度、幻觉率，写一份评测报告。
3. **W9**：加 Function Calling，让模型能自己决定「搜文档 / 查计算 / 反问澄清」，用 `while` 循环手写 Agent。
4. **W10**：用 LangGraph 重写编排逻辑，对比手写版的差异。
5. **W11**：FastAPI 包一层，SSE 输出，Docker 化，加日志与成本统计，部署成可访问的链接。

## 五个小练习（改代码比看代码学得快）

- [ ] 把 `TOP_K` 从 5 改成 1 和 10，观察答案质量与冗余的变化
- [ ] 把 `CHUNK_SIZE` 改成 100 和 1000，重新 `ingest.py`，看检索命中怎么变
- [ ] 改 `SYSTEM_PROMPT`，让模型在答案末尾额外输出「置信度」和「依据片段原文」
- [ ] 打印每轮 prompt 的字符数，估算 token 成本（`chat()` 里也能从响应的 `usage` 字段取准确值）
- [ ] 加一个 `/clear` 命令清空 `history`，再实现「历史超过 N 轮就自动摘要」

做完这些，你就已经能独立讲清 RAG 全流程了 —— 这正是 `16-week-plan.md` W6 的验收标准。
