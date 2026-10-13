# hx — release binaries

`hx` is an agentic coding harness: it runs a coding task through an implement → validate (your
tests) → review loop, using any OpenAI-compatible LLM endpoint. This repository holds only the
release binaries; the source is private.

> **For coding agents:** this README is written so you can install and configure hx end to end
> without asking the user anything except (1) whether they use subscriptions (Claude, ChatGPT,
> Google — Path A, recommended) or an API key (Path B), and (2) the browser logins or the key. Follow the steps in order. Every step has a check; do not move on until it
> passes. Do not print API keys.

## 1. Pick the right binary

| OS | CPU (`uname -m` / `$env:PROCESSOR_ARCHITECTURE`) | Asset |
|---|---|---|
| Linux | `x86_64` / `amd64` | `hx-linux-x64` |
| Linux | `aarch64` / `arm64` | `hx-linux-arm64` |
| Windows | `AMD64` | `hx-windows-x64.exe` |
| macOS, Windows ARM, 32-bit | — | **not supported**: stop and tell the user |

Each release `vX.Y.Z` also ships `CHECKSUMS-X.Y.Z.txt` (SHA-256 of every asset),
`manifest-X.Y.Z.json` and `hx-harness-vX.Y.Z.tar.gz` (project scaffolding that `hx init` downloads
by itself — you never fetch it by hand). The binary embeds its runtime: **no Bun, Node or Go is
needed** to run it.

## 2. Install

### Linux

```bash
set -eu
VERSION=$(curl -fsSL https://api.github.com/repos/robertof1lho/hx-releases/releases/latest \
  | sed -n 's/.*"tag_name": *"v\([^"]*\)".*/\1/p')
case "$(uname -m)" in
  x86_64|amd64)  ASSET=hx-linux-x64 ;;
  aarch64|arm64) ASSET=hx-linux-arm64 ;;
  *) echo "unsupported CPU: $(uname -m)"; exit 1 ;;
esac
BASE=https://github.com/robertof1lho/hx-releases/releases/download/v$VERSION
TMP=$(mktemp -d) && cd "$TMP"
curl -fsSLO "$BASE/$ASSET"
curl -fsSLO "$BASE/CHECKSUMS-$VERSION.txt"
sha256sum -c --ignore-missing "CHECKSUMS-$VERSION.txt"   # must print "<asset>: OK"
mkdir -p "$HOME/.local/bin"
install -m 755 "$ASSET" "$HOME/.local/bin/hx"
```

Make sure `~/.local/bin` is on `PATH` (add `export PATH="$HOME/.local/bin:$PATH"` to the shell rc
if `command -v hx` finds nothing).

### Windows (PowerShell)

