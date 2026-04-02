# Capture Coverage Matrix

**Last updated:** 2026-03-31
**Purpose:** Maps every activity an engineer might perform during a session to how MethodProof sees it, which agent is responsible, and where the gaps are.

---

## Coverage Legend

| Symbol | Meaning |
|--------|---------|
| **FULL** | We see the complete data with context |
| **PARTIAL** | We see something but miss key signals |
| **INFERRED** | Not directly observed — reconstructed from timing/heuristics |
| **GAP** | No visibility |

---

## 1. Writing Code

| Activity | Coverage | Agent(s) | What We See | What We Miss |
|----------|----------|----------|------------|-------------|
| Type code manually | FULL | FS Watcher | File diff with lines added/removed, language, path | — |
| Accept AI inline completion (Copilot tab, Cursor tab) | **PARTIAL** | FS Watcher | File diff shows new code appeared. No signal it was AI-generated. | Can't distinguish "typed this" from "accepted AI suggestion." No reject signal. |
| Reject AI inline completion | **GAP** | — | Nothing. No file change occurs. | Invisible — critical signal for AI usage analysis. |
| Accept/modify AI chat suggestion (via MCP) | FULL | MCP Server + FS Watcher | Prompt text, completion text, model, tokens, latency. Then file diff shows what was actually used. | — |
| Use Claude Code CLI | **PARTIAL** | Terminal Monitor + Network Proxy + FS Watcher | Terminal command visible. Proxy sees Anthropic API calls (encrypted payload). File diffs captured. | Prompt and completion text not extracted from proxy. |
| Use Cursor built-in AI | **PARTIAL** | Network Proxy + FS Watcher | Proxy sees API calls to OpenAI/Anthropic/custom endpoints. File diffs captured. | Prompt/completion text not decoded from proxy payload. |
| Use ChatGPT/Claude web UI | **PARTIAL** | Extension + FS Watcher | URL + time on page + copy events. If code is pasted into editor, file diff captured. | No prompt text, no completion text, no accept/reject signal. |
| Copy from AI chat → paste into editor | **INFERRED** | Extension + FS Watcher | Extension: copy event (text length + source URL). FS Watcher: file edit. Link inferred by timing + content-length correlation. | Inference can be wrong. No direct causal link. |
| Refactor (rename symbol, extract function) | FULL | FS Watcher | Multi-file diffs | — |
| Navigate code (go to definition, find references) | **GAP** | — | Nothing unless navigation leads to a file change. | No visibility into code reading/exploration within editor. |
| Undo/redo | FULL | FS Watcher | Net file diff | Individual undo steps collapsed into net result |
| Use code snippets/templates | FULL | FS Watcher | File diff | Can't distinguish snippet expansion from manual typing |

---

## 2. Searching and Reading

| Activity | Coverage | Agent(s) | What We See | What We Miss |
|----------|----------|----------|------------|-------------|
| Web search (Google, Bing, DuckDuckGo) | FULL | Extension | Query text, search engine, URL, timestamp | — |
| Stack Overflow | FULL | Extension | URL, time on page, copy events with length + source | Specific answer content read (unless copied) |
| GitHub (repos, issues, PRs, code search) | FULL | Extension | URL, time on page, domain categorized as "code hosting" | — |
| Documentation sites (MDN, React docs, etc.) | FULL | Extension | URL, time on page, domain categorized as "docs" | Specific section read |
| AI chat in browser (chat.openai.com, claude.ai) | PARTIAL | Extension | URL, time on page, domain categorized as "AI chat," copy events | No prompt/completion text |
| Package registries (npm, PyPI, crates.io) | FULL | Extension | URL, time on page | — |
| Blog posts / tutorials | FULL | Extension | URL, time on page | — |
| Man pages / CLI help in terminal | FULL | Terminal Monitor | `man` or `--help` command + output snippet | — |
| Reading code in editor without editing | **GAP** | — | No signal | No visibility into which files/functions were read. Screen capture could show this but is expensive to process. |
| Reading in a second monitor | **PARTIAL** | Extension (if in browser) | Browser activity captured regardless of monitor. Editor/terminal on second monitor: no signal. | Non-browser activity on secondary monitor is invisible. |

---

## 3. Terminal / Shell

