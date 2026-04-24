# evobase-plugin_副本 项目全面总结与复现指南

## 1. 项目定位

`evobase-plugin_副本` 是一个面向 Claude Code / Agent 工作流的 Repo Wiki 自动化插件。它不是传统的 Web 应用或后端服务，而是一套由 **命令定义 + 子代理定义 + API/元数据脚本 + 模板规范** 组成的文档自动化系统。

它的核心目标是：

- 自动分析目标代码仓库
- 按业务语义生成 Wiki 目录结构
- 批量生成业务 Wiki 文档
- 对已有 Wiki 做增量保鲜
- 接入 Evobase / KIS 知识库检索已有知识
- 从中心化 Repo Wiki 拉取文档到本地
- 自动维护文档 Front Matter、版本、校验和、更新时间等元数据

该项目适合在其他项目中复现为：

1. 面向代码仓库的自动化文档系统
2. 面向知识库的 AI 文档生成插件
3. 面向研发团队的“代码 -> 结构化知识”生成器

---

## 2. 项目整体架构

整体采用分层结构：

- **命令层（Commands）**：负责用户入口、流程编排、交互和批次控制
- **子代理层（Agents）**：负责具体执行目录生成、文档生成、文档保鲜、知识检索等任务
- **脚本层（Scripts）**：负责远端 API 调用、Front Matter 生成更新、环境变量注入
- **模板层（Templates）**：负责输出格式、目录 schema、日志模板、业务知识收集规范

推荐按下面的逻辑理解整个系统：

```text
用户命令
  -> 命令编排 Markdown
    -> 调用子代理 Markdown
      -> 使用 Read / Glob / Grep / Bash / Write / Edit 等工具
        -> 调用 Node/Python 脚本
          -> 读写 .evobase 知识库与业务wiki目录
```

---

## 3. 目录结构与模块说明

项目核心目录如下：

- `evobase-plugin_副本/commands/`
- `evobase-plugin_副本/agents/`
- `evobase-plugin_副本/agents/wiki-agents/`
- `evobase-plugin_副本/scripts/`
- `evobase-plugin_副本/templates/repowiki/`
- `evobase-plugin_副本/hooks.json`
- `evobase-plugin_副本/CLAUDE.md`

### 3.1 命令层

#### `evobase-plugin_副本/commands/gen_repowiki.md`

**功能**：生成 Repo Wiki 文档。

**职责**：
- 检查知识库是否已初始化
- 选择目标知识库
- 判断生成模式（全量 / 增量）
- 收集用户意图
- 调用目录生成代理
- 收集业务知识
- 分批并发调用文档生成代理
- 写入变更日志
- 清理中间产物

**实现方式**：
- 用 Markdown 定义完整工作流
- 通过 Agent 工具调用 `wiki-structure-generator`
- 通过 Agent 工具调用 `wiki-document-generator`
- 通过模板决定 changelog 的写法
- 通过 `_business_docs` 暂存业务知识文件

**复现建议**：
如果你要在别的项目中复现“自动生成文档”的能力，最先需要复制的就是这类“命令编排层”。不要把所有逻辑塞进一个 Agent，而是拆成：
- 前置检查
- 用户意图确认
- 结构规划
- 文档生成
- 日志记录

---

#### `evobase-plugin_副本/commands/update_repowiki.md`

**功能**：保鲜已有 Wiki 文档。

**职责**：
- 检查知识库
- 选择知识库
- 确定保鲜范围
- 按目录分组文档
- 并发调用 `wiki-document-refresher`
- 写入更新日志

**实现方式**：
- 用 Markdown 描述“保鲜而非重写”的流程
- 每批最多并发 3 个子代理
- 每个子代理最多处理 5 篇文档
- 更新完成后统一写 changelog

**复现价值**：
这是“持续文档同步”的关键。如果别的项目只实现生成，不实现保鲜，文档会很快过时。这个文件定义了如何把文档生成系统变成长期可维护系统。

---

### 3.2 Wiki 子代理层

#### `evobase-plugin_副本/agents/wiki-agents/wiki-structure-generator.md`

**功能**：根据用户意图与代码仓库实际内容，生成严格两级的 Wiki 目录结构。

**输入**：
- Wiki 文档根目录
- 用户意图（自然语言 / 指定目录）

**输出**：
- 符合 schema 的 JSON 目录结构

