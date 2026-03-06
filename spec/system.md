# System Design

## System Overview

CastaneaFS is a security-oriented FUSE filesystem with native ACID transaction support. It exposes a mostly POSIX-compliant interface (planned omissions: hardlinks, atime) to user-space processes and is designed around two core principles: strong access control guarantees and a capability-based transactional model.

### Transaction Model

Transactions are first-class filesystem primitives. When a client requests a new transaction, the CastaneaFS server issues two capability tokens:

- **Owner token**: Grants authority to commit or abort the transaction. Only the owner token holder can finalize the transaction's outcome.
- **Access token**: Grants authority to perform filesystem operations within the transaction, subject to the standard POSIX permission model. The access token can be distributed to multiple cooperating processes.

This two-token model separates the concern of *participating* in a transaction from *deciding its outcome*, enabling multi-process collaborative workflows where distinct processes contribute to a single atomic batch of filesystem changes while a designated coordinator retains commit authority.

> **Future feature**: Fine-grained token permissions (e.g., subtree-scoped access, read-only tokens, operation-type restrictions) are noted for potential future inclusion.

Initiating a transaction isolates a snapshot of the filesystem (or parent transaction) state at that point in time and issues the two tokens. All reads within the transaction see this snapshot, plus any changes committed by subtransactions since the snapshot was taken.

Transaction properties follow a modified ACID model:

- **Atomicity**: All operations within a transaction commit or none do.
- **Consistency**: Limited; the filesystem enforces structural invariants (e.g., directory tree integrity, permission coherence) but does not provide application-level constraint checking.
- **Isolation**: Concurrent transactions observe consistent snapshots and produce serializable outcomes.
- **Durability**: Committed transactions survive process and system crashes.

The temporary transaction-internal state serves dual purposes: it acts as a sandbox for security-sensitive work (see Threat Model below) and as a disposable scratchpad for exploratory workflows such as testing different design or implementation configurations with easy rollback.

### Closed Nested Transaction Model

CastaneaFS employs a closed nested transaction model. Transactions form a hierarchy:

- Any participant in a transaction can create **subtransactions** using the same ioctl-based API used for top-level transactions. Creating a subtransaction issues a new pair of owner and access tokens for the subtransaction, scoped to it. The subtransaction operates on an isolated copy of its parent's state at the time of creation.
- A subtransaction's effects become visible to the parent (and its other subtransactions) only when the subtransaction commits. The subtransaction's access token can be shared with other processes, forming a nested workgroup.
- The server implicitly wraps **every individual filesystem operation** in a single-operation subtransaction that commits immediately. If the calling process is not participating in any explicit transaction, this implicit subtransaction commits directly to the **hierarchy root** — the filesystem itself, which serves as the always-open implicit parent of all top-level transactions. This means that non-transactional filesystem access is a special case of the transactional model, and races between concurrent participants — whether inside a shared transaction or between independent top-level operations — are handled by the same conflict detection logic. The wrapping is transparent to the calling process — it issues standard POSIX operations and is unaware of the subtransaction mechanism.

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

Conflict detection is **optimistic**: operations within a subtransaction execute without blocking, and all validation occurs at commit time. A subtransaction's commit succeeds only if **all three** conditions hold:

1. The parent transaction is **open**.
2. The subtransaction's changes do not **conflict** with changes previously committed by a sibling subtransaction. This includes both data-level conflicts (concurrent writes to the same conflict domain of a file or directory entry — see Conflict Domains) and security conflicts (see Security Conflict Policy).
3. The subtransaction's changes do not violate **POSIX permission checks** at commit time (e.g., writing to a read-only file, modifying a file owned by a different user). Permissions are evaluated against the parent's state at the time of commit, not at the time the operation was performed within the subtransaction. For implicit single-operation subtransactions, commit is immediate, so this is equivalent to checking at operation time.

If any condition is violated, the committing subtransaction enters the **failed** state. This covers all failure cases uniformly:

- Parent is **failed**, **aborted**, or **committed** — condition 1 violated.
- Sibling write conflict — condition 2 violated.
- Permission violation — condition 3 violated.

**Implicit single-operation transactions and permission errors**: The server wraps every bare FUSE operation in an implicit single-operation subtransaction (see Closed Nested Transaction Model above). When such an implicit subtransaction fails condition 3, the server simply aborts the subtransaction without persisting the failed state and returns the appropriate POSIX error code (e.g., `EACCES`, `EPERM`) to the calling process. This is indistinguishable from a standard POSIX permission error — no failed transaction state is visible to the user. In contrast, when an explicit subtransaction fails condition 3, the failed state is persisted as usual.

Failure cascades only at commit boundaries: if `T` is failed and `T.S1` is a subtransaction of `T`, then committing `T.S1` into `T` will fail `T.S1`. But `T.S1` remains open until that commit is attempted, so `T.S1.S2` may still successfully commit into `T.S1` before `T.S1` itself attempts to commit. This is by design: subtransactions of a failed parent remain live, supporting use cases where the transaction serves as an experimentation sandbox with no intention of eventually committing (e.g., testing configurations, comparing alternatives). There is no mechanism to notify subtransaction participants of a parent's failure other than the failure they observe when they attempt to commit — the system does not proactively cascade failure or revoke access to open subtransactions.

### Hierarchy Root

The base filesystem — the hierarchy root into which top-level transactions commit — is modeled as an always-open transaction. It follows the same lifecycle rules as any other transaction, with differences arising only from external factors, not from special-casing in the transaction model:

- **State**: Always open. No process holds the owner token, so no process can commit or abort it. The state transitions defined above are theoretically applicable but can never be actuated.
- **Access**: The primary mount serves the root transaction's state as a transaction view mount. Unlike other transaction view mounts, the primary mount does not require an access token to create. Equivalently, the root transaction's access token can be thought of as publicly available — a kind of UUID identifying the filesystem — rather than a secret capability.
- **Durability**: The spec guarantees durability for transactions that commit into the hierarchy root (see constraint 8). This is the only level of the hierarchy for which the spec mandates durability. Implementations may provide durability for in-progress transactions at other levels, but the spec provides no mechanism to reissue transaction tokens after a crash, so applications are responsible for persisting their own tokens if they wish to attempt recovery.
- **Parent reference**: The root is the only transaction with no parent. This is an inherent structural property of the tree, not a behavioral difference — no caller can ever attempt to commit the root (the owner token is inaccessible), so the absence of a parent has no observable consequence. Implementations are free to represent this in whatever data model suits them (nullable reference, sentinel value, cyclic self-reference, etc.).

### Transaction Administration

There are no system-imposed limits on the number of open or failed transactions.

> **Future feature**: Configurable limits on transaction count, lifetime, and disk usage are noted for potential future inclusion.

As a stop-gap measure, the **`castaneafs` CLI tool** will provide administration commands that allow admin-privileged users to:

- List existing open and failed transactions with relevant metadata (disk usage, time of transaction start or failure, owner PID, etc.).
- Force cleanup of open or failed transactions.

### Actors

- **CastaneaFS server**: The FUSE daemon that manages the filesystem tree, enforces security policies, coordinates transactions, and persists data.
- **Client processes**: User-space programs that interact with the filesystem through standard POSIX syscalls and through a transaction control interface (for creating, joining, committing, and aborting transactions).
- **Transaction owner**: The process (or its delegates) holding the owner token for a transaction. Controls the transaction's lifecycle (commit/abort).
- **Transaction participants**: Processes holding an access token for a transaction. Can perform filesystem operations within the transaction's scope, subject to POSIX permission checks. May create subtransactions.
- **System administrator**: Privileged user who can inspect and force-cleanup transactions via the `castaneafs` CLI tool.

