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
2. The subtransaction's changes do not **conflict** with changes previously committed by a sibling subtransaction. This includes both data-level conflicts (concurrent writes to the same file or directory entry) and security conflicts (see Security Conflict Policy).
3. The subtransaction's changes do not violate **POSIX permission checks** at commit time (e.g., writing to a read-only file, modifying a file owned by a different user). Permissions are evaluated against the parent's state at the time of commit, not at the time the operation was performed within the subtransaction. For implicit single-operation subtransactions, commit is immediate, so this is equivalent to checking at operation time.

If any condition is violated, the committing subtransaction enters the **failed** state. This covers all failure cases uniformly:

- Parent is **failed**, **aborted**, or **committed** — condition 1 violated.
- Sibling write conflict — condition 2 violated.
- Permission violation — condition 3 violated.

**Implicit single-operation transactions and permission errors**: The server wraps every bare FUSE operation in an implicit single-operation subtransaction (see Closed Nested Transaction Model above). When such an implicit subtransaction fails condition 3, the server simply aborts the subtransaction without persisting the failed state and returns the appropriate POSIX error code (e.g., `EACCES`, `EPERM`) to the calling process. This is indistinguishable from a standard POSIX permission error — no failed transaction state is visible to the user. In contrast, when an explicit subtransaction fails condition 3, the failed state is persisted as usual.

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

### Transaction Control Interface

Transaction lifecycle operations (create, commit, abort, create subtransaction) are exposed via **ioctl calls** on the CastaneaFS mount. The call to create a transaction returns the access token; the owner token is returned separately. Specific ioctl numbers and argument structures are not yet finalized and will be defined during feature design.

Filesystem operations within a transaction's scope are performed through standard POSIX syscalls, with the transaction context identified by its access token.

### Technology Stack

- **Formal specification**: TLA+ (with potential TLAPS-powered proofs of formal properties) for modeling the filesystem, transaction mechanism, and security policies.
- **Implementation language**: F* (F-star), a proof-oriented programming language, for the server and reference client.
- **Integration testing**: Go with testcontainers for isolated behavioral tests and benchmarks, including demonstrations of security properties under concurrent multi-user scenarios (varying privilege levels, including root).

### Design Context

**Relationship to existing transactional systems.** CastaneaFS's transaction model combines multi-process shared transactions with closed nested subtransactions. Each of these mechanisms has precedent individually, but existing systems make different trade-offs:

- *Multi-process transaction coordination*: X/Open XA and WS-AtomicTransaction coordinate transactions across multiple participants, but each participant operates on its own local branch rather than a shared workspace. PostgreSQL's `PREPARE TRANSACTION` allows a transaction to be finalized by a different session, but cannot accept further operations after the prepare phase. Calvin (Thomson et al., 2012) batches operations from multiple clients but each transaction originates from a single client. CastaneaFS differs in that multiple processes contribute operations to a single shared workspace, mediated by capability tokens.
- *Nested transactions*: The theoretical framework for nested transactions was established by Moss (1981–1985), and the ARIES/NT recovery algorithm (Mohan et al., 1989) extends WAL-based recovery for this model with subtransaction tables, lock inheritance, and selective undo. However, production databases (PostgreSQL, Oracle, MySQL/InnoDB, SQL Server) universally implement savepoints rather than true nesting — savepoints provide partial rollback within a flat transaction but do not provide workspace isolation between regions. CastaneaFS requires genuine nesting semantics (workspace isolation, commit-time conflict detection, cascading failure) because every individual FUSE operation is implicitly wrapped in a subtransaction, making savepoint-style partial rollback insufficient.

**Compositionality of ACID properties.** Atomicity, Consistency, and Durability are compositional: pairwise satisfaction extends to arbitrary sets of transactions. Snapshot isolation's write conflict detection is pairwise (first-committer-wins on each item independently), but the security conflict extensions introduce cross-item conflict rules (e.g., a write to file F conflicts with a permission relaxation on F's ancestor directory) that span multiple objects. These cross-item rules must be evaluated against the full committed state of the parent at commit time, not just pairwise against individual sibling transactions, because the relevant permission state may have been established by a chain of sibling commits.

## Architectural Constraints

1. **No hardlinks**: The filesystem does not support hardlinks. Every file has exactly one parent directory. This simplifies reasoning about the directory tree as a true tree (not a DAG) and eliminates an entire class of access-control edge cases.

2. **No atime tracking**: Access time metadata is not maintained. This eliminates a source of write amplification on read-heavy workloads and removes a covert channel for information leakage about file access patterns.

