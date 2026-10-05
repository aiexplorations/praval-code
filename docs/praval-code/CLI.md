# praval-code Command Line and TUI Design

Status: draft 0.1. Part of the [specification](SPEC.md). Internal interfaces and schemas are in [API.md](API.md).

## 1. Synopsis

```
praval-code [OPTIONS] [PROMPT]        start the TUI; PROMPT, if given, is sent as the first message
praval-code -p [OPTIONS] PROMPT       print mode: run one prompt without the TUI and exit
praval-code run [OPTIONS] PROMPT      same as -p
praval-code undo [OPTIONS] [SEQ]      reverse recorded changes
praval-code history [OPTIONS]         show journal and command log
praval-code sessions COMMAND          list | show | resume | rm
praval-code config COMMAND            get | set | list | path
praval-code doctor [--probe]          check environment, provider and tool support
praval-code completion bash|zsh       print a completion script
praval-code gc                        remove sessions and blobs past retention
praval-code eval run SUITE            run an offline acceptance suite
```

Rules for choosing a mode:

- `praval-code` and `praval-code "prompt"` on a terminal start the TUI. `-p`/`--print` and `run` select print mode.
- If stdin is not a TTY, print mode is used. With a prompt, stdin is read (up to 1 MB) and supplied to the agent as input material, labelled `stdin`; without one, the whole of stdin is the prompt.
- The TUI needs a terminal on both stdin and stdout. If either is missing and print mode was not requested, `praval-code` exits with code 2 and suggests `-p`.

## 2. Global options

Short and long forms follow POSIX and GNU convention. Every long option has the form `--name VALUE` or `--name=VALUE`.

| Option | Meaning |
|---|---|
| `-h`, `--help` | Help for the current command. `praval-code --help` lists commands; `praval-code undo --help` covers only `undo`. |
| `-v`, `--version` | Print `praval-code X.Y.Z (praval A.B.C, python 3.x)` to stdout, exit 0. |
| `-p`, `--print` | Print mode: run one prompt without the TUI (section 3). |
| `-q`, `--quiet` | Print mode: suppress activity output on stderr (thinking, tool lines, usage). Errors and approval prompts remain. |
| `--thinking MODE` | `show` (default) or `hide`: whether the agent's thinking is displayed on stderr (SPEC.md, section 5.7). |
| `--verbose` | Add detail to stderr: tool arguments, timings, token usage per round. |
| `--debug` | Show internals: in print mode, log to stderr and print the trace timeline after a failed run; in the TUI, open a debug pane. |
| `--no-log` | Do not write the session log or traces for this run. The journal is still written. |
| `--no-color` | Disable ANSI colour. Also disabled by `NO_COLOR` set to any value, `TERM=dumb`, or a non-TTY stream. |
| `-o`, `--output FORMAT` | Print mode: `text` (default), `json`, `jsonl`. |
| `--json` | Alias for `--output json`. |
| `-C`, `--cwd DIR` | Working directory for the session. |
| `--config FILE` | Use this config file instead of discovery. |

Agent options (apply to the TUI and print mode):

| Option | Meaning |
|---|---|
| `--provider NAME` | `anthropic`, `openai`, `gemini`, `cohere`, `ollama`, `vllm`, `lmstudio`, `llama-cpp`, `openai-compatible`. |
| `-m`, `--model NAME` | Model identifier for the provider. |
| `--base-url URL` | Endpoint for OpenAI-compatible providers. |
| `--max-tool-rounds N` | Tool-call rounds per request (default 40, range 1 to 1000). |
| `--timeout SECONDS` | Wall-clock limit for the whole run in print mode (default none). |
| `--system FILE` | Append the file's text to the system message. |
| `-f`, `--file PATH` | Attach a document; repeatable. Equivalent to asking the agent to read it. |
| `-i`, `--image PATH` | Attach an image; repeatable. Requires an image-capable model. |
| `-c`, `--continue` | Resume the most recent session in the current directory. |
| `--session ID` | Resume or name a session. |
| `--shell PATH` | Shell used for commands (bash or zsh). |
| `--reasoning LEVEL` | `off` (default), `low`, `medium`, `high`: ask the provider for model reasoning. Costs tokens; the reasoning itself is not displayed. Hosted providers receive it as Praval `ReasoningConfig`, and exit 3 if the model does not support it. OpenAI-compatible providers receive it as `provider_options.reasoning_effort`, where `off` sends `"none"`: local reasoning models reason by default, and Praval drops their reasoning text, so it adds latency with nothing to show. |

