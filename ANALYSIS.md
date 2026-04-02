# SCL Code Analysis & Improvement Recommendations

> Generated: 2026-04-02
> Scope: SYSTORE_BYPASS, PATH_MANAGEMENT, PATH, PATH_MERGE, PATH_SPLIT

Priority Levels: **Critical** | **High** | **Medium** | **Low**

---

## 1. SYSTORE_BYPASS

### 1.1 Bugs / Logic Issues

| # | Priority | Issue | Location |
|---|----------|-------|----------|
| 1 | Critical | **Safety/Stop override gets overwritten in the same scan.** Region 3 sets `State := ST_BYPASS_EXIT`, but then the CASE in Region 4 runs `ST_BYPASS_EXIT` immediately on the same scan, which transitions straight to `ST_SYSTORE` (line 336). The exit state never persists, so cleanup logic never actually executes across multiple scans. | Lines 143-158 vs 336 |
| 2 | High | **`StateChanged` is computed too early.** `PrevState` is updated at line 131 before safety/stop checks may change `State`. If Region 3 changes the state, `StateChanged` will be FALSE on the first scan of the new state, so "on entry" logic (e.g., line 215) is skipped. **Fix:** Move `#PrevState := #State` to the END of the FB, after the CASE. | Lines 130-131 |
| 3 | High | **`StopRequested` is set and cleared in the same scan.** Line 135 sets it, Region 3 transitions to EXIT, and line 333 clears it all in one scan cycle. The latch has no practical effect. | Lines 135, 333 |
| 4 | Medium | **`SYS_HeartbeatHi` is declared but never used.** Only `SYS_HeartbeatLo` is read (line 109). If the counter wraps at 65535, the heartbeat will appear to "change" on wrap, masking a stale connection. | Line 29 vs 109 |
| 5 | Medium | **Hold-to-start timer gated by `NOT BypassStopPB`** can silently prevent bypass entry if the stop button bounces mechanically. No diagnostic is raised. | Lines 350-353 |
| 6 | Low | **Garbled comment on line 46.** Residual text from copy-paste in the VAR declaration block. | Line 46 |

### 1.2 Missing Features

| # | Priority | What to Add | Notes |
|---|----------|-------------|-------|
| 1 | Critical | **`ST_BYPASS_EXIT` must verify motors are stopped** before transitioning to `ST_SYSTORE`. Currently transitions unconditionally (line 336). Add a timer or check `PATH_MANAGEMENT_DB` outputs. | Lines 305-336 (placeholder) |
| 2 | High | **`ST_BYPASS_MOVECHECK` is a no-op.** `LineReady` is hardcoded TRUE (line 272). Implement actual conveyor readiness checks. | Lines 250-272 (placeholder) |
| 3 | High | **`FaultCode` output is never written.** Always 0. Should be set for each fault condition (comm loss, safety, timeout, etc.). | Line 39 |
| 4 | Medium | **`PendantAuto` input is unused.** Bypass should likely require Auto mode on the pendant. | Line 32 |
| 5 | Medium | **No comm-flapping protection.** If SYSTORE communication is intermittent, the state machine can rapid-cycle between SYSTORE/CHECK/INIT/EXIT. Add a cooldown timer or retry counter. | - |
| 6 | Medium | **`FaultMessage` only set for one condition** (PATH timeout, line 298). Comm loss, safety faults, etc. produce no message. | Line 298 |

### 1.3 Safety Concerns

| # | Priority | Concern |
|---|----------|---------|
| 1 | Critical | **No explicit E-Stop handling.** `SafetyOK` is the only proxy. Best practice: dedicated E-Stop input with direct motor-stop output, not routed through the state machine. |
| 2 | High | **Bypass can activate without pendant in Auto mode.** `PendantAuto` exists but is never checked. |
| 3 | Medium | **No watchdog or cycle-time monitoring.** If PLC scan time becomes excessive, the state machine cannot detect or react. |

### 1.4 Optimization

| # | Priority | Suggestion |
|---|----------|------------|
| 1 | Low | **`Timer.tMotorTimeout[0..31]` array is unused** in this FB. Only PATH uses motor timeouts. Wastes memory (32 TON_TIME instances). Remove it. |
| 2 | Low | **`StateName` string assignment every scan** is redundant. Move inside `IF StateChanged` to reduce memory writes. |

### 1.5 HMI / Diagnostics

| # | Priority | What's Missing |
|---|----------|----------------|
| 1 | High | **No bypass session duration or activation counter.** Operators cannot see how long bypass has been active or how many times it was triggered. |
| 2 | Medium | **No heartbeat counter value exposed to HMI.** Operators cannot see the actual heartbeat or time since last change for comm diagnostics. |

---

## 2. PATH_MANAGEMENT

### 2.1 Bugs / Logic Issues