### Transaction Control Interface

Transaction lifecycle operations (create, commit, abort, create subtransaction) are exposed via **ioctl calls** on the CastaneaFS mount. The call to create a transaction returns the access token; the owner token is returned separately. Specific ioctl numbers and argument structures are not yet finalized and will be defined during feature design.

Filesystem operations within a transaction's scope are performed through standard POSIX syscalls, with the transaction context identified by its access token.

### Technology Stack

- **Formal specification**: TLA+ (with potential TLAPS-powered proofs of formal properties) for modeling the filesystem, transaction mechanism, and security policies.
- **Implementation language**: F* (F-star), a proof-oriented programming language, for the server and the `castaneafs` CLI administration tool.
- **Integration testing**: Go with testcontainers for isolated behavioral tests and benchmarks, including demonstrations of security properties under concurrent multi-user scenarios (varying privilege levels, including root).

### Design Context

**Relationship to existing transactional systems.** CastaneaFS's transaction model combines multi-process shared transactions with closed nested subtransactions. Each of these mechanisms has precedent individually, but existing systems make different trade-offs:

- *Multi-process transaction coordination*: X/Open XA and WS-AtomicTransaction coordinate transactions across multiple participants, but each participant operates on its own local branch rather than a shared workspace. PostgreSQL's `PREPARE TRANSACTION` allows a transaction to be finalized by a different session, but cannot accept further operations after the prepare phase. Calvin (Thomson et al., 2012) batches operations from multiple clients but each transaction originates from a single client. CastaneaFS differs in that multiple processes contribute operations to a single shared workspace, mediated by capability tokens.
- *Nested transactions*: The theoretical framework for nested transactions was established by Moss (1981–1985), and the ARIES/NT recovery algorithm (Mohan et al., 1989) extends WAL-based recovery for this model with subtransaction tables, lock inheritance, and selective undo. However, production databases (PostgreSQL, Oracle, MySQL/InnoDB, SQL Server) universally implement savepoints rather than true nesting — savepoints provide partial rollback within a flat transaction but do not provide workspace isolation between regions. CastaneaFS requires genuine nesting semantics (workspace isolation, commit-time conflict detection, cascading failure) because every individual FUSE operation is implicitly wrapped in a subtransaction, making savepoint-style partial rollback insufficient.

**Compositionality of ACID properties.** Atomicity, Consistency, and Durability are compositional: pairwise satisfaction extends to arbitrary sets of transactions. Snapshot isolation's write conflict detection is pairwise (first-committer-wins on each item independently), but the security conflict extensions introduce cross-item conflict rules (e.g., a write to file F conflicts with a permission relaxation on F's ancestor directory) that span multiple objects. These cross-item rules must be evaluated against the full committed state of the parent at commit time, not just pairwise against individual sibling transactions, because the relevant permission state may have been established by a chain of sibling commits.

### Conflict Domains

CastaneaFS partitions the modifications a transaction can make to a single inode into independent **conflict domains**. Two concurrent modifications to the same inode conflict (under commit rule 2) only when they fall in the same domain. Cross-domain interactions on the same inode are mediated by the security conflict rules (Rules 1–3) and by POSIX permission checks at commit time (commit rule 3), not by standard write-write conflict detection.

**Content domain.** File data operations (reads, writes, truncates) and directory entry operations (create, delete, rename). Standard first-committer-wins applies within this domain: if a transaction modifies the content domain of an inode and that domain has changed in the parent since the transaction's snapshot, the commit fails.

**Permission metadata domain.** Permission bit changes (`chmod`) and ownership changes (`chown`, `chgrp`). This domain has reconciliation rules beyond simple first-committer-wins:

- **Permission bits (mode).** When a transaction T commits having modified an inode's permission bits, the server performs a current-state comparison using three values: T's snapshot start `S`, T's end state `E`, and the parent's current committed state `P` for that inode. If T's change is a *pure expansion* (`E | S == E`, meaning T only added bits, never removed any) and the parent's cumulative change since T's snapshot is also a pure expansion (`P | S == P`), the changes merge: the result is `E | P` (bitwise OR). If either delta removed any bit, standard first-committer-wins applies (T fails if `P ≠ S`). The parent's current state `P` already encapsulates all intermediate sibling commits regardless of when those transactions started or how many committed in between; no enumeration of individual siblings is needed. Rationale: a relative `chmod g+r` translates to an absolute mode where the group-read bit is added and all other bits are preserved — a pure expansion from the snapshot. An absolute `chmod 640` that removes bits the snapshot had is expressing intent to restrict, which may contradict a concurrent expansion. The server cannot distinguish "didn't care about this bit" from "deliberately removed this bit" since both produce the same absolute end state.

- **Ownership (uid, gid).** Ownership is a discrete value, not a bitmask; no natural merge operation exists. When a transaction T commits having modified an inode's ownership, the server compares T's final (uid, gid) against the parent's current committed (uid, gid). If they are equal (convergent), there is no conflict. If they differ (divergent), first-committer-wins applies (T fails if the parent's ownership has changed since T's snapshot).

The permission bit reconciliation is structurally a **state-based CRDT** (conflict-free replicated data type): the merge function (bitwise OR) is commutative, associative, idempotent, and monotone over the subset lattice of permission bits. These algebraic properties guarantee that the merged result is independent of the order in which transactions commit — two pure expansions produce the same merged state regardless of commit sequence. The monotonicity of OR over the subset relation is also what makes the security checks composable: if the current parent state does not exceed the writer's security reference, no future pure-expansion merge can cause it to do so, because OR can only add bits, and each merge is checked at its own commit time. This current-state comparison approach — validating the committing transaction against the parent's committed state rather than enumerating individual concurrent transactions — follows the same pattern used by optimistic concurrency control systems generally: FoundationDB's Resolver checks a transaction's read set against a unified map of recent committed writes; CockroachDB's read-refresh validates against the MVCC version history in a timestamp range; PostgreSQL's snapshot isolation checks a tuple's `xmax` against the transaction's snapshot. In all cases, the current state subsumes the chain of intermediate commits. CastaneaFS's contribution is applying reconciliation (not just abort) within this model, using the algebraic properties of the merge operation to guarantee that the reconciled result is safe without requiring knowledge of the individual transactions that produced the current state.

**Other inode metadata.** Timestamps and file size are side effects of content-domain operations and do not form an independent conflict domain. Link count changes are directory-entry operations in the content domain.

**Cross-domain interactions.** Content writes and permission metadata changes on the same inode do **not** produce a standard write-write conflict (they are in different domains). Instead, three independent checks mediate their interaction:

1. **Content domain conflict** (commit rule 2, condition 2): standard first-committer-wins on concurrent writes to the same file data or directory entry. Applies only within the content domain.
2. **Security conflict** (Rules 1–3): does the permission change expose the written content beyond what the writer approved? Evaluated using the reference permission state defined in the Security Conflict Policy section.
3. **POSIX permission check** (commit rule 3, condition 3): are the writer's operations still permitted under the parent's permission state at commit time? A permission tightening that commits first may cause a concurrent write to fail its POSIX check — this is the standard mechanism, not a special case.

