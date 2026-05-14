# **Project: NotArtemonim — LLM-driven Telegram group participant**

 [🇷🇺 Русский](README.md) | [🇬🇧 English](README.en.md)

## **Overview**

`NotArtemonim` is an asynchronous Python system in which a language model acts as a full participant and moderator of a group Telegram chat. The bot actively operates in 20+ groups, offloading work from human administrators: it accumulates separate context per chat, makes decisions in batches, replies, stays silent, or applies a moderation action — and gives the operator clear control levers.

The project prioritizes product/runtime maturity: local and cloud inference, multi-layer memory, decision explainability, a feedback loop, operator UX in both Telegram and the browser, strict local CI, and a convenient process lifecycle. The public description intentionally omits internal prompts, exact defensive heuristics, and low-level details that could facilitate abuse, exploitation, or reverse engineering.

![](/media/Router%20Reasoning.png)

*The project's source code is closed.*

*The Github Langbar is simulated using [gh-lang-mock](https://github.com/Artemonim/gh-lang-mock).*

## **Tech Stack**

*   **Language**: Python 3.13+
*   **Telegram runtime**: aiogram 3.26.x, asyncio, long polling
*   **Web layer**: aiohttp 3.13.x (local WebUI)
*   **Data**: PostgreSQL, asyncpg, Alembic, async SQLAlchemy 2.0
*   **Configuration**: pydantic 2.12.x + pydantic-settings 2.13.x
*   **HTTP & LLM integrations**: httpx, LM Studio, OpenRouter
*   **Tool gateway**: MCP (Model Context Protocol)
*   **Code quality**: Ruff, MyPy strict, Bandit, pip-audit
*   **Testing**: pytest 9.x, pytest-cov, 1200+ test cases with 75+% coverage
*   **Local CI**: three-stage pipeline powered by [Agent Enforcer 2](https://github.com/Artemonim/AgentEnforcer2)

## **What Is Already Implemented**

### **1. Chat Behavior**

*   **LLM as a conversation participant:** the system can listen to ordinary group conversation; an explicit mention of the bot only raises priority, but is not the sole trigger.
*   **Closed-batch routing:** incoming messages are first collected into batches and only then processed by the Router. This produces more natural participation in conversations and reduces impulsive replies.
*   **"Reply or stay silent" policy:** the model is not required to respond every time. The system can choose silence, a reply, indirect reactions, sub-agent invocations, and other scenarios.
*   **Per-chat context isolation:** memory and state are not mixed across different groups.
*   **Built-in moderation:** the system tracks violations, applies a warning ladder, and executes Telegram actions — warn, mute, ban, delete — based on the Router's decision. Every action is logged and remains transparent to operators.
*   **Multimodality:** the bot reads the chat and views GIFs and images; it listens to voice and video messages via [AskVLM]().

### **2. Memory, Knowledge, and Context**

*   **OpenClaw-like multi-layer memory:** active context window, working notes, compressed per-chat memory, global knowledge, and a curated knowledge base operate as distinct layers with distinct roles.
*   **PostgreSQL as the operational source of truth:** batch history, decisions, quality signals, chat settings, and the review loop live in the database, while compressed memory is rebuilt from this data.
*   **Periodic memory compaction:** a dedicated loop condenses context and keeps it useful for subsequent cycles.
*   **Curated knowledge base:** responses are grounded in a curated knowledge base with operator-level control.
*   **Knowledge base update suggestions:** based on chat analysis, the Librarian sub-agent can propose additions to persistent memory.
*   **Semantic hints:** admins and users can rate each chatbot response, and the system builds from this a hints mechanism with positive and negative examples for each individual reply.

### **3. Operator UX**

*   **Privileged DM channel:** the primary operator surface — management commands and free-form dialogue with global product context are available in DMs.
*   **Chat registry / onboarding wizard:** the operator loop includes a step-by-step chat setup flow, with contextual parameters such as timezone, so that responses better match the actual chat environment.
*   **CLI for operator and developer:** health checks, memory rebuilds, feedback statistics, access management, and other operational tasks are available without a GUI.
*   **Global and per-chat mute:** operator kill switches for outgoing behavior that do not break ingest or audit.
*   **Reminders and background service scenarios:** the project can not only react to messages but also support auxiliary workflows around chats.
*   **Runtime config updates without full restart:** configuration and knowledge base changes can be applied without restarting the process.
*   **Graceful shutdown/drain:** the process can terminate cleanly without cutting off active scenarios at an arbitrary point.

### **4. Reliability, Observability, and Security**

*   **Ingest/inference separation:** the incoming Telegram stream is not dependent on LLM call duration.
*   **Separate queues and throttling for outgoing actions:** rate control for outgoing messages and handling of Telegram limits are built into the runtime.
*   **Structured observability:** correlation between update, batch, decision case, and reply, plus dedicated runtime artifacts, logs, and diagnostic traces.
*   **LLM JSONL logging and session artifacts:** convenient for post-mortems, regressions, and manual diagnostics.
*   **Automatic delivery of high-signal errors to the operator:** important failures do not stay in the console alone.
*   **Feedback loop via reactions and votes:** response quality is assessed not only manually but also through passive product signals.
*   **RBAC / ACL:** a minimal product role matrix is already in place, separating the global operator from per-chat administrators.
*   **Explicit separation of trusted and untrusted content:** the system is deliberately designed so that user input, tool output, and external content cannot become privileged instructions.
*   **Validation checks before side effects:** actions go through a controlled loop of validation, logging, and policy enforcement.
*   **Local-first admin surface:** sensitive operator scenarios are oriented toward the local loop and controlled access.

### **5. Engineering Maturity**

*   **Multi-provider LLM routing:** the same product scenario can run on local inference via LM Studio or on a cloud route via OpenRouter.
*   **100+ test files:** coverage spans core/runtime, router, schema, telegram flow, knowledge base, webapp, operator UX, and auxiliary services.
*   **Strict typing and local quality gate:** mypy strict, ruff, bandit, pip-audit, and coverage monitoring are embedded in the everyday development cycle.
*   **3-stage local CI:** a fully featured local pipeline with reports, caching, and launch profiles.
*   **DB migrations and schema evolution:** 15+ Alembic migrations; the schema actively evolves alongside the product.

## **What Is Planned**

*   **Extended moderation UX:** human-in-the-loop approval for complex cases, a detailed audit trail, and operator override scenarios.
*   **Broader agentic loop:** the current runtime works safely with tools, but a multi-step agent loop with the full tool set remains on the roadmap.
*   **Additional specialized agents:** the architecture is designed for extension; some roles are still in planning.
*   **Operator analytics and service scenarios:** richer timeline, restart UX, bulk operations, and additional diagnostics.
*   **Further strengthening of the security/eval layer:** extensions in anti-abuse, red-team, and security automation.
*   **Userbot for warm lead handling (in development):** a separate subsystem based on the Telegram userbot API (Telethon) that will be able to import a lead list, reach out to them, answer questions from the knowledge base, and softly transition them into a managed chat.
*   **Telegram Mini App:** an optional mobile operator surface as a complement to the DM channel.

## **Skills Demonstrated**

*   Designing an LLM-driven group chat participant rather than a classic command bot
*   Async `Python` (`asyncio`, `aiogram`, `aiohttp`, `asyncpg`, `SQLAlchemy 2.0`, `httpx`)
*   OpenClaw-like multi-layer memory and rebuildable memory artifacts
*   Multi-provider LLM orchestration: `LM Studio` + `OpenRouter`
*   Multimodal and media-aware LLM pipelines
*   `MCP` integration as a safe tool layer for LLMs
*   Operator surfaces: CLI, Telegram DM review, local WebUI
*   Automated Telegram group moderation: warn, mute, ban, delete with a warning ladder and audit trail
*   RBAC / ACL and admin workflows for a Telegram product
*   Feedback loop, explainability, and operator audit of decisions
*   Observability, graceful lifecycle, hot reload of partial runtime config
*   Database schema design and evolution via `Alembic`
*   Local 3-stage CI following the [Agent Enforcer 2](https://github.com/Artemonim/AgentEnforcer2) philosophy
*   Strict typing, testing, and security checks in the daily development cycle