Safety options:

| Option | Meaning |
|---|---|
| `--approval MODE` | `ask` (default), `read-only`, `auto`. |
| `-y`, `--yes` | Same as `--approval auto`. Hard denies still apply. |
| `--non-interactive` | Print mode: never prompt. Approval-gated actions fail with exit code 4 unless `--yes`. Implied when no terminal is available. |
| `--dry-run` | Mutating tools validate and render, but apply nothing. |
| `--add-dir DIR` | Additional writable root; repeatable. |
| `--pass-env NAME` | Pass a named environment variable through to commands; repeatable. |
| `--allow-subagents` | Approve subagent rosters without prompting (needed in headless runs without `--yes`). |
| `--max-subagents N` | Alive subagents (default 8). |
| `--max-concurrent N` | Running subagents (default 4). |
| `--allow-private-net` | Let `fetch_url` and `web_search` reach private and link-local addresses. |
| `--search-backend NAME` | `duckduckgo` or `searxng` (SPEC.md, section 5.5). |
| `--no-mcp` | Do not start configured MCP servers for this run. |
| `--checkpoint MODE` | `off` (default) or `auto`. |

## 3. Print mode: streams and output

This section applies to print mode. The TUI owns the terminal and writes nothing to stdout (section 8).

Text output (`--output text`, the default):

- stdout: the agent's final answer only, as plain text when not a TTY, rendered Markdown when a TTY.
- stderr: thinking, tool-call lines (`> run_shell: du -sh *`), diffs, approval prompts, spinners, usage, warnings, errors.

Thinking is printed before the tool call it leads to, and before that call's approval prompt. On a TTY stderr it is dim italic behind a gutter; without colour each line is prefixed `thinking: ` instead, so it stays distinct from tool lines and the answer. Subagent lines carry the subagent name.

```
│ thinking  The user wants the largest directories; du at depth 1 is enough.
> run_shell: du -sh * | sort -rh | head
│ thinking (doc_reader#2)  report.docx has no headings; summarising by paragraph.
```

`praval-code "list large files" > answer.txt` therefore writes only the answer. `praval-code ... 2>/dev/null` shows only the answer.

JSON mode (`--output json`): stdout receives exactly one JSON object after the run ends. Nothing else is written to stdout. Stderr behaves as in human mode (use `-q` to silence it).

```json
{
  "schema": "praval_code.result/1",
  "session": "20261005-091500-a3f9",
  "status": "ok",
  "exit_code": 0,
  "answer": "string",
  "tool_calls": [
    {"id": "call_1", "agent": "praval_code", "tool": "run_shell", "args": {"command": "du -sh *"},
     "approved": true, "is_error": false, "duration_ms": 112}
  ],
  "changes": [
    {"seq": 14, "op": "write", "path": "/abs/path", "before": "sha256:...", "after": "sha256:..."}
  ],
  "subagents": [{"name": "doc_reader", "pattern": "replicas", "count": 4, "tool_rounds": 9, "tokens_in": 4210, "tokens_out": 880}],
  "usage": {"tokens_in": 9120, "tokens_out": 1530, "reasoning_tokens": 0, "tool_rounds": 6, "wall_ms": 18431},
  "error": null
}
```

On failure `status` is `error`, `error` is `{"code": "approval_required", "message": "...", "hint": "..."}`, and the exit code matches section 5.

JSON Lines mode (`--output jsonl`): stdout receives one event per line as the run progresses (event types in API.md, section 4), including `thinking` events. In `json` mode thinking is never part of `answer`. The last line is `{"type": "final", ...}` carrying the same object as `json` mode.

## 4. Help text (shape)

