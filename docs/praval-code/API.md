# praval-code Internal API Specification

Status: draft 0.1. Part of the [specification](SPEC.md). The user-facing contract is in [CLI.md](CLI.md). Praval calls referenced here are listed with their verification status in [REFERENCES.md](REFERENCES.md).

Python 3.12 to 3.14, full type hints, PEP 8.

## 1. Package layout

The code lives in the fork of `mistralai/mistral-vibe`. Vibe's UI and protocol packages are kept; the engine is a new package beside them.

```
vibe/cli/textual_ui/      Vibe UI (kept, rebranded, trimmed)
vibe/cli/                 launcher, commands, programmatic mode (kept, adapted)
vibe/app_server/          protocol, models, events, client, transport (kept as the contract);
                          Vibe's server-side modules removed
praval_code/
  server.py              app-server protocol handlers: sessions, turns, callbacks, config, MCP state
  projection.py          internal events (section 4) -> Vibe protocol notifications
  cli.py                 print mode, argument grammar additions, exit codes
  config.py              load/merge TOML, env, flags; dataclasses
  agent.py               builds the Praval Agent, runs one request with astream(), yields events
  events.py              internal Event dataclasses and JSON serialisation
  log.py                 session log writer, redaction, Praval trace configuration
  policy.py              approval modes, hard denies, read-only classifier
  journal.py             change journal, blob store, undo, checkpoints
  session.py             session log read/write, session ids
  render/                print mode: human.py, plain.py, jsonout.py
  tools/
    __init__.py          registry: build_tools(ctx) -> list of tool functions
    shell.py             run_shell
    files.py             read_file, write_file, edit_file, list_dir, find_files, grep, move_path, delete_path, make_dir
    documents.py         read_document
    images.py            describe_image
    web.py               web_search, fetch_url
    mcp.py               MCP server lifecycle, tool discovery and wrapping
    git.py               repository detection, branch and status checks
    subagents.py         spawn_subagents, delegate, map_subagents, evaluate, list_subagents, retire_subagents
  subagents/
    manager.py           roster, lifecycle, reef, budgets
    patterns.py          specialist, replicas, evaluator
  doctor.py              environment and provider checks
tests/
```

Module boundaries are the unit of work for implementation: `policy`, `journal`, `log`, `tools/*`, `render/*`, `subagents/*` depend only on `config` and `events`, so they can be built and tested independently. `server.py` and `projection.py` are the only modules that import Vibe's protocol types.

## 2. Core types

```python
@dataclass(frozen=True)
class RunContext:
    session_id: str
    cwd: Path
    roots: tuple[Path, ...]          # allowed write roots
    policy: Policy
    journal: Journal
    config: Config
    emit: Callable[[Event], None]    # event sink used by tools and agents
    agent_name: str = "praval_code"        # "praval_code" or a subagent name

class Policy:
    def check_shell(self, command: str, cwd: Path) -> Decision: ...
    def check_path_write(self, path: Path) -> Decision: ...
    def check_path_read(self, path: Path) -> Decision: ...        # ask outside roots
    def check_mcp(self, server: str, tool: str, annotations: Mapping[str, Any]) -> Decision: ...
    def check_net(self, url: str) -> Decision: ...

class Decision(NamedTuple):
    action: Literal["allow", "ask", "deny"]
    risk: Literal["low", "medium", "high"]
    reason: str

class Journal:
    def record_file_change(self, ctx: RunContext, op: str, path: Path, tool_call_id: str) -> Entry: ...
    def log_command(self, ctx: RunContext, result: CommandResult) -> None: ...
    def undo(self, *, last: int | None = None, to_seq: int | None = None,
             session: str | None = None, force: bool = False, dry_run: bool = False) -> UndoReport: ...
    def checkpoint(self, ctx: RunContext) -> CheckpointId | None: ...
    def restore_checkpoint(self, cp: CheckpointId, *, confirm_removals: Callable[[list[Path]], bool]) -> UndoReport: ...
```

