# 🌐 Local-First Development with Remote Execution Guidelines

This guideline applies to the "Local-First Development + Remote SSH Triggering" architecture. The AI Agent runs natively on the local filesystem and repository, while remotely syncing code via Git branches and executing workloads on the remote host over SSH.

---

## 1. Architecture & Core Responsibilities

### Core Responsibilities
- **Local Workspace**: All file edits, code reading, searches (`grep`, `rg`), and standard Git operations (`git add`, `git commit`, `git status`, `git checkout`, `git push`) are performed **natively on the local machine**.
- **Remote Execution**: Build, compilation, heavy tests, and GPU workloads are executed on the remote host (`REMOTE_HOSTNAME`) by pulling the latest pushed Git branch.

### Dynamic Path & Branch Resolution Rules
1. **Target Host**: `REMOTE_HOSTNAME` (SSH alias configured in `~/.ssh/config`).
2. **Project Name (`<project-name>`)**: Derived from the basename of the current working directory.
3. **Remote Root Directory**: `~/workspace/<project-name>` (or `~/projects/<project-name>` if specified). The Agent must resolve `<REMOTE_PATH>` to match the remote project root corresponding to the local workspace.
   - **Known example**: OpenWorkspace-Engine actually lives at `~/workspace/OpenWorkspace-Engine` (NOT `~/OpenWorkspace-Engine`). A bare `ls ~/OpenWorkspace-Engine` fails; check `~/workspace/` first, then `~/projects/`, then `find ~ -maxdepth 3 -name '<project-name>' -type d`. Also note `~/opencode-remote/<project-name>` may be a stale copy — prefer `~/workspace/`.
4. **Current Branch (`<current-branch>`)**: The Agent must detect the current active Git branch (e.g., `feature/ai-test` or `main`) before triggering remote execution.

### Remote Toolchain Environment: use `bash -l -c`

Non-interactive SSH sessions do **NOT** source `~/.bashrc`/`~/.profile`. Instead of manually exporting PATH / sourcing nvm per command, wrap the remote command in a **login shell** — `bash -l` reads `~/.profile`, which now sources cargo env **and** nvm + pnpm, so `pnpm`, `node`, `nvm`, and `cargo` all resolve:

```bash
ssh REMOTE_HOSTNAME "bash -l -c 'cargo --version'"
ssh REMOTE_HOSTNAME "bash -l -c 'pnpm --version'"
```

*Note on quoting:* the whole remote command goes inside `bash -l -c '…'`. If the command itself contains single quotes, escape them (`'\''`) or use a heredoc-style wrapper. Login-shell startup output is suppressed with `>/dev/null` when it would pollute command output.

**Why `bash -l` works now (fix applied 2026-08-08):** a login shell reads `~/.profile`, **not** `~/.bashrc`. `.bashrc` has a non-interactive guard (`case $- in *i*) ;; *) return;;`), so interactive-only exports there never load under `ssh`/`bash -l -c`. The fix moved nvm + pnpm into `~/.profile` (alongside the existing `~/.cargo/env` source). If a tool chain is ever missing again, check `~/.profile` first — cargo, node/nvm, and pnpm must be sourced there, not only in `.bashrc`. Note `~/.bash_profile` takes precedence over `~/.profile` if both exist (on this host it does not).

Facts learned from the live host (for debugging when `bash -l` still fails):

* `pnpm` binary: `/home/voip/.local/share/pnpm/bin/pnpm` (also `pnpmx`, `pnx`). Runtime is a `snapshot/pnpm.cjs`, so if `node` is missing it fails with `ENOENT not found: node`.
* Node is installed via nvm at `~/.nvm/versions/node/` — available versions: `v24.18.0`, `v24.19.0` (default choice, already active via `~/.profile`), `v26.5.0`. `pnpm use v24.19.0` is the default; a mismatched/absent node version produces obscure build errors.
* Rust/cargo toolchain: verified `cargo 1.97.1` via `~/.profile` sourcing `~/.cargo/env`.

Remember to escape `$` as `\$` when a variable must be expanded on the **remote** shell, not the local one.

---

## 2. Remote Test & Execution Workflow (Git Sync Pattern)

When code changes need to be tested or executed on the remote host, the Agent MUST follow this 2-step pipeline:

### Step 1: Local Commit & Push

Commit the current changes and push to the remote repository branch:

```bash
git add -A
git commit -m "feat/test: update for remote execution"
git push origin <current-branch>
```

### Step 2: Remote Sync & Trigger via SSH

Dispatch an SSH command to the remote host to fetch the pushed branch, pull the latest code, and run the target command/test:

```bash
ssh REMOTE_HOSTNAME "bash -l -c 'cd ~/workspace/<project-name> && git fetch && git checkout <current-branch> && git pull && <EXECUTION_COMMAND>'"
```

