---
name: reverse-swarm
description: >
  Only to be triggered by explicit /reverse-swarm commands. Cross-harness wave executor
  for reverse-engineering ports: Codex implements each task via `codex exec`, Claude
  Code verifies via Task subagents, variations kick back to Codex. Binary Ninja
  smart-diff MCP is the source of truth for both roles.
---

# Reverse-Swarm Executor (Codex implements, Claude Code verifies)

Wave-based executor for reverse-engineering ports. Each task runs a two-role pipeline:

1. **Implementer — Codex** (`codex exec` in a background shell).
   Reads Binary Ninja decomps via the smart-diff MCP and writes/updates the C port.
2. **Verifier — Claude Code** (Task subagent spawned by this orchestrator).
   Re-reads the same decomps and diffs them against the Codex output for structural
   and behavioral equivalence. Variations kick back to the implementer.

A task only commits when the verifier reports zero variations.

Pairs with the `reverse-planner` skill.

## Prerequisites

- Plan file exists (produced by `/reverse-planner`), default path:
  `~/openimp/libimp-reverse-plan.md`.
- The target port repo is the current working directory (`~/openimp` for libimp work).
- smart-diff MCP is running at `http://127.0.0.1:8011/mcp` and configured in **both**
  harnesses (verify: `~/swarms/.mcp.json` and `~/.codex/config.toml`).
- HLIL reference file exists: `~/openimp/libimp.so_hlil.txt`.
- Binary is loaded in Binary Ninja with its per-binary MCP server started (T31
  `libimp.so` for the default libimp work).
- `codex` CLI is on PATH and accepts `codex exec`.

If any prerequisite is missing, log which one and abort the run (no interactive prompt
— the loop runs autonomously).

## Sources of truth (both implementer and verifier)

1. **smart-diff MCP** — primary. `decompile_binary_function(binary_id, name)`.
2. **`~/openimp/libimp.so_hlil.txt`** — secondary / discrepancy resolution. Full HLIL
   dump of the T31 `libimp.so`. Grep by function name, address, or byte offset to
   resolve anything smart-diff leaves ambiguous (indirect calls, struct offsets,
   neighboring-function context).

Every implementer and verifier prompt references both. Never fall back to guessing or
asking the user.

## Process

### Step 1: Parse request

Extract from user input:
1. Plan file (default above)
2. Optional task subset (e.g. `T3 T7`)

### Step 2: Read & parse plan

1. Read the plan file.
2. For each `### T<id>:` block, extract:
   - `id`, `target_file`, optional `header`
   - `depends_on[]`
   - `functions[]` as `(name, address)` pairs
   - `status`
3. If a subset was requested, filter to those IDs plus any of their dependencies that
   are not yet `Not Completed` → blocked.

### Step 3: Launch wave — Codex implementer

For every **unblocked** task (all `depends_on` statuses are Completed), launch Codex in
the background. Each task runs in its own shell so they run in parallel.

**Collision rule**: tasks in a wave must target **different files**. The planner
enforces this; if you detect two wave-mates writing the same file, log the conflict,
drop the later task from this wave (re-queue it for the next wave), and continue.

**Foundation task (T0) is special.** It contains no function ports — only shared
globals, struct definitions, and constants. Use the Foundation prompt below for T0
instead of the per-function implementer prompt. T0 is always alone in Wave 1 when it
exists.

#### Foundation task prompt (T0 only)

```bash
codex exec "$(cat <<'PROMPT_EOF'
You are setting up shared state for a reverse-engineering port from <binary> into
<target-port>.

## Source of truth
smart-diff MCP at http://127.0.0.1:8011/mcp, binary_id: <from plan>.

## Task T0: Foundation

The plan's "Shared state" section lists:
- Globals to declare (name, owner file, consumer functions)
- Shared struct types (name, header, users)
- Shared constants / enums

## Rules
1. For each global listed: define it exactly once in its owner file (initial value
   from the binary's .data/.bss — inspect via smart-diff). Declare `extern` in the
   header.
2. For each struct type: use the exact layout the decomp shows (field offsets and
   sizes). If the binary has debug info or you can infer field names from usage,
   prefer named fields; otherwise use `uint8_t padN[size]` fillers to preserve offsets.
3. For each constant/enum: use the exact numeric value from the decomp.
4. Do NOT implement any functions in this task.
5. Do NOT commit. Do NOT push.

## Output
- Create/edit the header files and the globals defining translation unit as listed
  in the plan's Shared state table.
- At end, print:
  `WROTE_HEADER <path>`
  `WROTE_GLOBALS <path>` (for the .c that defines the globals)
  `DEFINED <global_name> in <path>`
  `TYPE <struct_name> in <header>`
  `CONST <name> = <value> in <header>`
PROMPT_EOF
)" \
  --full-auto \
  --cd <target-port> \
  > /tmp/reverse-swarm-T0-impl-round<N>.log 2>&1 &
```

