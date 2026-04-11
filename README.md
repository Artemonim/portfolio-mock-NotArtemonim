# **Проект: NotArtemonim — LLM-участник группового Telegram-чата**

[In English](./README_ENG.md)

## **Краткий обзор**

`NotArtemonim` — это асинхронная Python-система, в которой LLM-модель функционирует как полноправный *участник* группового Telegram-чата, а не как инструмент команд. Сообщения батчатся, после чего **closed-batch router** вызывает LLM, которая сама решает: участвовать ли в диалоге, ответить ли сейчас, как обновить долгосрочную память. Все решения фиксируются в PostgreSQL; производный слой «сжатой памяти» хранится в Markdown-файлах и пересобирается фоновым **librarian**. Для операторов предусмотрены: CLI, inline-review в личке, локальный WebUI и опциональный Telegram Mini App.

*Код проекта является закрытым. Название изменено.*

*Github Langbar симулирован при помощи [gh-lang-mock](https://github.com/Artemonim/gh-lang-mock)*

## **Технологический стек**

*   **Язык**: Python 3.13+
*   **Telegram framework**: aiogram 3.26.x (asyncio, long-polling)
*   **База данных**: PostgreSQL (asyncpg), schema v1+, миграции через Alembic
*   **ORM**: SQLAlchemy 2.0 (async)
*   **Admin WebUI**: aiohttp 3.13.x (REST API + статика; Telegram Mini App опционально)
*   **Конфигурация**: pydantic-settings 2.x (строгая типизация env-переменных)
*   **HTTP-клиент**: httpx (LM Studio Responses API, OpenRouter)
*   **MCP-интеграция**: mcp 1.2.x (Model Context Protocol — tool gateway для LLM)
*   **LLM-провайдеры**:
    *   LM Studio (локальный inference, on-demand model load, vision-capable модели)
    *   OpenRouter (облачный fallback; список моделей с приоритетом, предпочтения провайдеров)
*   **Инструменты качества кода**:
    *   Ruff (форматирование и линтинг)
    *   MyPy 1.20.x (строгая проверка типов)
    *   Bandit (SAST)
    *   pip-audit (проверка зависимостей на CVE)
*   **Тестирование**: pytest 9.x, pytest-cov (покрытие ≥75%); ~60 тестовых модулей
*   **CI/CD**: 3-уровневая локальная CI (`run.ps1` → `build.ps1` → `build.py`), совместима с Windows PowerShell 5.1+ и PowerShell 7+

## **Ключевые задачи и решения**

*   **«Участник», а не «бот»:** Задача — создать LLM-агента, который живёт в чате как собеседник: накапливает контекст без смешивания чатов, сам решает, когда вступить в разговор, и помнит договорённости между сессиями. Это принципиально отличается от типичных Telegram-ботов, отвечающих только на явные команды.
*   **Масштабируемый контекст при ограниченном контексте LLM:** Полная история переписки не помещается в окно LLM. Решение — двухслойная память: PostgreSQL хранит сырые данные, а sandboxed **MemoryManager** пересобирает компактные `summary.md` / `decisions.md` через **librarian**, не обращаясь к raw archive.
*   **Контроль качества решений LLM:** Router фиксирует в БД объяснения каждого решения (`router_explanation_json`), а операторы могут review пар «батч → ответ» прямо в Telegram (кнопки голос / Why? / отключить пару). Это создаёт замкнутый цикл feedback с удержанием audit trail.
*   **Prompt injection protection:** Система явно разделяет системную политику (доверенный контент) и untrusted-контент (память, батч, shortlist). Компонент `injection_guard` + `assembled_untrusted_provenance` отслеживает провенанс каждого фрагмента.
*   **Отказоустойчивость при деградации LLM:** При недоступности модели короткий fallback-текст отправляется только по парам, прошедшим policy «ответить». Бот не падает и не прекращает ingest.
*   **Продуктовые лимиты на исходящие:** `/mute` для product-admin подавляет исходящие видимые действия и модерацию, не останавливая ingest и батчинг, что позволяет «тихо» включать и настраивать бота без видимости в чате.

### **Архитектурные и технические особенности**

1.  **Closed-batch router:**
    *   Входящие сообщения буферизуются `BatchScheduler`, затем **RouterRuntime** вызывает LLM с собранным контекстом.
    *   Router принимает решение по нескольким осям: участвовать ли сейчас, что ответить, что сохранить в память (summary/decisions), какие tool-вызовы выполнить.
    *   Для групп поддерживаются два типа батчей: `group_listen_batch` (фоновое прослушивание) и `group_direct_addressed_batch` (явное обращение к боту).
    *   Router-pipeline реализован как цепочка стадий (`router_pipeline.py`): политика → shortlist → LLM-вызов → postprocess → persistence → tool execution.
    *   Политика `ROUTER_THINKING_POLICY=thinking_stripped` позволяет модели отдавать reasoning, который вырезается перед отправкой в Telegram.

2.  **Двухслойная память:**
    *   **PostgreSQL** — единственный источник истины: хранит пары (батч, ответ), `decision_cases`, `pair_signals`, `pair_feedback`, `chat_admin_access`.
    *   **Markdown-слой** (`memory/chats/<chat_id>/summary.md`, `decisions.md`) — производный, sandboxed: пересобирается librarian только из строк PostgreSQL, без чтения raw archive. Разрешённые пути записи контролируются `MemoryManager`.
    *   **Hot-state cache:** сводка `summary.md` кэшируется по grubому fingerprint (mtime+size), что исключает повторное чтение диска между батчами одного чата при неизменном файле; LRU-eviction и инвалидация при `/start`.
    *   **Periodic librarian:** фоновая задача компактификации `summary.md` / `decisions.md` из PostgreSQL с настраиваемыми лимитами по числу пар, decision cases и символам на файл.

3.  **MCP tool gateway:**
    *   Через `mcp_adapter.py` + `tool_gateway.py` LLM получает доступ к инструментам (knowledge base read/write, search, proposals) в рамках формального контракта (`tool_contracts.py`).
    *   **AgentScheduler** и **AgentCapabilities** управляют возможностями агента и расписанием выполнения действий.
    *   `post_tool_chat_guard.py` проверяет корректность состояния чата после выполнения tool-вызовов.

4.  **Knowledge Base с proposals:**
    *   Curated KB с allowlisted read/write (`knowledge_base.py`, `knowledge_base.py`).
    *   Workflow proposals (`knowledge_base_proposals.py`): LLM предлагает обновления KB, оператор — подтверждает.
    *   Librarian-passes (`librarian_passes.py`) и расписание (`librarian_schedule.py`) управляют циклами пересборки.

5.  **Vision и мультимодальные входы:**
    *   При `FEATURE_ENABLE_VISION_INPUTS=true` медиа-сообщения обрабатываются через `media_frame_sampling.py` и `vision_payload.py`, формируя мультимодальный prompt для vision-capable моделей.
    *   LLM-сервис (`llm_service.py`) поддерживает как текстовые, так и vision-маршруты через единый интерфейс.

6.  **Многоуровневая операторская поверхность:**
    *   **CLI-команды**: `check` (health), `librarian` (пересборка памяти), `feedback-stats`, `feedback-admin-vote`, `acl` (grant/revoke/list ролей).
    *   **Admin review в личке**: команды `/review` и `/reviewstats` после ingest; inline-кнопки для голоса (через `pair_signals`), кнопка **Why?** (снимок `router_explanation_json` без выдуманных score), отключение пары / keyword_rules, пагинация.
    *   **Локальный WebUI** (aiohttp): REST API + статика на `127.0.0.1:<port>` с token-аутентификацией; просмотр decision pairs, KB, proposals.
    *   **Telegram Mini App**: опциональный HTTPS-endpoint с валидацией `initData`.

7.  **Observability и отказоустойчивость:**
    *   `RuntimeSession` и `SubagentSessionLog` ведут лог каждой сессии с манифестом.
    *   `LLMJsonlUsage` сохраняет JSONL-лог всех LLM-вызовов с метриками (tokens, latency).
    *   `LlmConsoleTranscript` выводит human-readable транскрипт диалога в консоль.
    *   `RuntimeFailureNotify`: необработанные ошибки автоматически доставляются операторам в Telegram.
    *   `RuntimeErrorLog` сохраняет все runtime-ошибки с контекстом.
    *   `Metrics` + `Observability` для агрегированной статистики.
    *   `GlobalMuteService`: `/mute` подавляет исходящие без остановки ingest.

8.  **RBAC и ACL:**
    *   Продуктовые роли MVP: **product-admin** (глобальный доступ) и **chat-admin** (доступ только к своим чатам через `chat_admin_access`).
    *   `ACLService` проверяет права при feedback-голосовании, admin review, управлении KB.
    *   `DmGate` контролирует доступ к DM-интерфейсу.

9.  **Injection guard и безопасность:**
    *   `InjectionGuard` обнаруживает попытки prompt injection в user-контенте.
    *   `AssembledUntrustedProvenance` отслеживает провенанс каждого untrusted-фрагмента в промпте.
    *   Системная политика и untrusted-контент (память, батч, shortlist) физически разделены в prompt assembly.

10. **3-уровневая локальная CI:**
    *   `run.ps1` — тонкая обёртка с флагами (`-Fast`, `-SkipLaunch`) и help.
    *   `build.ps1` — оркестратор стадий, кэша (`.ci_cache/`), логов и итогового JSON-репорта.
    *   `build.py` — Python-логика стадий: `self-check`, `fmt`, `lint`, `line-limits`, `compile`, `test`, `coverage`, `security`.
    *   Профиль `-Fast` для быстрых итераций пропускает `coverage` и `security`; полный профиль — безопасный дефолт перед handoff.

## **Демонстрируемые навыки**

*   Проектирование LLM-агента с batch-роутером и детерминированным fallback-слоем
*   Асинхронный Python (asyncio, aiogram 3.x, aiohttp, asyncpg, SQLAlchemy 2.0 async, httpx)
*   Двухслойная персистентная память: PostgreSQL (source of truth) + sandboxed Markdown (production layer)
*   Интеграция MCP (Model Context Protocol) как tool-шлюза для LLM
*   Prompt assembly с явным разделением trusted / untrusted контента и injection guard
*   Мультимодальные LLM-пайплайны (vision inputs, multi-frame sampling)
*   Multi-provider LLM routing: LM Studio + OpenRouter с fallback-списком моделей
*   RBAC: product-admin / chat-admin с row-level ACL в PostgreSQL
*   Admin WebUI (aiohttp REST + статика + Telegram Mini App)
*   Inline-review в Telegram DM с audit trail и кнопкой объяснения решения
*   Observability: JSONL-логирование LLM, сессионные манифесты, runtime error notifications
*   Проектирование и тестирование DB schema (Alembic, schema v1 + migrations)
*   ~60 тестовых модулей: unit + интеграционные тесты всех подсистем
*   3-уровневая локальная CI с кэшем, инкрементальными проверками и JSON-репортом
*   Операторский CLI с полным набором команд и кодами завершения
