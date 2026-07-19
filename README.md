# AI-Stack-Trace-Investigator
# AI Stack Trace Investigator

LLM-assisted **root-cause analysis for Python stack traces**. It parses a
traceback, pulls in the surrounding source, ranks the likely causes with
calibrated probabilities, **asks for the evidence it's missing instead of
guessing**, and — when the failure is self-contained — **reproduces it in an
isolated Docker sandbox**.

Most AI debugging helpers do one-shot analysis and answer confidently even when
they're missing the one fact that matters. This tool is built around the
opposite instinct: *reason probabilistically, and say what you'd need to be
sure.*

---

## Why it's different

- **Ranked causes, not a single guess.** Debugging is probabilistic; the output
  is a ranked list with probabilities, not one confident (and often wrong) answer.
- **Asks for missing evidence.** When a diagnosis hinges on a runtime value, a
  dependency version, or the triggering input, it requests that explicitly.
- **Reproduces failures safely.** Best-effort reproduction runs in a locked-down
  Docker container (no network, non-root, read-only mount, memory/CPU caps,
  hard timeout) and degrades gracefully when the environment can't be rebuilt.
- **Every stage is independently useful and testable** — parse, context,
  rank, evidence, reproduce.

---

## Demo

Given this traceback (`examples/demo_traceback.txt`):

```
Traceback (most recent call last):
  File "app.py", line 15, in <module>
    start_server()
  File "app.py", line 11, in start_server
    port = config["port"]
KeyError: 'port'
```

Run:

```bash
investigate -f examples/demo_traceback.txt --project-root . --repro app.py
```

Output:

```
====================================================================
  AI STACK TRACE INVESTIGATOR
====================================================================
Exception : KeyError: 'port'
Origin    : app.py:11 in start_server()

LIKELY CAUSES (ranked)
--------------------------------------------------------------------
1. ●●● [  75%] Missing 'port' key in config
     load_config() returns a dict without 'port', so config['port']
     raises KeyError.
     Fix: Use config.get('port', DEFAULT_PORT) or ensure the key is set.

2. ●○○ [  25%] Config source not loaded
     The config may be read before env/file population.

MISSING EVIDENCE (would sharpen the diagnosis)
--------------------------------------------------------------------
• The full dict returned by load_config()
    why : To confirm whether 'port' is absent vs. mistyped
    how : Print the dict or paste the config file/env

SANDBOX REPRODUCTION
--------------------------------------------------------------------
Status: ✓ reproduced
        Reproduced KeyError.
====================================================================
```

---

## Install

```bash
git clone https://github.com/vivekselva91/ai-stack-trace-investigator.git
cd ai-stack-trace-investigator
pip install -e ".[dev]"

cp .env.example .env       # then add your ANTHROPIC_API_KEY
```

Reproduction requires Docker on the host. Everything else works without it.

---

## Usage

```bash
# Analyze a traceback piped straight from a failing command
python failing_app.py 2>&1 | investigate --project-root .

# Analyze a saved traceback and attempt Docker reproduction
investigate -f crash.txt --project-root . --repro app.py

# Parse-only, no API key required (great for CI or offline)
investigate -f crash.txt --no-llm --json
```

Key flags:

| flag                    | purpose                                                    |
|-------------------------|------------------------------------------------------------|
| `-f, --file`            | read the traceback from a file (else stdin)                |
| `--project-root`        | repo root; resolves source and mounts the sandbox          |
| `--repro ENTRYPOINT`    | script to run in Docker to attempt reproduction            |
| `--repro-requirements`  | `requirements.txt` to install before reproduction          |
| `--no-llm`              | skip the model; parse + context only                       |
| `--model`               | override the Claude model                                  |
| `--json`                | machine-readable output                                    |

---

## How it works

```
raw traceback
     │
     ▼
┌──────────┐   ┌───────────────┐   ┌──────────────────────┐   ┌────────────────┐
│ 1. parse │──▶│ 2. source     │──▶│ 3/4. rank causes &   │──▶│ 5. reproduce   │
│  frames  │   │    context    │   │   request evidence   │   │  in Docker     │
└──────────┘   └───────────────┘   └──────────────────────┘   └────────────────┘
 deterministic   reads your repo      one structured LLM call    isolated, capped
```

1. **Parse** — turn the raw traceback into structured frames + exception. Handles
   chained exceptions, log-line prefixes, and library vs. project frames.
2. **Context** — pull a window of real source around each frame from your repo.
3. **Rank + evidence** — a single Claude call returns ranked hypotheses (with
   normalized probabilities) *and* the missing evidence that would sharpen them.
4. **Reproduce** — best-effort run of your entrypoint in a hardened container;
   reports whether the same exception type came back.

---

## Scope (v1)

Focused on purpose:

- **Python only.** Reliable traceback parsing and reproduction beats shallow
  multi-language support.
- **Reproduction is best-effort** and works best on self-contained,
  deterministic failures. Bugs that need external databases, services, or
  specific runtime state will report `skipped` / `different failure` rather than
  pretend. This is by design, not a gap to paper over.

On the roadmap: more languages, pytest-native integration, and richer repro
harnesses (fixtures, inputs).

---

## Benchmarks

Diagnosis quality and reproduction are measured against a set of **known bugs
with known causes** in [`benchmarks/`](benchmarks/):

```bash
python -m benchmarks.run --repro-only   # parsing + reproduction (no API key)
python -m benchmarks.run                # full run incl. LLM diagnosis
```

Example output:

```
case                         diagnosis    reproduction
--------------------------------------------------------
case_01_null_config          ✓            ✓
case_02_index_error          ✓            ✓
--------------------------------------------------------
Diagnosis    : 2/2
Reproduction : 2/2
```

Adding a case is three files (`bug.py`, `traceback.txt`, `expected.json`) — see
[benchmarks/README.md](benchmarks/README.md).

---

## Security

Reproduction executes code, so the sandbox treats every target as
hostile-adjacent: `--network none`, non-root user, read-only project mount,
`no-new-privileges`, memory/CPU/pid caps, and a hard timeout. Don't disable
these when pointing the tool at code you don't trust.

---

## Development

```bash
pip install -e ".[dev]"
pytest          # unit tests (no API key needed)
ruff check src tests
```

CI runs the parser/ranker tests and the reproduction benchmark on every push.

---