`praval-code --help` lists commands and the global options only; each command's flags appear under its own help.

```
praval-code: a terminal agent for bash and zsh

Usage:
  praval-code [OPTIONS] [PROMPT]       start the TUI
  praval-code -p [OPTIONS] PROMPT      run one prompt and print the answer
  praval-code COMMAND [OPTIONS]

Commands:
  run         Run one prompt and exit (same as -p)
  undo        Reverse changes made by praval-code
  history     Show recorded changes and commands
  sessions    List, show, resume or remove sessions
  config      Read or change configuration
  doctor      Check the environment and provider
  completion  Print a shell completion script

Common options:
  -h, --help          Show help
  -v, --version       Show version
  -p, --print         Run one prompt without the TUI
  -q, --quiet         Suppress activity output (print mode)
  --provider NAME     Provider to use
  -m, --model NAME    Model to use
  -y, --yes           Approve all actions except hard-denied ones
  -o, --output FORMAT text, json or jsonl

Run 'praval-code COMMAND --help' for command options.
praval-code runs with your own permissions and never uses sudo.
```

`praval-code undo --help`:

```
Usage: praval-code undo [OPTIONS] [SEQ]

Reverse recorded changes, newest first.

Arguments:
  SEQ                 Undo back to and including this journal sequence number

Options:
  -n, --last N        Undo the last N changes (default 1)
  --all               Undo every change in the current session
  --session ID        Session to act on (default: latest here)
  --checkpoint N      Restore from shell-command checkpoint N
  --force             Restore even if a file changed since praval-code wrote it
  --dry-run           Show what would be restored
  -y, --yes           Do not ask for confirmation
```

## 5. Exit codes

| Code | Meaning | Example |
|---|---|---|
| 0 | Success | Run completed |
| 1 | General error | Unexpected failure after retries |
| 2 | Usage or validation error | Unknown flag, TUI requested without a terminal, bad `--approval` value |
| 3 | Configuration or provider error | Missing API key, unreachable local server, model lacks required capability, configured MCP server fails to start |
| 4 | Approval required or denied | `--non-interactive` run reached a gated action; user rejected and the agent could not proceed |
| 5 | Budget exhausted | `max_tool_rounds`, `--timeout` or subagent budget reached before an answer |
| 6 | Policy hard deny | Attempt to use `sudo`, write outside allowed roots, or run a command with its working directory outside them |
| 130 | Interrupted | SIGINT |

Exit codes 4 to 6 print a one-line reason on stderr, and in JSON mode fill `error.code` (`approval_required`, `budget_exhausted`, `policy_denied`).

## 6. Error message format

Errors on stderr have three parts: what failed, why, and what to do.

```
praval-code: error: no API key for provider 'anthropic'
  ANTHROPIC_API_KEY is not set and no key is configured.
  Set it with 'export ANTHROPIC_API_KEY=...', or choose a local model with
  'praval-code --provider ollama --model llama3.1'. Run 'praval-code doctor' to check.
```

```
praval-code: error: approval required for 'run_shell' (rm -r build)
  Running non-interactively without --yes, so the action was not run.
  Re-run with --yes to allow it, or run interactively to review it.
```

Stack traces are never shown without `--debug`. In debug mode they appear after the actionable message. In the TUI, errors appear in the conversation view in the same three-part form.

## 7. Configuration

Vibe reads `~/.vibe/config.toml` and project `.vibe/` directories. The fork reads only the `praval-code` files below; Vibe's config paths are not consulted, and the UI's config views are served from this configuration by the engine.

Precedence, highest first: command-line flags, environment variables, project file (`./.praval-code.toml` in the working directory or an ancestor up to `$HOME`), user file (`${XDG_CONFIG_HOME:-~/.config}/praval-code/config.toml`), defaults. `praval-code config path` prints the files in use.