**核心能力**：
- 理解用户意图
- 探索代码仓库边界
- 抽象业务语义目录
- 生成 `category_name + documents[]`

**关键实现规则**：
- 目录必须是两级结构
- 目录名必须是业务语义而不是技术文件夹名
- 文档粒度不能过大或过细
- 必须基于代码事实，不允许编造
- 只做适度探索，不深入代码实现细节

**配套实现文件**：
- `evobase-plugin_副本/templates/repowiki/wiki-structure-schema.json`
- `evobase-plugin_副本/templates/repowiki/repowiki-structure.md`

**复现建议**：
如果在别的项目中复现，建议保留“先规划目录、后写正文”的二阶段设计。不要一上来直接生成全文，否则文档粒度会很差。

---

#### `evobase-plugin_副本/agents/wiki-agents/wiki-document-generator.md`

**功能**：批量生成同一目录下的多篇 Wiki 文档。

**职责**：
- 汇总当前目录全部文档描述
- 一次性探索共享代码区域
- 逐篇生成文档
- 写入文档文件
- 调用 frontmatter 脚本生成元数据

**核心设计**：
- 同一目录的文档共用一次代码探索结果，避免重复分析
- 每一段描述必须基于代码证据
- 支持业务知识优先
- 支持 Mermaid 图辅助说明
- 不把文档正文输出到对话，直接写文件

**关键实现依赖**：
- `evobase-plugin_副本/scripts/wiki_frontmatter.py`
- `evobase-plugin_副本/templates/repowiki/mermaid-patterns.md`

**复现建议**：
在其他项目中复现时，不要设计成“一个文档一次全仓扫描”，而是尽量按主题目录批量处理，这样可以显著降低工具调用成本。

---

#### `evobase-plugin_副本/agents/wiki-agents/wiki-document-refresher.md`

**功能**：检测已有 Wiki 与当前代码仓库是否一致，并定向更新过时部分。

**职责**：
- 读取文档现状
- 抽取文档中引用的代码文件
- 对照代码验证文档是否过时
- 仅更新有差异的部分
- 更新后刷新 frontmatter 机械字段

**设计原则**：
- 不做全量重写
- 不做风格优化
- 不做无关润色
- 仅修改真正过时的描述和失效引用

**配套实现文件**：
- `evobase-plugin_副本/scripts/wiki_frontmatter.py`

**复现建议**：
这是文档系统长期可用的关键。复现时要明确区分：
- 生成代理：偏创建
- 保鲜代理：偏校验 + 定向编辑

二者最好分开，不要共用一个 prompt。

---

### 3.3 知识库辅助代理层

#### `evobase-plugin_副本/agents/knowledge-search-agent.md`

**功能**：在 Evobase 知识库中检索术语、规则、架构设计、背景知识。

**核心实现路径**：
1. 优先本地检索 `.evobase/`
2. 优先读 `AGENTS.md` 做导航
3. 如果本地不够，再调用远端语义检索

**依赖脚本**：
- `evobase-plugin_副本/scripts/evobase_api.js`

**为什么重要**：
自动生成文档时，光靠代码往往不够。术语定义、业务规则、历史背景很多不在代码中，这个代理承担“业务知识补充层”的作用。

**复现建议**：
如果你的目标项目已有内部文档库或知识库，建议直接按这个模式增加一个“知识检索代理”，把代码分析和知识检索组合起来。

---

#### `evobase-plugin_副本/agents/knowledge-conflict-resolver.md`

**功能**：在知识库发生 git merge 冲突时，对 Markdown 冲突块做语义融合并提交。

**职责**：
- 定位冲突文件
- 读取冲突块
- 合并本地与主库内容
- 移除冲突标记
- 提交 Git 结果

**适用场景**：
多人维护知识库时，经常会发生知识文档冲突。这个代理将冲突解决自动化。

**复现建议**：
如果你的知识库是多人协作 Git 仓库，这是一个非常值得复用的模块。

---

#### `evobase-plugin_副本/agents/knowledge-init.md`

**功能**：初始化知识库目录结构并复制模板文件。

**职责**：
- 创建标准目录
- 复制 AGENTS 模板
- 复制中间件知识模板
- 复制规范模板

**实现方式**：
- 完全用 Bash `mkdir -p` 和 `cp -n`
- 不使用交互、不覆盖已有文件

**复现建议**：
任何知识库系统都需要初始化阶段。复现时建议保留一个“只做脚手架”的轻量代理，避免把初始化逻辑混入主工作流。