| # | Priority | Issue | Location |
|---|----------|-------|----------|
| 1 | Critical | **`Fault_Timeout` feedback loop.** Lines 21-25 push `Fault_Timeout` INTO every PATH DB. Lines 136-141 read it back via OR. Result: when ONE path faults, the fault propagates to ALL paths on the next scan, causing a system-wide cascade. **Fix:** Use separate variables for fault-in vs fault-out per PATH instance. | Lines 20-26 vs 136-141 |
| 2 | High | **Missing handshake enables on `PATH_E0003->E0006`.** E0003 is a merge destination and E0006 is a split source, but this middle path has neither `HskSrcEna` nor `HskDestEna` set. This allows it to freely pull from E0003 during a merge or push to E0006 during a split, risking collisions. | Lines 55-63 |
| 3 | Medium | **Comment mismatch at line 76.** Says "E0007 -> E0008" but the path is E0006 -> E0007. | Line 76 |
| 4 | Medium | **One-scan handshake latency.** PATH instances are called before MERGE/SPLIT, so grants from MERGE/SPLIT aren't seen until the next scan. Usually acceptable, but adds latency. | Lines 114-131 |

### 2.2 Missing Features

| # | Priority | What to Add |
|---|----------|-------------|
| 1 | High | **No per-path fault identification.** When `Fault_Timeout` is TRUE, the operator cannot tell WHICH path faulted. Add individual fault flags or a fault-path index output. |
| 2 | Medium | **No per-path enable/disable.** Cannot take a single path offline for maintenance without stopping all bypass. |
| 3 | Medium | **No pallet counting.** Operators need transfer counts per path for monitoring and maintenance scheduling. |

### 2.3 Optimization

| # | Priority | Suggestion | Location |
|---|----------|------------|----------|
| 1 | Medium | **CFG region writes every scan.** All configuration (ModuleList, CommandList, NumberModules, timeouts) is written unconditionally every scan cycle. This overwrites any HMI runtime tuning and wastes CPU time. **Fix:** Move CFG writes into the `IF Init_MNG` block or use a write-once flag. | Lines 29-86 |

---

## 3. PATH

### 3.1 Bugs / Logic Issues

| # | Priority | Issue | Location |
|---|----------|-------|----------|
| 1 | High | **Duplicate line.** Line 222 is an exact copy of line 221: `#VARs.dbAny.DBW14 := CFG.CommandList[...]`. Possibly a missing write to a different register (DBW16?). | Lines 221-222 |
| 2 | High | **`Hsk.Done_SrcMov` / `Done_DstMov` set regardless of handshake enable.** Done signals fire even when `HskSrcEna` / `HskDestEna` is FALSE. Could cause unexpected behavior in MERGE/SPLIT arbiters that aren't expecting Done signals from unmanaged paths. | Lines 183-188 |
| 3 | Medium | **DB_ANY absolute addressing is fragile.** `%DBX4.7` (presence), `%DBX12.0` (run command), `DBW14` (command word) rely on exact byte offsets. Any module DB restructuring silently breaks all PATH instances. | Lines 109, 220-225 |
| 4 | Medium | **Transfer can restart immediately after completion** if the destination sensor bounces. No debounce or minimum-off-time between transfers on the same segment. | Lines 170-203 |
| 5 | Low | **INIT/FAULT loops iterate 0..31** regardless of actual module count. Minor waste for 2-3 module paths. | Lines 54-58, 87-91 |

### 3.2 Missing Features

| # | Priority | What to Add |
|---|----------|-------------|
| 1 | High | **No sensor debouncing.** Presence sensor (`%DBX4.7`) is read directly. Sensor noise or pallet bounce can cause false triggers or premature transfer completion. |
| 2 | High | **No jam detection.** If module N has a pallet but N+1's sensor is stuck TRUE (jam/sensor fault), no transfer is requested and no alarm is raised. The pallet sits indefinitely. Only active-transfer timeouts are detected. |
| 3 | Medium | **No "unexpected pallet" detection.** If a pallet appears on a module without an active transfer (manual placement, sensor fault), there is no detection or handling. |
| 4 | Medium | **No transfer counter.** No way to track completed transfers per segment for maintenance (roller/chain wear). |

### 3.3 Safety Concerns

| # | Priority | Concern |
|---|----------|---------|
| 1 | High | **Motors run for one extra scan after fault.** When `Fault_Exit` is set (line 195), it is acted upon next scan (line 85). During the current scan, ControlWord bits remain active (lines 200-203). Should be documented or handled. |
| 2 | Medium | **No maximum concurrent transfer limit.** All segments can transfer simultaneously. High simultaneous motor starts could exceed electrical capacity. |

### 3.4 Optimization

| # | Priority | Suggestion |
|---|----------|------------|
| 1 | Low | **SHL operations repeated for the same bit index.** `SHL(DWORD#1, BitIndex)` is computed 3-4 times per loop iteration (lines 166, 167, 201, 202). Store in a TEMP variable once. |

---