All three checks are independent. A commit fails if any check fails.

## Architectural Constraints

1. **No hardlinks**: The filesystem does not support hardlinks. Every file has exactly one parent directory. This simplifies reasoning about the directory tree as a true tree (not a DAG) and eliminates an entire class of access-control edge cases.

2. **No atime tracking**: Access time metadata is not maintained. This eliminates a source of write amplification on read-heavy workloads and removes a covert channel for information leakage about file access patterns.

3. **Two-token capability model**: Each transaction has two capability tokens: an owner token (commit/abort authority) and an access token (filesystem operation authority). Participation is governed solely by token possession. The access token grants full filesystem access within the transaction, restricted only by the standard POSIX permission model. Token secrecy is the security boundary.

4. **Token non-dissemination over network**: Transaction tokens are assumed to be distributed only among local processes (e.g., via Unix domain sockets, pipes, or shared memory). The system does not provide mechanisms for secure token transport over a network and does not guarantee security properties if tokens traverse network boundaries.

5. **Server-side transaction state**: Transaction state (operation queues, isolation bookkeeping, conflict detection) is maintained on the server side. Clients are stateless with respect to transaction coordination; they submit operations and receive results, but do not hold authoritative transaction state.

6. **Security-oriented conflict detection**: The filesystem's conflict detection extends beyond data-level conflicts to include security-relevant interactions between concurrent transactions. Content writes and permission metadata changes are in separate conflict domains (see Conflict Domains), so they do not trigger standard write-write conflicts with each other. Cross-domain interactions are mediated by the security conflict rules (see Security Conflict Policy) and by POSIX permission checks at commit time (commit rule 3).

7. **Snapshot isolation with security extensions**: Concurrent transactions operate on consistent snapshots and are subject to first-committer-wins write conflict detection within each conflict domain. Read-write conflicts are not tracked; write skew is permitted, consistent with the system's position that application-level invariants are outside its scope (see Consistency above). The permission metadata domain has additional reconciliation beyond first-committer-wins: concurrent pure-expansion permission bit changes merge via bitwise OR, and convergent ownership changes do not conflict (see Conflict Domains). The security conflict rules (see Security Conflict Policy) extend the base snapshot isolation model with additional conflict detection for security-relevant cross-domain operation combinations that SI alone would permit.