Praval mapping: the agent loop is async and uses `Agent.astream()`, because MCP tools are registered async-only. `agent.py` constructs `praval.Agent(name="praval_code", provider=..., model=..., system_message=SYSTEM, hitl_enabled=True, hitl_db_path=<state>/hitl.db, max_history=...)`, registers each tool with `Agent.tool` (or `add_tool_spec` for tools whose schema is built dynamically), and sets `max_tool_rounds` through `ModelRequest`/config (exact route fixed by spike S1).

## 3. Tools

Built-in tools are async functions, so they never block the TUI's event loop (SPEC.md, section 5.8). All tool results are strings (Praval `ToolResult.content`). Errors are returned as results with `is_error` true, never raised, so the model can correct itself. Paths are resolved against the session working directory; relative paths are allowed.

Every tool schema, including wrapped MCP tools, also accepts an optional `intent` string ("One sentence: why you are making this call."). The schemas below omit it for brevity. It is emitted as a `thinking` event before the tool runs: for gated tools the approval prompt reads it from the pending intervention's arguments, otherwise the wrapper emits it on entry. The wrapper strips it before calling the tool, and it is recorded in the session log (SPEC.md, section 5.7).

Approval column: `-` runs without approval in `ask` mode; `gated` requires approval in `ask` mode. `read-only` mode removes gated tools.