## 4. PATH_MERGE

### 4.1 Bugs / Logic Issues

| # | Priority | Issue | Location |
|---|----------|-------|----------|
| 1 | Medium | **No timeout on active grant.** Once `Busy = TRUE`, it stays until `Done` arrives. If the granted PATH stalls without setting Done, the merge is permanently locked. **Fix:** Add a watchdog timer; release grant and raise fault on timeout. | Lines 58-78 |
| 2 | Medium | **`DstReady` not rechecked during active transfer.** Once Busy, `DstReady` is bypassed (RETURN at line 78). If the destination becomes occupied mid-transfer (jam), the merge continues granting, risking collision. | Line 88 vs 58-78 |
| 3 | Low | **First-request bias after init.** `LastGranted = -1` means `(-1+1) MOD N = 0`, always checking path 0 first. | Line 40, 97 |

### 4.2 Missing Features

| # | Priority | What to Add |
|---|----------|-------------|
| 1 | High | **No fault output.** MERGE has no way to signal that it is stuck or faulted. Add a `Fault` output and timeout timer. |
| 2 | Medium | **No starvation detection.** If one path is consistently blocked, there is no reporting mechanism. |

### 4.3 Robustness

| # | Priority | Concern |
|---|----------|---------|
| 1 | Medium | **`Init_Merge` during active transfer** clears all grants immediately (lines 33-41), leaving a pallet mid-transfer with no motor command. PATH will eventually timeout, but merge state becomes inconsistent. |

---

## 5. PATH_SPLIT

### 5.1 Bugs / Logic Issues

| # | Priority | Issue | Location |
|---|----------|-------|----------|
| 1 | Medium | **No timeout on active grant.** Same as PATH_MERGE. Stuck Done permanently locks the split. | Lines 59-79 |
| 2 | Low | **First-request bias after init.** Same as PATH_MERGE. `LastGranted = -1` favors path 0. | Line 41, 126 |

### 5.2 Missing Features

| # | Priority | What to Add |
|---|----------|-------------|
| 1 | High | **No `SrcReady` check.** PATH_MERGE has `DstReady` (destination is empty), but PATH_SPLIT has no equivalent check that the source module actually has a pallet before granting. | - |
| 2 | High | **No fault output or timeout.** Same gap as PATH_MERGE. Add a `Fault` output and watchdog. | - |
| 3 | Medium | **No destination-availability check.** Split grants purely on request + round-robin. It does not verify the downstream path is clear. | - |

### 5.3 Robustness

| # | Priority | Concern |
|---|----------|---------|
| 1 | Medium | **`Init_Split` during active transfer** clears state abruptly, same risk as PATH_MERGE. |

---

## 6. Cross-Cutting Concerns (All FBs)

| # | Priority | Issue | Affected FBs |
|---|----------|-------|-------------|
| 1 | Critical | **Fault cascade feedback loop** in PATH_MANAGEMENT. One path's fault propagates to all paths via the `Fault_Timeout` read/write cycle. | PATH_MANAGEMENT, PATH |
| 2 | High | **No "all motors stopped" verification** anywhere. When bypass exits or a fault occurs, software flags are cleared but physical motor states are never confirmed. | SYSTORE_BYPASS, PATH |
| 3 | High | **Power cycle / warm restart behavior.** After PLC restart, PATH instances have `NumberModules = 0` and `ValidMask = 0`. If `Init_MNG` is not called, all PATHs silently do nothing with no error. | PATH, PATH_MANAGEMENT |
| 4 | Medium | **DB_ANY absolute addressing** throughout PATH. Hardcoded offsets (`%DBX4.7`, `%DBX12.0`, `DBW14`) break silently if module DB structure changes. Consider constants or a documented interface. | PATH |
| 5 | Low | **All FBs are VERSION 0.1** with multiple placeholder comments, confirming pre-production status. | All |

---

## Summary by Priority

| Priority | Count |
|----------|-------|
| Critical | 5 |
| High     | 18 |
| Medium   | 22 |
| Low      | 8 |
| **Total** | **53** |

### Recommended Fix Order
1. Fix the **fault cascade feedback loop** in PATH_MANAGEMENT (affects all paths)
2. Fix **`StateChanged` timing** and **`ST_BYPASS_EXIT` single-scan escape** in SYSTORE_BYPASS
3. Add **handshake enables** on `PATH_E0003->E0006`
4. Add **timeout watchdogs** to PATH_MERGE and PATH_SPLIT
5. Implement **`ST_BYPASS_EXIT` motor verification** and **`ST_BYPASS_MOVECHECK`** logic
6. Add **`SrcReady`** check to PATH_SPLIT
7. Guard **`Hsk.Done` signals** with handshake-enable checks in PATH
8. Move **CFG writes** to init-only in PATH_MANAGEMENT
9. Add **HMI diagnostics** (fault codes, counters, session info)
10. Address remaining Medium/Low items
