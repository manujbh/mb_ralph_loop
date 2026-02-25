# mb_ralph_loop Process Flow

This document traces the full execution path of the Ralph autonomous development loop, starting from `install.sh`.

---

## 1. Installation (`install.sh`)

**Entry point:** `./install.sh` or `./install.sh install`

### Step 1.1 – Dependency check (`install.sh:34–93`)

`check_dependencies()` validates required tools:

- `node` / `npx` — Claude Code CLI runtime
- `jq` — JSON parsing throughout the loop
- `git` — progress detection via git diff
- `timeout` (Linux) or `gtimeout` (macOS via `coreutils`) — execution timeout
- `tmux` — optional, for integrated monitoring

Missing dependencies print platform-specific install instructions and exit with code 1.

### Step 1.2 – Directory creation (`install.sh:96–105`)

`create_install_dirs()` creates:

```
~/.local/bin/     ← global command stubs (ralph, ralph-monitor, etc.)
~/.ralph/         ← runtime home for scripts and templates
~/.ralph/lib/     ← library scripts
~/.ralph/templates/  ← project templates
```

### Step 1.3 – Script installation (`install.sh:108–237`)

`install_scripts()` copies library files and generates thin command-stub wrappers in `~/.local/bin/` that delegate to `~/.ralph/*.sh`:

| Stub command       | Delegates to                     |
|--------------------|----------------------------------|
| `ralph`            | `~/.ralph/ralph_loop.sh`         |
| `ralph-monitor`    | `~/.ralph/ralph_monitor.sh`      |
| `ralph-setup`      | `~/.ralph/setup.sh`              |
| `ralph-import`     | `~/.ralph/ralph_import.sh`       |
| `ralph-migrate`    | `~/.ralph/migrate_to_ralph_folder.sh` |
| `ralph-enable`     | `~/.ralph/ralph_enable.sh`       |
| `ralph-enable-ci`  | `~/.ralph/ralph_enable_ci.sh`    |

`install_ralph_loop()` (`install.sh:224–237`) uses `sed` to patch path references in `ralph_loop.sh` before copying it to `~/.ralph/ralph_loop.sh`, ensuring scripts resolve library paths correctly when run globally.

### Step 1.4 – PATH check (`install.sh:254–269`)

If `~/.local/bin` is not in `$PATH`, the installer prints instructions for adding it to `~/.bashrc` / `~/.zshrc`.

---

## 2. Starting the Loop (`ralph_loop.sh` — top-level execution)

**Entry:** `ralph` command → stub → `~/.ralph/ralph_loop.sh "$@"`

### Step 2.1 – Library sourcing (`ralph_loop.sh:14–18`)

Four libraries are sourced immediately before any other logic:

```bash
source "$SCRIPT_DIR/lib/date_utils.sh"      # Cross-platform timestamps
source "$SCRIPT_DIR/lib/timeout_utils.sh"   # Cross-platform timeout
source "$SCRIPT_DIR/lib/response_analyzer.sh" # Output analysis + session mgmt
source "$SCRIPT_DIR/lib/circuit_breaker.sh"  # Stagnation detection
```

### Step 2.2 – Environment variable capture (`ralph_loop.sh:37–46`)

Before defaults are applied, the current environment values of key variables are saved into `_env_*` prefixed copies. This allows `load_ralphrc()` later to correctly give user-set environment variables higher precedence than `.ralphrc` file values.

### Step 2.3 – Default configuration (`ralph_loop.sh:49–98`)

Defaults are established for all configuration variables:

| Variable | Default | Purpose |
|---|---|---|
| `MAX_CALLS_PER_HOUR` | `100` | Rate limit |
| `CLAUDE_TIMEOUT_MINUTES` | `15` | Per-call timeout |
| `CLAUDE_OUTPUT_FORMAT` | `json` | Claude CLI output mode |
| `CLAUDE_ALLOWED_TOOLS` | `Write,Read,Edit,Bash(git *),Bash(npm *),Bash(pytest)` | Tool permissions |
| `CLAUDE_USE_CONTINUE` | `true` | Session continuity |
| `CLAUDE_SESSION_EXPIRY_HOURS` | `24` | Session TTL |

