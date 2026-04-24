---
description: "Incrementally update existing wiki documentation based on code changes"
---

# Incremental Wiki Update

你是 `repo-wiki` 的增量更新命令编排层，负责根据代码变更选择性更新 `docs/` 中的 wiki 文档。

收到 `/wiki-update` 命令后，不要只转交给 skill；请按本文档直接编排执行。

## 参数

用户可能提供：
- `--base <ref>`：用于比较的 Git 基线，默认 `main`
- `--lang <code>`：文档语言，默认 `en`

记录为：
- `BASE_REF`，默认 `main`
- `DOC_LANG`，默认 `en`

---

## 更新目标

根据代码差异，选择性更新以下文档：

- `docs/index.md`
- `docs/directory-structure.md`
- `docs/architecture.md`
- `docs/learning-path.md`
- `docs/modules/*.md`

## 配套模板

增量更新涉及以下模板文件；当对应文档需要重写时，必须先读取并按模板结构填充：

- `templates/wiki/index-template.md`
- `templates/wiki/directory-structure-template.md`
- `templates/wiki/architecture-template.md`
- `templates/wiki/learning-path-template.md`
- `templates/wiki/writing-rules.md`

若模板文件不存在，则终止并报告缺失文件。

---

## 全局执行规则

1. 只更新受影响的文档，不做整库全量重写
2. 若 `docs/` 不存在，则告知用户当前没有可增量更新的 wiki，建议先执行 `/wiki`
3. 对模块删除、模块重命名这类高风险变更，先征求用户确认再执行文件删除/重命名
4. 更新内容语言使用 `DOC_LANG`
5. 所有重写型更新在执行前都必须先读取 `templates/wiki/writing-rules.md`
6. 如仅个别模块变化，不要重建所有模块文档
7. 优先并发更新多个受影响模块

---

## 阶段 0：前置检查

### 步骤 0.1：确认文档目录存在

检查以下目录/文件：
- `docs/`
- `docs/modules/`

若 `docs/` 不存在，终止并告知用户：

“No existing wiki docs found under docs/. Please run /wiki first before using incremental update.”

### 步骤 0.2：确定比较基线

若用户提供了 `--base`，使用该值；否则 `BASE_REF = main`。

---

## 阶段 1：变更检测

### 步骤 1.1：获取 Git 变更列表

使用 Bash 执行：

```bash
git diff --name-status {BASE_REF}...HEAD
```

识别并分类：
- `A`：新增文件
- `M`：修改文件
- `D`：删除文件
- `R`：重命名文件

### 步骤 1.2：映射受影响模块

根据路径将变更映射到模块，并识别特殊变更：

- 配置文件变化（如 `package.json`、`pom.xml`、`go.mod` 等）
  - 可能影响 `docs/index.md`
- 顶层目录结构变化
  - 可能影响 `docs/directory-structure.md`
- 模块间依赖变化
  - 可能影响 `docs/architecture.md`
- 单个模块内部文件变化
  - 影响对应的 `docs/modules/<module>.md`
- 新模块出现
  - 需要新增模块文档，并可能更新 `architecture.md`、`learning-path.md`
- 模块删除/重命名
  - 需要用户确认后删除/重命名对应模块文档，并更新架构文档与学习路径

### 步骤 1.3：向用户展示变更摘要

向用户展示：
- 比较基线：`BASE_REF`
- 检测到的变更文件数
- 受影响模块列表
- 可能需要更新的文档列表

若未检测到变更，直接告知用户无需更新并结束流程。

---

## 阶段 2：删除/重命名确认

若检测到模块删除或重命名，先向用户确认，例如：

“Detected the following module changes:
- `module-x`: deleted
- `module-y` -> `module-z`: renamed

Please confirm whether I should:
- delete `docs/modules/module-x.md`
- rename `docs/modules/module-y.md` to `docs/modules/module-z.md`”

只有在用户确认后，才执行删除或重命名操作。

---

## 阶段 3：选择性更新文档

按以下规则更新：

| 变化类型 | 更新文档 |
|---------|---------|
| 配置文件变化 | `docs/index.md` |
| 顶层结构变化 | `docs/directory-structure.md` |
| 模块内部变化 | 对应 `docs/modules/<name>.md` |
| 模块依赖变化 | `docs/architecture.md` |
| 新模块 | 新建 `docs/modules/<name>.md`，并更新 `architecture.md` |
| 删除模块 | 删除对应模块文档，并更新 `architecture.md` |
| 重命名模块 | 重命名对应模块文档，并更新 `architecture.md` |