| Tool | Approval | Risk |
|---|---|---|
| `run_shell` | gated unless classified read-only | low/medium/high from classifier |
| `read_file`, `list_dir`, `find_files`, `grep`, `read_document`, `describe_image` | - inside roots; gated outside | low |
| `write_file`, `edit_file`, `make_dir` | gated | medium |
| `move_path`, `delete_path` | gated | high |
| `web_search`, `fetch_url` | gated | medium |
| MCP tools (`server__tool`) | gated unless allowlisted in user config | from annotations: high if destructive, else medium; never lowered below medium |
| `spawn_subagents` | gated | medium |
| `delegate`, `map_subagents`, `evaluate`, `list_subagents`, `retire_subagents` | - (children's own gated tools still gate) | low |

Because Praval's `requires_approval` is a static flag on the tool, conditional approval (a read-only shell command running without a prompt) is handled by registering `run_shell` with `requires_approval=True` and having `policy.check_shell` auto-approve through the HITL service when the decision is `allow`. If spike S2 shows that suspend/resume cannot be auto-resolved cheaply, `run_shell` is split into `run_shell_readonly` (ungated, rejects anything not classified read-only) and `run_shell` (gated).

### 3.1 run_shell

```json
{
  "name": "run_shell",
  "description": "Run a command in the user's shell (bash or zsh) as the current user. Each call is a fresh process: the working directory persists between calls, but exported variables, aliases and functions do not. Never use sudo or other privilege escalation; it is blocked.",
  "parameters": {
    "type": "object",
    "properties": {
      "command": {"type": "string", "description": "Command line to run."},
      "cwd": {"type": "string", "description": "Directory to run in. Defaults to the session directory."},
      "timeout_seconds": {"type": "integer", "minimum": 1, "maximum": 3600, "default": 120}
    },
    "required": ["command"]
  }
}
```

Result text: exit code, a combined stdout/stderr transcript (capped as in SPEC.md 5.1), and the new working directory if it changed.

### 3.2 File tools

```json
{
  "read_file":  {"properties": {"path": {"type": "string"}, "offset": {"type": "integer", "minimum": 1}, "limit": {"type": "integer", "minimum": 1, "maximum": 5000}}, "required": ["path"]},
  "write_file": {"properties": {"path": {"type": "string"}, "content": {"type": "string"}, "overwrite": {"type": "boolean", "default": false}}, "required": ["path", "content"]},
  "edit_file":  {"properties": {"path": {"type": "string"}, "old": {"type": "string"}, "new": {"type": "string"}, "replace_all": {"type": "boolean", "default": false}}, "required": ["path", "old", "new"]},
  "list_dir":   {"properties": {"path": {"type": "string", "default": "."}, "depth": {"type": "integer", "minimum": 1, "maximum": 5, "default": 1}}},
  "find_files": {"properties": {"pattern": {"type": "string", "description": "Glob such as **/*.py"}, "path": {"type": "string", "default": "."}}, "required": ["pattern"]},
  "grep":       {"properties": {"pattern": {"type": "string", "description": "Regular expression"}, "path": {"type": "string", "default": "."}, "glob": {"type": "string"}, "ignore_case": {"type": "boolean", "default": false}, "max_results": {"type": "integer", "default": 200}}, "required": ["pattern"]},
  "move_path":  {"properties": {"source": {"type": "string"}, "destination": {"type": "string"}}, "required": ["source", "destination"]},
  "delete_path":{"properties": {"path": {"type": "string"}, "recursive": {"type": "boolean", "default": false}}, "required": ["path"]},
  "make_dir":   {"properties": {"path": {"type": "string"}}, "required": ["path"]}
}
```

Behaviour: `edit_file` fails if `old` is absent or (without `replace_all`) not unique, and reports the match count. `write_file` fails on an existing path unless `overwrite` is true. `read_file` returns numbered lines and flags binary files instead of dumping them. `delete_path` on a directory requires `recursive` and journals every contained file. All mutating tools check `policy.check_path_write` first, create a journal entry before applying the change, and write via temp file and atomic rename.

### 3.3 Documents, images, web

```json
{
  "read_document": {"properties": {"path": {"type": "string"}, "max_chars": {"type": "integer", "default": 60000}, "pages": {"type": "string", "description": "PDF page range such as 1-5"}}, "required": ["path"]},
  "describe_image": {"properties": {"path": {"type": "string"}, "question": {"type": "string", "default": "Describe this image in detail."}}, "required": ["path"]},
  "fetch_url": {"properties": {"url": {"type": "string", "format": "uri"}, "max_chars": {"type": "integer", "default": 60000}}, "required": ["url"]},
  "web_search": {"properties": {"query": {"type": "string"}, "max_results": {"type": "integer", "minimum": 1, "maximum": 20, "default": 8}}, "required": ["query"]}
}
```

`web_search` returns a JSON array `[{"title": "...", "url": "...", "snippet": "..."}]` and names the backend used. Backend failures (blocked, rate limited, unreachable) return an error result naming the backend and the `searxng` alternative.

### 3.4 MCP tools

```python
class McpManager:
    async def start(self, servers: Mapping[str, McpServerSettings]) -> list[ServerStatus]: ...
    async def register(self, agent: praval.Agent, ctx: RunContext) -> list[ToolSpec]: ...
    async def close(self) -> None: ...
```

`start` builds one `praval.mcp.MCPServerConfig` per configured server (`require_approval=True` always) and connects an `MCPClient`. A server that fails to connect is reported in the status line and `/mcp` and skipped; the session continues. `register` calls `list_tools()`, adds the `intent` property to each schema, sets `requires_approval` (true unless the tool is allowlisted in user config), and registers a wrapper with `agent.add_tool_spec(spec, handler, async_only=True)`. Approval happens in Praval's HITL layer before the wrapper runs. The wrapper runs `policy.check_mcp`, emits events, strips `intent`, then awaits `client.call_tool(name, arguments)`. MCP results are truncated by the client at `max_result_size` (1 MiB default) before the tool-output cap applies.

### 3.5 Subagent tools

```json
{
  "spawn_subagents": {
    "properties": {
      "roster": {
        "type": "array", "minItems": 1, "maxItems": 8,
        "items": {
          "type": "object",
          "properties": {
            "name": {"type": "string", "pattern": "^[a-z][a-z0-9_]{1,31}$"},
            "purpose": {"type": "string", "description": "One line the user will read."},
            "pattern": {"enum": ["specialist", "replicas", "evaluator"]},
            "count": {"type": "integer", "minimum": 1, "maximum": 8, "default": 1},
            "system_message": {"type": "string"},
            "tools": {"type": "array", "items": {"type": "string"}},
            "model": {"type": "string"}
          },
          "required": ["name", "purpose", "pattern", "system_message"]
        }
      }
    },
    "required": ["roster"]
  },
  "delegate": {"properties": {"agent": {"type": "string"}, "task": {"type": "string"}, "inputs": {"type": "array", "items": {"type": "string"}, "description": "Paths or text the subagent needs."}}, "required": ["agent", "task"]},
  "map_subagents": {"properties": {"agent": {"type": "string", "description": "Replicas role to use."}, "task": {"type": "string", "description": "Instruction applied to every shard."}, "shards": {"type": "array", "items": {"type": "string"}, "minItems": 1, "maxItems": 200}}, "required": ["agent", "task", "shards"]},
  "evaluate": {"properties": {"agent": {"type": "string", "description": "Evaluator role."}, "artifact": {"type": "string", "description": "Text or file path to evaluate."}, "rubric": {"type": "string"}}, "required": ["agent", "artifact", "rubric"]},
  "list_subagents": {"properties": {}},
  "retire_subagents": {"properties": {"names": {"type": "array", "items": {"type": "string"}}}, "required": ["names"]}
}
```

Results:

- `delegate` returns the subagent's answer text.
- `map_subagents` returns a JSON array `[{"shard": "...", "ok": true, "answer": "...", "error": null}]` in shard order. Shards beyond the concurrency limit queue; failures are per-shard and do not abort the others.
- `evaluate` returns `{"verdict": "pass"|"fail", "score": 0.0-1.0, "notes": "..."}`. Evaluators are instructed to answer in that JSON shape; `praval-code` validates it and retries once on malformed output.

## 4. Event stream

The agent loop yields `Event` objects. In the TUI, `projection.py` maps them onto Vibe's protocol notifications and server callbacks (approval requests become callbacks the UI answers); print-mode renderers consume them directly, and in `jsonl` mode each event is one line. The event-to-notification mapping is fixed by spike S15.

Common fields: `type` (string), `ts` (ISO 8601 UTC), `session` (string), `agent` (string, `praval_code` or subagent name).

| `type` | Extra fields |
|---|---|
| `session.start` | `cwd`, `provider`, `model`, `approval`, `version` |
| `thinking` | `text`, `tool_call_id` |
| `message.delta` | `text` |
| `message.complete` | `text` |
| `tool.call` | `id`, `tool`, `args` |
| `approval.request` | `id`, `tool`, `summary`, `diff` (optional), `risk`, `reason` |
| `approval.decision` | `id`, `decision` (`approve`, `reject`, `edit`, `auto`, `denied`), `reason` |
| `tool.result` | `id`, `tool`, `is_error`, `duration_ms`, `summary` |
| `journal.entry` | `seq`, `op`, `path`, `before`, `after` |
| `subagent.roster` | `roster` (as approved) |
| `subagent.task` | `agent`, `task_id`, `state` (`started`, `done`, `error`), `shard` (optional) |
| `mcp.server` | `server`, `state` (`connected`, `failed`, `closed`), `tools` (count), `message` |
| `usage` | `tokens_in`, `tokens_out`, `reasoning_tokens`, `tool_rounds`, `context_percent` |
| `warning` | `message` |
| `error` | `code`, `message`, `hint` |
| `final` | the `praval_code.result/1` object (CLI.md, section 3) |

Mapping from Praval: `ModelEvent(type="delta")` becomes `message.delta`; `type="tool_call"` becomes `tool.call`; `type="tool_result"` becomes `tool.result`; `type="usage"` becomes `usage`. `approval.*`, `journal.*`, `mcp.*` and `thinking` events are produced by `praval-code`, not Praval. `reasoning_tokens` comes from Praval's usage normalisation (`runtime_observation.py`).

## 5. Journal format

Directory: `<state>/sessions/<session-id>/`

```
journal.jsonl      one JSON object per change or command
blobs/<sha256>     prior file contents, content-addressed
checkpoints/<n>/   cloned working-directory trees (when enabled)
transcript.jsonl   session log: messages and events, redacted (SPEC.md, section 8)
traces.db          Praval OpenTelemetry traces (content capture off until spike S14)
hitl.db            Praval HITL store (SQLite)
```

`journal.jsonl` entries:

```json
{"seq": 14, "kind": "file", "ts": "2026-10-05T09:15:02Z", "agent": "praval_code", "tool_call": "call_3",
 "op": "write", "path": "/abs/path/notes.md",
 "before": {"exists": true, "sha256": "ab12...", "mode": 420, "size": 311},
 "after":  {"exists": true, "sha256": "cd34...", "mode": 420, "size": 502},
 "undone": false}
{"seq": 15, "kind": "command", "ts": "...", "agent": "praval_code", "tool_call": "call_4",
 "command": "rm -r build/", "cwd": "/abs/path", "exit_code": 0, "duration_ms": 41,
 "classified": "mutating", "checkpoint": 3, "stdout_tail": "...", "stderr_tail": ""}
```

`op` is one of `create`, `write`, `delete`, `move` (with `from`/`to`), `mkdir`. Entries are appended with `O_APPEND` and fsynced; sequence numbers are per session and monotonic. Undo marks entries `"undone": true` rather than deleting them, and writes its own entry (`kind: "undo"`) so `history` shows the whole story. Undo of a `move` verifies the destination hash and that the source path is still free.

## 6. Session log format

Every event is passed through `log.redact()` before it is written or rendered. `transcript.jsonl` stores the event stream plus the user turns (`{"type": "user", "text": "...", "attachments": [...]}`). `--continue` rebuilds the conversation from `user` and `message.complete` events. Tool call and result events are summarised into the resumed context rather than replayed in full, so resumed sessions do not exceed the model's context window. `thinking` events are stored but not replayed.

## 7. Subagent manager

```python
class SubagentManager:
    def __init__(self, ctx: RunContext, parent_tools: Mapping[str, Callable]) -> None: ...
    def propose(self, roster: list[RosterEntry]) -> Proposal: ...         # validates, never creates
    def create(self, proposal: Proposal) -> list[str]: ...                # after approval
    def delegate(self, name: str, task: str, inputs: list[str]) -> str: ...
    def map(self, name: str, task: str, shards: list[str]) -> list[ShardResult]: ...
    def evaluate(self, name: str, artifact: str, rubric: str) -> Verdict: ...
    def retire(self, names: list[str]) -> None: ...
    def close(self) -> None: ...
```

Validation in `propose`: names unique and not `praval_code`; tool lists are subsets of the parent's registered tools and exclude all `subagents.*` tools and all MCP tools; counts respect `max_alive`; evaluator and replicas roles require `system_message`. Replicas are named `<name>_1` to `<name>_N`.

Transport: the manager owns one `praval.core.reef.Reef` (in-memory backend) per session. Each subagent is a `praval.Agent` created with its own `system_message` and the allowed tool subset, subscribed to a session channel, with a spore handler that runs the task and replies. `delegate` and `map` use the reef's request/reply calls (`request_and_wait`) with the task timeout. Spikes S5 and S6 determine whether this transport is used or the thread-pool fallback described in SPEC.md, section 7.4.

Each subagent's tool functions are wrapped so every call carries `agent_name` in its `RunContext`, which flows into events and journal entries.

## 8. Doctor checks

`praval-code doctor` prints a checklist to stdout (JSON with `--json`) and exits 3 if a blocking check fails:

| Check | Blocking |
|---|---|
| `praval`, `rich`, `textual` importable (versions shown) | yes |
| Python 3.12 to 3.14 | yes |
| `mcp` importable | yes if MCP is a required dependency, otherwise only when servers are configured (SPEC.md, open decision 10) |
| Effective UID is not 0 | yes |
| `bash` or `zsh` found | yes |
| State directory writable | yes |
| Provider configured, key present or endpoint reachable | yes |
| Model supports tools (`--probe` runs a real call) | warning for local models |
| Model supports image input | info |
| `rg`, `textutil`, copy-on-write `cp`, `git` (without an install prompt), `curl`, `wget` | info |
| Each configured MCP server connects and lists tools | warning |
| Search backend reachable | warning |