| Activity | Coverage | Agent(s) | What We See | What We Miss |
|----------|----------|----------|------------|-------------|
| Run application | FULL | Terminal Monitor | Command, exit code, truncated output (10KB) | Full output if >10KB |
| Run tests | FULL | Terminal Monitor | Command, framework detection (pytest/jest/go test/cargo test), pass/fail counts, duration | Individual test names (in truncated output) |
| Git commands | FULL | Terminal Monitor + FS Watcher | Command + exit code. Commits detected via `.git/refs` polling (hash, message, changed files). **Community challenges only:** full unified diff per commit (capped 50 KB, AES-256-GCM encrypted). **All other contexts (CLI, regular sessions):** structural only (no diff content). | — |
| Package install (npm, pip, cargo) | FULL | Terminal Monitor | Command + exit code | Package contents |
| Docker commands | FULL | Terminal Monitor | Command + exit code | Container-internal state changes |
| Build / lint / typecheck | FULL | Terminal Monitor | Command + exit code + truncated output | Full warning/error list if output exceeds 10KB |
| Database CLI (psql, cypher-shell, redis-cli) | **PARTIAL** | Terminal Monitor | Initial command visible. | Interactive session queries/responses inside the REPL are not captured. |
| curl / httpie | FULL | Terminal Monitor | Command + exit code + truncated response | Full response body if large |
| SSH into remote server | **PARTIAL** | Terminal Monitor | SSH command visible. | All activity within the remote session is invisible. |
| Environment setup (export, source, cd) | FULL | Terminal Monitor | Command + exit code | — |
| Piped/chained commands | FULL | Terminal Monitor | Full command string + final exit code | Intermediate pipe exit codes |

---

## 4. AI Tool Usage (The Big Complexity)

This is the most important coverage area and the one with the most gaps. Engineers use many different AI tools, most of which don't go through our MCP server.

| Tool | Coverage | How We See It | What We Miss |
|------|----------|--------------|-------------|
| LLM via MethodProof MCP server | **FULL** | Full prompt, completion, model, tokens, latency, temperature | — |
| GitHub Copilot (inline suggestions) | **PARTIAL** | Network Proxy: sees API calls to `api.github.com/copilot`. FS Watcher: sees accepted code in file diffs. | Prompt context sent to Copilot. Rejected suggestions. Accept/reject ratio. Ghost text that was shown but dismissed. |
| GitHub Copilot Chat | **PARTIAL** | Network Proxy: sees API calls. Terminal/panel output not captured. | Prompt and completion text. |
| Cursor AI (inline + chat) | **PARTIAL** | Network Proxy: sees API calls to Cursor's backend. FS Watcher: file diffs. | Prompt text, completion text, which model was used, accept/reject of inline suggestions. |
| Claude Code (CLI) | **PARTIAL** | Terminal Monitor: sees `claude` command invocation. Network Proxy: sees Anthropic API calls. FS Watcher: sees file changes Claude Code makes. | Prompt content, completion content, tool use details, multi-turn conversation. |
| Windsurf / Codeium | **PARTIAL** | Network Proxy: sees API calls. FS Watcher: file diffs. | Same as Cursor — no prompt/completion extraction. |
| Aider (CLI) | **PARTIAL** | Terminal Monitor: command. Network Proxy: API calls. FS Watcher: file diffs. | Prompt/completion content from the aider conversation. |
| ChatGPT web (chat.openai.com) | **PARTIAL** | Extension: URL, time on page, copy events. | Everything inside the chat — prompts, completions, conversation flow. |
| Claude web (claude.ai) | **PARTIAL** | Extension: URL, time on page, copy events. | Same as ChatGPT web. |
| Ollama / local models | **POTENTIALLY FULL** | Network Proxy: sees localhost API calls. Payload is unencrypted HTTP (not HTTPS). | Need to build decoders for Ollama's API format. |
| v0 / Bolt / other web AI tools | **PARTIAL** | Extension: URL, time, copy events. | Generated code content, prompts, iterations. |

---

## 5. Environment and Behavior

| Activity | Coverage | Agent(s) | What We See | What We Miss |
|----------|----------|----------|------------|-------------|
| Tab/file switching in editor | **GAP** | — | Nothing | Which files were open, switching frequency, reading patterns |
| Tab switching in browser | FULL | Extension | Active tab changes with timestamps | — |
| Idle time / thinking | **INFERRED** | Timeline gaps | Gap between actions in the temporal chain. Long gap = probable thinking or distraction. | Can't distinguish thinking from distraction/break. |
| Screen content | PARTIAL | Screen Capture | Periodic screenshots (5s interval, compressed) | Processing screenshots for semantic content is expensive. Raw storage is large. |
| Gaze / attention direction | PARTIAL | Attention Tracker | Head pose, gaze direction, eye open/closed (requires webcam + consent) | Accuracy depends on camera quality and positioning. |
| Multiple monitors | **PARTIAL** | Extension + Screen Capture | Browser activity on any monitor (extension). Primary screen captured. | Non-browser activity on secondary monitors. |
| Taking notes on paper | **GAP** | — | Nothing | — |
| Talking / rubber ducking | **GAP** | Attention Tracker (if enabled) | Face presence/absence, head movement | No audio capture (by design — privacy). |
| Diagramming (Excalidraw, Miro, etc.) | PARTIAL | Extension (if in browser) | URL + time on page | Diagram content |

---

## 6. Gap Prioritization

### Tier 1 — Must solve (these undermine the core value proposition)

**GAP 1: Non-MCP AI tool visibility**

The majority of engineers use Copilot, Cursor, Claude Code, or web AI interfaces — none of which go through our MCP server. If we only see AI interactions through our own MCP, we miss the real picture of how someone uses AI tools.

