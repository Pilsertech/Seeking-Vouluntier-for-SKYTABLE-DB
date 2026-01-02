# Skytable for Production Auth Systems: Critical Analysis

**TL;DR**: Skytable has fundamental reliability issues that make it unsuitable for authentication systems. This is well-documented in the code.

---

## Quick Facts

| Factor | Details |
|--------|---------|
| **Data Loss Risk** | Up to 5 minutes of writes lost on crash |
| **Transaction Support** | None (DDL-only, no DML transactions) |
| **Isolation Guarantees** | None (dirty reads, phantom reads possible) |
| **GNS Reliability** | Single point of failure - database offline if corrupted |
| **Crash Recovery** | Conservative but destructive (discards "suspicious" data) |
| **Target Use Case** | Caching and analytics, NOT critical data |

---

## The Five Critical Issues

### 1. Delayed Durability (Data Loss Risk)
**Evidence**: `server/src/engine/fractal/mgr.rs:490-520`

Writes stay in memory for up to 5 minutes before flushing to disk. Crash = data loss.

```
Timeline of a write:
T+0s    User executes write (INSERT user account)
T+0s    Data stored in memory buffer
T+0.1s  Client receives "success" response
T+0.1s - T+300s  Data in memory only
T+300s  Background flush writes to disk
OR
T+100s  SERVER CRASHES → Data lost
```

### 2. No DML Transaction Support (Data Corruption Risk)
**Evidence**: `server/src/engine/txn/mod.rs` (only GNS transactions) + `server/src/engine/ql/dml/` (no transaction context)

Multi-row operations not atomic. Crash mid-update leaves inconsistent state.

```
Scenario: Create user account + Initialize settings
Op 1: INSERT users (succeeds)
Op 2: INSERT user_settings (server crashes)
Result: Orphaned user with no settings
        OR: User exists but isn't active
        OR: Partial data visible to readers
```

### 3. GNS Single-Point-of-Failure (Database Crash Risk)
**Evidence**: `server/src/engine/storage/v2/impls/gns_log.rs` (no checksums)

All schema/metadata in one file. Corruption = entire database unreachable.

```
Corruption detected:
  Database won't start
  Must run: skyd repair
  Repair discards "suspicious" data
  User accounts lost
  Password hashes lost
  Sessions lost
  → Restore from backup (if available)
```

### 4. No Isolation Guarantees (Race Conditions)
**Evidence**: `server/src/engine/ql/dml/sel.rs` + `server/src/engine/core/index/row.rs:41` (RwLock per row)

Concurrent transactions see inconsistent state.

```
Race condition example:
Thread A reads: account_balance = 100
Thread B writes: account_balance = 50
Thread A writes: account_balance = 100 + 50 = 150 (should be 50!)

For auth:
Thread A counts failed logins: 4 attempts
Thread B inserts attempt #4 (while A is checking)
Thread A decides: not locked (count = 4)
Thread C inserts attempt #5 while A executing
Result: Account not locked, brute force attack works
```

### 5. Crash Recovery is Destructive (Data Recovery Risk)
**Evidence**: `server/src/engine/storage/v2/raw/journal/raw/mod.rs:100-150`

Recovery discards uncertain data rather than restore it.

```
Crash during batch flush:
  1,247 pending writes in queue
  GNS partially written
  Recovery scans to last certain point: 5 minutes ago
  Decision: discard last 5 minutes of changes
  Result: 1,247 inserts/updates lost
           1 month of auth logs lost
           All password resets from 5min window lost
```

---

## Recommendation

For critical systems, use the right tool for the job. Skytable is the wrong tool for auth systems.

---

**Questions? Discussion?** Create a GitHub issue referencing these documents.

**Want to help fix Skytable?** Start with [SKYTABLE_IMPLEMENTATION_GUIDE.md](SKYTABLE_IMPLEMENTATION_GUIDE.md) Phase 1.

**Want to migrate?** Consider SQLite or PostgreSQL for reliable auth systems.
