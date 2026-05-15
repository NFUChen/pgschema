# perf: parallelize current state fetching in multi-schema plan

## Summary

The multi-schema `plan` command previously processed each schema sequentially through the full pipeline (fetch current state → apply desired state → inspect → diff). This PR parallelizes the **current state fetching** phase across all schemas, reducing overall wall-clock time.

## Motivation

In multi-schema scenarios (e.g., multi-tenant architectures), `runPlanMultiSchema` runs the plan pipeline for N schemas. The original loop was fully sequential:

```
schema_1: connect to target DB for current state → apply desired → inspect desired → diff
schema_2: connect to target DB for current state → apply desired → inspect desired → diff
schema_3: connect to target DB for current state → apply desired → inspect desired → diff
```

Each "connect to target DB for current state" is an independent network round-trip (querying `pg_catalog`), with no dependencies between schemas. This is the most straightforward parallelization opportunity.

## Design

### Why two phases?

The core constraint is that `DesiredStateProvider` (embedded postgres or external DB) is a **stateful shared resource**.

The provider holds a single `tempSchema` name (e.g., `pgschema_tmp_20251030_154501_a3f9d2e1`). Each `ApplySchema` call performs:

```sql
DROP SCHEMA IF EXISTS "pgschema_tmp_xxx" CASCADE;
CREATE SCHEMA "pgschema_tmp_xxx";
SET search_path TO "pgschema_tmp_xxx", public;
-- Execute user's desired state SQL
```

If two goroutines called `ApplySchema` concurrently, they would overwrite each other's temp schema contents, causing the subsequent inspect to return incorrect IR and produce wrong diffs.

Therefore, this PR splits the pipeline into two phases:

```
Phase 1 (parallel):   All schemas concurrently fetch current state from target DB + compute fingerprint
                       ↓ sync.WaitGroup.Wait()
Phase 2 (sequential): For each schema, apply desired state → inspect → diff (using pre-fetched current state)
```

### Architecture diagram

```
                    ┌─ goroutine: inspect schema_1 from target DB ─┐
                    │                                               │
Phase 1 (parallel)  ├─ goroutine: inspect schema_2 from target DB ─┤  wg.Wait()
                    │                                               │
                    └─ goroutine: inspect schema_3 from target DB ─┘
                                        │
                                        ▼
                    ┌─ schema_1: apply desired → inspect provider → diff ─┐
                    │                                                      │
Phase 2 (sequential)├─ schema_2: apply desired → inspect provider → diff ─┤
                    │                                                      │
                    └─ schema_3: apply desired → inspect provider → diff ─┘
```

### Additional optimizations

- **Ignore config** and **desired state file** were previously loaded inside `GenerateSchemaPlan` on every iteration. They are now hoisted outside the loop and loaded once.

## Changes

| File | Change |
|------|--------|
| `cmd/plan/plan.go` | Add `generateSchemaPlanWithCurrentState` function that accepts pre-fetched current state IR and fingerprint |
| `cmd/plan/plan.go` | Refactor `runPlanMultiSchema` into two phases: Phase 1 parallel current state fetch, Phase 2 sequential desired state processing |
| `cmd/plan/plan.go` | Hoist ignore config and desired state file processing outside the loop (avoid redundant I/O) |

## Behavioral Notes

- `GenerateSchemaPlan` (public API) is **unchanged**. The single-schema code path is not affected.
- The new `generateSchemaPlanWithCurrentState` is an unexported function, used only internally by `runPlanMultiSchema`.
- No data race between Phase 1 goroutines: each goroutine writes to its own index in `currentStates[i]`, and all writes complete before `wg.Wait()` returns.
- Error handling behavior is unchanged: a failure in one schema prints an error and continues processing the remaining schemas.

## Future Work

To parallelize Phase 2 as well, the `DesiredStateProvider` interface would need to be redesigned:

- Have `ApplySchema` return an independent temp schema name (instead of storing it in a struct field)
- Or allow the provider to manage multiple temp schemas concurrently

This is a larger refactor and can be explored as a follow-up.

## Test Plan

- [ ] `go build ./...` compiles successfully
- [ ] Existing multi-schema integration tests pass
- [ ] Manual test with multiple schemas confirms results are identical to before the refactor