### Step 2.4 – CLI argument parsing (`ralph_loop.sh:1675–1781`)

Argument parsing runs before `main()`. Key flags set variables:

| Flag | Effect |
|---|---|
| `--monitor` | Sets `USE_TMUX=true` |
| `--live` | Sets `LIVE_OUTPUT=true` (streaming mode) |
| `--calls N` | Overrides `MAX_CALLS_PER_HOUR` |
| `--timeout N` | Overrides `CLAUDE_TIMEOUT_MINUTES` |
| `--output-format` | Validates and sets `CLAUDE_OUTPUT_FORMAT` |
| `--allowed-tools` | Validates via `validate_allowed_tools()` and sets `CLAUDE_ALLOWED_TOOLS` |
| `--no-continue` | Sets `CLAUDE_USE_CONTINUE=false` |
| `--reset-circuit` | Calls `reset_circuit_breaker()` and exits |
| `--auto-reset-circuit` | Sets `CB_AUTO_RESET=true` |

### Step 2.5 – tmux setup (if `--monitor`) (`ralph_loop.sh:191–298`)

`setup_tmux_session()` creates a tmux session with three panes:

```
┌─────────────────────┬──────────────────────┐
│                     │  Claude live output  │
│   Ralph loop        │  (tail -f live.log)  │
│   (left pane 0)     ├──────────────────────┤
│                     │  ralph-monitor       │
│                     │  (status dashboard)  │
└─────────────────────┴──────────────────────┘
```

The left pane re-runs `ralph --live` (plus forwarded flags) to avoid recursion while tmux handles the UI. After setup, the function attaches to the tmux session and exits — the loop itself runs inside the new session.

---

## 3. `main()` — Startup validation (`ralph_loop.sh:1413–1613`)

### Step 3.1 – Load `.ralphrc` (`ralph_loop.sh:1415–1418`, function at `101–156`)

`load_ralphrc()` sources `.ralphrc` from the project root if it exists. The function:

1. Sources the file, which may override defaults.
2. Maps `.ralphrc` aliases (`ALLOWED_TOOLS` → `CLAUDE_ALLOWED_TOOLS`, etc.).
3. Restores values from `_env_*` captures for any variables explicitly set in the environment — environment variables always win over `.ralphrc`.

### Step 3.2 – Migration check (`ralph_loop.sh:1426–1437`)

If `PROMPT.md` exists at the project root (old flat structure) but `.ralph/` does not exist, Ralph prints a migration notice and exits. Users must run `ralph-migrate` to upgrade to the `.ralph/` subfolder layout.

### Step 3.3 – Project validation (`ralph_loop.sh:1439–1462`)

If `.ralph/PROMPT.md` is missing, Ralph exits with actionable guidance (`ralph-enable`, `ralph-setup`, or `ralph-import`).

### Step 3.4 – Session tracking init (`ralph_loop.sh:1464–1465`)

`init_session_tracking()` (`ralph_loop.sh:881–928`) creates `.ralph/.ralph_session` if missing, containing a newly generated session ID plus timestamps. If the file exists but is corrupted JSON, it is recreated with a `corrupted_file_recovery` reset reason.

---

## 4. Main Loop — Per-iteration flow (`ralph_loop.sh:1469–1612`)

The loop runs `while true` and increments `loop_count` each iteration.

### Step 4.1 – Session heartbeat (`ralph_loop.sh:1473`)

`update_session_last_used()` updates the `last_used` timestamp in `.ralph/.ralph_session`, keeping a running record of loop activity.

### Step 4.2 – Call tracking init (`ralph_loop.sh:1476`, function at `301–325`)

`init_call_tracking()`:

