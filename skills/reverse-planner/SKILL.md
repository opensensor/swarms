---
name: reverse-planner
description: >
  [EXPLICIT INVOCATION ONLY] Plans a no-variation reverse-engineering port from a binary
  loaded in Binary Ninja into a C codebase, using the smart-diff MCP server as the
  source of truth. Pairs with the reverse-swarm executor. Optimized for libimp → openimp
  (T31) but generic.
metadata:
  invocation: explicit-only
---

# Reverse-Engineering Port Planner

Produce a dependency-aware port plan by enumerating functions in a target binary via the
Binary Ninja smart-diff MCP, grouping them by destination file in the target port
codebase, and ordering waves so callees land before their callers.

This skill plans. Execution is done by `reverse-swarm`.

## Core principles

1. **Sources of truth, in order:**
   1. **smart-diff MCP** — live Binary Ninja decomp (primary). Use for every function.
   2. **`~/openimp/libimp.so_hlil.txt`** — static HLIL dump of the T31 `libimp.so`,
      including segments, sections, and every function's HLIL. Used as the
      discrepancy-resolution reference when smart-diff output is ambiguous, when a
      call target is indirect, or when a symbol's surrounding context is needed.
   Never guess from existing headers or prior ports.
2. **No variations.** The plan's acceptance contract for each function is: port
   reproduces the decomp's control flow, constants, and struct offsets exactly. Safe
   struct member access is allowed where the struct layout is known.
3. **Leaves first.** Functions with no internal callees are scheduled first; callers
   wait for their callees.
4. **File-grouped tasks.** Multiple functions destined for the same target `.c` are
   bundled into a single task. This prevents parallel-edit collisions during execution,
   since the swarm does not use git worktrees.

## Prerequisites

- `smart-diff` MCP reachable at `http://127.0.0.1:8011/mcp` (the Binary Ninja bridge)
- The target binary is loaded in Binary Ninja with its per-binary MCP server started
- Target port codebase exists (e.g. `~/openimp`)

## Process

### 1. Collect the target function list

The methods to port come from one of:
- Inline in the invocation (`/reverse-planner  func_a, func_b, func_c ...`)
- A file path (e.g. `/reverse-planner  @methods.txt` — one function name per line, `#` comments allowed)
- Default: if no list is provided, seed = every non-external function exported from the
  binary (enumerate via smart-diff `list_binary_functions` and filter out imports/thunks)

Record the effective seed set in the plan.

### 2. Identify the target binary

Call `list_binja_servers` via smart-diff to list loaded binaries. Capture the
`binary_id` for the target.

Default selection: the `libimp.so` variant for T31. If multiple `libimp.so` binaries are
loaded, pick the T31 one (match on architecture `mipsel32` + the largest / most recent
match). Record the resolution reasoning in the plan's "Binary Ninja reference" section;
do not block.

### 3. Explore the target codebase

Walk the target port directory (default: `~/openimp`) to learn its file conventions:
- Which `.c` files exist under `src/` and any subdirs (e.g. `src/imp/`, `src/sysutils/`)
- Which headers exist under `include/`
- Any existing ported functions and which files they live in — these establish the
  module naming convention
- Any existing global variables / shared types already declared in headers

Record the layout snapshot at the top of the plan under "Target codebase layout".

### 4. Expand the seed set + build the call graph

For every function in the seed set, call `decompile_binary_function(binary_id, name)`
and extract the set of functions it calls (direct calls, plus indirect calls when the
target is resolvable).

Add those callees to the working set and repeat until closure. This produces the
transitive call tree rooted at the seed set.

Skip externals (imports/thunks — libc, syscalls, kernel helpers). List them once in the
plan under "External dependencies" for awareness.

Store:
- `callees[f]`: set of functions `f` calls
- `callers[f]`: reverse map
- `seed_set`: the user-supplied originals (flagged in the plan so they're
  distinguishable from auto-discovered callees)

### 4a. Identify global state and common types

Scan every function's decomp for:
- **Globals**: reads/writes to data-segment addresses. Collect `{global_name: [funcs]}`.
  A global is "shared" if more than one function touches it.
- **Struct types** referenced by multiple functions (same offsets, same size).
- **Magic constants / enums** repeated across functions.

These feed the Foundation task in the next step.