```powershell
$ErrorActionPreference = "Stop"
if ($env:PROCESSOR_ARCHITECTURE -ne "AMD64") { throw "unsupported CPU: $env:PROCESSOR_ARCHITECTURE" }
$v = (Invoke-RestMethod https://api.github.com/repos/robertof1lho/hx-releases/releases/latest).tag_name.TrimStart("v")
$base = "https://github.com/robertof1lho/hx-releases/releases/download/v$v"
$dir = "$env:USERPROFILE\.local\bin"; New-Item -ItemType Directory -Force $dir | Out-Null
Invoke-WebRequest "$base/hx-windows-x64.exe" -OutFile "$dir\hx.exe"
$sums = (Invoke-WebRequest "$base/CHECKSUMS-$v.txt").Content
$want = ($sums -split "`n" | Where-Object { $_ -match "hx-windows-x64.exe" }) -split "\s+" | Select-Object -First 1
$got = (Get-FileHash "$dir\hx.exe" -Algorithm SHA256).Hash.ToLower()
if ($got -ne $want) { throw "checksum mismatch" }
# add $dir to the user PATH if missing
$p = [Environment]::GetEnvironmentVariable("Path", "User")
if ($p -notlike "*$dir*") { [Environment]::SetEnvironmentVariable("Path", "$p;$dir", "User") }
```

Open a new terminal afterwards. Install **Git for Windows** too: hx uses its `bash` for shell
commands (it falls back to `cmd.exe` otherwise, which breaks most task commands).

### Alternative: npm (Linux/Windows, Node ≥ 18)

```bash
npm install -g @robertof1lho/hx
```

The package's postinstall downloads the binary. If your npm blocks install scripts, `hx` will say
`executável hx-<platform> não encontrado`: download the binary as above and either set
`HX_BINARY=/path/to/binary` or put it on `PATH` under its platform name (`hx-linux-x64`, …).

### Check

```bash
hx version     # → "hx version X.Y.Z", commit, target
```

## 3. Prerequisites

| Tool | Needed? |
|---|---|
| `git` | **Yes, in practice.** Without a git repo the reviewer gets no diff. Run hx inside a git repository. |
| An OpenAI-compatible LLM endpoint | **Yes.** CLIProxyAPI with your subscriptions (recommended) or a provider API key — step 5. |
| [ai-jail](https://github.com/akitaonrails/ai-jail) (bubblewrap, Linux only) | Optional: sandboxes the agent's shell. Without it hx runs commands directly on the machine (`backend = "local"`). |
| Docker | Only for `backend = "container"`. |
| `gh` (GitHub CLI) | Only for `hx pr`, `hx run --issue/--pr`, `hx fix-ci`, `hx review --pr`. |

## 4. Set up a project

Run inside the root of the user's git repository:

```bash
cd /path/to/project
git init            # only if it is not a git repo yet
hx init --no-onboard
```

`hx init` downloads the matching harness package from this repository (checksum-verified) and
creates: `.hx/config.toml`, `.hx/workers.toml`, `.hx/lib/`, `.hx.lock`, `.ai/agents/`,
`.ai/skills/`, `.ai/CONTEXT.md`, `AGENTS.md`, `CLAUDE.md`, a `docs/` wiki skeleton and
`.gitignore` entries (`.ai/runs/`, `.env`, …). It refuses to run twice; `--force` overwrites.
Without ai-jail it writes `backend = "local"` plus `[workers] unsandboxed_shell = true`. If it
says `nenhuma pilha detectada`, you must declare gates yourself (step 5).

Messages from hx are in Brazilian Portuguese; the meaning of the important ones is given below.

## 5. Connect the models

hx talks to one OpenAI-compatible endpoint. Pick a path:

- **Path A (recommended): your subscriptions through CLIProxyAPI.** No provider API keys to paste;
  you log in once per subscription (Claude Pro/Max, ChatGPT/Codex, Google Antigravity) and the proxy
  refreshes the tokens. hx's generated config already targets this proxy and its model names, so
  **no `.hx/config.toml` change is needed**. Do 5A, then skip to 5.3.
- **Path B: a provider API key** (OpenAI, OpenRouter, LiteLLM, Ollama…). Do 5.1 and 5.2.

### 5A. Subscriptions via CLIProxyAPI

[CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI) logs into your subscriptions over OAuth
and serves them as one OpenAI-compatible API on `http://127.0.0.1:8317/v1` — hx's default
`base_url`.

**Install** (Linux x64; use `linux_aarch64` on ARM, `windows_amd64.zip` on Windows):

```bash
set -eu
TAG=$(curl -fsSL https://api.github.com/repos/router-for-me/CLIProxyAPI/releases/latest \
  | sed -n 's/.*"tag_name": *"\([^"]*\)".*/\1/p')
V=${TAG#v}
mkdir -p "$HOME/.local/bin/cliproxy" && cd "$HOME/.local/bin/cliproxy"
curl -fsSL "https://github.com/router-for-me/CLIProxyAPI/releases/download/$TAG/CLIProxyAPI_${V}_linux_amd64.tar.gz" | tar xz
cp -n config.example.yaml config.yaml
```

**Configure `config.yaml`** (YAML, in that folder): set `server.host: "127.0.0.1"` (local only) and
replace the example list under `access.api-keys` with ONE key you generate
(`openssl rand -hex 24`). This key protects the proxy; hx sends it. Keep `server.port: 8317`.
Never commit `config.yaml`.

**Log in** (once per subscription; tokens go to `~/.cli-proxy-api/`):

| Subscription | Command | Serves hx defaults |
|---|---|---|
| Claude Pro/Max | `./cli-proxy-api -claude-login` | `claude-sonnet-5-5`, `claude-opus-5-5` (implementer, manager, planner) |
| ChatGPT Plus/Pro (Codex) | `./cli-proxy-api -codex-login` (headless/SSH: `-codex-device-login`) | `gpt-5.6-terra` (reviewer and all judges) |
| Google Antigravity | `./cli-proxy-api -antigravity-login` | `gpt-oss-120b-medium`, `gemini-3-flash` (reader, researcher, panel judge) |

Add `-no-browser` to print the login URL instead of opening a browser. Other providers exist
(`-xai-login`, `-kimi-login`, …; see `./cli-proxy-api -h`).

**Run it** (keep it running; a systemd user service is the easy way on Linux):

```bash
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/cliproxy.service << EOF
[Unit]
Description=CLIProxyAPI (local)
After=network-online.target
[Service]
WorkingDirectory=$HOME/.local/bin/cliproxy
ExecStart=$HOME/.local/bin/cliproxy/cli-proxy-api -config $HOME/.local/bin/cliproxy/config.yaml
Restart=always
RestartSec=5
[Install]
WantedBy=default.target
EOF
systemctl --user daemon-reload && systemctl --user enable --now cliproxy
```

**Give hx the proxy key:** `HX_PROXY_KEY` = the key you put in `access.api-keys` — export it in
the shell rc, or put `HX_PROXY_KEY=...` in the project's `.env`.

