# AI Agent Root Gateway & Framework Router

> **Universal Entrypoint**: This repository (`ai-guardrails-harness`) defines architecture patterns, coding standards, and testing policies across language stacks.

---

## 1. Global Mandates (Cross-Stack Execution Rules)

All AI agents (GitHub Copilot, Claude Code, Cursor, Aider, custom harnesses) MUST follow these rules regardless of target stack:

1. **Token Conservation & AST Navigation**:
   * Do NOT load full file trees or perform brute-force vector/grep searches.
   * Query the local AST graph database via MCP tools (`get_call_graph`, `get_impact_analysis`) to identify relevant package paths.
   * Read **only** the specific sub-directory `.md` guardrail files needed for the active task.

2. **Core Architectural Boundaries**:
   * **Strict Isolation**: Maintain clear layer separation (e.g., API $\rightarrow$ Service $\rightarrow$ Data access).
   * **No Direct Storage Access from API**: HTTP/API handlers MUST NEVER inject or call repositories/databases directly.
   * **Immutability First**: Default to immutable data transfer structures across all layers.
   * **Null Safety**: Avoid returning raw `null` values across business interfaces.

3. **Build & Self-Correction Feedback Loop**:
   * Run targeted local build/test checks after generating or modifying code.
   * If compilation or test failures occur, parse the terminal error trace, inspect affected files, and self-correct prior to presenting final output.

---

## 2. Language Gateway Index

Identify the target ecosystem from the codebase and traverse directly to its root router:

* **Java Ecosystem**: Navigate to [`java/index.md`](java/index.md)
* **Python Ecosystem** *(Future)*: Navigate to `python/index.md`
* **Go Ecosystem** *(Future)*: Navigate to `go/index.md`

---

## 3. Provider Wrappers

Ensure local IDE tools defer directly to this file:

* **GitHub Copilot**: `.github/copilot-instructions.md` $\rightarrow$ Defers to `/AGENTS.md`.
* **Claude Code**: `CLAUDE.md` / `.claude/CLAUDE.md` $\rightarrow$ `@AGENTS.md`.
* **Cursor**: `.cursor/rules/*.mdc` $\rightarrow$ Set `globs: **/*` referencing `/AGENTS.md`.