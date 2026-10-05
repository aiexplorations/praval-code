# praval-code References and Verification Log

Status: draft 0.1. Part of the [specification](SPEC.md).

Praval facts below were checked on 2026-10-05 against `praval 0.8.3` installed in the specification's original working venv (Python 3.14.6), by reading signatures and source. "Source" means the behaviour was read in the package code. "Docs" means it is stated in Praval's own documentation. "Run" means it was executed end to end. As of this draft, nothing is at "Run" status, because no provider key or local model server has been used yet. The spikes in section 3 move items to "Run" before any feature depends on them.

## 1. Notes that informed the design

The design incorporates three kinds of input: the CLI practices pasted into the request (POSIX flags, stdout/stderr split, JSON output, `NO_COLOR`, exit codes, bypass flags, signal handling), the agentic CLI architecture notes (interface, agent loop, tools; dry-run, checkpoints, context curation, error feedback), and the notes on building with Praval. Several statements in the last group were checked rather than taken as given:

| Statement in the notes | Finding |
|---|---|
| A single agent on the reef, with the UI broadcasting spores | Not adopted. `praval-code` uses `praval.Agent` directly (`astream`) with no reef for the main agent. The reef is used only for subagents. |
| `requires_approval=True` on `@tool` gives an approve/reject/edit boundary | Present in the `tool` signature and in `ToolSpec`; `InterventionDecision` has `APPROVE`, `EDIT`, `REJECT`. Round trip not yet run (S2). |
| `praval.mcp` connects to stdio MCP servers | Confirmed and adopted (SPEC.md, section 5.9): stdio and Streamable HTTP, tools only. Facts in section 2. |
| `show_recent_traces()` prints a timeline | Exported from `praval.observability`. |
| An evaluation layer for offline suites | `praval eval run SUITE` exists with `--config --db --run-id --module --case-id --tag --limit --seed --json`. Suite format not yet read (S8). |
| Local execution "cleanly handled" | Local providers exist, but their presets disable tools (section 2). Local tool use needs configuration and a probe. |
| Git snapshots for rollback | Not adopted: git is not guaranteed on a stock machine. `praval-code` keeps its own journal. |
| Use `ripgrep` and AST parsers for context curation | `rg` is used if present; searching is implemented in Python otherwise. AST parsing is out of scope for v1. |
| Textual or Rich for the terminal UI | Textual, through the Mistral Vibe fork (goal 8). It runs on asyncio, which the agent loop needs for MCP tools. |
| OpenCode as a fork base or reference (user notes, 2026-10-05) | The notes describe OpenCode as written in Go with auto-compaction at 95% of the context window. That describes `opencode-ai/opencode`, which was archived on 2025-09-18 and continues as Charm's Crush. The active OpenCode (`anomalyco/opencode`, formerly `sst/opencode`) is TypeScript on Bun, MIT, with an OpenTUI and Solid TUI that talks to a local server. Used as a design reference for features only. |
| Python agent CLIs as a fork base (2026-10-05) | Checked on GitHub: Mistral Vibe (Apache 2.0, Textual, MCP, ACP, active) chosen. OpenHands CLI (MIT, Textual) is tied to the OpenHands SDK throughout. Toad (Textual) is AGPL-3.0. Aider is a line-based prompt, not a full-screen TUI. Kimi CLI is archived. Open Interpreter is now Rust. |
| Porting Praval to Go to sit under a Go TUI | Rejected: about 12,000 to 15,000 lines of Praval would need porting and keeping in step with the Python original. |
| Fork parts of Claude Code | Not possible: the npm package's `LICENSE.md` reads "© Anthropic PBC. All rights reserved." Used as a UX reference only. |

## 2. Praval facts used by the spec

