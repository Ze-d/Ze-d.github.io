---
title: LangChain LCEL管道符技术参考
date: 2026-01-12 16:09:07
tags: [langchain,llm]
---

这是一份关于 **LangChain 表达式语言 (LCEL)** 核心组件——管道符 `|` 的综合技术参考文档。

## 1. 核心概述

在 LangChain 中，管道符 `|` 是一种**声明式**的组合操作符，用于构建 LangChain 表达式语言 (LCEL)。它的核心作用是将不同的 `Runnable` 组件串联起来，**将左侧组件的输出作为右侧组件的输入**。

### 数学表达

如果我们将组件视为函数，代码 chain = A | B | C 等价于复合函数：

$$Result = C(B(A(Input)))$$

### 协议基础

参与管道操作的所有对象（Prompt, LLM, Parser, Retriever 等）都必须实现 **`Runnable` 接口**。这意味着它们都支持统一的调用方法：

- `invoke()`: 同步调用
- `stream()`: 流式输出
- `batch()`: 批量处理
- `ainvoke()` / `astream()`: 异步方法

------

## 2. 语法与使用模式详解

### 2.1 线性串联 (Linear Chaining)

最基础的用法，按顺序传递数据。

**语法:** `Component_A | Component_B | Component_C`

Python

```
# 示例：Prompt -> Model -> OutputParser
chain = prompt | model | StrOutputParser()
```

### 2.2 参数注入与透传 (Passthrough)

当后续组件需要多个参数，或者需要保留原始输入时使用。

**组件:** `RunnablePassthrough`

- **`RunnablePassthrough()`**: 占位符，原样传递接收到的输入。
- **`RunnablePassthrough.assign(key=value)`**: 在输入字典中增加新的键值对，不覆盖旧数据。

Python

```
from langchain_core.runnables import RunnablePassthrough

# 示例：在 RAG 中，同时传递检索到的 context 和原始 question
rag_chain = (
    {"context": retriever, "question": RunnablePassthrough()} 
    | prompt 
    | model
)
```

### 2.3 并行执行 (Parallel Execution)

使用**字典**或 `RunnableParallel` 来定义分支，这些分支会并行运行。

**语法:** `{ "key1": chain1, "key2": chain2 }`

LangChain 会自动检测字典中的 Runnable 对象并并行执行它们，最后将结果合并为一个字典返回。

Python

```
# 示例：同时生成摘要和提取意图
map_chain = {
    "summary": summary_chain,
    "intent": intent_chain
}
# 输出: {"summary": "...", "intent": "..."}
```

### 2.4 运行时配置 (Runtime Configuration)

在管道中绑定静态参数（如 Stop Token、Temperature 或 Tools）。

**方法:** `.bind()`

Python

```
# 示例：告诉模型遇到 "END" 时停止生成
chain = prompt | model.bind(stop=["END"]) | output_parser
```

### 2.5 自定义函数 (Custom Functions)

将任意 Python 函数转换为管道的一部分。

**组件:** `RunnableLambda` 或 `@chain` 装饰器

Python

```
from langchain_core.runnables import RunnableLambda

def enhance_text(text: str):
    return text.upper() + "!!!"

# 使用 RunnableLambda 包装
chain = prompt | model | RunnableLambda(enhance_text)
```

------

## 3. 应用场景指南

在架构设计时，请根据**任务的确定性**来决定是否使用管道符。

| 场景类型 | 描述 | 推荐架构 | 为什么使用管道 (|) |

| :--- | :--- | :--- | :--- |

| 基础 RAG | 检索文档并回答 | Chain | 流程固定（检索-增强-生成），需极低延迟。 |

| 信息提取 | 从文本转 JSON | Chain | 结构化输出通常是一次性处理，无需循环。 |

| 聊天机器人 | 带记忆的对话 | Chain | 历史记录注入 + 调用模型是线性过程。 |

| 多重意图路由 | 分类后走不同逻辑 | Chain (Routing) | 使用 RunnableBranch 进行确定性分支跳转。 |

| 复杂推理/工具调用 | 需多步思考、查错 | Agent | 管道无法处理不确定次数的循环（While Loop）。 |

------

## 4. 完整实战代码示例 (RAG 模式)

以下代码展示了如何利用管道符构建一个完整的、包含检索和流式输出的问答系统。

Python

```
import bs4
from langchain_community.document_loaders import WebBaseLoader
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_text_splitters import RecursiveCharacterTextSplitter

# 1. 准备数据 (Retriever)
loader = WebBaseLoader("https://lilianweng.github.io/posts/2023-06-23-agent/")
docs = loader.load()
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
splits = text_splitter.split_documents(docs)
vectorstore = Chroma.from_documents(documents=splits, embedding=OpenAIEmbeddings())
retriever = vectorstore.as_retriever()

# 2. 定义 Prompt
template = """Answer the question based only on the following context:
{context}

Question: {question}
"""
prompt = ChatPromptTemplate.from_template(template)

# 3. 定义模型与解析器
model = ChatOpenAI(model="gpt-3.5-turbo")
output_parser = StrOutputParser()

# 4. === 核心：使用管道符构建 LCEL 链 ===
# logic: 
#   1. 并行: (检索 context) AND (保留 question)
#   2. 格式化 prompt
#   3. 模型推理
#   4. 解析字符串
rag_chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | model
    | output_parser
)

# 5. 调用 (支持流式)
print("--- 开始流式输出 ---")
for chunk in rag_chain.stream("What is Task Decomposition?"):
    print(chunk, end="", flush=True)
```

------

## 5. 调试技巧

由于管道符将逻辑高度抽象，调试时推荐使用 **LangSmith**。如果在本地调试，可以使用 `set_debug(True)` 查看中间环节。

Python

```
from langchain.globals import set_debug
set_debug(True) # 开启后，控制台会打印每一步的输入输出详细日志
```

------

下一步建议：

如果您希望深入了解如何处理复杂的条件分支（例如：如果检索结果为空，则联网搜索；否则直接回答），我可以为您讲解 RunnableBranch 的用法。