**Check:** `curl -s -H "Authorization: Bearer $HX_PROXY_KEY" http://127.0.0.1:8317/v1/models`
lists the model ids of every subscription you logged into.

**Missing a subscription?** The hard rule of 5.2 still holds: judges need a model outside the
implementer's family. Without Codex, map the judge profiles (5.2 list) to, e.g., a Gemini model
from Antigravity; without Antigravity, map `reader`/`researcher` to a model you have and set
`panel_reviewer = []`. Use ids from the `/models` list above.

**Re-auth:** the proxy refreshes tokens by itself. If a run fails with `503 auth_unavailable`, or
`hx doctor` says a model is no longer offered, run that subscription's login command again.

> Using a consumer subscription through a proxy may conflict with the provider's terms of use;
> that choice and its risk are yours. Path B avoids it.

### 5.1 Endpoint and key (Path B)

Edit `.hx/config.toml` — change the existing tables instead of adding duplicates (a TOML table
may appear only once). The generated config points at the proxy of Path A (`base_url` defaults to
`http://127.0.0.1:8317/v1`, key variable `HX_PROXY_KEY`); replace it:

```toml
[llm]
base_url = "https://api.openai.com/v1"   # any OpenAI-compatible /v1 base
api_key_env = "OPENAI_API_KEY"           # NAME of the env var holding the key, never the key
```

The endpoint must speak **OpenAI Chat Completions with function calling** (`POST /chat/completions`
with `tools`) and should serve `GET /models`. Known shapes: OpenAI, OpenRouter
(`https://openrouter.ai/api/v1`), LiteLLM, vLLM, Ollama (`http://localhost:11434/v1`). Other keys:
`auth_header` (default `"bearer"`; set e.g. `"x-api-key"` to send the raw key in that header),
`extra_headers = { ... }`, `timeout_s` (default 300).

Provide the key as an environment variable, or in a `.env` file in the directory where you run
`hx` (it is loaded automatically and `hx init` already git-ignores it):

```bash
echo 'OPENAI_API_KEY=...' >> .env    # the user pastes the key; never echo it back
```

A keyless local server works, but `hx doctor` complains about an empty variable: set it to any
dummy value.

### 5.2 Models — every role must exist on your endpoint (Path B)

hx uses several agent profiles, and some of them ship with the author's model names. Map **all**
of them, or the run stops before spending anything with `o proxy não oferece nenhum modelo de …`
("the endpoint offers no model for …").

**Hard rule: judges must not use the implementer's model.** The reviewer and judge profiles refuse
the model family of `[models] main`; with `family_guard = "prefer"` (default) they accept another
model of the same family as a degraded fallback. Best: judges from a different provider family
(e.g. main Claude, judges GPT). One single model for every role makes the review phase fail with
`BLOCKED: agente 'reviewer' … sem modelo permitido` — `hx doctor` does not catch this. If the user
truly has one model, use `hx run --no-review` or set `[validation] review_enabled = false`.

Template (example ids for OpenRouter; replace with ids from `GET {base_url}/models`):

```toml
[models]
main = "anthropic/claude-sonnet-4.5"   # implementer
family_guard = "prefer"                # "strict" = judges must be another family, or fail
panel_reviewer = []                    # disables the extra panel judge (default is an author alias)

[models.profiles]
# builders: may share main's family
manager = "anthropic/claude-sonnet-4.5"
planner = "anthropic/claude-sonnet-4.5"
reader = "openai/gpt-5-mini"
researcher = "openai/gpt-5-mini"
# judges: must differ from main (another family is best)
reviewer = "openai/gpt-5-mini"
completion-judge = "openai/gpt-5-mini"
selection-judge = "openai/gpt-5-mini"
proposal-arbiter = "openai/gpt-5-mini"
security-reviewer = "openai/gpt-5-mini"
silent-failure-hunter = "openai/gpt-5-mini"
oracle = "openai/gpt-5-mini"
```

Single-provider example (OpenAI only): `main = "gpt-4.1"`, builders `"gpt-4.1"`, judges
`"gpt-4.1-mini"`, `family_guard = "prefer"`. Runs complete with a `model_family_degraded`
warning, which is expected.

Model ids hx cannot place in a family (custom names) block the judges: declare them, e.g.
`[model_families]` `"my-model*" = "openai"`. Known prefixes (claude, gpt/o1-o9/codex, gemini,
deepseek, qwen, llama, mistral, grok, kimi, glm) are recognized automatically, also after a
`provider/` prefix.

Optional: `[fallback_models]` `"model" = ["alt1", "alt2"]` (fallback chain on errors/quota; quote
ids that contain dots); `[model_effort]` `"gpt-5*" = "medium"` (sends `reasoning_effort`; values
`none|minimal|low|medium|high|xhigh`).

