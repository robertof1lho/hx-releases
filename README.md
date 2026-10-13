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

```bash
hx setup
```

It asks for the endpoint URL (any OpenAI-compatible endpoint with tool calling), the API key
(saved to the project's `.env`, which is git-ignored; never shown), and two models picked from
the endpoint's list:

- **main** — writes the code;
- **judge** — reviews it. It must be a different model, ideally from another vendor; setup
  suggests one.

On Linux x64 it also offers a sandbox for the commands the agent runs; accept it (it needs the
`bubblewrap` package, and setup tells you how to install it if missing).

Without prompts (scripts, agents):

```bash
hx setup --llm-url https://your-endpoint/v1 --llm-key-env MY_LLM_KEY \
  --model <main-model> --judge-model <judge-model> --sandbox --yes
```

Then put the key in the project's `.env` as `MY_LLM_KEY=...`. Add `--global` to save these
settings for every project. Use model ids exactly as `GET <base_url>/models` lists them.

**Set your checks.** If `hx init` did not detect your test command, add it to `.hx/config.toml`:

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

`hx setup` ends by running the check; run it again any time:

```bash
hx doctor
```

Ready when it exits with code 0 and prints no line starting with `hx doctor:`. Each such line says
what is missing and how to fix it (messages are in Portuguese).

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
| `hx help` | all commands |
| `hx` | interactive mode |

In scripts, always pass a command (plain `hx` opens the interactive mode).

## Update

Download the binary again (step 1) or `npm install -g @robertof1lho/hx@latest`. Then, in each
project, `hx upgrade` refreshes the project files without overwriting your edits.

## Troubleshooting

| Problem | Fix |
|---|---|
| `backend não declarado` | run `hx setup --sandbox`, or add `[operations]` with `backend = "local"` to `.hx/config.toml` |
| review stops with `sem modelo permitido` | run `hx setup` again with a different `--judge-model` |
| spending limit never triggers | declare prices: `[prices]` `"model-a" = { input = 3.0, output = 15.0 }` (USD per 1M tokens) |