T0's verifier (next step) confirms each listed global/type/constant exists and its
value matches the binary. No function-level decomp diff applies to T0.

Use this shell pattern (tune flags to the local `codex` version — the intent is
non-interactive, full tool access, allow the smart-diff MCP):

```bash
codex exec "$(cat <<'PROMPT_EOF'
You are porting functions from <binary> into <target-port>.

## Sources of truth
1. smart-diff MCP at http://127.0.0.1:8011/mcp, binary_id: <from plan>.
   Use `decompile_binary_function(binary_id, function_name)` for EVERY function you
   write. Do not paraphrase, do not guess, do not rely on existing headers.
2. `~/openimp/libimp.so_hlil.txt` — full HLIL dump for discrepancy resolution. Grep
   it when smart-diff output is ambiguous (indirect calls, unresolved jump targets,
   struct offsets without known types). Never ask a human — always resolve from
   these two sources.

## Task <ID>: <target_file>
Functions to port (name @ address):
- <func1> @ 0x<addr>
- <func2> @ 0x<addr>
...

## Rules — NO VARIATIONS
1. Reproduce the decomp's control flow exactly: every branch, loop, goto, and early
   return.
2. Reproduce all numeric/string constants exactly.
3. Reproduce struct accesses at the same offsets. Prefer safe struct member access
   (`obj->field`) when the struct layout is known in the target port's headers;
   otherwise use the exact byte-offset form the decomp shows and leave a single-line
   comment with the offset.
4. Call other ported functions by their target-port name. Callees for this task are
   already implemented (the planner orders leaves first).
5. External calls (libc, syscalls, ioctls) go through the target port's existing
   wrappers if any exist; otherwise call them directly as the decomp does.
6. **Shared globals** are already defined by T0 in `<globals.c>` and declared `extern`
   in `<globals.h>`. Include the globals header and use the externs — do NOT re-declare
   or re-define any shared global. The plan's "Shared state" section lists which
   globals are shared.
7. **Shared struct types / enums / constants** are already defined by T0 in the
   appropriate headers. Include those headers — do NOT re-define shared types.

## Prior-round variations to fix
<Empty on first round. On kickback rounds, this section contains the verifier's diff
report. You MUST address every item.>

## Output
- Edit `<target_file>` and its header in place under `<target-port>`.
- Touch NO other files.
- Do NOT commit. Do NOT push.
- At end, print a one-line-per-function summary:
  `WROTE <func_name> @ <addr> -> <file>:<first_line>-<last_line>`
- If a function could not be ported (e.g. unresolved indirect call after consulting
  both smart-diff AND `~/openimp/libimp.so_hlil.txt`), print:
  `BLOCKED <func_name> @ <addr> reason: <short reason>` — but this should be rare;
  always try HLIL-grep first.
PROMPT_EOF
)" \
  --full-auto \
  --cd <target-port> \
  > /tmp/reverse-swarm-<TASK_ID>-impl-round<N>.log 2>&1 &
```

Record the PID per task. `<N>` is the kickback round (1 on first launch, 2/3 on
retries).

Note: the precise `codex exec` flag names may differ across Codex CLI versions. If the
above invocation errors, adapt flags (e.g. `--non-interactive`, `--approval
never`, `--cd`, `-C`) while preserving the intent: non-interactive run, full tool
access, smart-diff MCP available, working directory = target port.

### Step 4: Wait for the implementer

Poll each task's PID until exit. Then read the log and extract the per-function
`WROTE` / `BLOCKED` lines.

If a task produced only `BLOCKED` lines, skip verification, mark the task `blocked`
with the reason, and continue the wave.

### Step 5: Claude Code verifier

For each completed task, spawn a Task subagent.

**T0 verifier** uses the Foundation verifier prompt below. Per-function tasks use the
standard verifier prompt that follows.

#### Foundation verifier prompt (T0)

```
You are verifying the Foundation task of a reverse-engineering port. Read-only.

## Source of truth
smart-diff MCP at http://127.0.0.1:8011/mcp, binary_id: <from plan>.

## Verify every entry in the plan's Shared state section

1. For each global: confirm it is defined exactly once in the declared owner file, has
   the initial value from the binary's data segment, and is declared `extern` in the
   listed header. Use smart-diff to read the binary's initial value at the global's
   address.
2. For each struct type: confirm it is defined in the listed header, and its field
   offsets/sizes match the decomp's usage across consumer functions.
3. For each constant/enum: confirm the header has the exact value.

## Return format
First line: `APPROVED` or `VARIATIONS`.
If VARIATIONS, list every mismatch:
```
- kind: global|type|constant
  name: <name>
  expected: <from decomp / binary>
  found: <from header or .c>
  location: <path>:<line>
  fix_hint: <short>
```
```

#### Per-function verifier prompt

**description**: `Verify task <ID>: <target_file>`

