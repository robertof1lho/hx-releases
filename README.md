# hx

`hx` runs a coding task in your repository in three steps: it **implements** the change,
**validates** it with your own checks (tests, lint) and has a second model **review** it, fixing
and retrying until the work passes. It works with any OpenAI-compatible model endpoint.

This repository hosts the release binaries.

> **For coding agents:** follow the steps in order; each ends with a check. Ask the user only for
> the endpoint URL, the API key (never print it) and the two models to use.

## 1. Download

| System | Check | File |
|---|---|---|
| Linux, Intel/AMD | `uname -m` → `x86_64` | `hx-linux-x64` |
| Linux, ARM | `uname -m` → `aarch64` | `hx-linux-arm64` |
| Windows, Intel/AMD | `$env:PROCESSOR_ARCHITECTURE` → `AMD64` | `hx-windows-x64.exe` |

Other systems (macOS, Windows ARM) are not supported. The binary is self-contained: nothing else
to install to run it.

**Linux**

```bash
set -eu
V=$(curl -fsSL https://api.github.com/repos/robertof1lho/hx-releases/releases/latest \
  | sed -n 's/.*"tag_name": *"v\([^"]*\)".*/\1/p')
case "$(uname -m)" in x86_64|amd64) F=hx-linux-x64 ;; aarch64|arm64) F=hx-linux-arm64 ;;
  *) echo "unsupported"; exit 1 ;; esac
cd "$(mktemp -d)"
curl -fsSLO "https://github.com/robertof1lho/hx-releases/releases/download/v$V/$F"
curl -fsSLO "https://github.com/robertof1lho/hx-releases/releases/download/v$V/CHECKSUMS-$V.txt"
sha256sum -c --ignore-missing "CHECKSUMS-$V.txt"
mkdir -p ~/.local/bin && install -m 755 "$F" ~/.local/bin/hx
```

If `hx` is not found afterwards, add `~/.local/bin` to `PATH`.

**Windows (PowerShell)**

```powershell
$ErrorActionPreference = "Stop"
$v = (Invoke-RestMethod https://api.github.com/repos/robertof1lho/hx-releases/releases/latest).tag_name.TrimStart("v")
$base = "https://github.com/robertof1lho/hx-releases/releases/download/v$v"
$dir = "$env:USERPROFILE\.local\bin"; New-Item -ItemType Directory -Force $dir | Out-Null
Invoke-WebRequest "$base/hx-windows-x64.exe" -OutFile "$dir\hx.exe"
$want = ((Invoke-WebRequest "$base/CHECKSUMS-$v.txt").Content -split "`n" | Where-Object { $_ -match "hx-windows-x64.exe" }) -split "\s+" | Select-Object -First 1
if ((Get-FileHash "$dir\hx.exe" -Algorithm SHA256).Hash.ToLower() -ne $want) { throw "checksum mismatch" }
$p = [Environment]::GetEnvironmentVariable("Path", "User")
if ($p -notlike "*$dir*") { [Environment]::SetEnvironmentVariable("Path", "$p;$dir", "User") }
```

Open a new terminal. Install Git for Windows as well (hx uses its shell).

**Or with npm:** `npm install -g @robertof1lho/hx`

**Check:** `hx version` prints the version.

## 2. Set up your project

Inside your project's git repository:

```bash
hx init --no-onboard
```

This creates `.hx/config.toml` (the settings) plus a few support folders (`.ai/`, `docs/`).

## 3. Connect your models

Open `.hx/config.toml` and edit these tables (edit the existing ones; don't add duplicates):

```toml
[llm]
base_url = "https://your-endpoint/v1"   # any OpenAI-compatible endpoint
api_key_env = "MY_LLM_KEY"              # name of the environment variable that holds the key

[models]
main = "model-a"                        # writes the code
panel_reviewer = []

[models.profiles]                       # helper roles: use model-a
manager = "model-a"
planner = "model-a"
reader = "model-a"
researcher = "model-a"
                                        # reviewer roles: use model-b
reviewer = "model-b"
completion-judge = "model-b"
selection-judge = "model-b"
proposal-arbiter = "model-b"
security-reviewer = "model-b"
silent-failure-hunter = "model-b"
oracle = "model-b"
```

- **Two models are required.** The reviewer must be a different model from the one that writes
  the code — ideally from another vendor. With a single model, review cannot run (use
  `hx run --no-review`).
- Use model ids exactly as your endpoint lists them (`GET <base_url>/models`).
- hx recognizes the vendor from the id (`gpt…`, `claude…`, `gemini…`, `llama…`, `qwen…`,
  `mistral…`, `deepseek…`). For other ids, declare it:
  `[model_families]` `"my-model*" = "openai"`.
- The endpoint must support chat completions with tool calling.

Put the key in a `.env` file in the project (already git-ignored):

```
MY_LLM_KEY=...
```

**Set your checks.** If `hx init` did not detect your test command, add it:

```toml
[[validation.gates]]
name = "tests"
cmd = "npm test"
```

**Allow commands the agent needs** (by default it may only run git, `ls`, `cat` and your checks):

```toml
[tools]
allow_commands = ["npm install"]
```

## 4. Check

```bash
hx doctor
```

Ready when it exits with code 0 and prints no line starting with `hx doctor:`. Each such line says
what is missing (messages are in Portuguese): an empty key variable, an endpoint that does not
answer, a model the endpoint does not offer, or no checks configured.

## 5. Use

```bash
hx run "Add a --verbose flag and cover it with a test"
```

hx implements, runs your checks, reviews, and prints a summary. Details of each run are in
`.ai/runs/`.

| Command | What it does |
|---|---|
| `hx run "<task>"` | run a task (`--plan` to plan first, `--budget-usd N` to cap spend) |
| `hx resume <runId>` | resume an interrupted run |
| `hx check` | run your checks |
| `hx review --diff main..HEAD` | review existing changes |
| `hx observatorio` | dashboard of runs in the terminal |
| `hx` | interactive mode |

In scripts, always pass a command (plain `hx` opens the interactive mode).

## Update

Download the binary again (step 1) or `npm install -g @robertof1lho/hx@latest`. Then, in each
project, `hx upgrade` refreshes the project files without overwriting your edits.

## Troubleshooting

| Problem | Fix |
|---|---|
| `backend não declarado` | add `[operations]` with `backend = "local"` to `.hx/config.toml` |
| review stops with `sem modelo permitido` | the reviewer roles need a model different from `main` (step 3) |
| spending limit never triggers | declare prices: `[prices]` `"model-a" = { input = 3.0, output = 15.0 }` (USD per 1M tokens) |
