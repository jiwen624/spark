# Why the six `CANNOT_FIND_DATA` renders in `TableOutputResolver` are dead / unreachable

This is supporting evidence for SPARK-57223. While fixing `toSQLId` mis-quoting of
special-char column/field names, nine `toSQLId(<schema name>)` render sites in
`TableOutputResolver.scala` were candidates for the `toSQLId(Seq(name))` fix. Three
(lines 391, 425, 437 — `EXTRA_COLUMNS`, `EXTRA_STRUCT_FIELDS`, `STRUCT_MISSING_FIELDS`)
are reachable and were fixed + tested. The remaining six all render
`INCOMPATIBLE_DATA_FOR_TABLE.CANNOT_FIND_DATA` and were **left at base behavior**
because they are dead or unreachable.

Line numbers refer to `apache/spark@681afc2b500` (the base of this work).

The six sites:

| Line | Context |
|------|---------|
| 132 | `resolveOutputColumns`, `if (errors.nonEmpty)` |
| 405 | `reorderColumnsByName`, `else if (enforceFullOutput)` |
| 487 | `resolveColumnsByPosition`, `if (result.length != actualExpectedCols.size)` |
| 548 | `resolveStructType`, `else if (enforceFullOutput)` |
| 590 | `resolveArrayType`, `else if (enforceFullOutput)` |
| 657 | `resolveMapType`, `else if (enforceFullOutput)` |

> Note: these are genuine *user-facing* error messages (`AnalysisException`s shown to
> whoever runs the write) — the point is not that "users don't see this kind of
> message," but that **these particular lines never execute**, so nobody ever sees
> their output. The `CANNOT_FIND_DATA` a user *can* hit (a missing column) is rendered
> by the **unchanged** line 352 (`newColPath.quoted`), which already quotes special-char
> names correctly.

---

## 1. Line 132 is dead — static proof

```scala
val errors = new mutable.ArrayBuffer[String]()
... reorderColumnsByName(..., errors += _, ...)        // the callback is *passed*
... resolveColumnsByPosition(..., errors += _, ...)    //   but never *called*
if (errors.nonEmpty) {                                 // always false
  throw ...CannotFindDataError(tableName, expected.map(_.name).map(toSQLId)...)  // line 132
}
```

The `errors` buffer is written only through the `addError` callback. Grep shows
`addError` is declared and threaded through ~10 call sites but **never invoked**:

```
$ grep -rnP 'addError\(' \
    sql/catalyst/.../analysis/TableOutputResolver.scala \
    sql/catalyst/.../types/DataTypeUtils.scala
(no invocation — only `addError: String => Unit` declarations and pass-throughs)

$ grep -rnP 'addError\("|addError\([a-z]' <same files>
(none)
```

`DataTypeUtils.canWrite` (the leaf type-compatibility check the callback is handed to)
**throws** its errors directly (`CANNOT_SAFELY_CAST`, etc.) instead of calling
`addError`. So `errors` is always empty and `if (errors.nonEmpty)` is permanently
false. Line 132 cannot execute.

---

## 2. Lines 405 / 487 / 548 / 590 / 657 are unreachable — structural argument

Every one of these has the same shape:

```scala
if (resolved.length == expectedType.length) {
  ... Some(...)                          // success branch
} else if (enforceFullOutput) {
  ... toSQLId(... all expected names ...)  // <-- the reverted CANNOT_FIND_DATA render
  throw CannotFindDataError(...)
} else {
  None
}
```

The render fires only if **both** `resolved.length != expected.length` **and**
`enforceFullOutput == true`. Those cannot co-occur:

- **`enforceFullOutput == true`**: a nested resolution that cannot fill a column
  throws a *specific* error first — `EXTRA_*` (391/425), missing-column at 352, or
  `CANNOT_SAFELY_CAST` — rather than returning a short result. So lengths always match
  and the success branch is taken. (`reorderColumnsByName`/`resolveColumnsByPosition`
  only return a short/`Nil` result when `enforceFullOutput == false`; otherwise they
  throw.)
- **`enforceFullOutput == false`**: a nested resolution *may* return short, but then
  `else if (enforceFullOutput)` is false and control falls to `None` — the throw is
  skipped.

`enforceFullOutput` is `true` down the whole append/insert tree (set at
`resolveOutputColumns`) and `false` down the merge-update tree (`resolveUpdate`).
Both cases exclude the `else if (enforceFullOutput)` body.

---

## 3. Empirical confirmation

To verify the static/structural argument, all six render sub-branches and the
`addError` callback were instrumented with `System.err.println("MARK_…")`
(see `instrumentation.patch`), plus a **positive control** at line 391 (a reachable
line). The instrumented build was run over a broad battery of write / insert / merge /
update suites.

Result (see `test-run.log`):

```
19  MARK_391_POSITIVE_CONTROL       <- reachable line; proves the harness works
 0  MARK_ADDERROR
 0  MARK_132
 0  MARK_405
 0  MARK_487
 0  MARK_548
 0  MARK_590_657
Tests: succeeded 759, failed 0
```

The positive control firing 19× proves `System.err` is captured; the six zeros are
therefore real. A second run over a different suite set (762 tests) produced no
markers at all.

---

## 4. Reproduction

```bash
git checkout spark-57223-toSQLId-deadcode-evidence   # instrumentation already applied
# (or, on a clean base checkout:  git apply evidence/instrumentation.patch)

build/sbt 'sql/testOnly \
  *DataFrameWriterV2Suite *DataSourceV2SQLSuite *DataSourceV2DataFrameSuite \
  *ResolveDefaultColumnsSuite *SQLInsertTestSuite \
  *AlignUpdateAssignmentsSuite *AlignMergeAssignmentsSuite \
  *DeltaBasedMergeIntoSchemaEvolutionScalaSuite *GroupBasedMergeIntoSchemaEvolutionScalaSuite' \
  2>&1 | grep -oP 'MARK_[A-Z0-9_]+' | sort | uniq -c
```

Expected: only `MARK_391_POSITIVE_CONTROL` appears; none of the six `CANNOT_FIND_DATA`
markers nor `MARK_ADDERROR` appear.

---

## Conclusion

The three reachable renders (391/425/437) are fixed and covered by regression tests on
the fix branch. The six `CANNOT_FIND_DATA` renders are dead (132) or unreachable
(405/487/548/590/657): no real `INSERT`/write/`MERGE` reaches them, and the
user-visible `CANNOT_FIND_DATA` (missing column) is rendered by the unchanged,
already-correct line 352. They were intentionally left unchanged.