1. Reads the current hour from `date +%Y%m%d%H`.
2. Compares against `.ralph/.last_reset`. If the hour changed, resets `.ralph/.call_count` to `0`.
3. Initializes `.ralph/.exit_signals` JSON file if missing: `{"test_only_loops": [], "done_signals": [], "completion_indicators": []}`.
4. Calls `init_circuit_breaker()` (see §5).

### Step 4.3 – Circuit breaker pre-check (`ralph_loop.sh:1481–1486`)

`should_halt_execution()` (`circuit_breaker.sh:426–454`) reads the current circuit state. If `OPEN`, it prints a diagnostic summary with remediation steps, then returns 0 (signal to halt). The main loop calls `reset_session("circuit_breaker_open")` and `break`s.

### Step 4.4 – Rate limit check (`ralph_loop.sh:1489–1492`, functions at `370–423`)

`can_make_call()` reads `.ralph/.call_count` and returns 1 if the count equals or exceeds `MAX_CALLS_PER_HOUR`. If rate-limited, `wait_for_reset()` displays a countdown timer until the next hour boundary, then resets the counter and `continue`s the loop.

### Step 4.5 – Graceful exit check (`ralph_loop.sh:1495–1546`, function at `426–520`)

`should_exit_gracefully()` evaluates exit conditions **in priority order**:

| Priority | Condition | Threshold | Exit reason |
|---|---|---|---|
| 0 | Permission denials detected | `has_permission_denials == true` in `.response_analysis` | `permission_denied` |
| 1 | Too many test-only loops | `test_only_loops >= 3` | `test_saturation` |
| 2 | Repeated "done" signals | `done_signals >= 2` | `completion_signals` |
| 3 | Safety circuit breaker | `completion_indicators >= 5` | `safety_circuit_breaker` |
| 4 | Strong completion + EXIT_SIGNAL | `completion_indicators >= 2` **AND** `exit_signal == true` | `project_complete` |
| 5 | fix_plan.md all done | All checkboxes checked | `plan_complete` |

If `permission_denied`, Ralph prints a help panel showing current `ALLOWED_TOOLS` and exits. For other reasons, a success summary is printed and the loop breaks.

### Step 4.6 – Status update (`ralph_loop.sh:1549–1550`)

`update_status()` writes `.ralph/status.json` with the current loop count, call count, and `"running"` status. This file is consumed by `ralph-monitor`.

### Step 4.7 – Claude Code execution (`ralph_loop.sh:1552–1609`)

Calls `execute_claude_code "$loop_count"` (see §6). Exit codes from this function drive different outcomes:

| Exit code | Meaning | Action |
|---|---|---|
| `0` | Success | Update status → `success`, sleep 5s, continue |
| `3` | Circuit breaker opened | Reset session, break loop |
| `2` | Claude API 5-hour limit | Prompt user: wait 60min or exit |
| other | General failure | Log error, `sleep 30`, continue |

---

## 5. Circuit Breaker (`lib/circuit_breaker.sh`)

The circuit breaker implements Michael Nygard's "Release It!" pattern to prevent runaway loops.

### States

```
CLOSED ──(no progress × 2)──► HALF_OPEN ──(no progress × 3)──► OPEN
   ▲                               │                              │
   └──────── progress ─────────────┘                              │
                                                                   │
   ◄── cooldown (30min) or CB_AUTO_RESET=true ────────────────────┘
```

### `init_circuit_breaker()` (`circuit_breaker.sh:36–126`)

Called at the start of every iteration via `init_call_tracking()`. Validates or creates `.ralph/.circuit_breaker_state`. If state is `OPEN`, checks for auto-recovery:

- **`CB_AUTO_RESET=true`**: Immediately resets to `CLOSED`.
- **Cooldown timer**: Reads `opened_at` from state file, computes elapsed minutes via `parse_iso_to_epoch()` (`lib/date_utils.sh:53–97`). If `>= CB_COOLDOWN_MINUTES` (default 30), transitions to `HALF_OPEN`.

### `record_loop_result()` (`circuit_breaker.sh:150–325`)

Called after each successful Claude execution. Determines progress from three sources:

1. `files_changed > 0` — git tracked changes
2. `has_completion_signal == true` — `STATUS: COMPLETE` in response analysis
3. `ralph_files_modified > 0` — files reported in RALPH_STATUS block

State transitions after recording:

- `CLOSED` → `HALF_OPEN` if `consecutive_no_progress >= 2`
- `CLOSED` → `OPEN` if no progress for `CB_NO_PROGRESS_THRESHOLD` (3) loops, or same error for `CB_SAME_ERROR_THRESHOLD` (5) loops, or permission denials for `CB_PERMISSION_DENIAL_THRESHOLD` (2) loops
- `HALF_OPEN` → `CLOSED` on progress
- `HALF_OPEN` → `OPEN` if no progress for `CB_NO_PROGRESS_THRESHOLD` loops

When the circuit opens, `record_loop_result()` returns exit code `1`, which `execute_claude_code()` maps to its own return code `3`.

---

## 6. Claude Code Execution (`execute_claude_code()`, `ralph_loop.sh:1017–1396`)

### Step 6.1 – Git snapshot (`ralph_loop.sh:1026–1030`)

Records current `git rev-parse HEAD` into `.ralph/.loop_start_sha` to enable commit-aware progress detection later.

### Step 6.2 – Loop context build (`ralph_loop.sh:1037–1043`)

`build_loop_context()` (`ralph_loop.sh:603–638`) assembles a short context string injected as `--append-system-prompt`:

- Current loop number
- Count of remaining tasks from `fix_plan.md`
- Circuit breaker state (if not CLOSED)
- Previous loop work summary (from `.response_analysis`, truncated to 200 chars)

Total context is capped at 500 characters.

### Step 6.3 – Session resolution (`ralph_loop.sh:1047–1049`)

`init_claude_session()` (`ralph_loop.sh:699–733`):

1. Reads `.ralph/.claude_session_id`.
2. Calls `get_session_file_age_hours()` using platform-aware `stat` or `date -r`.
3. If age >= `CLAUDE_SESSION_EXPIRY_HOURS` (default 24h), deletes the file and returns empty string.
4. Returns the session ID for use with `--resume`.

### Step 6.4 – Command construction (`ralph_loop.sh:1060–1069`, function at `955–1014`)

`build_claude_command()` populates a global array `CLAUDE_CMD_ARGS`:

```
["claude", "--output-format", "json",
 "--allowedTools", "Write", "Read", "Edit", "Bash(git *)", ...,
 "--resume", "<session_id>",          # if session exists
 "--append-system-prompt", "<ctx>",   # loop context
 "-p", "<prompt file contents>"]
```

Using an array (not a string) prevents shell injection from prompt content or tool patterns. The `-p` flag feeds prompt content directly; `--prompt-file` does not exist in the Claude CLI.

### Step 6.5 – Execution (two modes)

#### Background mode (default) (`ralph_loop.sh:1206–1285`)

```bash
portable_timeout ${timeout_seconds}s "${CLAUDE_CMD_ARGS[@]}" \
    < /dev/null > "$output_file" 2>&1 &
```

`< /dev/null` prevents the Claude CLI from blocking on stdin when backgrounded (would cause SIGTTIN). The loop monitors `$!` with `kill -0`, updating a spinner and `.ralph/progress.json` every 10 seconds.

#### Live mode (`ralph_loop.sh:1097–1205`)

Live mode replaces `--output-format json` with `stream-json` and appends `--verbose --include-partial-messages`. The pipeline is:

```bash
portable_timeout Xs stdbuf -oL "${LIVE_CMD_ARGS[@]}" < /dev/null 2>&1 \
    | stdbuf -oL tee "$output_file" \
    | stdbuf -oL jq --unbuffered -j "$jq_filter" \
    | tee "$LIVE_LOG_FILE"
```

After streaming completes, the result-type message is extracted from `$output_file` and written back to the same file so downstream session saving and response analysis receive standard JSON.

`portable_timeout()` (`lib/timeout_utils.sh:99–133`) wraps `timeout` (Linux) or `gtimeout` (macOS) with cached detection.