| Fact | Evidence | Status |
|---|---|---|
| Version 0.8.3; Requires-Python `>=3.10,<3.15` | `importlib.metadata` | Source |
| Core dependencies: `openai`, `anthropic`, `cohere`, `pydantic`, `pydantic-settings`, `python-dotenv`, `opentelemetry-api` (plus `tomli` below 3.11) | `importlib.metadata` | Source |
| `Agent(name, provider, model, persist_state, system_message, config, memory_enabled, memory_config, knowledge_base, max_history, hitl_enabled, hitl_db_path)` | `inspect.signature` | Source |
| `Agent.chat(message) -> str`; `Agent.stream(message, **kw)` yields `ModelEvent`; `Agent.generate(message, **kw)` returns a richer response | signature and docstrings | Source |
| `ModelEvent.type` values emitted: `delta`, `tool_call`, `tool_result`, `usage`, and provider-level `tool_call_delta` | `model_runtime.py` lines 775-784, `providers/openai.py` lines 907-973 | Source |
| With tools registered, `ModelRuntime.stream()` and `astream()` run the whole tool loop through `_invoke_with_retries` and `_orchestrate_tool_calls`, then yields `tool_call`, `tool_result`, `delta` and `usage` from the completed response. Events are not live, and only the final round's text is emitted | `model_runtime.py` lines 566-574, 631-647, 761-785, 1172-1239 | Source |
| `ReasoningConfig(effort, summary, encrypted, budget_tokens, mode, display)` is accepted per request and mapped to Anthropic `thinking`, OpenAI `reasoning` (effort, summary) and Gemini `thinkingConfig` | `models/__init__.py` lines 243-251; `providers/anthropic.py` lines 436-447; `providers/openai.py` lines 775-780; `providers/gemini.py` lines 245-249 | Source |
| Reasoning text is not surfaced: `ModelEvent` has no reasoning type, the Anthropic stream reads only `delta.text`, the OpenAI Responses stream has no reasoning-summary branch. Anthropic non-streamed content blocks are kept whole via `model_dump`, so thinking blocks may survive in `response.messages` | `models/__init__.py` lines 358-369; `providers/anthropic.py` lines 330-334, 397-402; `providers/openai.py` lines 934-944 | Source |
| Local profiles declare no `reasoning` capability, so a `ReasoningConfig` on them raises `ProviderError("... does not support reasoning config")`. The chat-completions path ignores `ReasoningConfig` (only the Responses path maps it) and copies unreserved `provider_options` keys into the call, so `provider_options={"reasoning_effort": ...}` reaches a local server | `providers/registry.py` lines 257-262; `model_runtime.py` lines 906-909; `providers/openai.py` lines 751, 775-780, 794-811 | Source |
| No provider parses a `reasoning` or `reasoning_content` field from chat-completions responses or streams | grep over `providers/` | Source (absence) |
| Sibling projects on Praval 0.8.3 agree. `praval_research` treats any `tool_call` event in a stream as an error and streams only tool-free calls (`agent_streaming.py` lines 111-115). `vinn` (`agents/gateway.py` line 1174) and `praval_swarm` (`proof_swarm/factory.py` line 673) turn local reasoning off with `provider_options={"reasoning_effort": "none"}`, not `ReasoningConfig`. `vinn` notes reasoning tokens count toward usage but are never seen (`settings.py` lines 45-49). None of the three displays model reasoning | source of the three repositories | Source |
| `praval.mcp` (extra `praval[mcp]`, `mcp>=1.27,<2`) is not re-exported from `praval`. `MCPServerConfig(name, transport: stdio|streamable_http, command, args, env, cwd, url, headers, connection_timeout=30, tool_timeout=30, tool_name_prefix, require_approval=True, max_result_size=1 MiB)`. HTTP URLs must be https, or http on loopback | `mcp/__init__.py`; `mcp/client.py` lines 57-142 | Source |
| `MCPClient` is async: `connect`, `close`, `list_tools() -> list[ToolSpec]`, `register_tools(agent)`, `call_tool(name, args) -> ToolResult`. Tool names default to `server__tool`. Risk level comes from server annotations: `destructiveHint` gives high, `readOnlyHint` gives low, otherwise medium | `mcp/client.py` lines 145-589 | Source |
| `register_tools` registers each MCP tool with `add_tool_spec(..., async_only=True)`; sync tool execution raises "This tool is async-only; use Agent.agenerate() or Agent.astream()" | `mcp/client.py` line 306; `model_runtime.py` lines 244-247, 376-379 | Source |
| MCP scope in the 0.8 line: tools only; no resources, prompts, server hosting, managed OAuth, legacy SSE, sampling, elicitation, binary or image results, automatic reconnect, or sync bridging | Praval `docs/releases/RELEASE_NOTES_0.8.1.md` lines 93-103 | Docs |
| HITL has async paths: `execute_or_interrupt_async`, `execute_with_decision_async`, `Agent.aresume_run` | `hitl/runtime.py` lines 108, 226; `core/agent.py` line 749 | Source |
| On the async path, a provider without a native `ainvoke` runs in a thread executor, but tool functions are called directly on the event loop (`tool_func(**args)`, awaited only if awaitable), so a blocking sync tool blocks the loop | `model_runtime.py` lines 1150-1170; `hitl/runtime.py` lines 363-375 | Source |
| A stdio MCP server's `env` is passed to the SDK as given, or `None` when unset | `mcp/client.py` line 454 | Source |
| Usage tracks `reasoning_tokens` | `runtime_observation.py` line 150 | Source |
| `max_tool_rounds` defaults to 8 and accepts 1 to 1000 | `config.py`, `ModelRequest` annotation | Source |
| `tool(...)` accepts `requires_approval`, `risk_level`, `approval_reason`, `description`, `category`, `tags` | signature | Source |
| `ToolSpec` carries the same approval fields; `ToolResult(tool_call_id, name, content, is_error)` | signatures | Source |
| HITL: `Agent.configure_hitl`, `get_pending_interventions`, `approve_intervention(id, reviewer, edited_args)`, `reject_intervention`, `resume_run(run_id)`; `InterventionRequired(intervention_id, run_id, agent_name, tool_name, reason)` is raised from the model call and caught inside the runtime | signatures; `model_runtime.py`, `core/agent.py` references | Source |
| HITL store default `~/.praval/hitl.db`; `hitl_db_path` overrides | `hitl/store.py`, signature | Source |
| `praval hitl pending|show|approve|reject|resume` commands exist | `praval.cli --help` | Source |
| Provider capability gating raises `ProviderError("... does not support tools")` when tools are requested and `capabilities.tools` is false | `model_runtime.py` line 936 | Source |
| Image input is rejected unless `capabilities.multimodal and capabilities.image_input` | `model_runtime.py` line 1012 | Source |
| OpenAI provider declares `tools`, `streaming`, `multimodal`, `image_input`, `structured_outputs`, `audio_transcription`, `speech_generation` | `providers/openai.py` line 61 | Source |
| `ContentPart(type, text, data, url, mime_type)`; OpenAI provider handles types `image_url`, `image_base64`, `image` | `providers/openai.py` lines 852-872 | Source |
| `openai-compatible` provider; base-URL presets for `ollama` (`:11434`), `vllm` (`:8000`), `lmstudio` (`:1234`), `llama-cpp` (`:8080`); rejects credentials in URLs and metadata hosts | `providers/openai_compatible.py` | Source |
| Local profiles use `local_preset`, endpoint `chat.completions`, `downgrade_policy="error"`, and are described as conservative: "Enable tools/schema manually if server supports them" | `providers/registry.py` lines 400-440 | Source |
| `provider_options.capabilities` (a dict) overrides a provider's capabilities per request | `model_runtime.py` lines 880-891 | Source |
| `Reef(default_max_workers=4, backend=None, ...)`; default backend `InMemoryBackend`; `request_and_wait`, `reply`, `broadcast`, `create_channel`, `shutdown`, `wait_for_completion` exist | `core/reef.py` | Source |
| `Agent.subscribe_to_channel`, `set_spore_handler`, `send_knowledge`, `broadcast_knowledge`, `close()` (idempotent) | signatures, docstring | Source |
| `Spore` fields include `knowledge`, `correlation_id`, `reply_to`, `run_id`, `idempotency_key` | signature | Source |
| Observability: `PRAVAL_OBSERVABILITY` (`auto`/`on`/`off`), `PRAVAL_TRACES_PATH` (default `~/.praval/traces.db`), `PRAVAL_CAPTURE_CONTENT`, `PRAVAL_SAMPLE_RATE`, `PRAVAL_OTLP_ENDPOINT`; `show_recent_traces`, `print_traces`, `configure_observability` exported | `config.py`, `praval.observability` | Source |
| `PRAVAL_DEFAULT_PROVIDER` and `PRAVAL_DEFAULT_MODEL` are read | `config.py` lines 564-565 | Source |
| No image generation call: searches for `image_generation`, `generate_image` and `images.generate` found nothing; `Agent.speak` and `Agent.transcribe` exist | grep over the package; `dir(Agent)` | Source (absence of those terms) |
| `praval eval run SUITE [--config --db --run-id --module --case-id --tag --limit --seed --json]` | `praval.cli eval run --help` | Source |

