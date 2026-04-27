# LevelDB Code Reading Notes 

These are my rough notes from reading original LevelDB code before implementing the assignment APIs.

## 1. Write Path

- `DBImpl::Write` is the main write entry.
- First write goes to WAL using `log::Writer::AddRecord`.
- Then batch is inserted into memtable using `WriteBatchInternal::InsertInto`.
- So flow is: WAL + memtable update together.
- When memtable is full, it is rotated and old one becomes `imm_`.
- `imm_` is flushed to SSTable through `WriteLevel0Table`.
- After flush, `VersionSet::LogAndApply` updates visible file metadata.

## 2. Read Path

- `DBImpl::Get` checks in order:
  1. `mem_`
  2. `imm_`
  3. SSTables using `Version::Get`
- Internally read uses `LookupKey`.
- For range-like reading, `NewIterator` is used.
- Iterator result is merged from memtable + immutable + SSTables.

## 3. Iterator (DBIter)

- `DBIter` sits on merged internal iterator.
- It keeps key ordering while walking user keys.
- Same user key can have multiple internal versions.
- `DBIter` only returns latest visible version.
- Visibility is based on sequence numbers.
- This part is directly useful for implementing `Scan`.

## 4. Compaction

- SSTables are immutable, so old data stays unless compaction runs.
- Compaction merges files from level `L` and overlapping files in `L+1`.
- It removes obsolete versions.
- It also removes tombstones when safe.
- Important flow/functions I tracked:
  - `BackgroundCompaction`
  - `DoCompactionWork`
  - `VersionSet::PickCompaction`
- Level 0 files can have overlapping key ranges, unlike deeper levels.

## 5. Delete Semantics

- Delete is logical first, not physical erase.
- It writes a tombstone (`kTypeDeletion`) with sequence number.
- Actual physical cleanup happens in compaction.
- So delete behavior is connected to compaction correctness.
- Iterator skips deleted keys using tombstones during traversal.

## 6. Snapshot / Sequence Numbers

- Every write gets a sequence number.
- Snapshot captures a fixed sequence point.
- Read with snapshot should only see entries with seq <= snapshot seq.
- `DBIter` and read path use this filtering.
- This is important for scan snapshot semantics.

## 7. Observations

- Iterator already merges memtable and SSTables, so scan implementation can reuse it.
- Delete uses tombstones, not direct on-disk erase.
- Compaction logic is spread across multiple functions, not one place.
- DBIter hides older versions and gives latest visible one.
