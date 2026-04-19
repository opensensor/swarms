---
name: reverse-followup
description: >
  [EXPLICIT INVOCATION ONLY] Targeted follow-up work on a reverse-engineering port
  after the main port is running. Takes a single natural-language description of a
  specific defect / stall / divergence (e.g. "encoder stalls in AL_SchedulerCpu_
  CreateChannel"), grounds it in smart-diff HLIL + the existing C port, and produces
  a minimal focused fix plan + executes it. Unlike reverse-planner (which plans
  bulk function porting) or reverse-swarm (which executes waves), this is a single
  tightly-scoped debug pass.
metadata:
  invocation: explicit-only
---

# Reverse-Followup (Targeted Debug & Fix of a Ported Region)

Single-task follow-up driver for a reverse-engineering port. The main port (built by
`reverse-planner` + `reverse-swarm`) is already running; you hit a specific defect
— a stall, a wrong return, a SIGSEGV, an output mismatch — and you want the binary's
HLIL to be the tiebreaker.

This skill does not port new regions. It walks the **existing** port against the
decomp at a single suspected location and commits a minimal fix.

Pairs with `reverse-planner` (strategy) and `reverse-swarm` (bulk execution).

## When to use this

Use this when:
- The port is running end-to-end but **one specific region misbehaves** (stall,
  assert, wrong result, silent exit, data mismatch).
- You have a concrete entry point: a function name, a trace line, a stack frame, or
  a dmesg fault address.
- You want **one tight pass**: find the divergence, fix it, verify, commit.

Don't use this for:
- Porting new functions that aren't in the codebase yet → `/reverse-swarm`.
- Running a whole wave of tasks → `/reverse-swarm`.
- Producing a multi-task plan → `/reverse-planner`.
- General debugging without a binary to check against → just plain `/debug`-style work.

## Sources of truth (ordered)

1. **smart-diff MCP** — primary. `decompile_binary_function(binary_id, name)` and
   `list_binary_functions` for neighboring symbols.
2. **`~/openimp/libimp.so_hlil.txt`** — secondary. Grep for names/addresses when
   smart-diff is ambiguous (indirect calls, struct offsets, `$gp`-relative loads).
3. **Existing port source under the target repo (default `~/openimp`).** The C code
   is the THING we're fixing; it's not a reference, it's the subject.

Never guess from memory or prior ports. Every divergence claim is grounded in a
side-by-side HLIL ↔ C view.

## Prerequisites

- smart-diff MCP reachable at `http://127.0.0.1:8011/mcp` (verify via
  `list_binja_servers`).
- `~/openimp/libimp.so_hlil.txt` exists.
- The target port repo exists and builds (default `~/openimp`).
- At least one concrete symptom — name, address, or trace.

If any prerequisite is missing, say which one and stop.

## Process

### Step 1: Parse the complaint

The user invocation is a short natural-language description. Extract:

1. **Entry point** — a function name, a trace label, a dmesg PC, or a symptom phrase
   that maps to one. If multiple candidates, prefer the innermost (deepest in the
   call chain).
2. **Symptom class** — stall / crash / wrong-return / wrong-output / output-missing.
3. **Optional seed evidence** — stack frame, trace log, register values the user
   pasted. Attach these to the task notes verbatim.

If the complaint is vague ("encoder doesn't work"), **ask one focused question**:
"Which specific function or dmesg trace should we start from?" Then proceed.

### Step 2: Locate the entry point in both places

1. Find the entry point's HLIL via smart-diff
   (`decompile_binary_function(binary_id, name)`). Record the address.
2. Find the entry point's C implementation in the port (grep for the symbol).
3. If there are multiple C implementations (static-in-one-file vs global-in-another,
   multiple ports), list them — ambiguous naming is a common defect class.

### Step 3: Build a call-site frontier

From the entry point, enumerate the **outgoing** function calls in the HLIL version
(not the C version — the HLIL is the "what should happen"). Mark each as:
- `leaf` — simple pure logic, port body compared directly
- `indirect` — vtable slot / function pointer; record the slot offset + the vtable
  definition site
- `external` — libc / syscall / kernel ioctl
- `deep` — recursive port region beyond the current frontier

Stop expanding when the frontier is ~5–10 call sites. More than that, narrow the
complaint further.

### Step 4: Side-by-side diff

For the entry point and each leaf callee, produce a short diff:

```
=== <func_name> @ <addr> ===
HLIL:
  <compact pseudocode excerpt that covers the suspected divergence>
C (path/file.c:lineN-lineM):
  <corresponding port excerpt>
Divergence:
  <one-line description> — severity {critical|moderate|minor}
Fix hint:
  <one-line what-to-change>
```

Only include functions where divergence is plausible. A clean match gets a one-line
"Matches decomp."

**Common divergence classes to actively look for** (learned from real bugs):

- **Function-pointer indirection depth.** HLIL `(*(*$p + 4))(...)` = deref `p`, add
  4 bytes, deref again, call. Port `((T (*)(...))READ_S32(p, 4))(...)` = different
  (single deref). Count stars carefully.
- **Static vs global name collision.** If two TUs define `helper()` — one `static`,
  one non-`static` — gcc's identical-code-folding can route references to the
  global. Check for same-name twins across codec/format variants (AVC / HEVC / JPEG).
- **Signature mismatch at forward-decl boundary.** Forward decl in header says one
  thing, definition in .c says another. The call site reinterprets args. Check
  every forward decl you see near the divergence.
- **Missing symbol resolved to NULL.** If BUILD mode excludes a file and no stub
  exists, the dynamic linker resolves the call to NULL → SIGSEGV with `epc=0` and
  `ra` = just after the call site. Fix: add a minimal stub or re-include the file.
- **Off-by-4 layout due to nested wrapper.** When caller passes `s0_1 + 8` but data
  was memcpy'd to `s0_1 + 4`, validator sees the struct shifted by 4 bytes. Re-check
  every offset claim against the actual wrapper's layout.
- **Validator guards that look inverted.** HLIL `if ((x >> N) & 1) == 0 goto X`
  reads "bit is clear" → skip block. Easy to flip.
- **Uninitialized stack variable used on fall-through path.** HLIL has branches that
  assign a scalar on some paths and read it on others; port drops a branch →
  uninitialized read. Check every `$sN_M = ...` site in HLIL vs port's write
  coverage.
- **Volatile + register caching.** When `int32_t s1` stores increments across
  function calls, gcc -O2 can keep it in a caller-saved register and clobber via a
  call. Make it `volatile` if you're debug-tracing it.

### Step 5: Propose the minimal fix

Write the diff as an Edit (or Edit sequence) that only touches:
- The entry point function
- Any callee clearly identified as divergent in Step 4

Do not widen scope. If Step 4 shows multiple divergent callees, prefer one focused
pass per invocation — queue the rest as follow-up items in the run summary.

### Step 6: Build and verify

1. Build locally for the target (`./build-for-device.sh` or the repo's equivalent).
2. If the repo has a smoke test (e.g. trace output, unit test, on-device check that
   previously surfaced the defect), run it — or tell the user exactly what to run
   and what to look for.
3. On-device testing: this skill does not deploy. Output the exact deploy command
   and the expected trace line that proves the fix. If the user reports it still
   fails, **return to Step 4 with the new evidence** — do not start a new
   invocation.

### Step 7: Commit on pass

Only commit when the verification step (either local build + test, or user-reported
on-device trace) confirms the fix. Commit only the files touched in Step 5.

```
fix(<area>): <one-line summary of the divergence>

<body: HLIL vs port diff, what fixed it, trace that confirmed it>
```

Do not commit partial work. Do not push.

### Step 8: Write the run summary

Write `~/<target-repo>/reverse-followup-<entry-point>.md`:

```markdown
# Followup: <entry point>

Invocation: <original user complaint>
Symptom: <class + evidence>

## HLIL reference
binary_id: <id>, address: <addr>, function: <name>
Relevant HLIL excerpt: <pasted>

## Port reference
Files touched: <list>
Prior state: <summary>

## Divergences found
- <one per location with severity>

## Fix applied
<summary>

## Verification
<how confirmed>

## Residual follow-ups
<anything noticed but deliberately out of scope>
```

## Example invocations

```
/reverse-followup encoder stalls in AL_SchedulerCpu_CreateChannel
/reverse-followup dmesg shows SIGSEGV epc=0 ra=libimp+0x7ef4c
/reverse-followup CheckValidity returns s1=2, fields look correct
/reverse-followup JPEG path writes garbage at cp+0x7c after rc_param runs
```

Each of these gives a concrete entry point. The skill binds it to an HLIL address
and the C port location, diffs, fixes one place, and commits.

## Important

- **One focused pass per invocation.** If you find 4 divergences, fix the most
  upstream one; note the others as follow-ups.
- **HLIL is the tiebreaker.** If your memory says one thing and HLIL says another,
  HLIL wins. Always re-fetch the decomp via smart-diff; do not trust a cached
  snippet from a prior conversation.
- **Don't refactor.** If the port is ugly but correct, leave it. This skill is for
  fixing divergences, not beautifying.
- **Don't widen scope.** The user picked one defect. If you see a related one,
  mention it in the summary and stop. Let the user decide whether to run another
  pass.
- **Don't chase through a known-hardware wall.** If the divergence lands in a
  region the user has already called out as hardware-driver or AVPU-level work,
  report the wall and stop rather than speculating about IRQ flows — that's a
  different kind of work.