### 2.1 Mistral Vibe facts

Checked on 2026-10-05 against a shallow clone of `mistralai/mistral-vibe` at commit `7c19608` (2026-09-23), package version 2.25.8.

| Fact | Evidence | Status |
|---|---|---|
| Apache 2.0 licence; no `NOTICE` file in the repository | `LICENSE`; directory listing | Source |
| `requires-python = ">=3.12"`; 99 pinned runtime dependencies, including `textual 8.2.8`, `mcp 1.28.1`, `agent-client-protocol 0.11.0`, `mistralai`, `sentry-sdk` | `pyproject.toml` | Source |
| Entry points: `vibe` (`vibe.cli.launcher:main`), `vibe-acp`, `vibe-app-server` (stdio) | `pyproject.toml` `[project.scripts]` | Source |
| About 139,000 lines of Python: UI 25,000 (`vibe/cli/textual_ui`), app server 50,000 (`vibe/app_server`), core 63,000 (`vibe/core`) | `wc -l` over the clone | Source |
| The UI imports nothing from `vibe.core`; it imports the protocol, models and client from `vibe.app_server` | import scan of `vibe/cli/textual_ui` | Source |
| The app-server protocol names 109 methods (sessions, turns, config, MCP, review, rewind, skills, plugins, connectors, worktrees, telemetry, and more) | method strings in `vibe/app_server/protocol.py` | Source |
| In-process use connects the UI to the server through `memory_transport_pair` (`LocalHarness`) | `vibe/app_server/local.py` | Source |
| The UI's session API (`AppServerSession`) exposes about 30 operations: start, resume, act, events, turn queue, interrupt, callback responses, compact, clear history, close | `vibe/app_server/session.py` | Source |
| Telemetry is sent to Mistral-controlled endpoints when a Mistral provider key is configured; the Sentry DSN is `None` in the public source | `vibe/core/telemetry/send.py` lines 50-60; `vibe/observability/sentry.py` lines 15-16 | Source |
| Config is read from `~/.vibe/config.toml` (`VIBE_HOME`) and project `.vibe/` and `.agents/` directories | `vibe/core/paths/` | Source |

