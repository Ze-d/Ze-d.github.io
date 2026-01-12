---
title: langSmith功能与工作流
date: 2026-01-11 18:02:41
tags: [langchain,llm]
---

这份文档为您总结了 LangSmith 的核心技术能力及其在 LLM 应用开发全生命周期中的工作流。

## 1. 核心定位

LangSmith 是一个用于构建生产级 LLM 应用程序的 **DevOps 平台**。它弥补了传统软件开发工具在面对非确定性 LLM 时的不足，提供从**原型开发、调试、评估到生产监控**的全链路支持。

------

## 2. 核心功能详解 (Core Capabilities)

### 2.1 全链路追踪与调试 (Tracing & Debugging)

解决 LLM 应用的“黑盒”问题，提供极其细颗粒度的可视性。

- **功能描述：** 自动记录每一次 LLM 调用、Chain 执行、Agent 思考过程及 Tool 使用。
- **关键特性：**
  - **可视化调用树 (Run Tree)：** 以层级结构展示调用链，清晰查看输入、输出、Prompt 渲染结果。
  - **瀑布流视图 (Waterfall)：**  可视化每个步骤的延迟，快速定位是检索慢还是生成慢。
  - **元数据捕获：** 自动记录 Token 消耗、模型参数、延迟和错误堆栈。
  - **Playground 集成：** 在 Trace 界面直接修改 Prompt 参数并“重放” (Re-run)，快速验证修复方案。

### 2.2 评估与测试 (Evaluation & Testing)

将“凭感觉”调整转化为“基于数据”的优化。

- **功能描述：** 允许开发者构建数据集，定义评分标准，并批量运行测试。
- **关键特性：**
  - **数据集管理：** 支持上传 CSV/JSON 或从历史 Trace 中一键添加到数据集。
  - **自动化评估器 (Evaluators)：**
    - **内置评估器：** 正确性 (Correctness)、相关性 (Relevance)、有害性检测。
    - **自定义评估器：** 支持使用 Python 代码或 LLM-as-a-Judge 自定义逻辑。
    - **Ragas 集成：** 支持集成 Ragas 指标（如 Context Precision, Faithfulness）进行 RAG 专项评估。
  - **回归测试对比：** 自动对比新旧版本的测试结果，防止代码变动导致效果退化。

### 2.3 Prompt 管理 (Prompt Hub)

实现 Prompt 的版本控制与代码解耦。

- **功能描述：** 一个类似 GitHub 的 Prompt 仓库，用于存储、版本化和分发 Prompt。
- **关键特性：**
  - **代码解耦：** 代码中只需引用 `prompt_name`，无需硬编码 Prompt 字符串。
  - **版本控制：** 每次修改都会生成 Commit Hash，支持回滚。
  - **协同编辑：** 非技术人员（如产品经理）可在 UI 上调整 Prompt，无需修改代码库。
  - **Playground 测试：** 在浏览器中直接测试 Prompt 效果。

### 2.4 生产监控 (Monitoring)

实时掌握应用在生产环境的表现。

- **功能描述：** 聚合生产环境的 Trace 数据，提供宏观指标和微观反馈。
- **关键特性：**
  - **关键指标仪表盘：** 实时监控 P99 延迟、错误率、Token 成本、RPS (每秒请求数)。
  - **用户反馈收集：** 集成用户点赞/点踩 (Thumbs up/down) 数据，关联到具体的 Trace。
  - **数据漂移检测：** 监控输入分布是否发生变化。
  - **标注队列 (Annotation Queue)：** 将生产环境中的 Bad Case 发送到队列，进行人工审查和标注，转化为新的测试数据。

------

## 3. LangSmith 标准工作流 (The Workflow)

LangSmith 并非单点工具，而是贯穿整个开发循环。以下是基于 LangSmith 的 **RAG/Agent 开发迭代闭环**。

代码段

```
graph TD
    subgraph "Phase 1: 开发与原型 (Prototyping)"
        A[编写代码/Chain] -->|Tracing| B(LangSmith Trace)
        B -->|调试| C{发现问题?}
        C -->|是| D[在 Playground 调整 Prompt]
        D -->|保存| E[Prompt Hub]
        E --> A
        C -->|否| F[进入下一阶段]
    end

    subgraph "Phase 2: 评估与优化 (Evaluation)"
        F --> G[构建/扩充数据集]
        G --> H[批量运行评估 (Run Evaluation)]
        H --> I[分析指标 (结合 Ragas)]
        I -->|分数低| J[优化检索策略 / 修改 Prompt]
        J --> H
        I -->|分数达标| K[部署生产环境]
    end

    subgraph "Phase 3: 生产与监控 (Production)"
        K --> L[生产环境运行]
        L -->|Monitoring| M[监控仪表盘 (成本/延迟/错误)]
        L -->|Feedback| N[收集用户反馈 (👍/👎)]
    end

    subgraph "Phase 4: 数据闭环 (Data Loop)"
        N --> O[标注队列 (Annotation Queue)]
        O -->|人工修正/筛选| P[添加到测试数据集]
        P --> G
    end

    style F fill:#e1f5fe,stroke:#01579b
    style K fill:#e8f5e9,stroke:#1b5e20
    style P fill:#fff9c4,stroke:#fbc02d
```

### 工作流详细步骤：

1. **开发期 (Develop):**
   - 在本地编写代码，开启 LangSmith Tracing。
   - 运行代码，观察 Trace 瀑布图，优化 Prompt 和 Chain 的逻辑。将稳定的 Prompt 推送到 **Hub**。
2. **评估期 (Evaluate):**
   - 创建初始数据集（例如 20 个 QA 对）。
   - 运行评估任务，LangSmith 会计算分数（准确率、Ragas 指标等）。
   - **关键点：** 如果分数不达标，回到开发期修改，直到通过测试。
3. **部署期 (Deploy):**
   - 上线应用，持续将 Trace 发送到 LangSmith 的生产项目。
4. **监控与反馈 (Monitor & Improve):**
   - 查看监控面板，确保存本和延迟在预期内。
   - **数据飞轮：** 筛选出用户“点踩”的记录，或者延迟异常的记录。人工查看这些 Bad Case，修正其答案，然后**一键添加到“评估期”的数据集中**。
   - 下次代码更新时，这些曾经的 Bad Case 就变成了回归测试题，确保问题不再复发。

------

## 4. 总结

LangSmith 可以做什么？

简单来说，LangSmith 是 LLM 应用的 CT 扫描仪（Tracing）、考官（Evaluation）和 仪表盘（Monitoring）。

它将原本混乱的 Prompt 调试过程标准化，并建立了一个**“从生产数据中学习”**的自动化改进机制，是构建严肃企业级 AI 应用的必备基础设施。
