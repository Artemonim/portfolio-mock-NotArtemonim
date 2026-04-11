# **Project: NotArtemonim — LLM Participant in Group Telegram Chats**

[На русском](./README.md)

## **Overview**

`NotArtemonim` is an async Python system in which an LLM model operates as a full-fledged *participant* in a group Telegram chat — not a command-processing tool. Incoming messages are batched, then the **closed-batch router** invokes the LLM, which autonomously decides: whether to participate in the conversation, whether to reply now, and how to update long-term memory. All decisions are persisted in PostgreSQL; a derived "compressed memory" layer lives in Markdown files, rebuilt by a background **librarian**. Operators are provided with a CLI, inline review in DM, a local WebUI, and an optional Telegram Mini App.

*The project code is proprietary. The name has been changed.*

*Github Langbar simulated using [gh-lang-mock](https://github.com/Artemonim/gh-lang-mock)*

## **Technology Stack**

*   **Language**: Python 3.13+
*   **Telegram framework**: aiogram 3.26.x (asyncio, long-polling)
*   **Database**: PostgreSQL (asyncpg), schema v1+, migrations via Alembic
*   **ORM**: SQLAlchemy 2.0 (async)
*   **Admin WebUI**: aiohttp 3.13.x (REST API + static assets; Telegram Mini App optional)
*   **Configuration**: pydantic-settings 2.x (strict env-variable typing)
*   **HTTP client**: httpx (LM Studio Responses API, OpenRouter)
*   **MCP integration**: mcp 1.2.x (Model Context Protocol — tool gateway for LLM)
*   **LLM providers**:
    *   LM Studio (local inference, on-demand model load, vision-capable models)
    *   OpenRouter (cloud fallback; prioritized model list, provider preferences)
*   **Code Quality Tools**:
    *   Ruff (formatting and linting)
    *   MyPy 1.20.x (strict type checking)
    *   Bandit (SAST)
    *   pip-audit (dependency CVE scanning)
*   **Testing**: pytest 9.x, pytest-cov (coverage ≥75%); ~60 test modules
*   **CI/CD**: 3-level local CI (`run.ps1` → `build.ps1` → `build.py`), compatible with Windows PowerShell 5.1+ and PowerShell 7+

## **Key Challenges and Solutions**

*   **"Participant", not "bot":** The goal was to build an LLM agent that lives in a chat as a conversation peer: accumulating context without cross-chat contamination, autonomously deciding when to join a conversation, and retaining commitments between sessions. This is fundamentally different from typical Telegram bots that respond only to explicit commands.
*   **Scalable context within LLM context limits:** Full chat history does not fit in an LLM context window. The solution is a two-layer memory: PostgreSQL stores raw data, while the sandboxed **MemoryManager** rebuilds compact `summary.md` / `decisions.md` via the **librarian**, without accessing the raw archive.
*   **LLM decision quality control:** The router persists an explanation for every decision (`router_explanation_json`) in the database, and operators can review batch→response pairs directly in Telegram (vote / Why? / disable pair buttons). This creates a closed feedback loop with a full audit trail.
*   **Prompt injection protection:** The system explicitly separates system policy (trusted content) from untrusted content (memory, batch, shortlist). The `injection_guard` and `assembled_untrusted_provenance` components track the provenance of every fragment in the prompt.
*   **Graceful LLM degradation:** When the model is unavailable, a short fallback text is sent only for pairs that have already passed the "reply" policy. The bot does not crash and does not stop ingesting messages.
*   **Product-level outgoing limits:** The `/mute` command for product-admin suppresses outgoing visible actions and moderation without stopping ingest and batching, allowing "silent" onboarding and configuration of the bot without chat visibility.

### **Architectural and Technical Features**

1.  **Closed-batch router:**
    *   Incoming messages are buffered by `BatchScheduler`, then **RouterRuntime** invokes the LLM with the assembled context.
    *   The router makes decisions along multiple axes: whether to participate, what to respond, what to save to memory (summary/decisions), and which tool calls to execute.
    *   Two batch types are supported for groups: `group_listen_batch` (background listening) and `group_direct_addressed_batch` (direct bot address).
    *   The router pipeline is implemented as a stage chain (`router_pipeline.py`): policy → shortlist → LLM call → postprocess → persistence → tool execution.
    *   The `ROUTER_THINKING_POLICY=thinking_stripped` policy allows the model to emit reasoning tokens, which are stripped before delivery to Telegram.

2.  **Two-layer memory:**
    *   **PostgreSQL** is the single source of truth: stores pairs (batch, response), `decision_cases`, `pair_signals`, `pair_feedback`, `chat_admin_access`.
    *   **Markdown layer** (`memory/chats/<chat_id>/summary.md`, `decisions.md`) is derived and sandboxed: rebuilt by the librarian from PostgreSQL rows only, without reading the raw archive. Allowed write paths are enforced by `MemoryManager`.
    *   **Hot-state cache:** the `summary.md` digest is cached by a coarse fingerprint (mtime+size), eliminating repeated disk reads between batches of the same chat when the file is unchanged; LRU eviction and invalidation on `/start`.
    *   **Periodic librarian:** a background compaction task for `summary.md` / `decisions.md` from PostgreSQL with configurable limits on pair count, decision case count, and characters per file.

3.  **MCP tool gateway:**
    *   Via `mcp_adapter.py` + `tool_gateway.py`, the LLM gains access to tools (knowledge base read/write, search, proposals) within a formal contract (`tool_contracts.py`).
    *   **AgentScheduler** and **AgentCapabilities** manage agent capabilities and action execution scheduling.
    *   `post_tool_chat_guard.py` validates chat state consistency after tool calls.

4.  **Knowledge Base with proposals:**
    *   Curated KB with allowlisted read/write (`knowledge_base.py`).
    *   Proposals workflow (`knowledge_base_proposals.py`): the LLM proposes KB updates, the operator confirms them.
    *   Librarian passes (`librarian_passes.py`) and schedule (`librarian_schedule.py`) govern rebuild cycles.

5.  **Vision and multimodal inputs:**
    *   When `FEATURE_ENABLE_VISION_INPUTS=true`, media messages are processed through `media_frame_sampling.py` and `vision_payload.py`, assembling a multimodal prompt for vision-capable models.
    *   The LLM service (`llm_service.py`) supports both text-only and vision routes through a unified interface.

6.  **Multi-level operator surface:**
    *   **CLI commands**: `check` (health), `librarian` (memory rebuild), `feedback-stats`, `feedback-admin-vote`, `acl` (grant/revoke/list roles).
    *   **Admin review in DM**: `/review` and `/reviewstats` commands post-ingest; inline buttons for vote (via `pair_signals`), **Why?** button (snapshot of `router_explanation_json` without fabricated scores), disable pair / keyword_rules, pagination.
    *   **Local WebUI** (aiohttp): REST API + static assets at `127.0.0.1:<port>` with token authentication; decision pairs view, KB, proposals.
    *   **Telegram Mini App**: optional HTTPS endpoint with `initData` validation.

7.  **Observability and fault tolerance:**
    *   `RuntimeSession` and `SubagentSessionLog` maintain a per-session log with a manifest file.
    *   `LLMJsonlUsage` saves a JSONL log of all LLM calls with metrics (tokens, latency).
    *   `LlmConsoleTranscript` outputs a human-readable dialogue transcript to the console.
    *   `RuntimeFailureNotify`: unhandled errors are automatically delivered to operators in Telegram.
    *   `RuntimeErrorLog` persists all runtime errors with context.
    *   `Metrics` + `Observability` for aggregated statistics.
    *   `GlobalMuteService`: `/mute` suppresses outgoing messages without stopping ingest.

8.  **RBAC and ACL:**
    *   MVP product roles: **product-admin** (global access) and **chat-admin** (access limited to own chats via `chat_admin_access` row-level table).
    *   `ACLService` enforces permissions for feedback voting, admin review, and KB management.
    *   `DmGate` controls access to the DM interface.

9.  **Injection guard and security:**
    *   `InjectionGuard` detects prompt injection attempts in user content.
    *   `AssembledUntrustedProvenance` tracks the provenance of every untrusted fragment in the prompt.
    *   System policy and untrusted content (memory, batch, shortlist) are physically separated in prompt assembly.

10. **3-level local CI:**
    *   `run.ps1` — thin wrapper with flags (`-Fast`, `-SkipLaunch`) and help.
    *   `build.ps1` — stage orchestrator with cache (`.ci_cache/`), logs, and a final JSON report.
    *   `build.py` — Python stage logic: `self-check`, `fmt`, `lint`, `line-limits`, `compile`, `test`, `coverage`, `security`.
    *   `-Fast` profile skips `coverage` and `security` for rapid iteration; the full profile is the safe default before handoff.

## **Demonstrated Skills**

*   Designing an LLM agent with a batch router and deterministic fallback layer
*   Async Python (asyncio, aiogram 3.x, aiohttp, asyncpg, SQLAlchemy 2.0 async, httpx)
*   Two-layer persistent memory: PostgreSQL (source of truth) + sandboxed Markdown (production layer)
*   MCP (Model Context Protocol) integration as an LLM tool gateway
*   Prompt assembly with explicit trusted/untrusted separation and injection guard
*   Multimodal LLM pipelines (vision inputs, multi-frame sampling)
*   Multi-provider LLM routing: LM Studio + OpenRouter with fallback model list
*   RBAC: product-admin / chat-admin with row-level ACL in PostgreSQL
*   Admin WebUI (aiohttp REST + static assets + Telegram Mini App)
*   Inline review in Telegram DM with audit trail and decision explanation button
*   Observability: JSONL LLM logging, session manifests, runtime error notifications
*   DB schema design and testing (Alembic, schema v1 + migrations)
*   ~60 test modules: unit + integration tests for all subsystems
*   3-level local CI with caching, incremental checks, and a JSON report
*   Operator CLI with a full command set and exit codes
