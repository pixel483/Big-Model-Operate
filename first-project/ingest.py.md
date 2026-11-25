# first-project/ingest.py

> 本文件是 `first-project/ingest.py` 的 Markdown 版本，原始内容完整保留在下方代码块中。
> 若需恢复为可运行文件，把文件名改回 `first-project/ingest.py` 即可。

```python
# -*- coding: utf-8 -*-
"""RAG 第一步（离线）：把 data/ 里的文档切成小片段，存成 chunks.json。

运行：python ingest.py
产出：data/chunks.json

这一步对应 RAG 流水线里的「文档解析 + 切分」。生产环境会换成
Unstructured / LlamaParse + 语义切分，但原理完全一样：把长文档拆成
能塞进上下文窗口、且语义相对完整的小块。
"""
import json
import re
from pathlib import Path

DATA_DIR = Path(__file__).parent / "data"
OUT_FILE = DATA_DIR / "chunks.json"

CHUNK_SIZE = 400   # 每个片段的最大字符数
OVERLAP = 80       # 相邻片段重叠的字符数，避免一句话被拦腰截断


def read_text(path: Path) -> str:
    """读取单个文档，统一返回纯文本。"""
    if path.suffix.lower() == ".pdf":
        try:
            from pypdf import PdfReader  # 延迟导入：只处理 md/txt 时无需安装
        except ImportError:
            raise SystemExit("读取 PDF 需要先安装：pip install pypdf")
        return "\n".join(page.extract_text() or "" for page in PdfReader(path).pages)
    return path.read_text(encoding="utf-8")


def split_text(text: str, source: str) -> list[dict]:
    """按段落累积切分；超长段落用滑动窗口，保留 overlap。"""
    text = re.sub(r"\n{3,}", "\n\n", text).strip()
    paragraphs = [p.strip() for p in text.split("\n") if p.strip()]

    chunks: list[str] = []
    buf = ""
    for para in paragraphs:
        while len(para) > CHUNK_SIZE:
            if buf:
                chunks.append(buf.strip())
                buf = ""
            chunks.append(para[:CHUNK_SIZE])
            para = para[CHUNK_SIZE - OVERLAP:]
        if len(buf) + len(para) > CHUNK_SIZE:
            chunks.append(buf.strip())
            buf = buf[-OVERLAP:] if OVERLAP else ""
        buf += para + "\n"
    if buf.strip():
        chunks.append(buf.strip())

    pieces = [c for c in chunks if c.strip()]
    return [
        {"id": f"{source}#{i}", "source": source, "text": c}
        for i, c in enumerate(pieces)
    ]


def main() -> None:
    DATA_DIR.mkdir(exist_ok=True)
    files = [
        p for p in DATA_DIR.iterdir()
        if p.is_file() and p.suffix.lower() in {".txt", ".md", ".pdf"}
    ]
    if not files:
        raise SystemExit(f"请先把文档放进 {DATA_DIR}（支持 .txt / .md / .pdf）")

    all_chunks: list[dict] = []
    for path in sorted(files):
        pieces = split_text(read_text(path), path.name)
        all_chunks.extend(pieces)
        print(f"  {path.name}: {len(pieces)} 个片段")

    OUT_FILE.write_text(
        json.dumps(all_chunks, ensure_ascii=False, indent=2), encoding="utf-8"
    )
    print(f"\n完成：共 {len(all_chunks)} 个片段 -> {OUT_FILE}")
    print("下一步：python app.py")


if __name__ == "__main__":
    main()
```