Store:
- `callees[f]`: set of functions `f` calls
- `callers[f]`: reverse map

Functions with indirect calls whose target smart-diff cannot resolve: `grep` the
call-site address in `~/openimp/libimp.so_hlil.txt` to resolve the target from the
static HLIL. If still unresolvable, record the call site in the plan's "Risks" section
with the HLIL excerpt — the executor will re-attempt resolution at implementation time,
and fall back to a stub with a comment if still ambiguous. No human step.

### 4b. Define the Foundation task (T0)

Create a single Wave-1 task `T0: foundation` that owns:
- Shared **global declarations** (definition in `src/imp/globals.c` or equivalent;
  `extern` declarations in `include/imp/globals.h` or equivalent).
- Shared **struct type definitions** placed in the appropriate existing header, or a
  new `include/imp/types.h` if no fit exists.
- Shared **enums / magic constants** similarly placed.

Foundation task contains **no function ports**. Its deliverable is headers + the
globals' defining translation unit.

Every downstream file-level task **depends_on: [T0]** if it reads or writes any of the
shared globals or uses the shared types. Tasks that touch nothing shared can skip the
T0 dependency.

Record the global-ownership map in the plan under "Shared state" so the executor and
verifier can reference it.

### 5. Map functions to target files

For each function, decide which target `.c` file it belongs in. Heuristics, in order:
1. If the function already has a port in the target codebase, use that file.
2. Prefix / module convention (e.g. `IMP_Encoder_*` → `src/imp/encoder.c`,
   `IMP_ISP_*` → `src/imp/isp.c`, `SU_*` → `src/sysutils/*`). Derive conventions from
   what already exists in step 3.
3. If still ambiguous, consult the HLIL reference file (`~/openimp/libimp.so_hlil.txt`):
   search for the function to see its surrounding context — neighboring functions and
   shared data segments often disambiguate the module. Pick the file that contains the
   majority of its immediate callees/callers.
4. Final fallback: assign to the best-prefix-match file and record the decision under
   "Mapping decisions" with the reasoning. Do not block.

Any function whose prefix does not match any convention goes into `src/unmapped.c` and
is called out in the plan's "Unmapped functions" section (for the executor to continue
through; the verifier still enforces decomp equivalence regardless of file placement).

### 6. Group into file-level tasks

One task per target file. Each task implements every function mapped to that file.

### 7. Compute task dependencies

Task `A` depends on task `B` (where `A ≠ B`) if any function in `A` calls any function
in `B`. Collapse self-references.

If the task graph contains cycles (mutually recursive functions split across files):
- If the cycle is small (2–3 files), merge those tasks into one.
- Otherwise, break the cycle at the function with the fewest incoming cross-file edges
  and note the split in a `cycle_break` risk entry.

### 8. Produce waves

Topologically sort tasks into waves. Wave 1 = tasks with no dependencies. Every
subsequent wave includes tasks whose dependencies are all in prior waves.

### 9. Save the plan

Write to `<target-port>/<binary-basename>-reverse-plan.md` (default for libimp →
openimp: `~/openimp/libimp-reverse-plan.md`).

### 10. Subagent review

Spawn a Task subagent to review the plan for:
1. Missing call-graph edges that would cause a caller to run before its callee
2. Functions missing a target-file mapping
3. Cycles not broken or merged
4. Exported-symbol coverage — every exported symbol of the binary appears in at least
   one task
5. Risks missing for unresolved indirect calls

Revise the plan based on actionable feedback before yielding.

## Plan template

```markdown
# Reverse-Engineering Plan: <binary> → <target-port>

**Generated**: <date>

## Overview

Port the user-supplied seed set plus their transitive callees from `<binary>`
(binary_id `<id>`) into `<target-port>` with zero behavioral or structural variation
from the Binary Ninja decomp.

## Binary Ninja reference

- Binary: `<binary>` (`<variant>`)
- binary_id: `<id>`
- smart-diff MCP: `http://127.0.0.1:8011/mcp`
- Decomp access: `decompile_binary_function(binary_id, function_name)`
- HLIL reference file: `~/openimp/libimp.so_hlil.txt` (secondary / discrepancy
  resolution)
- Binary selection reasoning: <why this binary_id was chosen>

## Reference resolution policy