### Step 6.6 – Post-execution (success path) (`ralph_loop.sh:1288–1380`)

On exit code 0:

1. **Increment call counter** — writes to `.ralph/.call_count`.
2. **Save session** — `save_claude_session()` (`ralph_loop.sh:736–747`) extracts `sessionId` from the JSON output via `jq` and writes it to `.ralph/.claude_session_id`.
3. **Analyze response** — calls `analyze_response()` (see §7).
4. **Update exit signals** — calls `update_exit_signals()` (see §7.3).
5. **Log analysis summary** — `log_analysis_summary()` prints a formatted panel with exit signal, confidence, test-only flag, files changed, and work summary.
6. **Detect file changes** — checks git diff between `loop_start_sha` and current HEAD. Counts unique changed files across committed and uncommitted changes.
7. **Two-stage error detection** (`ralph_loop.sh:1354–1370`):
   - Stage 1: `grep -v '"[^"]*error[^"]*":'` — strips JSON field lines containing "error" (e.g., `"is_error": false`) to eliminate false positives.
   - Stage 2: `grep -qE '(^Error:|^ERROR:|^error:|\]: error|...)'` — matches actual error message patterns.
8. **Record circuit breaker result** — `record_loop_result()` with files changed, error flag, and output length.

---

## 7. Response Analysis (`lib/response_analyzer.sh`)

`analyze_response()` (`response_analyzer.sh:304–640`) is the central intelligence for exit signal detection.

### Step 7.1 – Format detection (`response_analyzer.sh:32–54`)

`detect_output_format()` checks the first character of the output file. If `{` or `[` and valid JSON per `jq`, returns `"json"`. Otherwise returns `"text"`.

### Step 7.2 – JSON parsing (`response_analyzer.sh:62–301`)

`parse_json_response()` handles three JSON shapes from the Claude CLI:

| Format | Indicator | Notes |
|---|---|---|
| **Flat** | No `result` field | Direct fields: `status`, `exit_signal`, etc. |
| **Claude CLI object** | Has `result` field | `result` is text; `sessionId` at top level; metadata nested |
| **Claude CLI array** | Top-level `[...]` | Array of typed messages; `type == "result"` is the final entry |

For the array format, the function extracts the result object and merges `sessionId` from the init message as a fallback.

**EXIT_SIGNAL extraction logic** (`response_analyzer.sh:127–159`):

If `exit_signal` is still `false` after checking top-level fields but a `result` text field exists, the function searches for `---RALPH_STATUS---` embedded in that text and parses `EXIT_SIGNAL:` and `STATUS:` lines directly. If `EXIT_SIGNAL: false` is explicit, it is respected over any `STATUS: COMPLETE` — this prevents premature exits when Claude marks a phase complete while indicating more work remains.

### Step 7.3 – Exit signals update (`response_analyzer.sh:643–694`)

`update_exit_signals()` maintains a rolling window (last 5 entries) of three arrays in `.ralph/.exit_signals`:

- **`test_only_loops`** — loop numbers where Claude only ran tests, no implementation.
- **`done_signals`** — loop numbers with any completion signal detected.
- **`completion_indicators`** — loop numbers where Claude's `EXIT_SIGNAL == true` **explicitly**. This is the gating field for the `project_complete` exit condition.

### Step 7.4 – Text fallback (`response_analyzer.sh:444–639`)

If JSON parsing fails or the format is `text`, heuristic analysis runs:

1. **Structured block** — looks for `---RALPH_STATUS---` marker and parses `STATUS:` and `EXIT_SIGNAL:` lines.
2. **Keyword scan** — scans for completion keywords: `done`, `complete`, `finished`, `all tasks complete`, etc. Each match adds 10 to confidence score.
3. **Test-only detection** — counts test-command vs implementation-keyword occurrences; flags loop as test-only if only testing happened.
4. **Error detection** — same two-stage filtering as `ralph_loop.sh`.
5. **No-work patterns** — checks for `nothing to do`, `no changes`, `already implemented`.
6. **Git file changes** — same commit-aware logic as the JSON path.
7. **Output length trend** — compares current output size to `.ralph/.last_output_length`; a drop below 50% adds 10 to confidence.
8. **Heuristic exit signal** — if `confidence_score >= 40` and no explicit EXIT_SIGNAL was found in RALPH_STATUS, sets `exit_signal=true`.