**Impact:** Individuals see an incomplete graph. The "how you use AI tools" value prop is hollow.

**Solutions:**
- **A. Proxy API decoders.** The network proxy already sees all HTTP(S) traffic from the dev environment. Build decoders for known AI API endpoints (OpenAI, Anthropic, Google, Ollama, Copilot). Extract prompt/completion from the request/response payloads. This works for any tool making API calls from within the environment.
- **B. VS Code extension (IDE-level, not browser).** Hook into `vscode.InlineCompletionItemProvider` to observe inline suggestions offered, accepted, and rejected. This captures the Copilot/Cursor tab-completion UX directly.
- **C. Claude Code integration.** Claude Code supports MCP and hooks. Investigate whether we can tap into its conversation stream programmatically.

**Recommendation:** Proxy decoders (A) give the broadest coverage with one implementation. VS Code extension (B) adds the accept/reject signal for inline completions that the proxy can't see. Both are needed.

---

**GAP 2: Inline completion accept/reject signal**

When an engineer sees a Copilot suggestion and presses Tab (accept) or Esc (reject), that's one of the most important signals in the entire dataset. We currently can't see it.

**Impact:** Can't calculate AI acceptance rate — a key metric for individual stats and a high-value signal for the training data corpus.

**Solution:** VS Code extension that observes the inline completion lifecycle. The `vscode.InlineCompletionItemProvider` API and the `InlineCompletionItem` events provide this.

**Recommendation:** Build the VS Code extension. This is also the right vehicle for editor-level events (file focus, navigation) that close other gaps.

---

**GAP 3: Copy/paste from web AI to editor**

Someone copies a function from ChatGPT and pastes it into their editor. We see both events but can't link them directly.

**Impact:** Can't attribute code to its AI source when the AI tool is in the browser.

**Solution:** Heuristic inference. If Extension records a copy event (with text length L from domain D) and FS Watcher records a file edit within T seconds where added content length is approximately L, create an INFERRED `PASTED_FROM` edge. Configurable thresholds: T=30s, L tolerance=20%.

**Recommendation:** Implement as a graph post-processing step. Low effort, good-enough accuracy.

---

### Tier 2 — Should solve (improves quality, not blocking)

**GAP 4: Editor navigation without edits.** Build into the VS Code extension — track active file, cursor position changes, go-to-definition events. Low priority but comes free with the extension.

**GAP 5: Interactive REPL sessions (psql, cypher-shell).** Terminal monitor could detect known REPL invocations and switch to a different capture mode that logs input lines. Moderate effort.

**GAP 6: Reading code in editor.** VS Code extension can report active editor and visible range. Passive data, useful for understanding reading-before-writing patterns.

### Tier 3 — Nice to have (defer)

**GAP 7:** Multi-monitor blind spots for non-browser apps.
**GAP 8:** Diagramming tool content.
**GAP 9:** Audio/voice interactions.
**GAP 10:** Paper notes / physical whiteboard.

---

## 7. Coverage Summary by Agent

| Agent | What It Covers Well | Key Gaps |
|-------|-------------------|----------|
| **MCP Server** | LLM calls routed through our server | Only captures AI usage that goes through our MCP — most real-world tools don't |
| **FS Watcher** | All file changes with diffs, language detection, git commits. **Community challenges only:** full git diffs per commit (encrypted). **All other contexts:** structural only. | Can't attribute changes to AI vs manual. No visibility into file reading. |
| **Terminal Monitor** | Commands, exit codes, test detection, truncated output | Interactive REPL sessions. Full output of verbose commands. |
| **Network Proxy** | All HTTP(S) traffic from the environment | Encrypted payloads need per-API decoders. High traffic volume requires filtering. |
| **Screen Capture** | Periodic visual record of the environment | Expensive to store. Expensive to process for semantic content. |
| **Attention Tracker** | Gaze, head pose, face presence | Requires webcam + consent. Accuracy varies. |
| **Extension** | Browser URLs, searches, tab switches, copy events, domain categorization | No page content. No AI chat content. Limited to browser. |
| **Telemetry Streamer** | Batching and transport | — (transport layer, not a capture gap) |

---

## 8. Planned Additions (Not Yet Built)

| Addition | Closes Gap | Priority | Effort |
|----------|-----------|----------|--------|
| **Proxy API decoders** (OpenAI, Anthropic, Google, Ollama, Copilot endpoints) | GAP 1: Non-MCP AI tools | P0 | Medium — parse known API formats from proxy traffic |
| **VS Code extension** (IDE-level, not browser) | GAP 2: Inline accept/reject. GAP 4: Editor navigation. GAP 6: Code reading. | P0 | Medium — new extension, hooks into VS Code APIs |
| **Copy-paste heuristic inference** | GAP 3: Web AI → editor attribution | P1 | Low — graph post-processing rule |
| **REPL session detection** | GAP 5: Interactive DB/shell sessions | P2 | Low — detect known REPL commands, switch capture mode |