---

### 3.4 脚本层

#### `evobase-plugin_副本/scripts/repowiki_api.js`

**功能**：Repo Wiki 中心化 HTTP API 的 Node CLI 封装。

**支持子命令**：
- `status`
- `tree`
- `build`
- `fetch-files`
- `pull`

**核心职责**：
- 访问远端 Repo Wiki 接口
- 获取 wiki 状态、结构树、子文档内容
- 触发构建
- 拉取远端文档到本地
- 清洗 Markdown 内容
- 为拉取文档生成 frontmatter

**关键实现点**：
- `repoWikiRequest()`：统一封装请求与鉴权
- `parseArgs()`：命令行参数解析
- `normalizeMdSpacing()`：标准化 Markdown 段落和标题空行
- `collectNodes()`：递归收集目录树中的文档与目录
- `cmdPull()`：完成整套“树获取 -> 内容拉取 -> 本地写入 -> 元数据生成”流程

**关键价值**：
这是“中心化文档平台”和“本地知识库目录”之间的桥梁。

**复现建议**：
如果别的项目也有远端文档中心或平台 API，建议仿照本脚本做成独立 CLI，而不是把 API 细节写死在 Agent prompt 里。

---

#### `evobase-plugin_副本/scripts/evobase_api.js`

**功能**：Evobase 知识库远端 API 的 Node CLI 封装。

**支持子命令**：
- `list`
- `cat`
- `glob`
- `grep`
- `search`

**核心职责**：
- 从 `.evobase/.repos.json` 解析 repo 与 `kisId`
- 调用远端知识库 API
- 返回目录结构、文件内容、通配符匹配、正则搜索、向量检索结果

**关键实现点**：
- `resolveKisId()`：解析 repo 与 kisId 的关系
- `apiRequest()`：统一请求入口
- `cmdSearch()`：语义检索实现
- `cmdList()/cmdCat()/cmdGlob()/cmdGrep()`：远端知识浏览工具化实现

**复现建议**：
如果需要把“知识库”作为文档生成上下文，最好提供这种独立 CLI，以便多个代理共用。

---

#### `evobase-plugin_副本/scripts/wiki_frontmatter.py`

**功能**：统一生成和更新 Wiki 文档的 YAML Front Matter。

**支持子命令**：
- `gen`
- `update`
- `pull`

**核心职责**：
- 生成文件头元数据
- 更新已有文件头中的机械字段
- 给拉取到本地的文档批量补文件头

**管理字段**：
- `title`
- `description`
- `created_at`
- `updated_at`
- `author`
- `updated_author`
- `file_name`
- `file_path`
- `version`
- `tags`
- `language`
- `encoding`
- `checksum`
- `source`

**关键实现点**：
- `get_git_author()`：获取作者信息
- `_get_nick_from_api()`：优先调用 API 获取花名
- `extract_tags()`：从 `##` 标题提取标签
- `extract_description()`：从正文抽取描述
- `compute_checksum()`：计算 SHA256 校验和
- `parse_file_path()`：从绝对路径推导相对 wiki 路径
- `cmd_gen()`：为新文档生成 frontmatter
- `cmd_update()`：为已有文档刷新更新时间、版本号、校验和
- `cmd_pull()`：为拉取文档批量生成 frontmatter

**为什么非常关键**：
很多项目只生成 Markdown，但没有元数据治理。这个脚本让 Wiki 具备：
- 可追踪性
- 可版本化
- 可校验
- 可检索标签化

**复现建议**：
如果在别的项目中复现，推荐优先复制这套 frontmatter 设计。即使不接 Evobase，也值得保留元数据治理能力。

---

#### `evobase-plugin_副本/scripts/init_env.sh`

**功能**：会话启动时注入环境变量。

**职责**：
- 注入 `EVOBASE_PROJECT_ROOT`
- 注入 `EVOBASE_PLUGIN_DIR`
- 尝试同步 PR 状态

**配套配置文件**：
- `evobase-plugin_副本/hooks.json`

**复现建议**：
任何多脚本、多代理系统都需要统一的运行时上下文。环境变量初始化最好通过 hook 自动完成，而不是在每个命令中重复推导。

---

### 3.5 模板层

#### `evobase-plugin_副本/templates/repowiki/repowiki-structure.md`

**功能**：定义项目整体结构说明。