---

## 8. Date Utilities (`lib/date_utils.sh`)

All timestamp operations use capability-based detection rather than OS-name checks, so macOS with Homebrew coreutils gets GNU behaviour automatically.

| Function | Used by | Purpose |
|---|---|---|
| `get_iso_timestamp()` | Status files, session files, circuit breaker | ISO 8601 with seconds |
| `get_next_hour_time()` | `update_status()` | Next rate-limit reset display |
| `get_epoch_seconds()` | Session expiry checks | Current Unix time |
| `parse_iso_to_epoch()` | `init_circuit_breaker()` cooldown timer | ISO → epoch conversion |

`parse_iso_to_epoch()` (`date_utils.sh:53–97`) tries GNU `date -d`, then BSD `date -j`, then manual regex extraction of date components, and finally falls back to current epoch.

---

## 9. Session Lifecycle Summary

Session state flows across two files:

| File | Contents | Purpose |
|---|---|---|
| `.ralph/.ralph_session` | `{session_id, created_at, last_used, reset_at, reset_reason}` | Ralph-level lifecycle tracking |
| `.ralph/.claude_session_id` | Raw session UUID string | Passed to Claude CLI `--resume` flag |

Session resets are triggered by:

- `reset_session("circuit_breaker_open")` — circuit opened
- `reset_session("circuit_breaker_trip")` — circuit opened mid-execution
- `reset_session("permission_denied")` — permission denial exit
- `reset_session("project_complete")` — successful completion
- `reset_session("manual_interrupt")` — SIGINT/SIGTERM via `cleanup()` trap (`ralph_loop.sh:1399–1407`)