## 3. Spikes

Each spike is a short script, run in the venv, with the result recorded in this file. A spike that fails changes the spec before dependent code is written. Spikes marked "needs key" require a provider API key; S3 needs a local server.

| ID | Question | Pass condition | If it fails | Needs |
|---|---|---|---|---|
| S1 | Does `Agent.astream()` with registered tools emit `tool_call` and `tool_result` events in order, and how is `max_tool_rounds` passed per request? Which environment variable names does each provider read for its key? | A two-tool task streams `delta`, `tool_call`, `tool_result`, `delta`, `usage`; round limit is honoured | Use `agenerate()` and derive activity from tool wrappers | key |
| S2 | HITL round trip in-process on the async path: a tool with `requires_approval=True`, including an async-only MCP tool, raises `InterventionRequired` out of `astream`; the pending intervention exposes the call's arguments, including `intent`; `approve_intervention` plus `aresume_run` completes; `edited_args` is honoured; the SQLite file lands at `hitl_db_path`. Can an `allow` decision be auto-approved cheaply, or must `run_shell` be split (API.md, section 3)? | Run completes after approval; rejection reaches the model as a result | Implement approval in `praval-code` tool wrappers and do not use Praval HITL | key |
| S3 | Local tool calling: with `provider="ollama"` and `provider_options.capabilities={"tools": true}`, does a named local model complete a tool call round trip? | Tool result reaches the final answer | Document that local mode is chat-only for that model; keep `doctor --probe` | local server (Ollama installed by the user) |
| S4 | Image input: `ContentPart(type="image_base64", ...)` through `generate`/`stream` with a vision model; message construction for `Agent` calls with parts | Model answers a question about a fixture image | Call the provider SDK directly inside `describe_image` | key, vision model |
| S5 | Dynamic subagents on an isolated `Reef`: create and `close()` agents at runtime; messages reach only their targets; `request_and_wait` fan-out to N replicas respects concurrency; no cross-talk with the default reef | 12 shards across 4 replicas complete, results in order, no leaked threads after `shutdown` | Thread-pool of `Agent` objects, reef used for events only | key |
| S6 | Can a prior message list be injected into an `Agent` (session resume)? Does HITL suspension raised in a subagent worker thread surface cleanly to the main thread? | History injection works; suspended subagent run can be approved from the main thread and resumed | Resume by summary message; subagents use read-only tools plus result-only mutation via the main agent | key |
| S7 | Subagent overhead: creation time, memory, and latency per delegated task against the same work done by the main agent alone | Report, no fixed threshold; the numbers decide the default subagent limits | Lower default `max_alive` or mark replicas as opt-in | key |
| S8 | Praval eval suite format: can the acceptance scenarios (SPEC.md, section 11) be expressed as cases with filesystem checks, run through `praval eval run`? | Scenarios 1, 6, 7 run as cases | Own minimal runner under `tests/acceptance/` with the same case files | none |
| S9 | Shell wrapper portability: capture working directory after a command under bash 3.2 (`/bin/bash` on macOS) and zsh with a POSIX `sh` wrapper; confirm process-group signalling | Same results on both shells | Simplify to cwd tracking via a trailing `pwd -P` marker | none |
| S10 | Copy-on-write checkpoint: `cp -c` (macOS) and `cp --reflink=auto` (Linux) behaviour and cost on a 200 MB, 20,000-file tree | Clone under 2 s on APFS; graceful fallback when unsupported | Checkpointing stays opt-in and size-limited, or is dropped | none |
| S11 | Thinking display (SPEC.md, section 5.7): does the optional `intent` argument appear reliably in tool calls from a hosted and a local model? With a local reasoning model on Ollama, does `provider_options.reasoning_effort` pass through to the server? | `intent` precedes each tool line in a five-call task; `reasoning_effort: "none"` changes local latency measurably | Make `intent` required in tool schemas; drop the `--reasoning` mapping for local providers | key, local server |
| S12 | MCP: connect a stdio and a Streamable HTTP test server; `list_tools`, wrap with `add_tool_spec(async_only=True)` and call through `astream` inside a Textual app's event loop; the TUI stays responsive during a 10 s model call and `sleep 10`; extra `intent` property stripped; which environment variables a stdio server receives when `env` is `None`; close on exit leaves no child processes | Tool call round trip on both transports, no leaked processes, env set recorded | Run MCP clients on a dedicated asyncio thread and bridge calls | none for the test server; key for the model |
| S13 | Web search: success rate and latency of the `duckduckgo` HTML backend over 50 queries from a home connection, against a local SearXNG instance | Recorded numbers decide open decision 9 | Default to `searxng` and document setup | none |
| S14 | Trace redaction: can `praval-code` redact span content before Praval's exporter writes it (a span processor through `configure_observability`), or scrub the trace store after each write? | A fake key in a tool result never reaches `traces.db` with content capture on | Keep content capture off; the session log holds full redacted content | none |
| S15 | Vibe seam: run Vibe's UI against a minimal Praval engine over the in-memory transport. Which protocol methods and notifications does the UI call at startup, in a basic turn, on an approval, and on `/compact` and `/rewind`? How does the UI behave when a method returns "not supported"? Record Vibe's key bindings and the kept dependency set | A two-tool turn with one approval renders correctly through the Praval engine; the list of required methods is recorded here | Keep more of Vibe's server and adapt it to Praval instead of replacing it | key |

