# **Project: NotArtemonim — LLM Participant in Group Telegram Chats**

[На русском](./README.md)

## **Overview**

`NotArtemonim` is an async Python system in which a language model operates as a full-fledged *participant* in a group Telegram chat — not a command-processing tool. Incoming messages are batched, then the System calls the **Router** — an LLM agent that autonomously decides: whether to participate in the conversation, whether to reply now, which tools to invoke, and how to update long-term memory. Operational data is persisted in PostgreSQL; multi-layer compressed memory in Markdown files is rebuilt by the **Librarian** agent. Operators are provided with a CLI, inline review in DM, a local WebUI, and an optional Telegram Mini App.

*The project code is proprietary.*

*Github Langbar simulated using [gh-lang-mock](https://github.com/Artemonim/gh-lang-mock)*

## **Technology Stack**

*   **Language**: Python 3.13+
*   **Telegram framework**: aiogram 3.26.x (asyncio, long-polling)
*   **Database**: PostgreSQL (asyncpg), schema v1+, migrations via Alembic
*   **ORM**: SQLAlchemy 2.0 (async)
*   **Admin WebUI**: aiohttp 3.13.x (REST API + static assets; Telegram Mini App deferred)
*   **Configuration**: pydantic-settings 2.x (strict env-variable typing)
*   **HTTP client**: httpx (LM Studio Responses API, OpenRouter)
*   **MCP integration**: mcp 1.2.x (Model Context Protocol — tool gateway for LLM)
*   **LLM providers**:
    *   LM Studio (local inference, on-demand model load, multimodal-capable models)
    *   OpenRouter (cloud inference; prioritized model list, provider preferences)
*   **Code Quality Tools**:
    *   Ruff (formatting and linting)
    *   MyPy 1.20.x (strict type checking)
    *   Bandit (SAST)
    *   pip-audit (dependency CVE scanning)
    *   [Agent Enforcer 2](https://github.com/Artemonim/AgentEnforcer2) (reference architecture for the three-tier local CI)
*   **Testing**: pytest 9.x, pytest-cov (coverage ≥75%); ~60 test modules
*   **CI/CD**: 3-level local CI (`run.ps1` → `build.ps1` → `build.py`), compatible with Windows PowerShell 5.1+ and PowerShell 7+

## **Key Challenges and Solutions**

*   **"Participant", not "bot":** The goal was to build an LLM agent that lives in a chat as a conversation peer: accumulating context without cross-chat contamination, autonomously deciding when to join a conversation, and retaining commitments between sessions. This is fundamentally different from typical Telegram bots that respond only to explicit commands.
*   **Scalable context within LLM context limits:** Full chat history does not fit in an LLM context window. The solution is a multi-layer OpenClaw-like memory: an active context window (constrained by message age, count, and batch size) assembled per batch; a Router scratch accumulating working notes per chat; the Librarian rebuilds compressed summaries and recorded decisions from PostgreSQL; global memory holds cross-chat facts and is populated only through explicit operator-approved promotion. The knowledge base serves as the source of truth for domain knowledge. PostgreSQL is the technical operational store.
*   **Router decision quality control:** The system persists an explanation for every decision, and operators can review batch→response pairs directly in Telegram (vote, "Why?", disable pair buttons). This creates a closed feedback loop with a full audit trail.
*   **Prompt injection protection:** The system explicitly separates trusted and untrusted content in the prompt. The provenance of every fragment is tracked; attempts to inject instructions through user messages are detected before the prompt reaches the model.

### **Architectural and Technical Features**

1.  **Closed-batch Router:**
    *   Incoming messages are buffered by the Batch Scheduler, then the System calls the Router — an LLM agent that makes decisions based on the assembled context.
    *   The Router itself decides: whether to participate, what to respond, what to save to memory, and which tools to call.
    *   Two group modes are supported: background listening and direct bot address — both pass through the same decision pipeline.
    *   The Router is implemented as a stage chain: policy → shortlist → LLM call → post-processing → persistence → tool execution.
    *   A reasoning-filter mode is available: only the final response is delivered to Telegram, stripping the model's internal reasoning block.
    *   The architecture is designed for agent roster expansion: in addition to the active Router and Librarian, Scout, Officer, and Triage agents are planned.

2.  **Multi-layer OpenClaw-like memory:**
    *   The memory architecture follows the OpenClaw principle: Markdown files as auditable, versioned artifacts with strict per-chat / global separation, rather than implicit history in the LLM context.
    *   **Active context window** — assembled dynamically for each batch: constrained by message age, count, and batch size. This is the Router's "working" memory for the current decision.
    *   **Router scratch** — per-chat: the Router accumulates working notes through tools; the Librarian deduplicates and compacts the scratch, extracting marked fragments into the recorded-decisions section.
    *   **Compressed per-chat memory** (summary + recorded decisions) — derived and sandboxed: the Librarian rebuilds it from PostgreSQL rows without accessing the raw archive. Allowed write paths are enforced by the Memory Manager.
    *   **Global memory** — cross-chat facts; enters here only through explicit operator-approved promotion from per-chat.
    *   **Knowledge base** — curated source of truth for domain knowledge: the System draws on it alongside chat memory when forming responses.
    *   **PostgreSQL** — technical operational store: batch→response pairs, decision cases, operator votes, chat settings.
    *   **Hot-state cache:** the Markdown summary is cached by a file fingerprint, eliminating repeated disk reads between batches of the same chat when the file is unchanged.
    *   **Periodic Librarian:** a background compaction task for memory from PostgreSQL with configurable limits on pair count, decision case count, and text volume.

3.  **MCP tool gateway:**
    *   Via the MCP adapter, the language model gains access to tools (knowledge base read/write, search, update proposals) within a formal contract.
    *   The Agent Scheduler and Agent Capabilities manage the available tool set and action execution scheduling.
    *   After each tool call, chat state consistency is validated.

4.  **Knowledge Base with proposals:**
    *   The curated knowledge base is the source of truth for domain knowledge: the System draws verified facts from it when forming responses. Access is controlled by an allowlisted set of paths.
    *   Proposals workflow: the System proposes knowledge base updates through tools, the operator confirms them.
    *   Librarian passes and schedule govern rebuild cycles.

5.  **Multimodal inputs:**
    *   When multimodal input support is enabled, media messages are processed through key-frame sampling and a multimodal prompt is assembled for supporting models.
    *   The LLM service supports both text-only and multimodal routes through a unified interface.
    *   Voice, audio, video, and video-note messages are transcribed via AskVLM — a proprietary Whisper-based transcription tool (closed source).

6.  **Multi-level operator surface:**
    *   **CLI commands**: dependency health check, memory rebuild, aggregated vote statistics, operator vote recording, role management (grant/revoke/list).
    *   **DM review**: `/review` and `/reviewstats` commands post-ingest; inline buttons for voting, viewing the decision explanation, disabling a pair, pagination.
    *   **Local WebUI** (aiohttp): REST API + static assets with token authentication; decision pairs view, knowledge base, proposals.
    *   **Telegram Mini App**: optional HTTPS endpoint with init-data validation.

7.  **Observability and fault tolerance:**
    *   Each session maintains a log with a manifest and action records.
    *   A JSONL log of all LLM calls is kept with metrics (tokens, latency).
    *   A human-readable dialogue transcript is printed to the console.
    *   Unhandled errors are automatically delivered to operators in Telegram.
    *   All runtime errors are persisted with context.
    *   Aggregated statistics and metrics are collected by a dedicated module.
    *   The `/mute` command (product-admin) deactivates the Router: the System stops processing batches without wasting resources, but continues downloading and storing messages in the database.

8.  **RBAC and ACL:**
    *   MVP product roles: **product-admin** (global access) and **chat-admin** (access limited to assigned chats via row-level table).
    *   The ACL service enforces permissions for voting, decision review, and knowledge base management.
    *   A dedicated gateway controls access to the DM interface.

9.  **Injection guard and security:**
    *   The system detects instruction-injection attempts in user content.
    *   The provenance of every untrusted fragment in the prompt is tracked by a dedicated component.
    *   Trusted system-policy content and untrusted user content are physically separated during prompt assembly.

10. **3-level local CI:**
    *   The CI is built following the philosophy of [Agent Enforcer 2](https://github.com/Artemonim/AgentEnforcer2) — a reference architecture for robust, language-agnostic local CI systems.
    *   `run.ps1` — thin wrapper with flags (`-Fast`, `-SkipLaunch`) and help.
    *   `build.ps1` — stage orchestrator with cache (`.ci_cache/`), logs, and a final JSON report.
    *   `build.py` — Python stage logic: `self-check`, `fmt`, `lint`, `line-limits`, `compile`, `test`, `coverage`, `security`.
    *   `-Fast` profile skips `coverage` and `security` for rapid iteration; the full profile is the safe default before handoff.

## **Demonstrated Skills**

*   Designing a multi-agent LLM system (Router, Librarian; extensible agent roster)
*   Async `Python` (`asyncio`, `aiogram 3.x`, `aiohttp`, `asyncpg`, `SQLAlchemy 2.0 async`, `httpx`)
*   OpenClaw-like multi-layer memory: active window + Router scratch + compressed per-chat memory + global memory + knowledge base + `PostgreSQL`
*   `MCP` (Model Context Protocol) integration as an LLM tool gateway
*   Prompt assembly with explicit trusted/untrusted separation and injection guard
*   Multimodal LLM pipelines (image processing, key-frame sampling, transcription tool integration)
*   Multi-provider LLM routing: `LM Studio` + `OpenRouter` with a fallback model list
*   `RBAC`: product-admin / chat-admin with row-level `ACL` in `PostgreSQL`
*   Admin WebUI (`aiohttp` REST + static assets + Telegram Mini App)
*   Inline decision review in Telegram DM with audit trail and explanation button
*   Observability: JSONL LLM logging, session manifests, automated error notifications
*   DB schema design and testing (`Alembic`, schema v1 + migrations)
*   ~60 test modules: unit + integration tests for all subsystems
*   3-level local CI following [Agent Enforcer 2](https://github.com/Artemonim/AgentEnforcer2) with caching, incremental checks, and a JSON report
*   Operator CLI with a full command set and exit codes
