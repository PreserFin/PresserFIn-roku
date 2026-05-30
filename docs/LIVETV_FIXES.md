# Live TV Fix Log

This document records Live TV guide/player bugs that were found and fixed, with root cause and file references.

## Scope

- Guide loading and navigation (`Schedule`, guide tasks)
- Live TV playback handoff and playback-info robustness
- Runtime memory-tier handling related to guide loading

## Fixes

### 1) Wrong Media Type on Live TV Replay

- Symptom:
  - After a TV channel finished, the app attempted replay using `recording` type.
- Root cause:
  - `onStateEvent()` created replay item with `type: ItemType.RECORDING` when `selectedItemType` was `tvchannel`.
- Fix:
  - Replay now uses `type: ItemType.TVCHANNEL`.
- File:
  - [source/MainEventHandlers.bs:1376](../source/MainEventHandlers.bs#L1376)

---

### 2) Unsafe Memory-Level Normalization

- Symptom:
  - Potential invalid numeric comparisons if `generalMemoryLevel` was not a clean numeric-like value.
- Root cause:
  - `ToInt()` result was used in comparisons without validity checks.
- Fix:
  - Added `isValid(memoryInteger)` guards before numeric comparisons.
- File:
  - [source/MainEventHandlers.bs:1445](../source/MainEventHandlers.bs#L1445)

---

### 3) Program Details Task Returned Wrong Failure Shape

- Symptom:
  - Potential runtime errors in guide details replacement path after failed detail lookup.
- Root cause:
  - `LoadProgramDetailsTask` returned `{}` on failure instead of `invalid`.
- Fix:
  - Failure now sets `m.top.programDetails = invalid`.
- File:
  - [components/liveTv/LoadProgramDetailsTask.bs:24](../components/liveTv/LoadProgramDetailsTask.bs#L24)

---

### 4) Schedule Task Did Not Guard Missing `Items`

- Symptom:
  - Potential iteration failure if `postJson()` returns a non-invalid object without `Items`.
- Root cause:
  - Only `data = invalid` was checked.
- Fix:
  - Added `or not isValid(data.Items)` guard before iterating.
- File:
  - [components/liveTv/LoadSheduleTask.bs:28](../components/liveTv/LoadSheduleTask.bs#L28)

---

### 5) Guide Focus/Details Path Missing Validity Guards

- Symptom:
  - Crash risk when focus metadata or indices were temporarily invalid during async guide updates.
- Root cause:
  - `programFocusedDetails`, `focusChannelIndex`, `focusIndex`, and some channel nodes were dereferenced without checks.
- Fix:
  - Added guards in:
    - `onProgramFocused()`
    - `onProgramDetailsLoaded()`
    - `onProgramSelected()`
  - Also guarded `group.lastFocus` assignment when active scene is invalid.
- File:
  - [components/liveTv/schedule.bs:205](../components/liveTv/schedule.bs#L205)
  - [components/liveTv/schedule.bs:270](../components/liveTv/schedule.bs#L270)
  - [components/liveTv/schedule.bs:306](../components/liveTv/schedule.bs#L306)

---

### 6) Schedule Iteration Assumed Array Shape

- Symptom:
  - Potential iteration issues if task output was empty/invalid-shaped while still entering handler.
- Root cause:
  - Iteration used `m.LoadScheduleTask.schedule` directly.
- Fix:
  - Normalized into local `scheduleItems` array first, then iterated that.
- File:
  - [components/liveTv/schedule.bs:205](../components/liveTv/schedule.bs#L205)

---

### 7) Batch Size Clamp Bug for Small Channel Lists

- Symptom:
  - For small lists, min-batch clamping could over-inflate batch size after list-size clamp.
- Root cause:
  - Clamp order allowed a later minimum clamp to override smaller list limits.
- Fix:
  - Reordered clamping logic in `getChannelScheduleBatchSize()` so final result respects list count.
- File:
  - [components/liveTv/schedule.bs:575](../components/liveTv/schedule.bs#L575)

---

### 8) Playback Info Structural Validation Gaps

- Symptom:
  - Playback could continue after malformed/partial playback-info response, causing dereference crashes on `MediaSources[0]`.
- Root cause:
  - Only top-level `isValid(m.playbackInfo)` checks existed in key paths.
- Fix:
  - Added `hasValidPlaybackInfo()` and used it:
    - immediately after first playback-info fetch
    - after transcode codec re-request flow
    - after subtitle burn-in re-request flow
  - Failures now set `video.content = invalid` and exit gracefully.
- File:
  - [components/ItemGrid/LoadVideoContentTask.bs:249](../components/ItemGrid/LoadVideoContentTask.bs#L249)
  - [components/ItemGrid/LoadVideoContentTask.bs:401](../components/ItemGrid/LoadVideoContentTask.bs#L401)

## Related Performance/Memory Changes (Context)

These are not bug fixes by themselves, but are relevant to current guide behavior:

- Adaptive guide channel batching based on runtime memory level and observed guide density.
- Time-window schedule loading in batches instead of one-shot loads.
- Startup memory tier seeded from Roku model RAM map.
- Dev-only guide profiling logs for batch size, payload estimate, and latency.

Primary files:

- [components/liveTv/schedule.bs](../components/liveTv/schedule.bs)
- [source/utils/session.bs](../source/utils/session.bs)
- [source/utils/rokuModelMemory.bs](../source/utils/rokuModelMemory.bs)
