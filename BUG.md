# Bug: `skip` silently ignored in the OpMsg protocol path

## Summary

When using `find`, `findOne`, or `findCommand` against a modern MongoDB server
(wire protocol version ≥ 17), the `skip` field of a `Query` is silently dropped
and has no effect. All results are returned from the beginning of the result set
as if `skip = 0`.

## Root Cause

`queryRequestOpMsg` in `Database/MongoDB/Query.hs` sets `qSkip = fromIntegral skip`
in the `P.Query` record, but `putOpMsg` in `Database/MongoDB/Internal/Protocol.hs`
never references `qSkip` when serializing an OpMsg `find` command. The command
document is built exclusively from `qProjector`, `"$db"`, and `qSelector`:

```haskell
-- Protocol.hs
Query{..} -> do
    let sec0 = foldl1' merge [qProjector, [ "$db" =: db ], qSelector]
    putDocument sec0   -- qSkip is never referenced
```

The `qSelector` for a non-command find query (Query.hs) is:

```haskell
c = ("filter" =: s) : special ++ maybeToList bSize ++ maybeToList mLimit
--  includes: filter, sort, snapshot, hint, batchSize, limit — but NOT "skip"
```

## Affected Code Paths

| Function      | Wire version | `skip` working? |
|---------------|-------------|----------------|
| `find`        | < 17        | ✅ Yes — legacy `putRequest` uses `putInt32 qSkip` |
| `find`        | ≥ 17        | ❌ `qSkip` set but never emitted |
| `findCommand` | < 17        | ✅ Yes — explicit `"skip" =: toInt32 skip` in command doc |
| `findCommand` | ≥ 17        | ❌ Delegates to `find`, inherits the bug |
| `findOne`     | ≥ 17        | ❌ Same `queryRequestOpMsg` code path |

## Relationship to `nextBatch` / Cursor

`nextBatch'` correctly omits `skip` from `GetMore` requests — that is intentional
MongoDB protocol behaviour, as the server cursor tracks position after the initial
query. However, because the initial `find` command does not apply `skip` on the
OpMsg path, the cursor is created at the wrong starting position, and every
subsequent `nextBatch` call continues from that wrong position.

## Fix

Add `"skip"` to the selector document in `queryRequestOpMsg` when it is non-zero,
matching how `findCommand` already handles it for older servers:

```haskell
-- Database/MongoDB/Query.hs — queryRequestOpMsg
mSkip = if skip == 0 then Nothing else Just ("skip" =: (fromIntegral skip :: Int32))
c = ("filter" =: s) : special ++ maybeToList mSkip ++ maybeToList bSize ++ maybeToList mLimit
```