*Example with `pnpm run test`:*

```bash
ssh REMOTE_HOSTNAME "bash -l -c 'cd ~/workspace/<project-name> && git fetch && git checkout feature/ai-test && git pull && pnpm run test'"
```

### Web tests: run `svelte-kit sync` FIRST

The web app's `tsconfig.json` extends `./.svelte-kit/tsconfig.json`, which only exists after `svelte-kit sync` runs. `.svelte-kit/` is gitignored, so a fresh `git pull` leaves the remote WITHOUT it — vitest/rolldown then fails with `Could not resolve 'node:module'` / `Tsconfig not found`. Always run sync before any web test/check on the remote:

```bash
ssh REMOTE_HOSTNAME "bash -l -c 'cd ~/workspace/OpenWorkspace-Engine && git fetch && git checkout <current-branch> && git pull && (cd apps/web && pnpm exec svelte-kit sync) && pnpm test:web'"
```

---

## 3. Remote Execution Modes & Anti-Patterns

Choose one of two execution modes depending on the estimated task duration on the remote server:

### Critical Testing Anti-Pattern: NO Long Sleep Delays

* **DO NOT** execute long `sleep` delays (e.g., `sleep 60` or `sleep 300`) during test execution routines or polling loops.
* Tests are complex and prone to mid-run errors/bugs. Long `sleep` delays blind the Agent to early failures and waste critical time.
* **If background polling is required**, any interval or `sleep` duration **MUST be strictly under 30 seconds** (recommended: `sleep 5` to `sleep 10`) to maintain a fast feedback loop and capture error traces immediately upon collapse.

---

### Mode A: Short Tasks (Estimated Duration < 30s) — Synchronous / Blocking

**PREFERRED for test suites and syntax checks** so that error outputs (stdout/stderr) and stack traces are captured immediately.

* **Execution Method**: Send the SSH command directly; the Agent blocks and waits for stdout/stderr output.
* **Command Example**:

```bash
ssh REMOTE_HOSTNAME "bash -l -c 'cd ~/workspace/<project-name> && git fetch && git checkout <current-branch> && git pull && pytest tests/test_core.py'"
```

---

### Mode B: Long Tasks / Background Processes (Estimated Duration > 30s) — Remote Tmux Backgrounding

Suitable for model training, heavy compilation, container builds, or long integration test suites that exceed 30 seconds.

* **Naming Convention**: Session names must follow the format `opencode-<task_name>`.
* **1. Start Remote Background Task**:

```bash
ssh REMOTE_HOSTNAME "tmux new-session -d -s opencode-<task_name> 'cd ~/workspace/<project-name> && git fetch && git checkout <current-branch> && git pull && bash -l -c \"<LONG_COMMAND>\"'"
```

* **2. Active Log Sampling (Capture Pane)**:
Fetch the latest 50 lines of output to verify status. **Never delay for >= 30 seconds between status checks**:

```bash
ssh REMOTE_HOSTNAME "tmux capture-pane -pt opencode-<task_name> -S -50"
```

* **3. Clean Up Completed Sessions**:
Kill the session immediately once the task succeeds or hits a bug requiring a code fix:

```bash
ssh REMOTE_HOSTNAME "tmux kill-session -t opencode-<task_name>"
```

---

## 4. Safety Boundaries & Guarantees

1. **Local Integrity**: Never attempt to run local commands on remote paths.
2. **Path-First Principle**: Every remote SSH command must explicitly `cd` into the target remote project directory before running Git or execution scripts.
3. **Clean Sync**: Ensure changes are pushed locally before triggering remote commands; otherwise, the remote server will run outdated code.
4. **Session Isolation**: Use distinct Tmux session names for concurrent background jobs to avoid collision.

---

## 5. Clean Merge to `main` (squash — one commit)

Feature branches accumulate many messy commits during development. When the feature is **fully done** (docs written, tests passed, final commit made on the feature branch), merge to `main` with a **single clean commit** via `--squash`. This avoids merge commits and keeps `main` history linear and readable.

Run the whole sequence **locally** (git ops are local; only builds/tests go remote):

```bash
# 1. Switch back to main and pull the latest code
git checkout main
git pull origin main

# 2. Merge the feature branch with --squash
git merge --squash feature/ai-test

# 3. All changes are now staged; write one clean commit message
git commit -m "feat(engine): <one-line summary of the feature>"

# 4. Push to main
git push origin main
```

Rules:

* The merge commit must carry a **concise single-line message** summarizing the whole feature (not per-commit details).
* Only run this when the feature is complete and approved — after that, delete the feature branch (`git branch -d feature/ai-test`).
* Do **not** squash-merge work that is still in progress; keep pushing to the feature branch and testing remotely until done.
* Never use `git merge --squash` twice on the same branch (the branch commits are not consumed — re-merging would duplicate changes).