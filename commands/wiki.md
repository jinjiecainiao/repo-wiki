---
description: "Generate structured wiki documentation for the current repository"
---

# Generate Repository Wiki

你是 `repo-wiki` 的命令编排层，负责把“仓库扫描 → 架构分析 → 模块深挖 → 学习路径”串成一个可执行的多阶段流程。

收到 `/wiki` 命令后，不要只转交给 skill；请按本文档直接编排执行。

## 参数

用户可能提供：
- `--lang <code>`：文档语言，默认 `en`

若用户未提供 `--lang`，默认：`en`

记录为：
- `DOC_LANG`

所有生成内容写入当前仓库的 `docs/` 目录。

---

## 目标输出

生成以下文档：

```text
docs/
├── index.md
├── directory-structure.md
├── architecture.md
├── learning-path.md
└── modules/
    ├── <module-a>.md
    └── ...
```

## 配套模板

在生成文档前，必须先读取以下模板文件，并严格按模板结构填充内容：

- `templates/wiki/index-template.md`
- `templates/wiki/directory-structure-template.md`
- `templates/wiki/architecture-template.md`
- `templates/wiki/learning-path-template.md`
- `templates/wiki/writing-rules.md`

若模板文件不存在，则终止并报告缺失文件。

---

## 全局执行规则

1. 所有文档内容使用 `DOC_LANG` 指定语言编写
2. 代码标识符、文件路径、类名、函数名保留原文
3. 所有文档在生成前都必须先读取 `templates/wiki/writing-rules.md` 并遵守其中的写作规范
4. 跳过生成产物和无关目录：
   - `node_modules`
   - `vendor`
   - `build`
   - `dist`
   - `target`
   - `.git`
   - `__pycache__`
   - `.venv`
   - `.idea`
   - `.cursor`
5. 每完成一个阶段，都要先向用户展示结果摘要并征求确认，再进入下一阶段
6. 若 `docs/` 已存在且已有目标文件，先告知用户将覆盖，并等待用户确认后再继续
7. 对模块分析任务，优先并发执行

---

## 阶段 0：初始化与覆盖检查

### 步骤 0.1：确定输出语言

若用户提供了 `--lang`，使用该值；否则 `DOC_LANG = en`。

### 步骤 0.2：检查 `docs/` 目录

1. 检查当前仓库下是否存在 `docs/`
2. 检查下列文件是否已存在：
   - `docs/index.md`
   - `docs/directory-structure.md`
   - `docs/architecture.md`
   - `docs/learning-path.md`
   - `docs/modules/*.md`
3. 如果存在任意目标文件，向用户明确说明将覆盖这些文档，并等待确认
4. 若用户拒绝覆盖，则终止流程
5. 若 `docs/` 不存在，则创建：
   - `docs/`
   - `docs/modules/`

---

## 阶段 1：仓库扫描与首批文档生成

### 步骤 1.1：调用扫描代理

使用 Agent 工具调用子代理 `repo-wiki:repo-scanner`，让其输出结构化扫描结果。

**SubAgent**：`repo-wiki:repo-scanner`

**Prompt 模板**：

```text
Analyze the current repository and produce a structured scan report.

Target language for final docs: {DOC_LANG}

Focus on:
- Tech stack detection
- Top-level and second-level directory structure
- Candidate core modules
- Entry points
- Build/test/run clues

Skip generated or vendored directories.
```

### 步骤 1.2：生成 `docs/index.md`

使用 Agent 工具调用子代理 `repo-wiki:index-writer` 生成最终的项目总览文档。

在调用前，先读取：
- `templates/wiki/index-template.md`
- `templates/wiki/writing-rules.md`

**SubAgent**：`repo-wiki:index-writer`

**Prompt 模板**：

```text
Generate the final `docs/index.md` content.

Target language: {DOC_LANG}
README findings: {README 摘要或关键内容}
Repository scan summary: {阶段1扫描结果}
Config/build/run clues: {配置文件与运行方式摘要}
Index template content: {index-template.md 全文}
Writing rules content: {writing-rules.md 全文}
```

生成结果必须：
- 严格遵循模板结构
- 遵守写作规范
- 替换全部占位符
- 不要保留任何 `{{...}}`

### 步骤 1.3：生成 `docs/directory-structure.md`

使用 Agent 工具调用子代理 `repo-wiki:directory-writer` 生成最终的目录结构文档。

在调用前，先读取：
- `templates/wiki/directory-structure-template.md`
- `templates/wiki/writing-rules.md`

**SubAgent**：`repo-wiki:directory-writer`

**Prompt 模板**：

```text
Generate the final `docs/directory-structure.md` content.

Target language: {DOC_LANG}
Repository scan summary: {阶段1扫描结果}
Directory template content: {directory-structure-template.md 全文}
Writing rules content: {writing-rules.md 全文}
```

生成结果必须：
- 严格遵循模板结构
- 遵守写作规范
- 替换全部占位符
- 不要保留任何 `{{...}}`

### 步骤 1.4：展示阶段结果并确认

向用户汇报：
- 识别出的技术栈
- 候选模块列表
- 已生成文件：
  - `docs/index.md`
  - `docs/directory-structure.md`

然后询问：

“Phase 1 complete. Please review the overview and directory structure. Any corrections before I continue to architecture analysis?”

若用户提出修正，先修改对应文档再进入阶段 2。

---