Each reset also clears `.ralph/.exit_signals` and `.ralph/.response_analysis` to prevent stale state from leaking into the next session (Issue #91 fix, `ralph_loop.sh:803–809`).

Session history is appended to `.ralph/.ralph_session_history` (last 50 transitions) via `log_session_transition()`.

---

## 10. End-to-End Flow Summary

```
./install.sh
  ├── check_dependencies()          [install.sh:34]
  ├── create_install_dirs()         [install.sh:96]
  ├── install_scripts()             [install.sh:108]  → ~/.local/bin/ralph stubs
  ├── install_ralph_loop()          [install.sh:224]  → ~/.ralph/ralph_loop.sh (patched)
  └── install_setup()               [install.sh:239]  → ~/.ralph/setup.sh

ralph [--monitor] [flags]
  └── ralph_loop.sh
        ├── source lib/date_utils.sh, timeout_utils.sh,
        │         response_analyzer.sh, circuit_breaker.sh  [ralph_loop.sh:14]
        ├── capture _env_* variables                          [ralph_loop.sh:37]
        ├── set defaults                                       [ralph_loop.sh:49]
        ├── parse CLI args                                     [ralph_loop.sh:1675]
        ├── [--monitor] setup_tmux_session() → exit          [ralph_loop.sh:191]
        └── main()                                             [ralph_loop.sh:1413]
              ├── load_ralphrc()                               [ralph_loop.sh:1415]
              ├── validate project structure                   [ralph_loop.sh:1426]
              ├── init_session_tracking()                      [ralph_loop.sh:1464]
              └── while true:
                    ├── update_session_last_used()             [ralph_loop.sh:1473]
                    ├── init_call_tracking()                   [ralph_loop.sh:1476]
                    │     └── init_circuit_breaker()           [circuit_breaker.sh:36]
                    ├── should_halt_execution()                [circuit_breaker.sh:426]
                    ├── can_make_call() / wait_for_reset()     [ralph_loop.sh:1489]
                    ├── should_exit_gracefully()               [ralph_loop.sh:1495]
                    │     ├── permission_denied check
                    │     ├── test_saturation check
                    │     ├── completion_signals check
                    │     ├── safety_circuit_breaker check
                    │     ├── project_complete check (EXIT_SIGNAL gate)
                    │     └── plan_complete check (fix_plan.md)
                    ├── update_status()                        [ralph_loop.sh:1550]
                    └── execute_claude_code()                  [ralph_loop.sh:1017]
                          ├── record git HEAD → .loop_start_sha
                          ├── build_loop_context()             [ralph_loop.sh:603]
                          ├── init_claude_session()            [ralph_loop.sh:699]
                          ├── build_claude_command()           [ralph_loop.sh:955]
                          │     → CLAUDE_CMD_ARGS array
                          ├── [live] streaming pipeline        [ralph_loop.sh:1105]
                          ├── [background] portable_timeout()  [timeout_utils.sh:99]
                          └── on success:
                                ├── save_claude_session()      [ralph_loop.sh:736]
                                ├── analyze_response()         [response_analyzer.sh:304]
                                │     ├── detect_output_format()
                                │     ├── parse_json_response() (JSON path)
                                │     │     └── extract EXIT_SIGNAL from RALPH_STATUS
                                │     └── text heuristics (fallback)
                                ├── update_exit_signals()      [response_analyzer.sh:643]
                                ├── log_analysis_summary()     [response_analyzer.sh:697]
                                ├── detect file changes (git)
                                ├── two-stage error detection
                                └── record_loop_result()       [circuit_breaker.sh:150]
```

---

## 11. CI/CD Pipeline (`.github/workflows/`)

Four GitHub Actions workflows govern automated testing, code review, and on-demand Claude assistance. They run entirely outside the Ralph loop itself; Ralph only executes locally. The workflows are triggered by GitHub events (pushes, PRs, issue comments) and use GitHub-hosted `ubuntu-latest` runners.

---

### 11.1 — `test.yml` — Main Test Suite

**Trigger:** `push` to `main` or `develop`; `pull_request` targeting `main`.

Two sequential jobs run every time code changes reach those branches.

#### Job 1: `test`

| Step | Action |
|---|---|
| Checkout | `actions/checkout@v3` |
| Node.js 18 | `actions/setup-node@v3` |
| Install deps | `npm install` + `sudo apt-get install -y jq` |
| Unit tests | `npm run test:unit` (must pass — no `|| true`) |
| Integration tests | `npm run test:integration || true` (non-blocking) |
| E2E tests | `npm run test:e2e || true` (non-blocking) |
| Step summary | Appends "✅ Unit tests passed" to `$GITHUB_STEP_SUMMARY` |

Unit tests are the enforced gate — integration and E2E failures are logged but do not fail the job.

#### Job 2: `coverage` (runs after `test`)

Builds **kcov v42** from source (the Ubuntu package is too old), then runs two kcov instrumentation passes:

1. `kcov coverage/cli-parsing` over `test_cli_parsing.bats`
2. `kcov coverage/all-unit` over all `tests/unit/` files

Coverage is extracted from `coverage/all-unit/kcov-merged/coverage.json`. The enforcement threshold is set to `COVERAGE_THRESHOLD: 0` (disabled) because kcov cannot instrument subprocesses spawned by bats — the coverage number is informational only. The real quality gate is 100% unit test pass rate.

Artifacts are uploaded (`coverage-report`, 7-day retention) and optionally pushed to Codecov (`continue-on-error: true`).

---

### 11.2 — `claude.yml` — On-Demand Claude Code Assistant

**Trigger:** Any of:
- Issue comment containing `@claude`
- PR review comment containing `@claude`
- PR review body containing `@claude`
- Issue opened/assigned with `@claude` in title or body

**Concurrency:** One active run per issue or PR number; in-progress runs are cancelled when a new trigger arrives for the same thread.

The job uses `anthropics/claude-code-action@v1` with a `CLAUDE_CODE_OAUTH_TOKEN` secret. Claude reads the comment/issue body that triggered it and performs the requested action (e.g., answer a question, draft code, update PR description).

Permissions are minimally scoped: `contents: read`, `pull-requests: read`, `issues: read`, `id-token: write`, `actions: read`.

---

### 11.3 — `claude-code-review.yml` — Automated PR Code Review

**Trigger:** `pull_request_target` on `opened` or `synchronize`. Uses `pull_request_target` (not `pull_request`) so the workflow runs with base-repo secrets even for fork PRs — the workflow only reads PR code, it does not execute it.

**Skipped for:** Markdown-only, `.github/**`, `.gitignore`, or `pyproject.toml` changes.

**Size gate:** Both the checkout and review steps are skipped unless the PR has **5+ changed files OR 20+ total lines changed** (additions + deletions). This prevents Claude from reviewing trivial single-line fixes.

**Concurrency:** One active review per PR; in-progress reviews are cancelled on new pushes.

The review step uses `anthropics/claude-code-action@v1` with:

- `CLAUDE_CODE_OAUTH_TOKEN` and explicit `GITHUB_TOKEN` (required for `pull_request_target` — OIDC does not work there)
- A prompt covering code quality, bugs, performance, security, and test coverage
- Instructions to be consistent with prior reviews and avoid repeating already-addressed feedback
- A reference to `CLAUDE.md` for project style conventions
- Allowed tools restricted to read-only `gh` subcommands plus `gh pr comment` for posting the review:
  ```
  Bash(gh issue view:*), Bash(gh search:*), Bash(gh issue list:*),
  Bash(gh pr comment:*), Bash(gh pr diff:*), Bash(gh pr view:*), Bash(gh pr list:*)
  ```

Claude posts its review as a PR comment via `gh pr comment`.

---

### 11.4 — `opencode-review.yml` — OpenCode PR Review (GLM-4.7)

**Trigger:** `pull_request` on `opened` or `synchronize`. Uses `pull_request` (not `pull_request_target`) because the `anomalyco/opencode` action does not support `pull_request_target`. This means it only runs for PRs from branches within this repo, not forks; fork PRs fall back to `claude-code-review.yml`.

**Skipped for:** Markdown-only, `.github/workflows/opencode-review.yml`, `.gitignore`, or `pyproject.toml` changes. The workflow file itself is excluded to prevent self-triggering loops.

**Size gate:** Same threshold as the Claude review — 5+ files or 20+ lines.

**Job timeout:** 10 minutes (prevents hanging runners).

A credential-clearing step runs before checkout to remove all GitHub auth headers and credential helpers from both global and local git config, avoiding token conflicts between the `actions/checkout` injected credentials and the OpenCode action's own auth.

The review uses `anomalyco/opencode/github@latest` with:

- Model: `zai-coding-plan/glm-4.7` (ZhipuAI GLM-4.7)
- `ZHIPU_API_KEY` secret for the ZhipuAI API
- `GITHUB_TOKEN` for posting PR comments
- Same review criteria as the Claude workflow (code quality, bugs, performance, security, test coverage)
- An explicit instruction to post **exactly one** `gh pr comment` then stop

---

### 11.5 — Workflow Interaction Summary

```
GitHub Event
│
├── push → main / develop
│     └── test.yml
│           ├── job: test          (unit gate — must pass)
│           └── job: coverage      (informational — kcov)
│
├── pull_request → main
│     ├── test.yml (same jobs as above)
│     └── opencode-review.yml     (repo branches only, size-gated)
│
├── pull_request_target → main (fork or branch PRs)
│     └── claude-code-review.yml  (fork-safe, size-gated)
│
└── issue_comment / PR review comment / issues (with @claude)
      └── claude.yml              (on-demand assistant)
```

The two AI review workflows (`claude-code-review.yml` and `opencode-review.yml`) are complementary: Claude handles fork PRs via `pull_request_target`, while OpenCode handles same-repo branch PRs via `pull_request`. Both apply the same size gate and post a single comment to the PR.