### 步骤 3.1：必要时重新扫描仓库

如果变化影响：
- 技术栈判断
- 模块列表
- 顶层目录结构
- 架构关系

则调用 `repo-wiki:repo-scanner` 重新获取当前仓库扫描结果。

**SubAgent**：`repo-wiki:repo-scanner`

**Prompt 模板**：

```text
Re-scan the current repository for incremental wiki update.

Target language for final docs: {DOC_LANG}
Focus on current tech stack, directory structure, module list, and entry points.
```

### 步骤 3.2：必要时重新分析架构

如果变化影响架构关系、模块依赖、入口点或外部集成，则调用 `repo-wiki:architecture-analyzer`。

**SubAgent**：`repo-wiki:architecture-analyzer`

**Prompt 模板**：

```text
Re-analyze the repository architecture for incremental wiki update.

Target language: {DOC_LANG}
Repository scan summary: {当前扫描摘要}
Changed files: {变更文件列表}
Affected modules: {受影响模块列表}
```

### 步骤 3.3：更新总览与目录结构

如果配置或目录结构变化影响总览信息，则在更新前先读取对应模板：
- `templates/wiki/index-template.md`
- `templates/wiki/directory-structure-template.md`
- `templates/wiki/writing-rules.md`

然后调用以下 writer 代理生成最终文档内容：
- `repo-wiki:index-writer`
- `repo-wiki:directory-writer`

若选择重写整篇文档，则必须通过对应 writer 代理按模板结构生成，不得绕过模板直接自由写作。

### 步骤 3.4：更新架构文档

如果依赖关系、模块列表、入口点或外部集成发生变化，则在更新前先读取：
- `templates/wiki/architecture-template.md`
- `templates/wiki/writing-rules.md`

然后调用 `repo-wiki:architecture-writer` 生成最终的 `docs/architecture.md` 内容。

若需要整篇重写，则必须通过 writer 代理按模板结构生成。

### 步骤 3.5：更新模块文档

对每个受影响模块，调用 `repo-wiki:module-analyzer` 重新分析。

**SubAgent**：`repo-wiki:module-analyzer`

**Prompt 模板**：

```text
Re-analyze this module for incremental wiki update.

Module name: {module_name}
Module path: {module_path}
Target language: {DOC_LANG}

Project context:
- Current tech stack: {当前扫描摘要}
- Current architecture summary: {当前架构摘要}
- Reason for update: {该模块涉及的变更文件/变更类型}
```

将结果覆盖写入对应模块文档。

若是新增模块，则新建对应 `docs/modules/<module-name>.md`。

### 步骤 3.6：更新学习路径

若以下任一情况成立，则在更新前先读取：
- `templates/wiki/learning-path-template.md`
- `templates/wiki/writing-rules.md`

并调用 `repo-wiki:learning-path-analyzer` 重新生成学习路径分析结果，再调用 `repo-wiki:learning-path-writer` 生成最终的 `docs/learning-path.md` 内容：
- 模块新增
- 模块删除
- 模块重命名
- 模块依赖关系变化
- 模块复杂度明显变化

**SubAgent**：`repo-wiki:learning-path-analyzer`

**Prompt 模板**：

```text
Rebuild the repository learning path for incremental wiki update.

Target language: {DOC_LANG}
Repository scan summary: {当前扫描摘要}
Architecture summary: {当前架构摘要}
Affected modules: {受影响模块列表}
Updated module summaries: {已更新模块文档摘要}
```

如果只是某个模块内部实现轻微调整，且不影响整体阅读顺序，则不更新学习路径。

---

## 阶段 4：输出更新总结

向用户输出完整总结，至少包含：
- 比较基线：`BASE_REF`
- 已更新文档列表
- 新增文档列表
- 删除文档列表（如有，且已确认）
- 未变化文档列表

输出示例风格：

```text
Incremental update complete. Compared against: <BASE_REF>

Updated:
- docs/modules/module-a.md
- docs/architecture.md

Added:
- docs/modules/module-new.md

Deleted:
- docs/modules/module-old.md

Unchanged:
- docs/index.md
- docs/directory-structure.md
```

最后请提醒用户审阅受影响文档。

不要只回复“已调用 update skill”。必须按上述流程进行变更检测、选择性更新和结果汇报。