Order of work: S9, S10, S8, S13 need no credentials and can run first. S1, S2, S4, S12 and S15 come next because they gate the core loop and the UI seam; S11 and S14 can follow. S5 to S7 answer the subagent experiment.

## 4. External references

- Praval repository: https://github.com/aiexplorations/praval; docs: https://pravalagents.com
- POSIX Utility Conventions (Guidelines 1 to 14): https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap12.html
- NO_COLOR convention: https://no-color.org
- XDG Base Directory Specification: https://specifications.freedesktop.org/basedir-spec/latest/
- Command Line Interface Guidelines: https://clig.dev
- Python `tomllib` (3.11+): https://docs.python.org/3/library/tomllib.html
- `rich`: https://rich.readthedocs.io
- Textual: https://textual.textualize.io
- Mistral Vibe: https://github.com/mistralai/mistral-vibe
- OpenCode: https://github.com/anomalyco/opencode (archived Go predecessor: https://github.com/opencode-ai/opencode)
- Model Context Protocol: https://modelcontextprotocol.io
- SearXNG: https://docs.searxng.org
- Bash 3.2 compatibility: avoid associative arrays, `mapfile`/`readarray`, `${var,,}`, `|&`, `coproc`.

## 5. Change log

- 0.1 (2026-10-05): initial draft of SPEC, CLI, API and REFERENCES.
- 0.1 (2026-10-05): added the thinking display requirement (SPEC.md 5.7) using the tool `intent` argument, `--thinking` and `--reasoning`, the `thinking` event, Praval streaming and reasoning facts, spike S11. Streaming model reasoning is out of scope for v1.
- 0.1 (2026-10-05): applied the new goals and non-goals: full-screen Textual TUI with print mode, OpenCode as architectural reference, MCP in v1 (async agent loop), web search, git practice, verified-output rule, logging on by default with redaction, directory scope without the temp-dir exception; spikes S12 to S14.
- 0.1 (2026-10-05): decided to fork Mistral Vibe, keep its Textual UI, and replace its app server and core with a Praval engine behind Vibe's protocol; Python 3.12+; Vibe facts (section 2.1); spike S15.