```toml
# ~/.config/praval-code/config.toml
[agent]
provider = "anthropic"
model = "claude-sonnet-5-5"
max_tool_rounds = 40
approval = "ask"          # ask | read-only | auto
reasoning = "off"         # off | low | medium | high (billed by the provider)
checkpoint = "off"        # off | auto

[shell]
path = ""                  # empty: $SHELL if bash or zsh, else /bin/bash
timeout_seconds = 120
pass_env = []

[provider]                 # used for local and OpenAI-compatible servers
base_url = ""

[provider.capabilities]    # overrides passed to Praval as provider_options.capabilities
tools = false              # set true for models that support tool calling
image_input = false
streaming = true

[display]
thinking = "show"         # show | hide

[web]
search_backend = "duckduckgo"   # duckduckgo | searxng
searxng_url = ""                # e.g. "http://localhost:8888"; exempt from the private-network refusal

[log]
enabled = true                  # session log and traces (SPEC.md, section 8)
redact_patterns = []            # extra regular expressions to redact

[tui]
context_warn_percent = 85

[subagents]
max_alive = 8
max_concurrent = 4
task_timeout_seconds = 300
task_max_tool_rounds = 20

[paths]
extra_roots = []

[state]
retention_days = 30

[mcp.servers.docs]              # one table per MCP server (SPEC.md, section 5.9)
transport = "stdio"             # stdio | streamable_http
command = "npx"
args = ["-y", "@example/word-mcp-server"]
env = { }                       # the only extra variables the server receives
allow_tools = []                # tools that run without approval; empty means all gated
tool_timeout_seconds = 30

[mcp.servers.mail]
transport = "streamable_http"
url = "https://mcp.example.com/mail"   # https, or http on loopback only
headers = { Authorization = "env:MAIL_MCP_TOKEN" }   # "env:NAME" reads the value from the environment
```

The server package names above are placeholders, not recommendations.

Example local setup:

```toml
[agent]
provider = "ollama"
model = "llama3.1"

[provider.capabilities]
tools = true
```

Project files may not set `approval = "auto"`, add `extra_roots`, declare `[mcp.servers.*]`, or change `searxng_url`; those keys are honoured only from the user file, command line or environment, so a cloned repository cannot loosen the policy or start a command.

### Environment variables

