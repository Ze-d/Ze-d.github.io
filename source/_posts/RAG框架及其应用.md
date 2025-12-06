---
title: RAG框架及其应用
date: 2025-11-24 20:03:37
tags: RAG
---

# RAG 技术总结

## 1. RAG 是什么？

RAG（Retrieval-Augmented Generation，检索增强生成）是一种将**信息检索系统**与**大语言模型（LLM）**结合的AI架构。模型在回答问题前，先从外部知识库检索相关内容，并将检索结果作为上下文供模型生成答案。

主要解决的问题：

- 训练数据过时导致知识陈旧
- 无法访问领域专有/内部知识
- 减少大模型“幻觉”（胡编乱造）

---

## 2. RAG 的典型框架

### 🧱 离线阶段（构建知识库）

1. **数据接入** → 企业文档、API 数据库、代码、会议记录等  
2. **内容清洗与切分（Chunking）**  
3. **Embedding 向量化**  
4. **索引构建（向量数据库：Qdrant / Milvus / Pinecone 等）**

### 🔍 在线阶段（实时问答）

1. Query 预处理 / 重写  
2. 检索（向量检索 / Hybrid 检索 + Reranker）  
3. 构造增强上下文（文档片段 + Prompt）  
4. LLM 生成答案  
5. 后处理（格式整理 / 安全审查）

### 常见技术栈

| 层级       | 可选技术                                      |
| ---------- | --------------------------------------------- |
| 框架       | LangChain / LangGraph / LlamaIndex / Haystack |
| 向量数据库 | Qdrant / Milvus / Pinecone / Elasticsearch    |
| Embedding  | BGE / E5 / text-embedding-3-large             |
| LLM        | GPT / Claude / Llama / Gemini / Minimax 等    |

---

## 3. RAG 的应用场景

### 📚 企业知识库 / 内部搜索

- 查询 HR 政策、技术方案、项目要求等

### 🤖 客服 & 帮助中心

- FAQ 自动化、产品询问、工单辅助

### ⚖ 合同 / 法务 / 合规

- 查找相关条款并总结风险点

### 💻 代码与技术助手

- 基于代码库检索解释、调试建议（如 GitHub Copilot Chat）

### 📊 业务数据问答（自然语言 BI）

- “上季度华东地区营收是多少？”

### 🏥 医疗、制造、公共服务

- 检索行业标准或批准流程，辅助分析与决策

---

## 4. 已经落地的 RAG 项目案例

| 领域      | 项目示例                                      |
| --------- | --------------------------------------------- |
| 政府      | *GOV.UK Chat（英国政府）*；e-Albania 电子政务 |
| 企业      | *Audi 内部文档助手*；*NVIDIA RAG Chatbots*    |
| 开发      | *GitHub Copilot Chat*（检索代码）             |
| SaaS 办公 | Microsoft 365 Copilot / ChatGPT Enterprise    |
| 教育      | *Harvard Business School – AI Professor*      |

---

## 5. 如何自己构建一个 RAG 系统？

1. 明确使用者 & 场景（如内网知识、生产数据）  
2. 选存储系统（Qdrant / ES / Mongo + Vector）  
3. 选架构方式（LangChain or SpringAI 架构）  
4. 规范文档结构和索引策略  
5. 实现检索增强 Prompt  
6. 引入反馈机制，迭代检索策略

---

## 6. 一句话总结

> **RAG 是让大模型“会查资料再回答”的外挂系统。**  
> 它不追求让模型“记住一切”，而是通过检索增强推理能力，实现**可解释性、更准确、更可靠**的问答、分析及知识应用。

---

如需：

- 架构图  
- Prompt 模板  
- RAG 项目 README & 设计文档  
  可以告诉我具体项目（如使用 Spring AI + Qdrant 来做 Java API 文档助手），我可以直接生成项目构建方案。