When smart-diff output is ambiguous or missing:
1. `grep` the function name in `~/openimp/libimp.so_hlil.txt` to find its HLIL block.
2. For unresolved indirect calls, grep the call-site address (printed in the decomp)
   in the HLIL file — neighboring lines usually resolve the target.
3. For unknown struct offsets, grep the offset (e.g. `0x1c`) in the HLIL file in
   combination with the function name.
4. No human prompt — always resolve from these two sources, record the resolution in
   "Mapping decisions" or the task log, and continue.

## Target codebase layout

<snapshot of relevant directories from step 3>

## Seed set (user-supplied)

<list of functions the user explicitly requested, flagged distinctly from
auto-discovered transitive callees>

## Shared state

### Globals (owned by T0 foundation)

| Global | Owner file | Consumer functions |
|--------|-----------|--------------------|
| g_name | src/imp/globals.c | func_a, func_b |

### Shared types (owned by T0 foundation)

| Type | Header | Used by |
|------|--------|---------|
| struct Foo | include/imp/types.h | func_a, func_c |

### Shared constants / enums (owned by T0 foundation)

- `IMP_MAX_FOO = 0x42` — used by func_a, func_c

## Acceptance contract (applies to every task)

For every function in a task, the port must reproduce the decomp:
- Same control flow (branches, loops, gotos, early returns)
- Same numeric/string constants
- Same struct offsets and field access (safe member access allowed where layout known)
- Same call targets (by ported name)

Variations are not allowed. Verifier (`reverse-swarm`) enforces this.

## Dependency graph

<ascii or list form>

## Tasks

### T0: foundation
- **depends_on**: []
- **target_file**: (none — headers + globals definition unit)
- **owns**:
  - Globals definition: `src/imp/globals.c`
  - Globals declarations: `include/imp/globals.h`
  - Shared types: `include/imp/types.h` (or existing header)
  - Shared constants / enums
- **functions**: []
- **validation**: Headers compile standalone; globals have exactly one definition; all
  subsequent tasks can include the headers without conflict.
- **status**: Not Completed

### T1: src/<file>.c
- **depends_on**: [T0]  # omit T0 only if this task touches no shared state
- **target_file**: src/<file>.c
- **header**: include/<file>.h (if applicable)
- **functions**:
  - `func_a` @ 0x<addr>
  - `func_b` @ 0x<addr>
- **validation**: reverse-swarm verifier reports APPROVED for every function
- **status**: Not Completed
- **log**: (filled at execution)
- **files edited/created**: (filled at execution)

[... remaining tasks ...]

## Parallel execution groups

| Wave | Tasks | Starts When |
|------|-------|-------------|
| 1 | T1, T2 | Immediately |
| 2 | T3 | T1 complete |
| ... | ... | ... |

## Unmapped functions

Functions the planner could not confidently place. User to redirect before execution.

- `func_x` — candidate files: [...]

## External dependencies (not implemented)

Imports / thunks surfaced by Binary Ninja; listed for awareness only.

- `malloc`, `free`, `ioctl`, ...

## Risks

- Unresolved indirect call in `func_y` at 0x<addr> — executor must grep
  `libimp.so_hlil.txt` for the call-site address to resolve the target
- cycle_break: moved `func_z` out of `src/a.c` into `src/b.c` to break cycle
```

## Example usage

```
# Inline list of seed functions
/reverse-planner IMP_Encoder_CreateGroup, IMP_Encoder_CreateChn, IMP_Encoder_RegisterChn

# File of seed functions (one name per line, # for comments)
/reverse-planner @methods.txt

# File + explicit target port dir
/reverse-planner @methods.txt --target ~/openimp
```

The planner expands the seed set to include transitive callees, then produces the
dependency-ordered plan.

## Important

- Do NOT implement — only plan.
- Do NOT invent functions not in the binary.
- Every function's **source of truth is its smart-diff decomp**. If a decomp is missing
  or errors out, flag the function as a risk; do not guess.
- For indirect / virtual calls whose target the decomp cannot resolve, **do not** fake
  a dependency edge — note the call site and flag as a risk.
- The seed set is what the user asked for. Transitive callees are what must also be
  ported to make the seed set work. Both appear in the plan, but the **Seed set**
  section marks which is which so the user can verify coverage of their request.