| Variable | Meaning |
|---|---|
| `PRAVAL_CODE_PROVIDER`, `PRAVAL_CODE_MODEL`, `PRAVAL_CODE_BASE_URL` | Override provider, model, endpoint |
| `PRAVAL_CODE_APPROVAL` | Approval mode |
| `PRAVAL_CODE_THINKING` | Thinking display, `show` or `hide` |
| `PRAVAL_CODE_NO_LOG` | Same as `--no-log` when set to any value |
| `PRAVAL_CODE_CONFIG` | Config file path |
| `PRAVAL_CODE_STATE_DIR` | State directory (journal, sessions, blobs) |
| `NO_COLOR` | Disable colour when set to any value |
| `SHELL` | Default shell for commands |
| `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, `COHERE_API_KEY` | Provider credentials (names as read by the Praval providers; to be confirmed per provider in spike S1) |
| `XDG_CONFIG_HOME`, `XDG_STATE_HOME` | Base directories |

`PRAVAL_CODE_PROVIDER` and `PRAVAL_CODE_MODEL` are mapped to Praval's `PRAVAL_DEFAULT_PROVIDER` and `PRAVAL_DEFAULT_MODEL` (both present in Praval 0.8.3).

## 8. TUI

The TUI is Mistral Vibe's Textual UI, trimmed and rebranded (SPEC.md, section 4.1). Its layout, keys, history, completion, theming and approval dialogs are kept as Vibe implements them; this section lists only what changes. Vibe's exact key bindings are recorded from the fork during spike S15 rather than restated here.

Status area: provider and model, approval mode, working directory and git branch, context used, session cost, running subagents.

Thinking: the `intent` lines (SPEC.md, section 5.7) render through Vibe's reasoning display, directly above the tool call they belong to.

Lines starting with `!` run a shell command directly under the same policy and journal, without involving the model. Lines starting with `/` are slash commands.

Slash commands kept from Vibe, with their behaviour backed by the Praval engine:

| Command | Action |
|---|---|
| `/help` | List slash commands and keys |
| `/exit` | Leave the TUI |
| `/clear`, `/new` | Clear the conversation context, or start a new session (the journal and log are kept) |
| `/compact` | Replace the conversation history with a summary to free context |
| `/model` | Show or switch model |
| `/config`, `/reload` | Show configuration; reload it from disk |
| `/mcp` | List MCP servers, connection state and tools |
| `/resume`, `/continue`, `/rename` | Resume or rename a session |
| `/rewind` | Rewind the conversation; file changes are reverted through the journal |
| `/retry` | Re-run the last turn |
| `/copy` | Copy the last answer |
| `/status` | Session, provider and context summary |
| `/thinking` | Show or switch thinking display |
| `/theme` | Choose a theme |
| `/todo` | Show the agent's task list |
| `/log`, `/log-level`, `/debug` | Show the session log; change log level; open debug output |

Slash commands added by this spec:

| Command | Action |
|---|---|
| `/provider [NAME]` | Show or switch provider |
| `/approval [MODE]` | Show or switch approval mode (cannot be switched to `auto` if configuration forbids it) |
| `/tools` | List tools with approval requirements |
| `/agents`, `/agents retire NAME` | List subagents with purpose, pattern, state and usage; close one |
| `/diff` | Show all changes made in this session |
| `/undo [N]` | Undo the last N changes (default 1) |
| `/history` | Show recent changes and commands |
| `/checkpoint` | Create a checkpoint of the working directory now |
| `/attach PATH` | Attach a document or image to the next message |
| `/cost` | Token and tool-round usage for this session |
| `/trace` | Show the trace timeline for the last request |
| `/session` | Show session id, state directory and log path |

Removed from Vibe in v1: `/connectors`, `/data-retention`, `/leanstall`, `/unleanstall`, `/loop`, `/plugins`, `/reload-plugins`, `/proxy-setup`, `/remote-project`, `/skills`, `/teleport`, `/voice`, `/whoami`, `/branch`. Kept or removed is decided per command during the fork; `/paste-image` is folded into `/attach`.

## 9. Approval prompt (shape)

In the TUI the prompt is Vibe's approval dialog, raised by a server callback from the engine; in print mode it is written to stderr and read from `/dev/tty`. The shapes below are the content each prompt carries.

```
? Run shell command   [risk: medium]
    rm -r build/
  Reason: removes a directory tree (mutating command).
  [y] run   [n] reject   [e] edit   [a] always allow run_shell this session   [?] help
```

For file changes the prompt shows a unified diff first (rendered with rich on a TTY, plain `diff -u` style otherwise). For a subagent roster it shows the table below, with `[d]` to expand any subagent's system message.

```
? Create 3 subagents
    NAME         PATTERN     COUNT  TOOLS                    PURPOSE
    doc_reader   replicas    4      read_document, grep      summarise one file each
    combiner     specialist  1      (none)                   merge summaries into a digest
    reviewer     evaluator   1      read_file                check digest covers every file
  [y] create   [n] reject   [e] edit counts/tools   [d] details
```

## 10. Signals and cleanup

| Signal | Behaviour |
|---|---|
| Ctrl+C in the TUI | Clear the input; on an empty input, a second press within 2 s exits |
| Esc or Ctrl+C while a model request runs | Cancel the request, keep the session, return focus to the input (print mode: exit 130) |
| Esc or Ctrl+C while a command runs | Signal the command's process group, then return focus to the input |
| SIGTERM | Close MCP servers and subagents, restore the terminal, exit 143 |
| SIGPIPE / closed stdout | Stop writing, finish journaling, exit quietly |

On any exit, praval-code closes agents and MCP server connections, restores the terminal, removes temporary files, flushes the journal and log, and leaves no partial writes (file writes are temp-file plus atomic rename).

## 11. Completion

`praval-code completion bash` and `praval-code completion zsh` print scripts for static completion of commands, flags and enumerated values. Scripts avoid bash 4 features (no associative arrays) so they load in bash 3.2. Installation:

```
praval-code completion zsh  > "${fpath[1]}/_praval-code"
praval-code completion bash > ~/.local/share/bash-completion/completions/praval-code
```