### 5.3 Budget and prices

`[budget] max_usd` (default 1.0 per run) only works for models with a known price; unknown models
count as $0. Declare prices in USD per 1M tokens:

```toml
[prices]
"anthropic/claude-sonnet-4.5" = { input = 3.0, output = 15.0 }
"openai/gpt-5-mini" = { input = 0.25, output = 2.0 }

[budget]
max_usd = 2.0
```

### 5.4 Validation gates (your tests)

Gates are the commands that prove the work is correct (tests, typecheck, lint). `hx init` fills
them from `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml` or `Makefile`; otherwise add
them, one table per gate:

```toml
[[validation.gates]]
name = "tests"
cmd = "npm test"
```

### 5.5 Sandbox and permissions

```toml
[operations]
backend = "local"        # "jail" if ai-jail is installed (Linux); "container" for Docker

[workers]
unsandboxed_shell = true # required with backend = "local"
```

`[permissions] mode` defaults to `"strict"`: the agent's shell may only run `git status/diff/add/commit`,
`ls`, `cat` and your gate commands; extend with `[tools] allow_commands = ["npm install", ...]`.
`"auto"` lets agents run commands without approval behind a deterministic safety floor — use it
only with `backend = "jail"`.

### 5.6 Global defaults (optional)

The same keys in `~/.hx/config.toml` (Windows: `%USERPROFILE%\.hx\config.toml`) apply to every
project; the project's `.hx/config.toml` wins key by key. Putting `[llm]` and `[models]` there
saves repeating them.

## 6. Verify

```bash
hx doctor
```

Healthy = **exit code 0 and no lines starting with `hx doctor:`**; it prints `proxy: OK` and `ok`
next to each probed model (it makes one tiny real call per model). Common failures:

| Message (pt-BR) | Meaning → fix |
|---|---|
| `a variável $X está vazia` | key env var not set → step 5.1 |
| `proxy inacessível em …` / `não respondeu` / `recusou a chave` | wrong `base_url` or key (the word "proxy"/"CLIProxy" just means "your endpoint") |
| `o proxy não oferece nenhum modelo de <perfil>` | that profile's model is not served → map it in `[models.profiles]` |
| `nenhum comando configurado para conferir o trabalho` | no gates → step 5.4 |
| `sem preço: …` | info only; models without `[prices]` |

`hx doctor --no-probe` skips the real model calls.

## 7. Use

```bash
hx run "Add a --verbose flag to the CLI and cover it with a test"
```

- Flow: IMPLEMENT → VALIDATE (gates) → REVIEW (judges), with automatic correction rounds.
- Prints a JSON summary; exit 0 = completed, 1 = failed, 2 = usage error, 75 = halted for resources.
- Trace and report: `.ai/runs/<runId>/` (`report.json`, `events.jsonl`).
- Useful flags: `--plan` (plan first), `--agentic` (manager delegating to sub-agents),
  `--budget-usd N`, `--no-review`, `--pr` (open a PR via `gh`), `--issue <n>`.
- Resume/control: `hx resume <runId>`, `hx pause <runId>`, `hx continue <runId>`,
  `hx run --revert <runId>`.
- Other: `hx check` (run the gates), `hx review --diff main..HEAD`, `hx observatorio` (terminal
  dashboard of runs), `hx onboard` (interactive interview that refines gates and project docs; costs
  LLM calls), `hx mcp` (MCP server for Claude Code).

Bare `hx` (and `hx --help`) opens an interactive terminal UI; there is no `help` command. In scripts
and agents always pass a subcommand and, without a TTY, redirect stdin: `hx run "…" </dev/null`.

On the first run in a repository with no commits, hx makes an initial commit `hx init` as
`hx <hx@localhost>` (set `[git] user_name` / `user_email` to change it).

## 8. Update

- **Binary:** repeat step 2 (or `npm install -g @robertof1lho/hx@latest`).
- **Project harness** (profiles/skills in `.hx/` and `.ai/`): `hx upgrade` refreshes from the
  installed binary without overwriting your edits; `hx upgrade vX.Y.Z` pulls that release's package,
  runs your gates and rolls back on failure. `--dry-run` previews.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `backend não declarado … ai-jail não está no PATH` | add `[operations] backend = "local"` (step 5.5) or install ai-jail |
| run ends `halted_llm_error` in `REVIEW` | judges have no allowed model → step 5.2 hard rule |
| `implementer_family_unknown` | add `[model_families]` for your model ids |
| budget never stops a run | add `[prices]` (step 5.3) |
| `cgroup: indisponível … sem teto` | harmless: no systemd user session, runs without memory/CPU cap |
| need a stack trace | `HX_DEBUG=1 hx …` |
| endpoint has no `/models` and the preflight blocks | `HX_NO_PREFLIGHT=1` |