3. **Two-token capability model**: Each transaction has two capability tokens: an owner token (commit/abort authority) and an access token (filesystem operation authority). Participation is governed solely by token possession. The access token grants full filesystem access within the transaction, restricted only by the standard POSIX permission model. Token secrecy is the security boundary.

4. **Token non-dissemination over network**: Transaction tokens are assumed to be distributed only among local processes (e.g., via Unix domain sockets, pipes, or shared memory). The system does not provide mechanisms for secure token transport over a network and does not guarantee security properties if tokens traverse network boundaries.

5. **Server-side transaction state**: Transaction state (operation queues, isolation bookkeeping, conflict detection) is maintained on the server side. Clients are stateless with respect to transaction coordination; they submit operations and receive results, but do not hold authoritative transaction state.

6. **Security-oriented conflict detection**: The filesystem's conflict detection extends beyond data-level conflicts to include security-relevant interactions between concurrent transactions. See the Security Conflict Policy section below for the full treatment.

7. **Snapshot isolation with security extensions**: Concurrent transactions operate on consistent snapshots and are subject to first-committer-wins write conflict detection. Read-write conflicts are not tracked; write skew is permitted, consistent with the system's position that application-level invariants are outside its scope (see Consistency above). The security conflict rules (see Security Conflict Policy) extend the base snapshot isolation model with additional conflict detection for security-relevant operation combinations that SI alone would permit.

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
> - Directory FD capability attenuation (ioctl to narrow subtree-capable directory FDs to listing-only before delegation)

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

**Transactional semantics of delegated operations.** Operations on delegated FDs are still subject to the implicit subtransaction mechanism. The server wraps a write through a delegated FD in an implicit single-operation subtransaction within the transaction identified by the FD's file handle, and it goes through conflict detection as usual. Permission checks (commit rule 3) use the access mode established at `open()` time — the daemon does not re-check the caller's uid/gid on each read/write, consistent with Unix FD semantics where the access grant is bound to the file description, not the process.

**File handle lifecycle and transaction state.** Processes sharing a file description (via `fork()` or `SCM_RIGHTS`) share the same `fh` and see each other's modifications in real time through the transaction workspace. A delegated FD is a live shared view into the transaction workspace, not a copy-on-delegate snapshot. This is standard Unix shared-file-description behavior.

**File deletion (unlink) while an FD is open.** Standard Unix semantics apply: `unlink()` removes the directory entry but the file data persists in the transaction workspace until `FUSE_RELEASE` (last FD closed). The delegated FD continues to work for reads and writes. The unlink is a transaction-local operation — the file is deleted within the transaction's workspace, not from the committed state.

**Transaction state changes with open FDs.** The transaction state definitions (see Transaction States above) determine what happens to operations through open FDs when the underlying transaction changes state:

- **Open → Failed**: The transaction becomes read-only (failed state: "read-only... remains readable"). Reads through existing FDs continue to work — the data is still there and the `fh` is still valid. Writes return an error (e.g., `EROFS`) — the transaction cannot accept new work (open state is "the only state in which a transaction can receive new work").
- **Open/Failed → Aborted**: The transaction's changes are discarded and resources cleaned up. The workspace is gone. The `fh` references state that no longer exists. Subsequent operations through the FD return an error (e.g., `ESTALE`).
- **Open → Committed**: The transaction's changes are merged into the parent. Unlike abort, the data is not destroyed — it now lives in the parent's workspace. The `fh` becomes stale and the server returns `ESTALE`, requiring the process to re-open the file through the parent's context. This applies uniformly to all FDs originating in the transaction, even those referring to files that were never modified during the transaction. The capability expires with its transaction — silent redirection to the parent's namespace would widen the access scope without re-authentication, which is inconsistent with CastaneaFS's security orientation.

**Access scope across transaction state transitions.** The FD staleness rule above is an instance of a general principle that applies equally to session-based access: when a transaction reaches a terminal state (committed, aborted, or failed→aborted), all access privileges scoped to that transaction are revoked. Subtransaction tokens are scoped to the subtransaction (see Closed Nested Transaction Model: "Creating a subtransaction issues a new pair of owner and access tokens for the subtransaction, scoped to it"), so when the subtransaction reaches a terminal state, those tokens reference a closed transaction. Participants are not implicitly granted access to the parent — their sessions become invalid and they must independently present a parent token to continue working. This preserves a clean security boundary: access is always explicitly granted, never silently inherited from a completed transaction.

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