## 阶段 2：架构分析与架构文档生成

### 步骤 2.1：调用架构分析代理

使用 Agent 工具调用子代理 `repo-wiki:architecture-analyzer`，由其生成仓库级架构分析结果。

**SubAgent**：`repo-wiki:architecture-analyzer`

**Prompt 模板**：

```text
Analyze the repository architecture and generate architecture documentation content.

Target language: {DOC_LANG}
Repository scan summary: {阶段1扫描结果}
Candidate modules: {阶段1模块列表}

Please focus on:
- repository shape
- entry points and bootstrap
- module relationships
- architectural layers
- important data/control flows
- external integrations
- confidence notes for inferred conclusions
```

### 步骤 2.2：生成 `docs/architecture.md`

使用 Agent 工具调用子代理 `repo-wiki:architecture-writer`，将架构分析结果落成最终文档。

在调用前，先读取：
- `templates/wiki/architecture-template.md`
- `templates/wiki/writing-rules.md`

**SubAgent**：`repo-wiki:architecture-writer`

**Prompt 模板**：

```text
Generate the final `docs/architecture.md` content.

Target language: {DOC_LANG}
Architecture analysis summary: {阶段2架构分析结果}
Repository scan summary: {阶段1扫描结果}
Architecture template content: {architecture-template.md 全文}
Writing rules content: {writing-rules.md 全文}
```

生成结果必须：
- 严格遵循模板结构
- 遵守写作规范
- 替换全部占位符
- 不要保留任何 `{{...}}`
- 区分确认事实与推断结论

### 步骤 2.3：展示阶段结果并确认

向用户汇报：
- 识别出的主要架构层次
- 关键模块依赖关系
- 已生成文件：`docs/architecture.md`

然后询问：

“Phase 2 complete. Please review the architecture doc. Any corrections before I continue to module deep-dives?”

若用户提出修正，先修改 `docs/architecture.md` 再进入阶段 3。

---

## 阶段 3：核心模块深挖

### 步骤 3.1：确认模块范围

基于阶段 1 的模块识别结果，向用户列出候选核心模块，并询问：

“I identified these core modules: [模块列表]. Should I document all of them, or focus on specific ones?”

如果用户未缩小范围，则默认文档化全部核心模块。

### 步骤 3.2：并发调用模块分析代理

对每个确认模块，调用子代理 `repo-wiki:module-analyzer`。

**SubAgent**：`repo-wiki:module-analyzer`

**Prompt 模板**：

```text
Analyze this module and generate a complete documentation page.

Module name: {module_name}
Module path: {module_path}
Target language: {DOC_LANG}

Project context:
- Tech stack: {阶段1识别结果摘要}
- Architecture summary: {阶段2摘要}
```

### 步骤 3.3：写入模块文档

将每个子代理结果写入：
- `docs/modules/{module-name}.md`

文件名使用稳定的模块标识；必要时将空格转为 `-`，统一小写。

### 步骤 3.4：展示阶段结果并确认

向用户汇报：
- 已生成模块文档数量
- 文档路径列表

然后询问：

“Phase 3 complete. I generated docs for [N] modules. Please review them and let me know if any need adjustments.”

若用户要求修改某个模块文档，只修改指定文件，不重做全部阶段。

---

## 阶段 4：学习路径生成

### 步骤 4.1：调用学习路径分析代理

使用 Agent 工具调用子代理 `repo-wiki:learning-path-analyzer`，由其基于前 3 个阶段的结果生成面向新人的阅读路径。

**SubAgent**：`repo-wiki:learning-path-analyzer`

**Prompt 模板**：

```text
Generate a newcomer-friendly repository learning path.

Target language: {DOC_LANG}
Repository scan summary: {阶段1扫描结果}
Architecture summary: {阶段2架构摘要}
Confirmed modules: {阶段3确认模块列表}
Module summaries: {阶段3模块文档摘要或列表}
```

### 步骤 4.2：生成 `docs/learning-path.md`

使用 Agent 工具调用子代理 `repo-wiki:learning-path-writer`，将学习路径分析结果落成最终文档。

在调用前，先读取：
- `templates/wiki/learning-path-template.md`
- `templates/wiki/writing-rules.md`

**SubAgent**：`repo-wiki:learning-path-writer`

**Prompt 模板**：

```text
Generate the final `docs/learning-path.md` content.

Target language: {DOC_LANG}
Learning path analysis summary: {阶段4学习路径分析结果}
Repository scan summary: {阶段1扫描结果}
Architecture summary: {阶段2架构摘要}
Learning path template content: {learning-path-template.md 全文}
Writing rules content: {writing-rules.md 全文}
```

生成结果必须：
- 严格遵循模板结构
- 遵守写作规范
- 替换全部占位符
- 不要保留任何 `{{...}}`
- 保持阅读顺序具备依赖感知

### 步骤 4.3：展示完成结果

向用户汇报：
- 全部生成的文档列表
- 各模块的推荐阅读顺序

然后询问：

“Phase 4 complete — all wiki documentation is generated. Please review the learning path. Want any adjustments?”

---

## 最终输出要求

流程完成后，给出总结，至少包含：
- 生成语言
- 已生成文件数
- 输出目录：`docs/`
- 核心模块数
- 若有用户指定的重点模块，说明已覆盖

不要只回复“已调用 skill”。必须按上述阶段进行编排与推进。
