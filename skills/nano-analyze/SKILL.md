---
name: nano-analyze
description: "Run nano-analyzer, a minimal LLM-powered zero-day vulnerability scanner, against a file or directory of source code. Triggers on: /nano-analyze, \"scan for vulnerabilities\", \"run nano-analyzer\", \"security scan\", \"find zero-day bugs\", \"vulnerability scan\", \"scan this code\""
argument-hint: "path:<file-or-dir> [model:<model>] [parallel:<n>] [triage-threshold:<level>] [triage-rounds:<n>] [min-confidence:<float>] [project:<name>] [repo-dir:<path>] [verbose-triage]"
---

# nano-analyze

You run **nano-analyzer**, a minimal LLM-powered zero-day vulnerability scanner, against a file or directory of source code.

Nano-analyzer uses a three-stage LLM pipeline:
1. **Context generation** — a model writes a security briefing about each file (what it does, where untrusted data flows, which buffers exist)
2. **Vulnerability scan** — the same model, primed with that context, hunts for zero-day bugs function by function
3. **Skeptical triage** — each finding is challenged over multiple rounds by a reviewer that can grep the codebase to verify or refute defenses; an arbiter makes the final call

Results are saved as Markdown and JSON to `~/nano-analyzer-results/<timestamp>/` for human review.

---

## Phase 1: Parse Arguments

**Parse the arguments** provided by the user. Recognized arguments:

| Argument | Example | Effect |
|----------|---------|--------|
| `path:<file-or-dir>` | `path:./src` | **(Required)** File or directory to scan |
| `model:<name>` | `model:gpt-5.4` | LLM for all stages (default: `gpt-5.4-nano`) |
| `parallel:<n>` | `parallel:30` | Max concurrent scan calls (default: 50) |
| `triage-threshold:<level>` | `triage-threshold:high` | Triage findings at or above this severity: `critical`, `high`, `medium`, `low` (default: `medium`) |
| `triage-rounds:<n>` | `triage-rounds:3` | Triage rounds per finding (default: 5) |
| `min-confidence:<float>` | `min-confidence:0.7` | Only surface findings above this confidence 0.0–1.0 (default: 0.0, show all) |
| `project:<name>` | `project:openssl` | Project name used in triage prompts (default: directory name) |
| `repo-dir:<path>` | `repo-dir:./` | Repo root for grep lookups during triage |
| `verbose-triage` | `verbose-triage` | Show per-round triage progress |

**If no `path:` argument is provided**, ask the user what to scan using AskUserQuestion with up to 4 options:
- **Current directory** — scan all supported source files in `.`
- **Specific file** — ask for a file path
- **Specific directory** — ask for a directory path
- **Cancel** — exit without scanning

**API key requirements — check before running:**
- For OpenAI models (name has no `/`, e.g. `gpt-5.4-nano`): `OPENAI_API_KEY` must be set
- For OpenRouter models (name has `/`, e.g. `qwen/qwen3-32b`): `OPENROUTER_API_KEY` must be set

If the required key is missing, tell the user:
```
Set export OPENAI_API_KEY=sk-...    # for OpenAI models
# or
Set export OPENROUTER_API_KEY=sk-or-...  # for OpenRouter models
```

---

## Phase 2: Run the Scanner

Build the command from parsed arguments and run it:

```bash
python3 <SKILL_BASE_DIR>/scripts/scan.py <PATH> \
  [--model <MODEL>] \
  [--parallel <N>] \
  [--triage-threshold <LEVEL>] \
  [--triage-rounds <N>] \
  [--min-confidence <FLOAT>] \
  [--project <NAME>] \
  [--repo-dir <PATH>] \
  [--verbose-triage]
```

Where `<SKILL_BASE_DIR>` is the directory containing this SKILL.md file.

**Only include flags for arguments the user explicitly provided.** Omit flags for defaults — the scanner has sensible defaults built in.

**Example invocations:**

```bash
# Minimal — scan a directory with all defaults
python3 <SKILL_BASE_DIR>/scripts/scan.py ./src/

# Scan a single file
python3 <SKILL_BASE_DIR>/scripts/scan.py ./lib/parser.c --model gpt-5.4

# High-confidence findings only, with triage grep over the full repo
python3 <SKILL_BASE_DIR>/scripts/scan.py ./lib/ --repo-dir ./ --min-confidence 0.7

# Fast scan: fewer rounds, higher threshold, lower parallelism
python3 <SKILL_BASE_DIR>/scripts/scan.py ./src/ \
  --triage-threshold high \
  --triage-rounds 2 \
  --parallel 20
```

**While the scanner runs**, narrate progress to the user — the scanner emits live output showing files scanned, severity indicators (🔴🟠🟡🔵⬜), and triage verdicts as they complete.

---

## Phase 3: Present Results

After the scan completes, present a structured summary:

### Summary
- Files scanned, wall time
- Severity breakdown: 🔴 critical, 🟠 high, 🟡 medium, 🟢 clean
- Output directory path

### Findings that survived triage (VALID verdict)
List in descending confidence order:

```
🔥 95% [VVVVV] src/parser.c: Stack buffer overflow via unchecked len
   📄 ~/nano-analyzer-results/<timestamp>/findings/VULN-001_parser.c.md

✅ 70% [VVIVV] src/auth.c: NULL deref on failed session lookup
   📄 ~/nano-analyzer-results/<timestamp>/findings/VULN-002_auth.c.md
```

Confidence bar: 🔥 = 90%+, ✅ = 70–89%, 🤔 = 50–69%, ❓ = below 50%.

Confidence is computed as `(VALID rounds) / (total rounds)` including the arbiter round.

### Next steps
After the summary, offer:
- Explain or drill into a specific finding file
- Re-scan with adjusted parameters (more rounds for confidence, lower threshold for coverage)
- Scan a different path

---

## Error Handling

| Error | Response |
|-------|----------|
| API key not set | Tell user to set `OPENAI_API_KEY` or `OPENROUTER_API_KEY` as appropriate |
| Path not found | Confirm path relative to cwd; suggest `ls` to verify |
| No scannable files | Report the path; note supported extensions: `.c .h .cc .cpp .cxx .hpp .hxx .java .py .go .rs .js .ts .rb .swift .m .mm .cs .php .pl .sh` |
| Rate limit errors | Suggest reducing `--parallel`; scanner retries automatically |
| Scanner error | Show stderr output; suggest checking API key and network access |

---

## Notes

- **C/C++ bias.** Prompts, few-shot examples, and heuristics are tuned for C/C++ memory safety bugs. Other languages are scanned but with lower effectiveness.
- **Multi-round triage** (default: 5 rounds + arbiter) significantly reduces false positives. Reduce for speed, increase for confidence.
- **Use `--repo-dir`** when scanning a subdirectory so triage grep can search the full codebase to verify defenses.
- **OpenRouter support.** Any model accessible via OpenRouter can be used by setting `OPENROUTER_API_KEY` and using a `provider/model` name (e.g. `qwen/qwen3-32b`).
- Results persist in `~/nano-analyzer-results/<timestamp>/` after the scan; individual finding files include full triage reasoning chains.