**内容涵盖**：
- 业务wiki目录结构
- `_business_docs` 中间产物目录
- `wiki_changelog.md` 的用途
- 命令职责
- 子代理职责
- 脚本职责

**作用**：
这是整个插件的架构参考文档，很多命令在开始执行前都会先读取它。

---

#### `evobase-plugin_副本/templates/repowiki/wiki-structure-schema.json`

**功能**：定义 Wiki 目录结构 JSON 的标准格式。

**核心字段**：
- `categories`
- `category_name`
- `documents`
- `title`
- `description`
- `output_file`
- `business_knowledge`

**作用**：
保证目录生成代理输出格式稳定，便于后续代理消费。

---

#### `evobase-plugin_副本/templates/repowiki/references/collect-business-knowledge.md`

**功能**：定义“业务知识收集与分配”的流程。

**实现思路**：
- 引导用户输入术语、规则、背景、格式要求
- 区分短文本 / 长文本 / 文件输入
- 长文本写入 `_business_docs`
- 再将业务知识分配到各文档的 `business_knowledge` 字段

**为什么重要**：
它把“用户脑中的隐性知识”转成“可被生成代理消费的结构化输入”。

---

#### `evobase-plugin_副本/templates/repowiki/mermaid-patterns.md`

**功能**：给文档生成代理提供 Mermaid 图表模板参考。

**作用**：
提升输出 Wiki 的可读性，使文档不仅是文字摘要，还能表达流程、依赖、结构关系。

---

#### `evobase-plugin_副本/templates/repowiki/changelog-template/`

**功能**：定义不同操作的变更日志模板。

**包括**：
- `changelog_gen_full.md`
- `changelog_gen_incremental.md`
- `changelog_update.md`
- `changelog_pull.md`

**作用**：
每次执行生成、更新、拉取后，都会写入 `wiki_changelog.md`，形成操作审计和知识演进记录。

---

### 3.6 配置与说明文件

#### `evobase-plugin_副本/hooks.json`

**功能**：定义 Claude 插件会话启动钩子。

**当前用途**：
- 在 SessionStart 时执行 `scripts/init_env.sh`

---

#### `evobase-plugin_副本/CLAUDE.md`

**功能**：给 Claude / Agent 提供项目说明、架构约束和关键约定。

**内容包括**：
- 项目定位
- 命令和代理分层结构
- 并发执行约定
- 知识库依赖
- 文档结构与日志策略
- 运行环境说明

**复现建议**：
如果你在其他项目里复现类似插件，强烈建议保留 `CLAUDE.md` 这种“给 AI 看”的架构说明文件。它能极大提升代理行为稳定性。

---

## 4. 核心业务流程

## 4.1 生成 Wiki 流程

入口文件：`evobase-plugin_副本/commands/gen_repowiki.md`

流程如下：

1. 检查环境变量与知识库是否存在
2. 读取 `.evobase/.repos.json` 选择目标 repo
3. 检查 `业务wiki/` 目录中是否已有文档
4. 判断本次是全量生成还是增量新增
5. 收集用户生成意图
6. 调用 `wiki-structure-generator` 生成结构 JSON
7. 让用户确认目录树
8. 通过 `collect-business-knowledge.md` 可选收集业务知识
9. 按一级目录分组，每组最多 5 篇文档
10. 每批最多 3 个子代理并发调用 `wiki-document-generator`
11. 每篇文档写入目标目录，并调用 `wiki_frontmatter.py gen`
12. 写入 `wiki_changelog.md`
13. 删除 `_business_docs` 中间产物

**这个流程的精髓**：
先定结构，再填内容；先收业务知识，再生成文档；按主题分组复用探索结果；最后再统一补元数据和日志。

---

## 4.2 保鲜 Wiki 流程

入口文件：`evobase-plugin_副本/commands/update_repowiki.md`

流程如下：

1. 检查知识库
2. 选择要更新的知识库
3. 让用户指定要保鲜的文件或目录
4. 收集最终文档列表
5. 按目录分组，每组最多 5 篇
6. 每批最多 3 个代理并发调用 `wiki-document-refresher`
7. 每个代理对比代码与文档，做定向更新
8. 调用 `wiki_frontmatter.py update`
9. 写入 `wiki_changelog.md`

**这个流程的精髓**：
它把“代码变更 -> 文档同步”做成了可持续的运维流程，而不是一次性生成。

---

## 4.3 远端拉取 Wiki 流程

核心实现文件：`evobase-plugin_副本/scripts/repowiki_api.js`

