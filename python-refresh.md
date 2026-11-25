# Python 恢复速查：只补这 10 件事

> 给「学过但忘得差不多」的人。目标不是重学 Python，是**两周内能写出调 API、处理 JSON、读写文件的小脚本**。
> 类、继承、装饰器、生成器、asyncio、NumPy、Pandas —— 现在全部跳过，第 11 周之后用到再补。

配套计划见 `16-week-plan.md` 的 W1–W2。每天 20–40 分钟，够用。

## 1. 变量、类型、f-string

```python
name = "知识库"
count = 3
price = 0.0035
ok = True
print(f"共 {count} 个文档，单次成本约 {price} 元：{name}")
```

最容易忘：字符串和数字不能直接拼，用 f-string 最省事；`f"{price:.4f}"` 控制小数位。

## 2. 列表与字典（大模型开发里 80% 的数据都长这样）

```python
messages = [
    {"role": "system", "content": "你是一个助手"},
    {"role": "user", "content": "你好"},
]
messages.append({"role": "assistant", "content": "你好！"})
for m in messages:
    print(m["role"], m["content"][:10])

print(messages[0]["role"])        # 取第一个元素的某个键
print(len(messages))              # 长度
print(m["role"] for m in messages)  # 注意：这是生成器，别直接 print
```

最容易忘：字典取值用 `d["key"]`，键不存在会报 `KeyError`；不确定时用 `d.get("key", 默认值)`。**API 返回的 JSON 解析出来就是嵌套的字典和列表**，这一节要练到熟。

## 3. 判断与循环

```python
for m in messages:
    if m["role"] == "user":
        print("用户说：", m["content"])
    elif m["role"] == "system":
        continue
    else:
        print("模型说：", m["content"])
```

最容易忘：冒号 `:`、缩进（统一 4 个空格）、`==` 是比较而 `=` 是赋值。

## 4. 函数

```python
def build_message(role: str, content: str) -> dict:
    return {"role": role, "content": content}

msg = build_message("user", "讲讲 RAG")
```

最容易忘：`def` 后面也要冒号；函数体要缩进；不写 `return` 就返回 `None`。类型标注（`: str`、`-> dict`）只是给人看的提示，不影响运行——但读别人的代码时很有用，要能看懂。

## 5. 读写文件

```python
from pathlib import Path

Path("out.txt").write_text("hello", encoding="utf-8")
text = Path("out.txt").read_text(encoding="utf-8")
```

最容易忘：Windows 上**一定要写 `encoding="utf-8"`**，否则中文乱码。`Path` 比老式 `open()` 简洁，路径拼接用 `Path("data") / "a.txt"`。

## 6. JSON（调 API 的核心）

```python
import json

data = {"model": "deepseek-chat", "messages": messages}
s = json.dumps(data, ensure_ascii=False, indent=2)   # 对象 -> 字符串
obj = json.loads(s)                                  # 字符串 -> 对象
print(obj["model"])
Path("req.json").write_text(s, encoding="utf-8")
```

最容易忘：`dumps` 是转字符串，`dump` 是写文件；`loads` 是解析字符串，`load` 是读文件。带 s 的都是「和字符串打交道」。中文要加 `ensure_ascii=False`。

## 7. 发 HTTP 请求

```python
import requests

resp = requests.post(
    "https://api.example.com/v1/chat/completions",
    headers={"Authorization": "Bearer sk-xxx"},
    json={"model": "m", "messages": messages},
    timeout=60,
)
print(resp.status_code)
print(resp.json()["choices"][0]["message"]["content"])
```

最容易忘：用 `json=` 参数（不是 `data=`）会自动序列化并设好请求头；一定要带 `timeout`，否则网络卡住时程序永远不返回。

## 8. 异常处理

```python
try:
    content = resp.json()["choices"][0]["message"]["content"]
except (KeyError, IndexError) as e:
    print("响应结构不对：", e)
except requests.RequestException as e:
    print("网络请求失败：", e)
```

最容易忘：只捕获你能处理的异常，别写裸 `except:` 把错误全吞掉——那会让调试变得极其痛苦。

## 9. 虚拟环境与装包

```bash
python -m venv .venv
.venv\Scripts\activate          # Windows；macOS/Linux 用 source .venv/bin/activate
pip install requests python-dotenv pypdf
pip freeze > requirements.txt
```

最容易忘：每次新开终端都要重新激活；看到 `ModuleNotFoundError` 先检查是不是装到了别的环境里。用 `python -m pip` 比直接 `pip` 更稳。

## 10. 读报错（最值钱的一项技能）

从下往上看 traceback：最后一行是错误类型和原因，倒数几行会告诉你**哪个文件第几行**。把这三样原样发给大模型——报错全文、相关代码、你想做什么——基本都能得到可直接用的答案。别自己盯着看半小时。

## 7 天恢复练习

| 天 | 任务（20–40 分钟） | 用到了 |
|---|---|---|
| D1 | 写脚本打印 1–20 里的偶数；再统计一个字符串里每个字出现的次数 | 循环、判断、字典 |
| D2 | 定义 `build_message(role, content)`，用它拼出一段 3 条消息的对话并遍历打印 | 函数、列表、字典 |
| D3 | 新建 `note.txt`（随便写几行中文），读进来按行切分，只保留含「模型」的行，写回 `out.txt` | 文件读写、字符串、编码 |
| D4 | 把一个嵌套字典（模拟 API 响应）转成格式化的 JSON 文件，再读回来取出最深层的文本 | json、Path |
| D5 | 用 `requests.get` 调一个公开 API（如 `https://api.github.com`），打印状态码并解析 JSON | HTTP、异常 |
| D6 | 给 D5 加上 `try/except` 和 `timeout`，让断网时打印友好提示而不是崩溃 | 异常处理 |
| D7 | 建虚拟环境、用 `pip freeze` 导出依赖、建 Git 仓库并 commit + push | 工程基础 |

## 自测清单（W1 结束前应该全打勾）

- [ ] 不看示例能写出一个函数，参数是字符串，返回字典
- [ ] 能遍历一个「列表套字典」并取出指定字段
- [ ] 能把字典存成 JSON 文件再读回来
- [ ] 能发一次 POST 请求并从响应里取出嵌套字段
- [ ] 看到 `KeyError` / `ModuleNotFoundError` / `IndentationError` 知道该往哪查
- [ ] 能自己建虚拟环境、装包、把代码 push 到 GitHub

做完这些就够了，直接进 `16-week-plan.md` 的 W3。