8. **Crash recovery without data loss**: Data that has been committed to the base filesystem (the hierarchy root) must survive crashes. The server must implement write-ahead logging or an equivalent mechanism to guarantee that committed data can be recovered after an unclean shutdown. Subtransactions that have committed into an open parent are durable only in the sense that the parent's workspace reflects them — if the parent's workspace is lost (e.g., the server crashes and the implementation does not persist in-progress transaction state), those commits are lost with it. This is consistent with the uniform treatment of hierarchy levels: a subtransaction commits "durably" into its parent, just as a top-level transaction commits durably into the base filesystem — the difference is that the parent may itself be volatile. Implementations may provide stronger guarantees (e.g., persisting in-progress transaction state for crash recovery), but the spec does not require this and provides no mechanism to reissue transaction tokens after a crash — applications that wish to resume in-progress transactions across crashes must store their tokens securely through their own means.

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
> - Directory FD capability attenuation (ioctl to narrow subtree-capable directory FDs to listing-only before delegation)
> - Preemptive cascading cleanup (aborting a transaction forcibly aborts all its open descendant subtransactions)
> - Read-only transaction view mounts (mount a transaction's state as a read-only filesystem, preventing writes through the mount regardless of POSIX permissions)
> - Automatic unmount of inert transaction view mounts (detecting when no process references the mount and unmounting without manual intervention)
>
> **Implementation note**: Implementations may impose a maximum nesting depth for subtransactions as a resource management measure. This is not a design-level constraint — the abstract model permits unbounded nesting.

### Link Types: Design Rationale

**Hardlinks** are not supported (constraint 1). A hardlink creates a second directory entry pointing to the same inode, giving a file multiple parents. This breaks the tree invariant that CastaneaFS relies on: with a single parent per file, the ancestor chain is unique and unambiguous, and security conflict rules (Rule 2: permission relaxation on an ancestor directory) can be evaluated by walking a single path to the root. Hardlinks would require evaluating Rule 2 against *every* ancestor chain, and a permission relaxation on a directory in *any* of them would need to be detected — turning a single-path check into a multi-path check whose cost grows with the number of links. Worse, hardlinks can be created concurrently, so the set of ancestor chains is itself subject to transactional races. Eliminating hardlinks removes this complexity entirely.

**Symlinks** are supported and receive no special treatment in the security conflict rules. A symlink is a file whose content is a path string — the kernel resolves it transparently during path traversal, and permission checks are evaluated on the *target's* real path, not the symlink's location. Creating a symlink in a world-readable directory to a restricted file does not grant access to the file's content; the reader must still pass permission checks on the target and its ancestor directories.

However, symlinks do create an indirect discoverability channel: a symlink's content (the target path) is readable via `readlink()` by anyone with access to the symlink, and making the symlink more discoverable (e.g., by relaxing permissions on its parent directory) increases the exposure of the target's path. This is a form of metadata leakage — the target's *existence and location* are revealed, even though its *content* is not.

CastaneaFS does not guard against this because it does not inspect file content, and a symlink's target path is file content. A regular file containing the same path string would pose the same discoverability risk, and CastaneaFS cannot distinguish the two without content inspection, which is outside its scope. This is a known limitation: the security conflict rules protect against unintended content exposure through permission changes, but do not protect against metadata leakage through symlink (or file content) discoverability.

> **Warning — symlinks and data discoverability**: Symlinks can make restricted file paths discoverable to users who lack access to the target. CastaneaFS's security conflict rules do not detect or prevent this because they operate on filesystem operations and metadata, not file content. Users working with sensitive file paths should be aware that creating symlinks to those paths in more permissive locations will expose the paths (though not the content) to a wider audience.

## Threat Model

### Trust Boundaries

- **Trusted**: The kernel, the FUSE kernel module, and the CastaneaFS server process. These are assumed to be correct and uncompromised.
- **Untrusted**: All client processes. The server must validate every request and never rely on client-side state or client-side enforcement.
- **Security boundary**: Token secrecy. A process can access a transaction's workspace if and only if it possesses a valid access token. The system does not authenticate processes by identity (uid/pid); it authenticates by token possession.

### Unauthorized Access to Transaction State

Transaction-internal state must be invisible to processes without the access token. Two naive approaches are rejected:

- **Path-based namespaces** (e.g., `/mountpoint/.txn/<token>/...`) would leak the token through `/proc/<pid>/fd` entries, `lsof`, and similar introspection tools.
- **Uncontrolled per-transaction mounts** where any token holder can create a globally-visible mount would expose transaction state to all processes that can traverse the mount path.

CastaneaFS uses a **multi-mount architecture** with two distinct access patterns:

**1. Primary mount (session-based, multiplexed views).** The server serves one primary mount point that shows the committed filesystem state by default. A process joins a transaction on this mount by calling the join-transaction ioctl with an access token. The server establishes a session binding that process to the transaction, so subsequent POSIX operations from that process are routed to the transaction's workspace. Processes without a session see the committed state.

This access pattern is process-specific: a subprocess (e.g., `diff`) spawned by a session-bound process does **not** inherit the session. The subprocess would need to independently present a token to access transaction state. This is a deliberate security property — session binding is non-transferable — but it means standard tools cannot operate on transaction state through the primary mount alone.

The exact session binding mechanism (PID-based mapping, FD-based scoping, or a hybrid) is a feature-design concern with security trade-offs:

- *PID-based*: Simple but vulnerable to PID reuse attacks (a new process inheriting a recycled PID could gain access).
- *FD-based*: The ioctl returns a directory FD; the process uses `openat()`-family syscalls relative to that FD. More secure (FD lifetime is tied to the process) but requires client-side adoption of `*at()` syscalls.
- *Hybrid*: PID-based with epoch/start-time validation to mitigate reuse.

The choice will be made during feature design. The system-level requirement is: **the session mechanism must not grant transaction access to any process that has not presented a valid access token**.

**File descriptor passing and inheritance.** When a session-bound process opens a file within a transaction, the FUSE daemon stores the transaction context in the kernel-level file handle (`fuse_file_info.fh`). This handle is bound to the kernel file description, not to the opening process's PID. Consequently, if the process passes the FD to another process (via Unix socket, `fork()`/`exec()`, etc.), the receiving process can **read and write through that FD** within the transaction — the daemon sees the original file handle and routes operations to the transaction's workspace. This is consistent with standard Unix FD semantics: access checks happen at `open()` time, not on each `read()`/`write()`.

This creates a per-file access delegation mechanism. A CastaneaFS-aware wrapper can selectively share access to specific transaction files with standard tools by opening files and passing FDs (e.g., via process substitution). This is not only a security concern but also a useful feature — it enables lightweight integration with non-CastaneaFS-aware tools without requiring a full transaction view mount.

**Directory FD delegation and subtree access.** A delegated directory FD provides access to the directory's children and recursively the entire subtree. In FUSE, `openat(dirfd, "child")` resolves via `FUSE_LOOKUP(parent_inode, "child")`. If the parent inode belongs to a transaction namespace, the daemon resolves the child within the transaction — the inode graph carries the context. Recursive tools (e.g., `diff -r` via `/proc/self/fd/N/...`) work with delegated directory FDs.

This default is permissive: subtree access is granted on every directory FD opened within a transaction. The alternative — defaulting to restrictive (listing-only) and requiring an ioctl to unlock subtree traversal — would impose an extra call on every `opendir()` within a transaction, including the session-bound process's own internal use. Since the common case is a process using its own directory FDs (where subtree access is expected), the permissive default avoids this usability tax.

> **Future feature**: A capability-attenuation ioctl that narrows a directory FD from subtree-capable to listing-only (`readdir()` without `openat()`) before delegating the FD to a subprocess. The narrowing would be irreversible on that file handle — the recipient cannot widen it back — providing per-delegation access granularity as a capability attenuation. The FUSE `open`/`opendir` path does not support custom per-open parameters (the `flags` field carries only standard POSIX open flags), so this attenuation must be a separate ioctl on the FD after open.

**Transactional semantics of delegated operations.** Operations on delegated FDs are still subject to the implicit subtransaction mechanism. The server wraps a write through a delegated FD in an implicit single-operation subtransaction within the transaction identified by the FD's file handle, and it goes through conflict detection as usual. The server manages the implicit subtransaction's full lifecycle transparently — the calling process never sees the owner or access tokens. The implicit owner token is attributed to the calling process's identity (from the FUSE request context), so commit-time checks use that process's uid/gid. For FD-based read/write operations specifically, permission checks (commit rule 3) use the access mode established at `open()` time rather than re-checking the caller's uid/gid, consistent with Unix FD semantics where the access grant is bound to the file description, not the process.

**File handle lifecycle and transaction state.** Processes sharing a file description (via `fork()` or `SCM_RIGHTS`) share the same `fh` and see each other's modifications in real time through the transaction workspace. A delegated FD is a live shared view into the transaction workspace, not a copy-on-delegate snapshot. This is standard Unix shared-file-description behavior.

**File deletion (unlink) while an FD is open.** Standard Unix semantics apply: `unlink()` removes the directory entry but the file data persists in the transaction workspace until `FUSE_RELEASE` (last FD closed). The delegated FD continues to work for reads and writes. The unlink is a transaction-local operation — the file is deleted within the transaction's workspace, not from the committed state.

**Transaction state changes with open FDs.** The transaction state definitions (see Transaction States above) determine what happens to operations through open FDs when the underlying transaction changes state:

- **Open → Failed**: The transaction becomes read-only (failed state: "read-only... remains readable"). Reads through existing FDs continue to work — the data is still there and the `fh` is still valid. Writes return an error (e.g., `EROFS`) — the transaction cannot accept new work (open state is "the only state in which a transaction can receive new work").
- **Open/Failed → Aborted**: The transaction's changes are discarded and resources cleaned up. The workspace is gone. The `fh` references state that no longer exists. Subsequent operations through the FD return an error (e.g., `ESTALE`).
- **Open → Committed**: The transaction's changes are merged into the parent. Unlike abort, the data is not destroyed — it now lives in the parent's workspace. The `fh` becomes stale and the server returns `ESTALE`, requiring the process to re-open the file through the parent's context. This applies uniformly to all FDs originating in the transaction, even those referring to files that were never modified during the transaction. The capability expires with its transaction — silent redirection to the parent's namespace would widen the access scope without re-authentication, which is inconsistent with CastaneaFS's security orientation.

**Access scope across transaction state transitions.** The FD staleness rule above is an instance of a general principle that applies equally to all access patterns — session-based, FD delegation, and transaction view mounts: when a transaction reaches a terminal state (committed, aborted, or failed→aborted), all access privileges scoped to that transaction are revoked. Subtransaction tokens are scoped to the subtransaction (see Closed Nested Transaction Model: "Creating a subtransaction issues a new pair of owner and access tokens for the subtransaction, scoped to it"), so when the subtransaction reaches a terminal state, those tokens reference a closed transaction. Participants are not implicitly granted access to the parent — their sessions become invalid and they must independently present a parent token to continue working. This preserves a clean security boundary: access is always explicitly granted, never silently inherited from a completed transaction.

**Transaction view mount behavior after transaction termination.** Transaction view mounts follow the same access revocation principle. When the underlying transaction reaches a terminal state, the mount becomes inert: all operations return `ESTALE`. The mount remains mounted — auto-unmount is not possible because a FUSE unmount fails with `EBUSY` if any process has open FDs or a working directory inside the mount, and the daemon cannot guarantee this. The inert mount must be manually unmounted by the user or the system administrator (via `fusermount -u` or the `castaneafs` CLI tool).

**`/proc` visibility and process execution.** When a process holds an FD to a file inside a transaction, the kernel exposes path information through `/proc` entries (`/proc/PID/fd/N`, `/proc/PID/cwd`, `/proc/PID/exe`, `/proc/PID/maps`). These entries are symlinks to paths on the FUSE mount. Another process reading through these paths triggers a new FUSE lookup with the *reading* process's identity — the reader sees its own view (committed state if it has no session), not the target process's transaction state. File *contents* are not leaked, but file *paths* are visible, which may reveal the existence of files created within the transaction.

The `/proc/PID/fd/N` entries are kernel magic symlinks with special `open()` semantics: opening one creates a *new* file description via a new `FUSE_OPEN` with the opener's PID, rather than sharing the original file description. Only actual FD inheritance (`fork()`) or explicit FD passing (Unix socket `SCM_RIGHTS`) shares the original file description and its transaction context.

**Executing transaction-state binaries.** When a session-bound process `fork()`s and the child calls `execve()` on a path inside the FUSE mount, the kernel opens the binary via `FUSE_OPEN` with the *child's* PID. The child has no session (sessions are non-transferable), so the daemon serves the **committed-state** version of the binary — not the transaction version, even if the parent modified the binary within the transaction. To execute the transaction version, the parent must use `fexecve()` (or equivalently, `execveat(fd, "", ..., AT_EMPTY_PATH)`), which uses the existing file description and its transaction-bound `fh`. After exec, the kernel demand-pages the binary through the file description established during exec; the `fh` is stable, so demand paging continues to serve the correct version.

**Comparison of access patterns.** The three access patterns — session-based access on the primary mount, FD delegation, and transaction view mounts — differ in access scope, subprocess behavior, and security properties:

| | Session-based (primary mount) | FD delegation | Transaction view mount |
|---|---|---|---|
| **Who sees transaction state** | Only the session-bound process | Any process holding the FD (via inheritance or passing) | All processes with POSIX access to the mount point |
| **Access scope** | All paths on the mount | Per-file or per-subtree (directory FDs grant subtree access by default) | All paths on the mount |
| **Subprocess `execve()` of transaction binary** | Child has no session; executes committed-state version unless `fexecve()` is used | Via `fexecve()` on the delegated FD: executes transaction version | Executes transaction version (mount serves it to everyone) |
| **Subprocess path-based access** | Child sees committed state (no session) | Child without the FD sees committed state | Child sees transaction state |
| **`/proc` content leakage** | Paths visible; contents not leaked (reader gets own view) | Paths visible; contents not leaked (magic symlink creates new file description) | Paths visible; contents accessible to anyone who can read the mount |
| **Security model** | Token-based capability (per-process session) | Capability delegation (access grant bound to file description) | POSIX permissions (OS-level access control) |
| **Tool compatibility** | CastaneaFS-aware tools only | Standard tools via `fexecve()`, process substitution, or piping | Standard tools work transparently |
| **Revocation** | End the session | Close/do not pass the FD (but cannot revoke from a process that already holds it) | Unmount (affects all users of the mount) |

The patterns form a spectrum from fine-grained-but-restrictive (session-based) to coarse-but-convenient (transaction view mount), with FD delegation as a middle ground that enables per-file access sharing with standard tools while preserving capability-based security semantics.

**2. Transaction view mounts (fixed view, OS-level access control).** A user (or the `castaneafs` CLI tool) can request an additional mount point that exposes a single fixed transaction state to **all processes** that can access the mount point. Token presentation is required once, to create the mount. Access control is then handled by standard OS permissions on the mount point directory (ownership, mode bits) — the same mechanism that protects any private mount.

```
castaneafs mount --transaction <token> /mnt/castanea-txn123
diff /mnt/castanea/path/to/file /mnt/castanea-txn123/path/to/file
```

Because the transaction view mount shows the same view to all callers, standard POSIX tools (`diff`, `rsync`, `find`, etc.) work transparently — a subprocess like `diff` simply accesses the mount point and sees the transaction state. A specialized `castaneafs diff` command is additionally desirable for atomic snapshot comparison (guaranteeing the state does not change during the diff), but standard tool support is the baseline.

> **Security model shift.** Creating a transaction view mount represents a deliberate transition from CastaneaFS's token-based capability model to OS-level POSIX permission-based access control. On the primary mount, transaction state is accessible only to processes that possess the access token. Once a token holder mounts a transaction view, the state becomes accessible to *any* process with appropriate POSIX permissions on the mount point — regardless of whether that process holds a CastaneaFS token. A token holder should only mount a transaction view after verifying that the transaction state does not contain sensitive information that should remain restricted to token holders.

**FUSE implementation note**: In libfuse, `fuse_context.private_data` is a `void*` set once in the filesystem's `init` callback — it stores per-filesystem-instance state, not per-request state. Per-request caller identification comes from `fuse_context.pid`, `.uid`, and `.gid`, which the kernel fills in for each request. The multi-mount approach uses separate FUSE instances (each with its own `private_data`) for each mount point: the primary mount's instance handles session multiplexing, while a transaction view mount's instance serves a fixed view without per-request identity checks.

**Multiple independent filesystem trees** are orthogonal to multi-view mounts. The server manages mount points with metadata identifying both the tree and the view (committed state vs. specific transaction). This is comparable to ZFS pools/datasets, each with their own mount points. The configuration or CLI distinguishes "mount tree X's committed state at /mnt/a" from "mount tree X's transaction T at /mnt/b" from "mount tree Y at /mnt/c."

### Token Leakage

Tokens are the sole credential. If a token is leaked, the holder gains access. Mitigations:

- **Architectural constraint 4** (token non-dissemination over network) limits the attack surface to local processes.
- Token distribution channels (Unix domain sockets, pipes, FD passing) are protected by OS-level process isolation.
- The server does not log tokens in plaintext.
- Tokens are cryptographically random and of sufficient length to resist brute-force guessing.

**Out of scope**: CastaneaFS does not protect against a compromised process intentionally leaking its own token, nor against a local attacker with root access reading another process's memory. These require OS-level security measures (e.g., SELinux, seccomp) that are outside CastaneaFS's control.

### Information Leakage through Metadata

Even without access to file contents, metadata can leak information:

- The existence of a transaction itself may be sensitive.
- Timing side-channels (e.g., observing lock contention delays) could reveal transaction activity.

CastaneaFS's position: transaction existence metadata is visible to the system administrator (via the `castaneafs` CLI tool). It is **not** visible to unprivileged processes that do not hold a token for the transaction. Timing side-channels are out of scope for the initial threat model.

### Accidental Data Exposure through Permission Changes

Addressed by the Security Conflict Policy (Rules 1–4). The conflict detection mechanism is the primary defense against this class of threat.

### Scope Exclusions

The following are explicitly out of scope for CastaneaFS's threat model:

- Kernel or FUSE module compromise
- Physical access attacks
- Network-based attacks (tokens are local-only per constraint 4)
- Side-channel attacks (timing, cache, power analysis)
- Denial-of-service (resource exhaustion attacks on the transaction system are a liveness concern, not a confidentiality concern; addressed separately by transaction administration)
- Application-level logic bugs (CastaneaFS protects filesystem-level invariants, not application semantics)

## Security Conflict Policy

### Motivation

Standard transactional conflict detection operates at the data level: two transactions conflict when they access the same data and at least one is a write. CastaneaFS extends this with **security-oriented conflict detection** that treats certain combinations of concurrent operations as conflicts even when they do not touch the same data, because the combination could result in unintended information exposure.

The core observation is that application code has no general mechanism to detect or prevent security violations arising from concurrent permission changes. By the time a process learns that file permissions have changed, the data it wrote under the old permission assumptions may already be exposed. CastaneaFS addresses this by treating the conflict as a transaction-level concern: the filesystem does not attempt to understand whether written content is genuinely sensitive, but conservatively treats the combination of content modification and permission relaxation as a conflict.

### Established Rules

**Rule 1: Content write + permission relaxation on the same file.** If transaction A writes to a file and transaction B makes that file's permissions more permissive (e.g., adding world-readable bits), the two transactions conflict. Transaction A may have written data under the assumption that only the file's current permission set governs access; transaction B's change would retroactively violate that assumption upon commit.

**Rule 2: Content write + permission relaxation on an ancestor directory.** The conflict extends to the directory hierarchy. If transaction A creates or writes a file, and transaction B relaxes permissions on a directory anywhere in the file's ancestor chain, the transactions conflict. The new file or content could become inadvertently accessible through the newly-permissive directory, even though the file's own permission bits are restrictive.

CastaneaFS does not inspect file content to determine sensitivity. It operates at the level of file operations: any file creation or write — including the creation of an empty file — combined with permission relaxation constitutes a conflict. File presence alone can carry information (e.g., through the filename), so the conflict trigger is the operation itself, not whether content was written.

**Reference permission state.** "More permissive" for Rules 1–2 is evaluated relative to the writing transaction's **final permission state** for the object in question. A transaction that does not modify an object's permissions has a final state identical to its snapshot start, so this single definition covers both cases uniformly. For Rule 2 (ancestor directories), the same definition applies per ancestor independently: each ancestor's reference is the writing transaction's final permission state for that ancestor.

**Security check against current state.** At commit time, security Rules 1–2 compare the parent's current permission state (after any permission metadata merges) against the writing transaction's reference state. This is a single comparison against the current committed state, not a per-sibling enumeration. The monotonicity of bitwise OR over the subset relation guarantees soundness: each pure-expansion merge is checked at its own commit time, and since OR can only add bits, if the parent's permissions do not exceed the writer's reference at any individual merge step, no subsequent merge can cause them to exceed it.

**Definition: "more permissive" for Rules 1–2.** The following permission changes are considered "more permissive" and trigger a security conflict when combined with concurrent content writes:

- **Adding read bits** (+r at user, group, or other level). This is the canonical case Rules 1–2 describe: granting read access to principals who previously lacked it is a direct confidentiality risk, since new content written by the concurrent transaction could be exposed.
- **Setting setuid or setgid bits.** Setuid/setgid is a qualitatively different threat from read-bit changes: rather than exposing *data*, it causes anyone who can execute the file to run it as the file owner (setuid) or group (setgid), risking *privilege escalation*. A concurrent write + setuid change means the writer's code could execute with elevated privileges the writer did not intend. Most operating systems ignore setuid on scripts, but CastaneaFS treats the bit change uniformly regardless of file type — the filesystem does not inspect file content to distinguish scripts from binaries.
- **Adding execute bits at a level where read is not set** (+x without +r at the same level). Execute-without-read is a deliberate access restriction pattern used for proprietary binaries: it allows execution but prevents copying or inspection of the file's contents. Adding +x at such a level grants a capability (execution) that cannot be obtained by copying the file, because the file cannot be read. This combination is uncommon in practice, making the conflict low-cost.

The following permission changes are **not** considered "more permissive" and do **not** trigger a security conflict with concurrent content writes:

- **Adding write bits** (+w at any level). Write permission is an integrity concern — it allows modification or truncation — not a confidentiality concern. It does not expose existing content to new readers. CastaneaFS's security conflict rules address data exposure, not data integrity.
- **Adding execute bits at a level where read is already set** (+x with +r at the same level). A user with read access can copy the file and execute the copy independently, so +x grants no capability that +r did not already effectively provide.

**Directory permission semantics for Rules 1–2.** POSIX directory permissions differ from file permissions: the read bit (`r`) controls listing directory entries (`readdir`), while the execute bit (`x`) controls traversal (resolving path components through the directory). Both bits independently gate access to the directory's contents — a directory with `r` but not `x` allows listing filenames but not accessing them; a directory with `x` but not `r` allows accessing children by name but not enumerating them. Because both bits independently contribute to discoverability and reachability, adding either `r` or `x` to a directory is considered "more permissive" for the purpose of Rules 1–2. As with files, adding write bits (`w`) to a directory is not considered a security relaxation — directory write permission controls creation and deletion of entries (an integrity concern), not visibility of existing content.

Currently, any permission relaxation on any single ancestor directory in the chain triggers a security conflict with a concurrent content write, regardless of whether other directories in the chain still prevent access. This is conservative: even if a parent directory remains restrictive, the relaxation of a grandparent still increases the set of principals who could potentially reach the parent.

> **Future refinement**: A more nuanced analysis could examine the ancestor chain as a whole and determine that a relaxation on one directory does not increase effective access when an intermediate directory still prohibits traversal. For example, if `/a/b/c/file` is written and `/a` is relaxed from 700 to 755, but `/a/b` remains 700, no new principal can reach `/a/b/c/file` through the relaxation alone. This refinement is deferred because it requires evaluating the *minimum effective access* across the entire ancestor chain at commit time, accounting for all concurrent permission changes to any directory in the chain — a more complex computation than the current per-object check.

**Rule 3: Content write + ownership change on the same file or ancestor directory.** If transaction A writes to (or creates) a file and transaction B changes the owner or group of that file or a directory in its ancestor chain (`chown`, `chgrp`), the two transactions conflict. An ownership change alters *who* has access: a writer may have assumed that only the current owner or group members can read the file, and a concurrent ownership transfer could violate that assumption by granting access to a different user or group. This rule applies to all ownership changes, regardless of whether the new owner/group demonstrably broadens or narrows access — the conservative treatment avoids requiring the filesystem to reason about group membership relationships at conflict detection time.

With domain separation, Rule 3 also uses the writing transaction's final ownership as the reference state. When both transactions agree on the final owner/group (the writing transaction's final ownership equals the other transaction's final ownership), no divergence exists and Rule 3 does not trigger. When they diverge — including the case where the writing transaction did not change ownership but the other transaction did — Rule 3 applies conservatively. For ancestor directories, the same per-object logic applies independently.

> **Future refinement**: An ownership change that is *strictly more restrictive* (the new owner/group is a subset of the previous access set) could be exempted from this rule, since it cannot widen exposure. This requires the filesystem to evaluate group membership at commit time to determine whether access was narrowed, which adds complexity. This refinement is deferred to a future version; the initial implementation treats all write + ownership change combinations as conflicts.

**Non-conflicts.** The security conflict rules are directional: they protect against the combination of *new content* being exposed through *widened permissions*. The following combinations are **not** security conflicts:

- **Read + permission relaxation**: A file read concurrent with permission relaxation on the same file or its ancestors. A read does not introduce new content — neither participant violates implicit assumptions made by the other. The reader observed data that was already present under the pre-relaxation permissions, and the permission change does not retroactively alter what was read.
- **File deletion + permission relaxation**: A file unlink concurrent with permission relaxation on the file's parent directory or ancestors. After the unlink commits, there is no POSIX-observable trace of former file existence at the deleted path — the file is simply absent from `readdir()`, which is indistinguishable from "was never there." The only residual is the directory's mtime change, which is non-specific (any directory modification updates mtime) and does not identify what changed or whether it was a deletion. No content is exposed because the content no longer exists.
- **Permission restriction + content write**: A permission tightening (making permissions *more restrictive*) concurrent with a content write is not a security conflict — tightening permissions cannot expose data. If the tightening commits first, the write may fail the POSIX permission check at commit time (commit rule 3), just as a race between writing to a file and `chmod`-ing it read-only could fail in a conventional filesystem. This is handled by the standard commit rules, not the security conflict policy.
- **Convergent permission writes**: Transaction A writes to a file and chmods it to 0644; transaction B chmods the same file to 0644. The permission metadata changes are convergent (identical final state), so no metadata conflict. The security check compares B's end state (0644) against A's reference state (A's final permissions, also 0644) — not more permissive. No conflict.
- **Compatible permission expansions (no content writes)**: Transaction A chmods a file from 0600 to 0644; transaction B chmods the same file from 0600 to 0640. Both are pure expansions from the snapshot start (0600). Merged result is bitwise OR: 0644. No content writes, so security conflict rules do not apply. No conflict.
- **Content writer approves wider permissions**: Transaction A writes to a file and chmods it to 0644; transaction B chmods the same file from 0600 to 0640. B's expansion (0640) is a subset of A's reference state (0644), so the security check passes. Both permission changes are pure expansions from snapshot start (0600), so permission metadata merges to 0644 via bitwise OR. No conflict.

**Scenario verification.** The following table exercises the interaction of metadata reconciliation, security conflict rules, and POSIX permission checks across representative cases. All cases start from file permissions 0600 unless noted. "Reference" denotes the writing transaction's final permission state (identical to snapshot start when the writer does not chmod). The metadata check uses the current-state comparison model: the committing transaction's delta from its snapshot and the parent's cumulative delta from that snapshot are each tested for pure expansion; merge is via bitwise OR when both are pure expansions. Security and POSIX checks compare against the parent's current committed state.

| # | Scenario | Metadata check | Security check | POSIX check | Result |
|---|---|---|---|---|---|
| 1 | T1 write + chmod 644, T2 chmod 644 | Convergent (P=E=644), no conflict | P(644) vs T1-ref(644): not more permissive | — | **Accept**, perms 644 |
| 2 | T1 chmod 644, T2 chmod 640 (no writes) | T2: E=640, S=600, pure expansion. P=644, P\|S=644, pure expansion. Merge: 640\|644=644 | No content write, N/A | — | **Accept**, perms 644 |
| 3 | T1 write + chmod 640, T2 chmod 644 | T2: E=644, S=600, pure expansion. P=640, P\|S=640, pure expansion. Merge: 644\|640=644 | P(644) vs T1-ref(640): adds other-read → more permissive | — | **Conflict** (security) |
| 4 | T1 write + chmod 644, T2 chmod 640 | T2: E=640, S=600, pure expansion. P=644, P\|S=644, pure expansion. Merge: 640\|644=644 | P(644) vs T1-ref(644): not more permissive | — | **Accept**, perms 644 |
| 5 | T1 write (no chmod), T2 chmod 644 | T2: E=644, S=600, pure expansion. P=600 (T1 didn't chmod), trivially pure expansion. Merge: 644\|600=644 | P(644) vs T1-ref(600): adds read → more permissive | — | **Conflict** (security) |
| 6 | T1 write (no chmod), T2 chmod a-w (→400) | T2: E=400, S=600, 400\|600=600≠400 → not pure expansion. P=600, no change → first-committer-wins | P(400) vs T1-ref(600): not more permissive | T2 first → file 400, T1 write needs owner-write → EACCES | **Conflict** (POSIX if T2 first) or **Accept** (T1 first) |
| 7 | T1 chmod 640, T2 chmod 604 (no writes) | T2: E=604, S=600, pure expansion. P=640, pure expansion. Merge: 604\|640=644 | No content write, N/A | — | **Accept**, perms 644 |
| 8 | T1 write + chmod 640, T2 chmod 620 | T2: E=620, S=600, pure expansion. P=640, pure expansion. Merge: 620\|640=660 | P(660) vs T1-ref(640): adds group-write → write bits not "more permissive" | — | **Accept**, perms 660 |
| 9 | T1 write + chown to B, T2 chown to B | Convergent ownership (both → B) | Rule 3: T1's final ownership = T2's → no divergence | — | **Accept** |
| 10 | T1 write (no chown), T2 chown to B | T2's ownership ≠ P's current (T1 didn't change it) → divergent | Rule 3: writer's final ownership ≠ T2's → conflict | — | **Conflict** (Rule 3) |
| 11 | File at 644, T1 chmod 600, T2 chmod 604 | T1 commits first (P=600). T2: E=604, S=644, 604\|644=644≠604 → not pure expansion | — | — | **Conflict** (first-committer-wins metadata) |

Note on case 6: commit order matters because the security and POSIX checks have different directional sensitivity. The security check passes either way (restriction is not "more permissive"). The POSIX check is order-dependent — if the restriction commits first, the write is no longer permitted. This order-dependence is inherent in POSIX semantics and matches non-transactional behavior (a race between writing and chmoding a file read-only). Note on case 11: the metadata check on T2 uses the current-state model. T2's snapshot was 644; T2's end state is 604. Since 604 | 644 = 644 ≠ 604, T2's change is not a pure expansion (it removed group-read). First-committer-wins applies; T1 committed first, so T2 fails.

**Rule 4: Uniform application across the transaction hierarchy.** Security conflict rules apply uniformly between all transactions — whether they are independent top-level transactions or sibling subtransactions of the same parent. CastaneaFS has no special rules distinguishing different levels of the transaction hierarchy. This falls out naturally from the general principle that all conflict detection uses the same mechanism regardless of nesting depth.

**Rule 5: Root exemption from security conflict checks.** The root user (uid 0) is exempt from security conflict checks, consistent with the POSIX convention that root is exempt from permission checks but bound by structural constraints. The reasoning is as follows:

- CastaneaFS's conflict detection addresses three categories of concern: *structural integrity* (e.g., creating a file in a deleted directory), *permission violations* (e.g., writing to a read-only file), and *security policy violations* (the rules above).
- In POSIX, structural constraints are impossible to violate (the operation has no meaning), while permission restrictions are merely access controls that root may bypass. Security policy violations — like permission violations — still leave the filesystem in a structurally valid state, so root is permitted to bypass them.
- For shared transactions, the effective user for all commit-time checks — both POSIX permission checks (commit rule 3) and security conflict checks (commit rule 2) — is the user who presents the owner token and issues the commit. If root holds the owner token and commits, permission checks and security conflict checks are both bypassed for that commit, consistent with the POSIX root exemption.

> **Warning — shared transactions with root**: Because root bypasses security conflict checks, sharing a transaction's access token with root-owned processes means those processes' modifications will not be security-checked at commit time. Users who are concerned about unintended data exposure should avoid sharing transactions with root-owned processes, or isolate root operations inside dedicated subtransactions whose results can be inspected before committing to the parent. This mirrors general security practice: limit root-level operations within user-owned activities.

### Open Questions

The following questions refine the boundaries of the security conflict detection rules. Answers will be incorporated as the policy is finalized during feature design.

1. **Subtransaction creation from a failed transaction**: Can new subtransactions be opened within a failed transaction? The failed state is defined as read-only ("cannot receive new work"), but subtransaction creation could be considered an administrative operation rather than new work — the subtransaction would snapshot the failed state and provide a workspace for experimentation or retry logic. Permitting it supports the sandbox use case; prohibiting it simplifies the state model.

Note: Rename/move across directories concurrent with content writes was considered as a potential security policy question but is already covered by standard write-write conflict detection. Under CastaneaFS's transactional model, rename is equivalent to an atomic delete at the source path and create at the destination path. This equivalence, which is only approximate in conventional filesystems, is made precise by CastaneaFS's atomicity guarantees. A concurrent content write to the file at the source path conflicts with the delete component (both modify the same path's directory entry), so the rename and write cannot both commit — no additional security-specific rule is needed.

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
- **2026-03-01**: Clarified snapshot timing (isolated at transaction creation), subtransaction token model (new token pair per subtransaction), transaction control interface (ioctl-based), permission check timing (at commit time), hierarchy root semantics, and security conflicts in commit rules.
- **2026-03-02**: Added Threat Model section: trust boundaries, multi-mount architecture (primary mount with session-based access, transaction view mounts with OS-level access control), token leakage mitigations, metadata leakage position, and scope exclusions.
- **2026-03-02**: Added FD sharing semantics: file descriptor passing and inheritance, directory FD delegation and subtree access trade-offs, and transactional semantics of delegated operations. Clarified that implicit subtransaction wrapping is a server-side mechanism transparent to calling processes.
- **2026-03-03**: Documented `/proc` visibility semantics, transaction-state binary execution behavior, and comparative analysis of the three access patterns (session-based, FD delegation, transaction view mount).
- **2026-03-03**: Documented FD lifecycle semantics: live shared view (not snapshot), unlink-while-open behavior, FD behavior across transaction state transitions (failed, aborted, committed), and access revocation policy — all FDs and sessions scoped to a transaction are invalidated when it reaches a terminal state, with no implicit promotion to the parent.
- **2026-03-03**: Settled that empty file creation conflicts with ancestor permission relaxation — security conflict rules operate at the operation level, not content level; file presence alone carries information. Settled directory FD delegation scope: subtree access by default (permissive); capability-attenuation ioctl to narrow to listing-only deferred as a future feature.
- **2026-03-03**: Revised isolation level from serializability to snapshot isolation with security extensions — write skew is permitted, consistent with no application-level invariant enforcement. Clarified that conflict detection is optimistic (commit-time). Clarified that the committing user's identity (owner token holder) governs all commit-time checks, not just security conflict checks.
- **2026-03-03**: Documented that subtransactions of a failed parent remain live by design (sandbox use case). Added preemptive cascading cleanup as a future feature; nesting depth limits noted as an implementation detail. Added open question on subtransaction creation from failed state. Clarified rename conflict analysis: rename is equivalent to atomic delete+create under CastaneaFS's atomicity, making concurrent write conflicts explicit without additional security rules.
- **2026-03-03**: Clarified crash recovery scope: durability is guaranteed only for data committed to the base filesystem. Subtransaction commits into open parents are durable relative to the parent but volatile if the parent is lost. No spec mechanism for token reissue after crash; applications manage their own token persistence.
- **2026-03-03**: Added open questions: symlink traversal and security conflict scope (#9), caller identity for non-FD operations (#10).
- **2026-03-04**: Settled transaction view mount lifecycle: mount becomes inert (all operations return `ESTALE`) when the underlying transaction terminates, consistent with access revocation for all other access patterns. Mount remains mounted; manual unmount required. Automatic unmount detection deferred as a future feature.
- **2026-03-04**: Clarified implicit subtransaction identity attribution: the server manages the implicit transaction lifecycle transparently, attributing the implicit owner token to the calling process's FUSE request context identity. This follows from the general rule that the owner token holder's identity governs commit-time checks. Removed former open question on caller identity for non-FD operations.
- **2026-03-04**: Settled symlink treatment: no special security conflict rules for symlinks. Symlinks are files whose content is a path; POSIX evaluates permissions on the resolved target, not the symlink's location. Discoverability of target paths via symlinks is a known metadata leakage limitation, documented with a warning. Added Link Types design rationale section contrasting hardlinks (unsupported — break tree invariant and multi-path ancestor checks) with symlinks (supported — no content exposure, metadata leakage accepted as out of scope). Removed symlink-related open questions (#5, #9).
- **2026-03-04**: Settled hierarchy root semantics: the base filesystem is an always-open transaction with an inaccessible owner token and a publicly available access token (the primary mount). Same lifecycle rules as any other transaction; differences arise only from external factors (durability guarantee, no token required for mount, no parent). Removed open question on root of the transaction hierarchy.
- **2026-03-04**: Settled that read + permission relaxation and file deletion + permission relaxation are not security conflicts. Security conflict rules are directional (new content + widened permissions); reads and deletions do not introduce new content. Added Non-conflicts section to the established rules.
- **2026-03-04**: Added Rule 3 (content write + ownership change). All chown/chgrp operations concurrent with writes are treated as conflicts — ownership changes alter who has access and can violate the writer's expectations about visibility. Exemption for strictly-more-restrictive ownership changes deferred to a future refinement.
- **2026-03-04**: Settled that permission tightening + content write is not a security conflict — tightening cannot expose data. If tightening commits first, the write may fail via the standard POSIX permission check (commit rule 3). Added to Non-conflicts section.
- **2026-03-04**: Settled the "more permissive" definition for Rules 1–2. Triggers: adding read bits (direct confidentiality risk), setting setuid/setgid (privilege escalation risk, treated uniformly regardless of file type), adding execute bits where read is not set (grants non-copyable execution capability). Non-triggers: adding write bits (integrity concern, not confidentiality), adding execute bits where read is already set (read already enables copy-and-execute). Removed open question #1.
- **2026-03-05**: Introduced conflict domains separating content (file data, directory entries) from permission metadata (chmod, chown/chgrp) on the same inode. Content and permission changes no longer produce standard write-write conflicts with each other; cross-domain interactions are mediated by security conflict rules (Rules 1–3) and POSIX permission checks (commit rule 3). Added permission metadata reconciliation: concurrent pure-expansion permission bit changes merge via bitwise OR; convergent ownership changes do not conflict; any restriction or divergent ownership uses first-committer-wins. Revised the security reference state for "more permissive" evaluation: the reference is uniformly the writing transaction's final permission state for the object (a transaction that does not modify permissions has a final state equal to its snapshot start, so no case split is needed). Rule 3 (ownership) uses the same principle — convergent final ownership produces no conflict. Added pairwise sufficiency argument for merged permission expansions. Added reconciliation examples to Non-conflicts section.
- **2026-03-06**: Reframed conflict detection from pairwise sibling comparisons to current-state comparison: all domain checks (content, permission bits, ownership, security rules) compare the committing transaction's state against the parent's current committed state, which encapsulates all intermediate sibling commits regardless of their start times. This follows the pattern used by optimistic concurrency control systems generally (FoundationDB, CockroachDB, PostgreSQL SI). Documented that the permission bit reconciliation is structurally a state-based CRDT — the merge function (bitwise OR) is commutative, associative, idempotent, and monotone over the subset lattice, guaranteeing order-independent convergence and composable security checks. Replaced the pairwise sufficiency argument with a direct current-state security comparison. Added scenario verification table (11 cases) exercising the interaction of metadata reconciliation, security conflict rules, and POSIX permission checks, with worked metadata check derivations using the S/E/P model.
- **2026-03-07**: Added directory permission semantics for Rules 1–2: both `r` (listing) and `x` (traversal) bits are independently security-relevant for directories; `w` is not (integrity concern). Documented that current ancestor-chain analysis is conservative (any single relaxation triggers conflict) with a future refinement for minimum-effective-access analysis across the chain.
- **2026-03-07**: Renamed CLI tool from `castaneafs-admin` to `castaneafs`, following the naming convention of `btrfs` and `zfs`. Replaced "reference client" mention with explicit reference to the `castaneafs` CLI administration tool — there is no client library; CastaneaFS is accessed via standard POSIX syscalls and ioctls.