**2. Transaction view mounts (fixed view, OS-level access control).** A user (or the admin tool) can request an additional mount point that exposes a single fixed transaction state to **all processes** that can access the mount point. Token presentation is required once, to create the mount. Access control is then handled by standard OS permissions on the mount point directory (ownership, mode bits) — the same mechanism that protects any private mount.

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

CastaneaFS's position: transaction existence metadata is visible to the system administrator (via the admin CLI utility). It is **not** visible to unprivileged processes that do not hold a token for the transaction. Timing side-channels are out of scope for the initial threat model.

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

CastaneaFS does not inspect file content to determine sensitivity. It operates at the level of file operations: any file creation or write — including the creation of an empty file — combined with permission relaxation constitutes a conflict. File presence alone can carry information (e.g., through the filename), so the conflict trigger is the operation itself, not whether content was written. "More permissive" is evaluated relative to the permission state at the start of the conflicting transaction's snapshot.

**Rule 3: Uniform application across the transaction hierarchy.** Security conflict rules apply uniformly between all transactions — whether they are independent top-level transactions or sibling subtransactions of the same parent. CastaneaFS has no special rules distinguishing different levels of the transaction hierarchy. This falls out naturally from the general principle that all conflict detection uses the same mechanism regardless of nesting depth.

**Rule 4: Root exemption from security conflict checks.** The root user (uid 0) is exempt from security conflict checks, consistent with the POSIX convention that root is exempt from permission checks but bound by structural constraints. The reasoning is as follows:

- CastaneaFS's conflict detection addresses three categories of concern: *structural integrity* (e.g., creating a file in a deleted directory), *permission violations* (e.g., writing to a read-only file), and *security policy violations* (the rules above).
- In POSIX, structural constraints are impossible to violate (the operation has no meaning), while permission restrictions are merely access controls that root may bypass. Security policy violations — like permission violations — still leave the filesystem in a structurally valid state, so root is permitted to bypass them.
- For shared transactions, the effective user for all commit-time checks — both POSIX permission checks (commit rule 3) and security conflict checks (commit rule 2) — is the user who presents the owner token and issues the commit. If root holds the owner token and commits, permission checks and security conflict checks are both bypassed for that commit, consistent with the POSIX root exemption.

> **Warning — shared transactions with root**: Because root bypasses security conflict checks, sharing a transaction's access token with root-owned processes means those processes' modifications will not be security-checked at commit time. Users who are concerned about unintended data exposure should avoid sharing transactions with root-owned processes, or isolate root operations inside dedicated subtransactions whose results can be inspected before committing to the parent. This mirrors general security practice: limit root-level operations within user-owned activities.

### Open Questions

The following questions refine the boundaries of the security conflict detection rules. Answers will be incorporated as the policy is finalized during feature design.

1. **Does the reverse direction conflict?** If transaction A *reads* a file and transaction B relaxes that file's permissions, is that a conflict? The read itself does not change content, but the reader may have made decisions based on the assumption that the file was restricted.

2. **Ownership changes (chown)**: Are ownership transfers treated equivalently to permission bit changes? Changing a file's owner or group can effectively widen access if the new owner/group has broader membership. Should `chown` always be treated as a permission relaxation, or only when the new owner/group demonstrably broadens access?

3. **What constitutes "more permissive"?** Is any addition of permission bits sufficient (e.g., adding group-read to a file that was owner-only), or is the conflict limited to specific transitions (e.g., adding world-readable/writable)? How are setuid/setgid bit changes treated?

4. **File deletion and permission relaxation**: If transaction A deletes a file and transaction B relaxes permissions on the directory, is that a conflict? The file no longer exists, so there is no content to expose — but the unlink operation itself reveals that a file existed at that path.

5. **Symlinks**: Creating a symlink to a file in a more permissive directory could expose the target. Should symlink creation be treated as a permission-relevant operation on the target?

6. **Permission restriction (tightening)**: If transaction A makes permissions *more restrictive* while transaction B writes content, is that a conflict? Tightening permissions doesn't expose data, but it could cause transaction B's subsequent operations to fail unexpectedly if they depend on the original permission state.

7. **Root of the transaction hierarchy**: The filesystem itself serves as the root of the transaction hierarchy — the implicit parent into which top-level transactions commit. What state model applies to this root? Does it have a meaningful lifecycle (e.g., can it be "closed" during shutdown), how does it interact with crash recovery, and what are the semantics of committing a top-level transaction into it?

Note: Rename/move across directories concurrent with content writes was considered as a potential security policy question but is already covered by standard write-write conflict detection (the move modifies the directory entry and file location, conflicting with concurrent writes to the file's content or its original parent directory).

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
