# first-project/app.py

> 本文件是 `first-project/app.py` 的 Markdown 版本，原始内容完整保留在下方代码块中。
> 若需恢复为可运行文件，把文件名改回 `first-project/app.py` 即可。

```python
# -*- coding: utf-8 -*-
"""RAG 第二步（在线）：检索相关片段 -> 拼进 prompt -> 调大模型 -> 带出处回答。

第一版故意不用 LangChain、不用向量库、不用数据库。检索用 30 行手写的
TF-IDF，目的是让你看清 RAG 的每一步在干什么。跑通之后再逐块替换成
生产级组件（见 README 的「升级路线」）。

运行：
    cp .env.example .env     # Windows 用 copy .env.example .env
    python ingest.py
    python app.py
"""
import json
import math
import os
import re
import sys
from collections import Counter
from pathlib import Path

import requests
from dotenv import load_dotenv

try:
    sys.stdout.reconfigure(encoding="utf-8")  # Windows 控制台中文不乱码
except (AttributeError, OSError):
    pass

BASE_DIR = Path(__file__).parent
CHUNKS_FILE = BASE_DIR / "data" / "chunks.json"
TOP_K = 5

# 这就是「上下文工程」：明确告诉模型能用什么、不能用什么、怎么标注来源。
SYSTEM_PROMPT = """你是一个只依据给定资料回答问题的助手。
规则：
1. 只用【资料】里的内容回答，不要编造，不要用自己的知识补充。
2. 每个关键结论后标注来源，格式 [文件名#片段号]。
3. 如果资料里没有答案，直接说「资料中没有相关信息」，并说明缺什么。
4. 回答用中文，简洁分点。
"""


def tokens(text: str) -> list[str]:
    """零依赖分词：英文数字按词，中文按相邻两字（bigram）。"""
    text = text.lower()
    words = re.findall(r"[a-z0-9]+", text)
    han = re.findall(r"[\u4e00-\u9fff]+", text)
    bigrams = [s[i:i + 2] for s in han for i in range(len(s) - 1)]
    return words + bigrams


def build_index(chunks: list[dict]):
    """离线建索引：计算每个片段的词频向量，并预存向量长度。"""
    docs_tf: list[Counter] = []
    df: Counter = Counter()
    for c in chunks:
        tf = Counter(tokens(c["text"]))
        docs_tf.append(tf)
        df.update(tf.keys())

    n = len(chunks)
    idf = {t: math.log((n + 1) / (d + 1)) + 1 for t, d in df.items()}
    norms = []
    for tf in docs_tf:
        vec = {t: (1 + math.log(v)) * idf[t] for t, v in tf.items()}
        norm = math.sqrt(sum(x * x for x in vec.values())) or 1.0
        norms.append((vec, norm))
    return norms, idf


def retrieve(query: str, chunks: list[dict], index, top_k: int = TOP_K) -> list[dict]:
    """检索：把问题也变成向量，与每个片段算余弦相似度，取前 K 个。

    这是整个项目最值得动手改的地方：换成 Embedding + Chroma 后，
    只需要替换本函数，其他代码一行都不用动。
    """
    norms, idf = index
    qtf = Counter(tokens(query))
    qvec = {t: (1 + math.log(v)) * idf.get(t, 0) for t, v in qtf.items()}
    qnorm = math.sqrt(sum(x * x for x in qvec.values())) or 1.0

    scored = []
    for (vec, norm), chunk in zip(norms, chunks):
        score = sum(qvec.get(t, 0) * w for t, w in vec.items()) / (qnorm * norm)
        scored.append((score, chunk))
    scored.sort(key=lambda item: item[0], reverse=True)
    return [c for score, c in scored[:top_k] if score > 0]


def build_messages(query: str, hits: list[dict], history: list[dict]) -> list[dict]:
    """把检索结果拼进 prompt —— RAG 里的 A（Augmented）就是这一步。"""
    context = "\n\n".join(f"【{c['id']}】{c['text']}" for c in hits)
    user = f"【资料】\n{context}\n\n【问题】\n{query}"
    return (
        [{"role": "system", "content": SYSTEM_PROMPT}]
        + history[-6:]                       # 只带最近 3 轮，控制 token
        + [{"role": "user", "content": user}]
    )


def chat(messages: list[dict], base_url: str, api_key: str, model: str) -> str:
    """裸调 OpenAI 兼容接口，用 SSE 实现打字机式流式输出。"""
    url = base_url.rstrip("/") + "/chat/completions"
    resp = requests.post(
        url,
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
        },
        json={
            "model": model,
            "messages": messages,
            "temperature": 0.2,
            "stream": True,
        },
        stream=True,
        timeout=60,
    )
    if resp.status_code != 200:
        raise SystemExit(f"API 返回 {resp.status_code}：{resp.text[:500]}")
    resp.encoding = "utf-8"

    answer = ""
    for line in resp.iter_lines(decode_unicode=True):
        if not line or not line.startswith("data: "):
            continue
        payload = line[6:].strip()
        if payload == "[DONE]":
            break
        try:
            delta = json.loads(payload)["choices"][0]["delta"].get("content", "")
        except (json.JSONDecodeError, KeyError, IndexError, TypeError):
            continue
        answer += delta
        print(delta, end="", flush=True)
    print()

    if not answer:
        raise SystemExit(
            "没有收到任何内容。该端点可能不支持流式输出：把 chat() 里的 "
            '"stream": True 改为 False，然后直接用 resp.json() 取内容。'
        )
    return answer


def main() -> None:
    load_dotenv(BASE_DIR / ".env")
    base_url = os.getenv("LLM_BASE_URL", "https://api.deepseek.com/v1")
    api_key = os.getenv("LLM_API_KEY", "")
    model = os.getenv("LLM_MODEL", "deepseek-chat")

    if not CHUNKS_FILE.exists():
        raise SystemExit("找不到索引，请先运行：python ingest.py")
    local = "localhost" in base_url or "11434" in base_url
    if not api_key and not local:
        raise SystemExit("请先复制 .env.example 为 .env，并填写 LLM_API_KEY")

    chunks = json.loads(CHUNKS_FILE.read_text(encoding="utf-8"))
    index = build_index(chunks)
    print(f"已加载 {len(chunks)} 个片段。输入问题开始提问，输入 quit 退出。\n")

    history: list[dict] = []
    while True:
        try:
            query = input("你 > ").strip()
        except (EOFError, KeyboardInterrupt):
            break
        if query.lower() in {"quit", "exit", "q", "退出"}:
            break
        if not query:
            continue

        hits = retrieve(query, chunks, index)
        if not hits:
            print("没有检索到相关内容，换个说法试试。\n")
            continue

        print("[检索命中：" + "、".join(c["id"] for c in hits) + "]")
        answer = chat(build_messages(query, hits, history), base_url, api_key, model)
        history += [
            {"role": "user", "content": query},
            {"role": "assistant", "content": answer},
        ]
        print()


if __name__ == "__main__":
    main()
```