核心过程：

1. 获取远端 Wiki 树
2. 提取目录与文档节点
3. 分批获取所有子文档内容
4. 在本地创建目录结构
5. 清理非标准 Markdown 内容
6. 写入本地 `.md` 文件
7. 调用 `wiki_frontmatter.py pull` 生成文件头
8. 输出新增 / 跳过 / 失败统计

**为什么值得复现**：
很多系统只做“生成”，但实际团队经常需要“把平台内容拉回本地仓库”做版本化管理。这个脚本直接给出了通用做法。

---

## 4.4 知识检索流程

入口文件：`evobase-plugin_副本/agents/knowledge-search-agent.md`

流程如下：

1. 优先读取本地 `.evobase/<repo>/AGENTS.md`
2. 如果命中目录地图，则定位相关文件和章节
3. 若未命中，用 `Glob + Grep + Read` 搜索本地知识库
4. 如果本地没有或不完整，再调用 `evobase_api.js search`
5. 返回本地或远端知识片段供生成代理使用

**这个流程的精髓**：
不是简单做全文搜索，而是结合结构化导航和语义检索，提高命中率和可解释性。

---

## 5. 关键设计亮点

### 5.1 命令与执行分离

命令只负责编排，不承担具体文档写作和代码分析；执行细节全部下沉到子代理。这样做的好处是：
- 可维护性高
- 提示词职责明确
- 更适合并发执行
- 更容易在别的项目中替换部分能力

### 5.2 结构规划与正文生成分离

先由 `wiki-structure-generator` 决定“写什么”，再由 `wiki-document-generator` 决定“怎么写”。这能避免：
- 文档重复
- 文档粒度混乱
- 目录结构不稳定

### 5.3 文档保鲜独立建模

保鲜不是重生成，而是独立的“比对 + 定向修改”流程。这比直接重写文档更稳，更适合长期维护。

### 5.4 业务知识显式注入

项目意识到“代码 != 全部知识”，因此设计了专门的业务知识收集与分配机制。这是很多自动文档系统容易缺失的一环。

### 5.5 元数据治理完整

通过 `wiki_frontmatter.py`，每篇文档都有：
- 来源
- 版本
- 作者
- 更新时间
- 校验和
- 标签

这让知识库具备了治理能力，而不仅是若干散落 Markdown 文件。

### 5.6 批次并发控制明确

并发规则是显式定义的：
- 每个子代理最多处理 5 篇文档
- 每批最多 3 个子代理并发
- 逐批执行并记录 TODO

这在复现时非常值得保留，因为它控制了上下文规模和生成稳定性。

---

## 6. 在其他项目中复现时的最小实现集

如果你要在其他项目中复现这套能力，推荐按下面优先级实现。

### 第一阶段：最小可用版

至少复制这些模块：

1. `commands/gen_repowiki.md`
2. `agents/wiki-agents/wiki-structure-generator.md`
3. `agents/wiki-agents/wiki-document-generator.md`
4. `scripts/wiki_frontmatter.py`
5. `templates/repowiki/wiki-structure-schema.json`
6. `templates/repowiki/repowiki-structure.md`
7. `hooks.json`
8. `scripts/init_env.sh`

这样你就能做出：
- 自动规划目录
- 自动写文档
- 自动维护 frontmatter

### 第二阶段：增强版

再加入：

1. `commands/update_repowiki.md`
2. `agents/wiki-agents/wiki-document-refresher.md`
3. `templates/repowiki/changelog-template/*`

这样你就具备：
- 文档保鲜
- 变更日志追踪

### 第三阶段：知识增强版

再加入：

1. `agents/knowledge-search-agent.md`
2. `agents/knowledge-init.md`
3. `agents/knowledge-conflict-resolver.md`
4. `scripts/evobase_api.js`
5. `templates/repowiki/references/collect-business-knowledge.md`

这样你就具备：
- 知识库初始化
- 本地 / 远端知识检索
- 多人协作冲突解决
- 业务知识增强生成

### 第四阶段：平台集成版

最后加入：

1. `scripts/repowiki_api.js`

这样你就具备：
- 远端 Repo Wiki 平台对接
- 一键 pull 到本地
- 平台与本地知识库双向配合能力

---

## 7. 推荐复现步骤

### 步骤 1：先定义你的知识存储目录

建议保留类似结构：

