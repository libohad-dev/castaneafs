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

### Failed Transaction State

When a transaction's commit fails (due to a conflict or other error), it enters a **failed** state with the following properties:

- The transaction becomes **read-only**. No further writes are accepted.
- **Commit cannot be retried**. The server rejects subsequent commit attempts immediately without performing expensive conflict checks.
- **Subtransaction commits are rejected**. Committing a subtransaction into a failed parent marks the subtransaction as failed as well (cascading failure).
- **Deeper nesting is unaffected**: A sub-subtransaction may still successfully commit into its (non-failed) parent subtransaction. For example, if transaction `T` is failed and `T.S1` is a subtransaction, committing `T.S1.S2` into `T.S1` may still succeed — but committing `T.S1` into `T` will fail and cascade `T.S1` into the failed state.
- Committing into an **aborted** transaction also produces the failed state on the committing subtransaction.
- The failed transaction's **filesystem content is persisted** and remains readable by access token holders. This allows participants to compare the failed state with the live committed state and retry application-level actions with full knowledge of what conflicted.
- The owner token holder may **abort** a failed transaction, which triggers cleanup of the persisted filesystem content.

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

## Architectural Constraints

1. **No hardlinks**: The filesystem does not support hardlinks. Every file has exactly one parent directory. This simplifies reasoning about the directory tree as a true tree (not a DAG) and eliminates an entire class of access-control edge cases.

2. **No atime tracking**: Access time metadata is not maintained. This eliminates a source of write amplification on read-heavy workloads and removes a covert channel for information leakage about file access patterns.

3. **Two-token capability model**: Each transaction has two capability tokens: an owner token (commit/abort authority) and an access token (filesystem operation authority). Participation is governed solely by token possession. The access token grants full filesystem access within the transaction, restricted only by the standard POSIX permission model. Token secrecy is the security boundary.

4. **Token non-dissemination over network**: Transaction tokens are assumed to be distributed only among local processes (e.g., via Unix domain sockets, pipes, or shared memory). The system does not provide mechanisms for secure token transport over a network and does not guarantee security properties if tokens traverse network boundaries.

5. **Server-side transaction state**: Transaction state (operation queues, isolation bookkeeping, conflict detection) is maintained on the server side. Clients are stateless with respect to transaction coordination; they submit operations and receive results, but do not hold authoritative transaction state.

6. **Isolation from privilege escalation during concurrent policy changes**: The filesystem must guarantee that concurrent modifications to security policies (e.g., permission changes, ownership transfers) cannot create transient windows where file content becomes accessible to unprivileged processes. Policy changes and data access must be ordered such that security invariants hold at every observable state.

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

## Feature Index

*No features designed yet. Use `/formspec.1.design <feature-name>` to add features.*

## Changelog

- **2026-02-28**: Initial system design. Established two-token capability model, closed nested transaction model, failed transaction state semantics, POSIX-only permission model, CLI admin utility, and deferred persistence backend.
