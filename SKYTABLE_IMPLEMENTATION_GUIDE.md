# Skytable Reliability Fixes: Implementation Guide

> Detailed code locations, specific changes required, and implementation roadmap

---

## Part 1: Critical Code Locations Map

### Issue #1: Delayed Durability

#### Primary Location: Batch Flush Timer
**File**: `server/src/engine/fractal/mgr.rs`  
**Lines**: 490-520  
**Function**: `general_executor_svc`

```rust
// CURRENT IMPLEMENTATION (Delayed)
pub async fn general_executor_svc(
    &'static self,
    global: super::Global,
    mut lpq: UnboundedReceiver<Task<GenericTask>>,
    mut sigterm: broadcast::Receiver<()>,
    rs_window: u64,  // ← Delay in seconds (default 300)
) {
    let dur = std::time::Duration::from_secs(rs_window);
    loop {
        tokio::select! {
            _ = sigterm.recv() => { ... },
            _ = tokio::time::sleep(dur) => {  // ← Timer-based, not immediate!
                let global = global.clone();
                let _ = tokio::task::spawn_blocking(
                    || self.general_executor(global)
                ).await;
            },
            task = lpq.recv() => { ... }
        }
    }
}

// general_executor() iterates all models and flushes
fn general_executor(&'static self, global: super::Global) {
    for (model_id, model) in global.state().namespace().idx_models().read().iter() {
        let observed_len = model.data().delta_state().__fractal_take_full_from_data_delta(...);
        // Only writes if observed_len > 0
        match self.try_write_model_data_batch(...) {
            Ok(()) => { ... },
            Err((e, stats)) => { ... } // Retry on error
        }
    }
}
```

**What needs to change:**
1. Add `sync_mode: bool` configuration option
2. When `sync_mode=true`, flush immediately after every write
3. Remove the `tokio::time::sleep(dur)` branch or make it optional
4. Call fsync() on journal file after each batch commit

---

## Part 2: Implementation Roadmap

### Phase 1: Foundation (1-2 weeks)
**Goal**: Setup testing infrastructure and create transaction skeleton

**Tasks:**
1. Create `server/src/engine/txn/dml/` module structure
2. Create `DMLTransaction` struct with basic fields
3. Create comprehensive test suite for each phase
4. Add sync mode configuration option

**Acceptance Criteria:**
- ✅ Module structure compiles
- ✅ Basic transaction tests pass
- ✅ Configuration loads correctly

---

### Phase 2: Sync Durability (1-2 weeks)
**Goal**: Guarantee writes reach disk immediately

**Tasks:**
1. Implement `sync_mode` flush logic in fractal manager
2. Add fsync() to journal writer
3. Test with failure injection
4. Benchmark performance impact

**Acceptance Criteria:**
- ✅ Data reaches disk before write completes
- ✅ Crash immediately after write leaves no data in memory
- ✅ Recovery test passes
- ✅ Performance acceptable (define threshold)

---

### Phase 3: GNS Hardening (1-2 weeks)
**Goal**: Prevent GNS corruption from crashing database

**Tasks:**
1. Add checksums to GNS metadata
2. Implement backup GNS file
3. Add recovery strategy
4. Test corruption scenarios

**Acceptance Criteria:**
- ✅ Checksum verified on every GNS load
- ✅ Backup created on every GNS write
- ✅ Recovery uses backup if primary corrupt
- ✅ Corruption test: corrupt primary, verify backup restores

---

### Phase 4: DML Transactions (3-4 weeks)
**Goal**: Multi-key operations atomic and isolated

**Tasks:**
1. Wire transaction context through DML execution
2. Implement row lock acquisition in deadlock-free order
3. Implement write buffering (don't commit until tx_end)
4. Implement undo logging for rollback
5. Comprehensive transaction tests

**Acceptance Criteria:**
- ✅ All rows locked before any changes
- ✅ All changes buffered, not applied until commit
- ✅ Rollback available until commit
- ✅ Concurrent txn test: no lost updates
- ✅ Concurrent txn test: no phantom reads

---

### Phase 5: Isolation Levels (2-3 weeks)
**Goal**: Enforce isolation guarantees per transaction

**Tasks:**
1. Implement isolation level enum
2. MVCC for snapshot consistency
3. Conflict detection
4. Test each isolation level

**Acceptance Criteria:**
- ✅ READ_UNCOMMITTED: allows dirty reads
- ✅ READ_COMMITTED: prevents dirty reads
- ✅ REPEATABLE_READ: prevents phantom reads
- ✅ SERIALIZABLE: equivalent to serial execution
- ✅ Test suite verifies each level

---

### Phase 6: Crash Consistency (3-4 weeks)
**Goal**: No data loss even with crash at any point

**Tasks:**
1. Implement 2-phase commit protocol
2. Add pre-image logging
3. Add commit barriers
4. Comprehensive crash injection tests

**Acceptance Criteria:**
- ✅ Undo log created for all changes
- ✅ Prepare marker written before data
- ✅ Commit marker written after data
- ✅ fsync() called at commit point
- ✅ 20+ crash injection tests pass
- ✅ Recovery test: crash at each step, verify correct behavior

---

### Phase 7: Comprehensive Testing & Hardening (4-6 weeks)
**Goal**: Production-ready reliability

**Tasks:**
1. Stress testing with concurrent workloads
2. Chaos engineering (random failures)
3. Performance profiling & optimization
4. Documentation & training

**Test Matrix:**
- 10,000+ concurrent transactions
- Random crash injection at each byte of write
- GNS corruption injection
- Disk I/O delays
- Memory pressure scenarios
- Deadlock detection scenarios

**Success Criteria:**
- ✅ All tests pass under stress
- ✅ No data loss under any crash scenario
- ✅ Performance overhead <20% vs original
- ✅ Zero deadlock scenarios found

---

## Conclusion

The implementation requires careful planning and execution across 7 phases, with comprehensive testing at each step. This is a significant undertaking requiring experienced database engineers.

---

**Document Status**: Ready for development  
**Last Updated**: January 2, 2026