```text
.evobase/
  <repo-name>/
    业务wiki/
    wiki_changelog.md
```

如果不想绑定 `.evobase` 命名，也至少保留：
- 文档根目录
- 中间产物目录
- changelog 文件

### 步骤 2：先实现 Front Matter 工具

优先把 `scripts/wiki_frontmatter.py` 迁移过去，保证文档生成后立刻拥有可治理元数据。

### 步骤 3：实现目录规划代理

先做 `wiki-structure-generator`，确保输出结构稳定，再做正文生成。

### 步骤 4：实现文档生成代理

复现 `wiki-document-generator` 的设计思路：
- 主题聚合
- 共享探索
- 分批生成
- 文档证据引用

### 步骤 5：补上保鲜能力

实现 `wiki-document-refresher`，保证代码演进后文档还能同步更新。

### 步骤 6：再接知识库

如果你的团队有内部文档库，再引入：
- 本地知识目录
- AGENTS 导航地图
- 语义检索 CLI

### 步骤 7：最后接平台 API

如果存在中心化文档平台，再把 `repowiki_api.js` 这类脚本接入，实现平台到本地的桥接。

---

## 8. 复现时的关键注意事项

### 8.1 不要把所有逻辑写进一个 prompt

这个项目成功的关键之一，是把系统拆成：
- 编排层
- 执行层
- 脚本层
- 模板层

复现时如果把所有逻辑放进一个大 prompt，系统会变得不稳定、难调试、难维护。

### 8.2 文档生成一定要基于代码证据

如果没有明确要求“基于代码文件引用”，生成结果会很容易漂移成空泛总结。

### 8.3 业务知识必须有显式入口

纯代码分析适合解释实现，不适合还原业务规则。复现时最好保留业务知识输入通道。

### 8.4 元数据和日志不要省略

很多系统只关注“能不能生成出来”，但真正可用的系统还要能回答：
- 这篇文档是谁生成的
- 什么时候更新的
- 版本是什么
- 内容有没有变化
- 最近一次变更是什么

### 8.5 并发规模必须受控

多代理并发确实能提速，但并发过高会带来：
- 上下文竞争
- 失败率上升
- 文件写入混乱

这个项目给出的 5 篇 / 3 并发规则是一个非常实用的经验值。

---

## 9. 适合直接复用的代码文件清单

### 核心必复用
- `evobase-plugin_副本/commands/gen_repowiki.md`
- `evobase-plugin_副本/agents/wiki-agents/wiki-structure-generator.md`
- `evobase-plugin_副本/agents/wiki-agents/wiki-document-generator.md`
- `evobase-plugin_副本/scripts/wiki_frontmatter.py`
- `evobase-plugin_副本/templates/repowiki/wiki-structure-schema.json`
- `evobase-plugin_副本/templates/repowiki/repowiki-structure.md`

### 强烈建议复用
- `evobase-plugin_副本/commands/update_repowiki.md`
- `evobase-plugin_副本/agents/wiki-agents/wiki-document-refresher.md`
- `evobase-plugin_副本/templates/repowiki/references/collect-business-knowledge.md`
- `evobase-plugin_副本/templates/repowiki/changelog-template/*`
- `evobase-plugin_副本/hooks.json`
- `evobase-plugin_副本/scripts/init_env.sh`
- `evobase-plugin_副本/CLAUDE.md`

### 按场景选用
- `evobase-plugin_副本/agents/knowledge-search-agent.md`
- `evobase-plugin_副本/agents/knowledge-init.md`
- `evobase-plugin_副本/agents/knowledge-conflict-resolver.md`
- `evobase-plugin_副本/scripts/evobase_api.js`
- `evobase-plugin_副本/scripts/repowiki_api.js`

---

## 10. 一句话总结

`evobase-plugin_副本` 本质上是一套 **“代码仓库 -> 结构化 Wiki -> 持续保鲜 -> 知识库增强 -> 元数据治理”** 的 AI 文档自动化体系。

它最值得复现的不是某一个脚本，而是这一整套设计原则：

- 用命令编排复杂流程
- 用子代理拆分职责
- 用 schema 保证结构稳定
- 用 frontmatter 管理文档元数据
- 用 changelog 记录演进历史
- 用知识库补足代码之外的业务信息
- 用保鲜机制让文档长期有效

如果在其他项目中按这个思路落地，你复现出来的将不是“一次性的文档生成器”，而是一个真正可持续运营的 Repo Wiki 系统。

