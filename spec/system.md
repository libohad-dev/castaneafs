# System Design

## System Overview

CastaneaFS is a security-oriented FUSE filesystem with native ACID transaction support. It exposes a mostly POSIX-compliant interface (planned omissions: hardlinks, atime) to user-space processes and is designed around two core principles: strong access control guarantees and a capability-based transactional model.

### Transaction Model

Transactions are first-class filesystem primitives. When a client requests a new transaction, the CastaneaFS server issues two capability tokens:

- **Owner token**: Grants authority to commit or abort the transaction. Only the owner token holder can finalize the transaction's outcome.
- **Access token**: Grants authority to perform filesystem operations within the transaction, subject to the standard POSIX permission model. The access token can be distributed to multiple cooperating processes.

This two-token model separates the concern of *participating* in a transaction from *deciding its outcome*, enabling multi-process collaborative workflows where distinct processes contribute to a single atomic batch of filesystem changes while a designated coordinator retains commit authority.

> **Future feature**: Fine-grained token permissions (e.g., subtree-scoped access, read-only tokens, operation-type restrictions) are noted for potential future inclusion.

Transaction properties follow a modified ACID model:

- **Atomicity**: All operations within a transaction commit or none do.
- **Consistency**: Limited; the filesystem enforces structural invariants (e.g., directory tree integrity, permission coherence) but does not provide application-level constraint checking.
- **Isolation**: Concurrent transactions observe consistent snapshots and produce serializable outcomes.
- **Durability**: Committed transactions survive process and system crashes.

