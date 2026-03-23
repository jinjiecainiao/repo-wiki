---
name: generate-wiki
description: Use when the user wants to generate wiki documentation for a repository to help newcomers understand the codebase - analyzes project structure, architecture, modules and produces structured docs
---

# Generate Repository Wiki

## Overview

Analyze the current repository and progressively generate structured wiki documentation into `docs/`. Designed to help newcomers understand an open-source project through guided learning.

## When to Use

- User runs `/wiki` or asks to generate wiki/documentation for a repository
- User wants to create onboarding docs for a codebase
- User wants to document project architecture for newcomers

## Parameters

- `--lang <code>`: Documentation language. Default: `en`. Examples: `zh-CN`, `ja`, `ko`, `en`.

## Output Structure

```
docs/
├── index.md                # Project overview
├── directory-structure.md  # Directory structure guide
├── architecture.md         # Architecture (with Mermaid diagrams)
├── learning-path.md        # Recommended learning path
└── modules/
    ├── <module-a>.md       # Module deep-dive
    └── ...
```

## Process

Execute four phases sequentially. After each phase, present output to user for review before proceeding.

### Phase 1: Quick Scan → Project Overview + Directory Structure

**Goal:** Give the user a fast first result.

**Steps:**

1. Dispatch the `repo-scanner` agent to analyze the repository:
   - Scan for config files (pom.xml, package.json, go.mod, Cargo.toml, etc.)
   - Map top-level and second-level directory structure
   - Identify tech stack, build tools, CI/CD, containerization
   - List candidate modules with estimated purposes

2. Using the scanner's output, generate two files:

   **`docs/index.md`** — Project Overview:
   - Project name and one-paragraph description (from README)
   - Tech stack summary (languages, frameworks, databases, infrastructure)
   - Quick start guide (how to build/run, extracted from README or config)
   - Project status (license, CI badges if applicable)

   **`docs/directory-structure.md`** — Directory Guide:
   - Tree view of key directories (skip node_modules, build outputs, etc.)
   - One sentence explaining each directory's purpose
   - Highlight which directories contain the core business logic

3. Present both files to the user. Ask: "Phase 1 complete. Review the project overview and directory structure. Any corrections before I continue to architecture analysis?"

### Phase 2: Architecture Analysis

**Goal:** Map how modules connect and data flows.

**Steps:**

1. Based on Phase 1 scanner results, analyze module dependencies:
   - Use Grep to find import/require/use statements across the codebase
   - Map which modules depend on which
   - Identify architectural layers (e.g., Controller → Service → Repository)
   - Find entry points (main, index, app files)
   - Identify configuration centers

2. Generate **`docs/architecture.md`**:
   - High-level architecture description
   - Mermaid diagram showing module relationships:
     ```mermaid
     graph TD
       A[Module A] --> B[Module B]
       A --> C[Module C]
     ```
   - Layer diagram (if applicable):
     ```mermaid
     graph TD
       subgraph Presentation
         Controllers
       end
       subgraph Business
         Services
       end
       subgraph Data
         Repositories
       end
       Controllers --> Services --> Repositories
     ```
   - Data flow description for key operations
   - External integrations (APIs, databases, message queues)

3. Present to user. Ask: "Phase 2 complete. Review the architecture doc. Any corrections before I dive into individual modules?"

### Phase 3: Core Module Deep-Dive

**Goal:** Document each core module in detail.

**Steps:**

1. From the module list identified in Phase 1, confirm with user:
   "I identified these core modules: [list]. Should I document all of them, or focus on specific ones?"

2. For each confirmed module, dispatch a `module-analyzer` agent (parallel when possible). Provide each agent with:
   - Module name and path
   - Target language (from --lang parameter)
   - Project context (tech stack, architecture summary from Phase 2)

3. Collect results and write each to `docs/modules/<module-name>.md`

4. Present module list to user. Ask: "Phase 3 complete. I've generated docs for [N] modules. Review them and let me know if any need adjustments."

### Phase 4: Learning Path

**Goal:** Guide newcomers on where to start reading code.

**Steps:**

1. Analyze all previous outputs to determine:
   - Module complexity (file count, dependency count)
   - Dependency order (which modules are prerequisites for others)
   - Which modules are most self-contained (good starting points)

2. Generate **`docs/learning-path.md`**:
   - Recommended reading order as a numbered list
   - For each step:
     - Module name with link to its doc
     - Why read this now (what you'll learn)
     - Prerequisites (which modules/concepts to understand first)
     - Difficulty estimate (Beginner / Intermediate / Advanced)
     - Key files to focus on (2-3 most important)
   - A Mermaid diagram showing the dependency/learning flow:
     ```mermaid
     graph LR
       A[1. Config] --> B[2. Models]
       B --> C[3. Services]
       C --> D[4. Controllers]
     ```

3. Present to user. Ask: "Phase 4 complete — all wiki documentation is generated. Review the learning path. Want me to make any adjustments?"

## Language Handling

All generated documentation content MUST be written in the language specified by `--lang`:
- Section headings, descriptions, explanations → target language
- Code identifiers, file paths, technical terms → keep original (English)
- Mermaid diagram labels → target language

Default is English (`en`) if `--lang` is not specified.

## Important Notes

- Always create `docs/` directory if it doesn't exist
- Skip generated/vendored directories: node_modules, vendor, build, dist, target, .git, __pycache__, etc.
- If the repository already has `docs/` content, warn the user before overwriting
- For unrecognized languages, fall back to directory structure + README + comment analysis (skip import parsing)
- Keep each document concise — a newcomer should be able to read the entire wiki in under 1 hour
