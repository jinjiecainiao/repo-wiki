# repo-wiki

A Claude Code plugin that analyzes any repository and generates structured wiki documentation to help newcomers understand the codebase.

## Features

- **Universal** — Works with any programming language (Java, TypeScript, Python, Go, Rust, C#, Ruby, and more)
- **Progressive Generation** — Four phases with user review between each:
  1. Project Overview + Directory Structure
  2. Architecture Analysis (with Mermaid diagrams)
  3. Core Module Deep-Dive
  4. Learning Path for Newcomers
- **Multi-language Docs** — Generate documentation in any language via `--lang` flag
- **Parallel Analysis** — Module analysis runs concurrently via subagents

## Installation

```bash
# Add the marketplace (first time only)
/plugin marketplace add jinjiecainiao/repo-wiki-marketplace

# Install the plugin
/plugin install repo-wiki@jinjiecainiao
```

## Usage

Navigate to any repository and run:

```
/wiki
```

With options:

```
/wiki --lang zh-CN
```

### Supported Languages

`en` (default), `zh-CN`, `ja`, `ko`, `es`, `fr`, `de`, and any other language code.

## Output

The plugin generates documentation in the `docs/` directory:

```
docs/
├── index.md                # Project overview & quick start
├── directory-structure.md  # What each directory does
├── architecture.md         # Architecture diagrams & data flow
├── learning-path.md        # Recommended reading order
└── modules/
    ├── auth.md             # Module deep-dive
    ├── api.md
    └── ...
```

## How It Works

1. **Phase 1** scans config files and directory structure to identify tech stack and modules
2. **Phase 2** analyzes import statements to map module dependencies and architectural layers
3. **Phase 3** dispatches parallel agents to deeply analyze each core module
4. **Phase 4** synthesizes a learning path based on module complexity and dependencies

Each phase produces output for user review before proceeding.

## License

MIT