**prompt**:
```
You are verifying a reverse-engineered port against its Binary Ninja decomp. You MUST
NOT edit files. You MUST NOT commit. Read-only.

## Sources of truth
1. smart-diff MCP at http://127.0.0.1:8011/mcp, binary_id: <from plan>.
   Use `decompile_binary_function(binary_id, func_name)` for each function below.
2. `~/openimp/libimp.so_hlil.txt` — grep for the function name / address to
   cross-check anything smart-diff leaves unclear.

## Task <ID>: <target_file>
Functions to verify:
- <func1> @ 0x<addr>
- <func2> @ 0x<addr>

## Procedure — for each function
1. Fetch the decomp via smart-diff.
2. Read the function's implementation in <target_file>.
3. Compare statement-by-statement:
   - Control flow: every branch, loop, goto, early return in decomp appears in port
   - Constants: every numeric/string literal matches
   - Struct access: every offset/field access matches (safe member access allowed when
     the field is confirmed in a target-port header; otherwise raw offset must match)
   - Call targets: every call matches by name (or equivalent ported name)

## Return format
First line: `APPROVED` or `VARIATIONS`.

If VARIATIONS, follow with one item per variation:
```
- function: <name>
  decomp_line: <short snippet from decomp>
  port_line: <short snippet from port>
  location: <target_file>:<line_number>
  severity: <critical|moderate|minor>
  fix_hint: <one sentence on the correct behavior>
```

Do not return narrative. Do not suggest refactors. Only report variations from the
decomp.
```

### Step 6: Kickback loop

If the verifier returns `VARIATIONS`:

1. Re-launch the implementer for the same task with the verifier's full report pasted
   into the `## Prior-round variations to fix` section of the prompt. Increment round
   counter.
2. Re-verify.
3. Rounds **1–3**: standard kickback (just the verifier's diff report).
4. Round **4 (HLIL assist)**: before relaunching, `grep` each still-failing function's
   name out of `~/openimp/libimp.so_hlil.txt` and include the matched HLIL block
   inline in the prompt under `## HLIL reference for <function>`. This gives the
   implementer the authoritative static HLIL alongside smart-diff.
5. Re-verify after round 4.

If still VARIATIONS after round 4:
- Mark the task `blocked` in the plan with the final variation report attached.
- Do NOT commit the partial work; leave it in the working tree.
- Revert any uncommitted file changes that would collide with later waves (so the
  task's files return to pre-task state) so the wave can move on cleanly.
- Continue other tasks. No human prompt.

### Step 7: Commit on pass

When the verifier returns `APPROVED`:

1. `git add <target_file> <header-if-any>` — stage ONLY the files for this task, since
   other tasks in the wave are working in parallel.
2. `git commit -m "port: T<ID> <target_file> (verified vs <binary> decomp)"` — NEVER
   push.
3. Update the plan task entry with:
   - `status: Completed`
   - `log`: round count, functions ported, verifier final verdict
   - `files edited/created`: list

### Step 8: Next wave

After every task in the current wave is `Completed` or `blocked`, re-parse the plan and
launch the next set of unblocked tasks.

A `blocked` task blocks all tasks that depend on it. Skip those dependents (mark them
`deferred`) and continue with tasks whose deps are all Completed.

### Step 9: Final summary

Report:
- Total tasks assigned / completed / blocked / deferred
- Per-blocked-task: final variation or BLOCKED reason
- Per-completed-task: kickback rounds used
- Files modified across the run
- Any unresolved indirect-call sites surfaced by implementers

Write the final summary to `~/openimp/reverse-swarm-last-run.md` for later review.
Do not block waiting for a human. The orchestrator exits after the last wave.

## Error handling

- `codex exec` not found: stop, instruct user to install / PATH the Codex CLI.
- `codex exec` fails with an MCP/connection error: verify `~/.codex/config.toml` has
  the `[mcp_servers.smart-diff]` entry and the bridge is reachable.
- `codex exec` errors with `multi_agents not enabled`: this skill only uses
  single-shot exec, not subagents; the error likely indicates a different underlying
  issue — surface the log.
- Plan parse failure: log the failing block to the run summary and skip the
  unparseable task. Do not block the run for a human fix.
- Task subset contains unknown IDs: list the available IDs to the run summary and
  drop the unknown ones; proceed with what's valid.

## Example usage

```
/reverse-swarm
/reverse-swarm libimp-reverse-plan.md
/reverse-swarm libimp-reverse-plan.md T3 T7
```

## Important

- Both harnesses share filesystem state (no worktrees). Same-wave tasks MUST write
  different files.
- Never allow the implementer or verifier to edit files outside its task's
  `target_file` + header.
- Never push. Only commit.
- The verifier must pull a **fresh** decomp from smart-diff for each round — do not
  cache across rounds; the implementer may have altered callee names that affect the
  decomp's cross-reference display.