The temporary transaction-internal state serves dual purposes: it acts as a sandbox for security-sensitive work (within the filesystem's threat model) and as a disposable scratchpad for exploratory workflows such as testing different design or implementation configurations with easy rollback.

### Closed Nested Transaction Model

CastaneaFS employs a closed nested transaction model. Transactions form a hierarchy:

- Any participant in a transaction can create **subtransactions**, which operate on an isolated copy of their parent's state.
- A subtransaction's effects become visible to the parent (and its other subtransactions) only when the subtransaction commits.
- **Every individual filesystem operation** by a FUSE client is implicitly wrapped in a single-operation subtransaction that commits immediately. This means that non-transactional filesystem access is a special case of the transactional model, and races between concurrent participants — whether inside a shared transaction or between independent top-level operations — are handled by the same conflict detection logic.

This design yields several properties by construction:

- **Operation ordering**: Determined by the commit order of subtransactions, not by the arrival order of individual operations.
- **Intra-transaction visibility**: A participant sees another participant's writes only after the writing participant commits its subtransaction. Visibility is arbitrated by the timing relationship between one participant committing and another starting a new subtransaction.
- **Write conflict handling**: Intra-transaction write conflicts are a special case of conflict between subtransactions, which are handled identically to conflicts between any transactions (see Failed State below).

### Transaction States

A transaction (including subtransactions) is in exactly one of four states:

- **Open**: Accepts reads, writes, and subtransaction commits. This is the only state in which a transaction can receive new work.
- **Committed**: The transaction's changes have been merged into its parent's workspace (or, for a top-level transaction, into the filesystem). Terminal state.
- **Aborted**: The transaction's changes have been discarded and its resources cleaned up. Terminal state. The owner token holder may abort any open or failed transaction.
- **Failed**: The transaction is read-only. Its filesystem content is persisted and remains readable by access token holders, allowing them to compare the failed state with the live committed state and retry application-level actions. The owner token holder may abort a failed transaction to trigger cleanup.

State transitions:

```
Open ──commit succeeds──► Committed
Open ──commit fails─────► Failed
Open ──abort────────────► Aborted
Failed ─abort───────────► Aborted
```

No other transitions are possible. Committed and Aborted are terminal. Failed can only transition to Aborted (via the owner token holder).

### Commit Rules and Failure

A subtransaction's commit succeeds only if **all three** conditions hold:

1. The parent transaction is **open**.
2. The subtransaction's changes do not **conflict** with changes previously committed by a sibling subtransaction.
3. The subtransaction's changes do not violate **POSIX permission checks** (e.g., writing to a read-only file, modifying a file owned by a different user).

If any condition is violated, the committing subtransaction enters the **failed** state. This covers all failure cases uniformly:

- Parent is **failed**, **aborted**, or **committed** — condition 1 violated.
- Sibling write conflict — condition 2 violated.
- Permission violation — condition 3 violated.

**Implicit single-operation transactions and permission errors**: Every bare FUSE operation is wrapped in an implicit single-operation subtransaction (see Closed Nested Transaction Model above). When such an implicit subtransaction fails condition 3, the FUSE client simply aborts the subtransaction without persisting the failed state and returns the appropriate POSIX error code (e.g., `EACCES`, `EPERM`) to the calling process. This is indistinguishable from a standard POSIX permission error — no failed transaction state is visible to the user. In contrast, when an explicit subtransaction fails condition 3, the failed state is persisted as usual.

Failure cascades only at commit boundaries: if `T` is failed and `T.S1` is a subtransaction of `T`, then committing `T.S1` into `T` will fail `T.S1`. But `T.S1` remains open until that commit is attempted, so `T.S1.S2` may still successfully commit into `T.S1` before `T.S1` itself attempts to commit.

### Transaction Administration

There are no system-imposed limits on the number of open or failed transactions.

> **Future feature**: Configurable limits on transaction count, lifetime, and disk usage are noted for potential future inclusion.

As a stop-gap measure, a **CLI administration utility** (`castaneafs-admin` or similar) will be implemented that allows admin-privileged users to:

- List existing open and failed transactions with relevant metadata (disk usage, time of transaction start or failure, owner PID, etc.).
- Force cleanup of open or failed transactions.

### Actors

- **CastaneaFS server**: The FUSE daemon that manages the filesystem tree, enforces security policies, coordinates transactions, and persists data.
- **Client processes**: User-space programs that interact with the filesystem through standard POSIX syscalls and through a transaction control interface (for creating, joining, committing, and aborting transactions).
- **Transaction owner**: The process (or its delegates) holding the owner token for a transaction. Controls the transaction's lifecycle (commit/abort).
- **Transaction participants**: Processes holding an access token for a transaction. Can perform filesystem operations within the transaction's scope, subject to POSIX permission checks. May create subtransactions.
- **System administrator**: Privileged user who can inspect and force-cleanup transactions via the CLI administration utility.

### Technology Stack

- **Formal specification**: TLA+ (with potential TLAPS-powered proofs of formal properties) for modeling the filesystem, transaction mechanism, and security policies.
- **Implementation language**: F* (F-star), a proof-oriented programming language, for the server and reference client.
- **Integration testing**: Go with testcontainers for isolated behavioral tests and benchmarks, including demonstrations of security properties under concurrent multi-user scenarios (varying privilege levels, including root).

### Design Context

**Relationship to existing transactional systems.** CastaneaFS's transaction model combines multi-process shared transactions with closed nested subtransactions. Each of these mechanisms has precedent individually, but existing systems make different trade-offs:

- *Multi-process transaction coordination*: X/Open XA and WS-AtomicTransaction coordinate transactions across multiple participants, but each participant operates on its own local branch rather than a shared workspace. PostgreSQL's `PREPARE TRANSACTION` allows a transaction to be finalized by a different session, but cannot accept further operations after the prepare phase. Calvin (Thomson et al., 2012) batches operations from multiple clients but each transaction originates from a single client. CastaneaFS differs in that multiple processes contribute operations to a single shared workspace, mediated by capability tokens.
- *Nested transactions*: The theoretical framework for nested transactions was established by Moss (1981–1985), and the ARIES/NT recovery algorithm (Mohan et al., 1989) extends WAL-based recovery for this model with subtransaction tables, lock inheritance, and selective undo. However, production databases (PostgreSQL, Oracle, MySQL/InnoDB, SQL Server) universally implement savepoints rather than true nesting — savepoints provide partial rollback within a flat transaction but do not provide workspace isolation between regions. CastaneaFS requires genuine nesting semantics (workspace isolation, commit-time conflict detection, cascading failure) because every individual FUSE operation is implicitly wrapped in a subtransaction, making savepoint-style partial rollback insufficient.

**Compositionality of ACID properties.** Atomicity, Consistency, and Durability are compositional: pairwise satisfaction extends to arbitrary sets of transactions. Isolation (serializability) is **not** — it is a global property of the conflict graph (acyclicity over all transactions, not just pairs). A set of three transactions can each be pairwise serializable while the triple admits no serial order (cycle of length 3 in the conflict graph). Implication: CastaneaFS must enforce serializability by protocol construction (e.g., two-phase locking, timestamp ordering, serializable snapshot isolation) rather than relying on pairwise checks.

## Architectural Constraints

1. **No hardlinks**: The filesystem does not support hardlinks. Every file has exactly one parent directory. This simplifies reasoning about the directory tree as a true tree (not a DAG) and eliminates an entire class of access-control edge cases.

2. **No atime tracking**: Access time metadata is not maintained. This eliminates a source of write amplification on read-heavy workloads and removes a covert channel for information leakage about file access patterns.

3. **Two-token capability model**: Each transaction has two capability tokens: an owner token (commit/abort authority) and an access token (filesystem operation authority). Participation is governed solely by token possession. The access token grants full filesystem access within the transaction, restricted only by the standard POSIX permission model. Token secrecy is the security boundary.

4. **Token non-dissemination over network**: Transaction tokens are assumed to be distributed only among local processes (e.g., via Unix domain sockets, pipes, or shared memory). The system does not provide mechanisms for secure token transport over a network and does not guarantee security properties if tokens traverse network boundaries.

5. **Server-side transaction state**: Transaction state (operation queues, isolation bookkeeping, conflict detection) is maintained on the server side. Clients are stateless with respect to transaction coordination; they submit operations and receive results, but do not hold authoritative transaction state.

6. **Security-oriented conflict detection**: The filesystem's conflict detection extends beyond data-level conflicts to include security-relevant interactions between concurrent transactions. See the Security Conflict Policy section below for the full treatment.

7. **Serializable transaction isolation**: Concurrent transactions must produce outcomes equivalent to some serial execution order. The system may employ snapshot isolation, two-phase locking, or serializable snapshot isolation internally, but the externally observable behavior must be serializable.

8. **Crash recovery without data loss**: Committed transactions must survive crashes. The server must implement write-ahead logging or an equivalent mechanism to guarantee that committed data can be recovered after an unclean shutdown.

9. **Transaction liveness**: Every transaction must eventually either commit or abort. The system must prevent indefinite transaction stalls through timeout-based expiration, deadlock detection, or a combination of both. Orphaned tokens (from crashed processes) must not hold resources indefinitely.

10. **POSIX compliance (modulo stated omissions)**: All standard filesystem operations (open, read, write, close, mkdir, rmdir, rename, chmod, chown, stat, readdir, truncate, symlink, readlink, unlink, etc.) must behave according to POSIX semantics except where explicitly stated otherwise (hardlinks, atime).

11. **Closed nested transactions**: All filesystem operations — whether explicit transactions or bare FUSE operations — pass through the same transactional conflict detection mechanism. Bare operations are implicitly wrapped in single-operation subtransactions. This provides a uniform concurrency model with no special cases.

12. **Failed transactions are preserved**: A failed transaction's content persists in a read-only state until explicitly aborted by the owner. This enables application-level inspection and retry without requiring the server to support automatic conflict resolution or merge semantics.

13. **POSIX permission model only (initially)**: Access control uses standard POSIX uid/gid/mode bits. No ACLs, mandatory access control, or extended attributes in the initial implementation.

> **Future features noted for potential inclusion**:
> - Fine-grained token permissions (subtree-scoped, read-only, operation-type restrictions)
> - Configurable transaction count, lifetime, and disk usage limits
> - Extended security models (POSIX ACLs, MAC/label-based access control)
> - Extended attributes (xattr)
> - POSIX advisory locks / flock
> - File change notifications (inotify-style)
> - Network-safe token transport

## Security Conflict Policy

### Motivation

Standard transactional conflict detection operates at the data level: two transactions conflict when they access the same data and at least one is a write. CastaneaFS extends this with **security-oriented conflict detection** that treats certain combinations of concurrent operations as conflicts even when they do not touch the same data, because the combination could result in unintended information exposure.

The core observation is that application code has no general mechanism to detect or prevent security violations arising from concurrent permission changes. By the time a process learns that file permissions have changed, the data it wrote under the old permission assumptions may already be exposed. CastaneaFS addresses this by treating the conflict as a transaction-level concern: the filesystem does not attempt to understand whether written content is genuinely sensitive, but conservatively treats the combination of content modification and permission relaxation as a conflict.

### Established Rules

**Rule 1: Content write + permission relaxation on the same file.** If transaction A writes to a file and transaction B makes that file's permissions more permissive (e.g., adding world-readable bits), the two transactions conflict. Transaction A may have written data under the assumption that only the file's current permission set governs access; transaction B's change would retroactively violate that assumption upon commit.

**Rule 2: Content write + permission relaxation on an ancestor directory.** The conflict extends to the directory hierarchy. If transaction A creates or writes a file, and transaction B relaxes permissions on a directory anywhere in the file's ancestor chain, the transactions conflict. The new file or content could become inadvertently accessible through the newly-permissive directory, even though the file's own permission bits are restrictive.

CastaneaFS does not inspect file content to determine sensitivity. It treats any combination of "file content changed" and "file access made more permissive" as a conflict, where "more permissive" is evaluated relative to the permission state at the start of the conflicting transaction's snapshot.

**Rule 3: Uniform application across the transaction hierarchy.** Security conflict rules apply uniformly between all transactions — whether they are independent top-level transactions or sibling subtransactions of the same parent. CastaneaFS has no special rules distinguishing different levels of the transaction hierarchy. This falls out naturally from the general principle that all conflict detection uses the same mechanism regardless of nesting depth.

**Rule 4: Root exemption from security conflict checks.** The root user (uid 0) is exempt from security conflict checks, consistent with the POSIX convention that root is exempt from permission checks but bound by structural constraints. The reasoning is as follows:

- CastaneaFS's conflict detection addresses three categories of concern: *structural integrity* (e.g., creating a file in a deleted directory), *permission violations* (e.g., writing to a read-only file), and *security policy violations* (the rules above).
- In POSIX, structural constraints are impossible to violate (the operation has no meaning), while permission restrictions are merely access controls that root may bypass. Security policy violations — like permission violations — still leave the filesystem in a structurally valid state, so root is permitted to bypass them.
- For shared transactions, the effective user for security conflict checks is the user who issues the commit with the owner token. If root holds the owner token and commits, security conflict checks are skipped for that commit.

> **Warning — shared transactions with root**: Because root bypasses security conflict checks, sharing a transaction's access token with root-owned processes means those processes' modifications will not be security-checked at commit time. Users who are concerned about unintended data exposure should avoid sharing transactions with root-owned processes, or isolate root operations inside dedicated subtransactions whose results can be inspected before committing to the parent. This mirrors general security practice: limit root-level operations within user-owned activities.

### Open Questions

The following questions refine the boundaries of the security conflict detection rules. Answers will be incorporated as the policy is finalized during feature design.

1. **Does the reverse direction conflict?** If transaction A *reads* a file and transaction B relaxes that file's permissions, is that a conflict? The read itself does not change content, but the reader may have made decisions based on the assumption that the file was restricted.

2. **Ownership changes (chown)**: Are ownership transfers treated equivalently to permission bit changes? Changing a file's owner or group can effectively widen access if the new owner/group has broader membership. Should `chown` always be treated as a permission relaxation, or only when the new owner/group demonstrably broadens access?

3. **What constitutes "more permissive"?** Is any addition of permission bits sufficient (e.g., adding group-read to a file that was owner-only), or is the conflict limited to specific transitions (e.g., adding world-readable/writable)? How are setuid/setgid bit changes treated?

4. **File creation vs. file write**: Rule 2 treats file creation as a content write. Should the creation of an empty file (no content written) also conflict with ancestor permission relaxation, or only files with content?

5. **File deletion and permission relaxation**: If transaction A deletes a file and transaction B relaxes permissions on the directory, is that a conflict? The file no longer exists, so there is no content to expose — but the unlink operation itself reveals that a file existed at that path.

6. **Rename / move across directories**: If a file is renamed or moved into a directory with more permissive access, does that count as a permission relaxation on the file? Does moving a file *out of* a permissive directory and into a restrictive one interact with concurrent content writes?

7. **Symlinks**: Creating a symlink to a file in a more permissive directory could expose the target. Should symlink creation be treated as a permission-relevant operation on the target?

8. **Permission restriction (tightening)**: If transaction A makes permissions *more restrictive* while transaction B writes content, is that a conflict? Tightening permissions doesn't expose data, but it could cause transaction B's subsequent operations to fail unexpectedly if they depend on the original permission state.

## References

### Nested Transactions

- Moss (1981–1985). *Nested Transactions: An Approach to Reliable Distributed Computing*. MIT Press. — Theoretical framework for transaction trees, lock inheritance, partial rollback, and concurrency within transactions. [Archive.org](https://archive.org/details/nestedtransactio00jeli)
- Mohan et al. (1989). "ARIES/NT: A Recovery Method Based on Write-Ahead Logging for Nested Transactions." — Extends ARIES for nested transactions with subtransaction tables, lock inheritance, and selective undo. [VLDB proceedings](https://www.vldb.org/conf/1989/P337.PDF)
- Weikum. "Multi-level Transactions." — Layered architectures with different isolation granularities at each level. [ACM DL](https://dl.acm.org/doi/abs/10.1145/103140.103145)

### Transaction Isolation and Serializability

- Weikum & Vossen (2001). *Transactional Information Systems: Theory, Algorithms, and the Practice of Concurrency Control and Recovery*. Morgan Kaufmann. — Canonical reference on serializability theory, concurrency control, and recovery. [Archive.org](https://archive.org/details/transactionalinf0000weik)
- Berenson et al. (1995). "A Critique of ANSI SQL Isolation Levels." — Defines snapshot isolation, write skew, and phenomena not captured by the ANSI SQL standard.
- Cahill et al. (2008). "Making Snapshot Isolation Serializable." — The SSI algorithm, now in PostgreSQL's SERIALIZABLE isolation level.
- Fekete et al. "Read-Only Anomaly." — Anomalies in snapshot isolation beyond write skew. [Paper](https://www.cs.umb.edu/~poneil/ROAnom.pdf)

### Crash Recovery

- Mohan et al. (1992). "ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging." — The standard for WAL-based crash recovery. [Paper](https://web.stanford.edu/class/cs345d-01/rl/aries.pdf)

### POSIX Filesystem Semantics

- Ridge et al. (2015). "SibylFS: Formal Specification and Oracle-Based Testing for POSIX and Real-World File Systems." SOSP. — Rigorous formal specification of POSIX filesystem semantics using Lem (translates to OCaml). Defines the POSIX baseline that CastaneaFS must match. [Site](https://sibylfs.github.io/), [Paper](https://www.cl.cam.ac.uk/~pes20/SOSP15-paper102-submitted.pdf)

### TLA+ Specification Patterns

- **TLA+ Examples Repository**: [TCommit.tla](https://github.com/tlaplus/Examples/blob/master/specifications/transaction_commit/TCommit.tla) (transaction commit), [TwoPhase.tla](https://github.com/tlaplus/Examples/blob/master/specifications/transaction_commit/TwoPhase.tla) (two-phase commit with refinement proof). [Repository](https://github.com/tlaplus/Examples)
- **MongoDB Distributed Transactions (VLDB 2025)**: MultiShardTxn.tla and Storage.tla — snapshot isolation with model-based testing infrastructure. [Repository](https://github.com/mongodb-labs/vldb25-dist-txns), [Paper](https://www.vldb.org/pvldb/vol18/p5045-schultz.pdf)
- **Snapshot Isolation Specification**: TLA+ spec for SI with write skew and read-only anomaly examples. [Repository](https://github.com/will62794/snapshot-isolation-spec)
- **UseCON**: TLA+ specifications for Usage Control (UCON) access control models. [Repository](https://github.com/agouglidis/UseCON-TLA_PLUS), [Foundational paper](https://profsandhu.com/it862/it862s05/ucon-tla1.pdf)

### Verification Practices

- AWS formal methods: TLA+ applied to DynamoDB, S3, and EBS. [Paper](https://lamport.azurewebsites.net/tla/formal-methods-amazon.pdf)
- MongoDB: Conformance checking of implementation against TLA+ specs. [Blog](https://www.mongodb.com/company/blog/engineering/conformance-checking-at-mongodb-testing-our-code-matches-our-tla-specs)
- Jepsen: Black-box transactional consistency testing. Includes [Elle](https://github.com/jepsen-io/elle) checker and [consistency models reference](https://jepsen.io/consistency). [Site](https://jepsen.io/)

## Feature Index

*No features designed yet. Use `/formspec.1.design <feature-name>` to add features.*

## Changelog

- **2026-02-28**: Initial system design. Established two-token capability model, closed nested transaction model, failed transaction state semantics, POSIX-only permission model, CLI admin utility, and deferred persistence backend.
- **2026-02-28**: Added Design Context (novelty analysis, ACID compositionality, nested transaction adoption) and References section (foundational literature, TLA+ specifications, testing tooling, industry practice, learning resources).
