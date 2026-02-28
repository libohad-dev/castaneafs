# System Design

## System Overview

CastaneaFS is a security-oriented FUSE filesystem with native ACID transaction support. It exposes a mostly POSIX-compliant interface (planned omissions: hardlinks, atime) to user-space processes and is designed around two core principles: strong access control guarantees and a capability-based transactional model.

### Transaction Model

Transactions are first-class filesystem primitives. When a client requests a new transaction, the CastaneaFS server issues a cryptographic capability token. That token can be distributed to multiple cooperating processes, each of which can enqueue filesystem operations against the transaction by presenting the token. This enables multi-process collaborative workflows where distinct processes with different expertise contribute to a single atomic batch of filesystem changes.

Transaction properties follow a modified ACID model:

- **Atomicity**: All operations within a transaction commit or none do.
- **Consistency**: Limited; the filesystem enforces structural invariants (e.g., directory tree integrity, permission coherence) but does not provide application-level constraint checking.
- **Isolation**: Concurrent transactions observe consistent snapshots and produce serializable outcomes.
- **Durability**: Committed transactions survive process and system crashes.

The temporary transaction-internal state serves dual purposes: it acts as a sandbox for security-sensitive work (within the filesystem's threat model) and as a disposable scratchpad for exploratory workflows such as testing different design or implementation configurations with easy rollback.

### Actors

- **CastaneaFS server**: The FUSE daemon that manages the filesystem tree, enforces security policies, coordinates transactions, and persists data.
- **Client processes**: User-space programs that interact with the filesystem through standard POSIX syscalls and through a transaction control interface (for creating, joining, committing, and aborting transactions).
- **Transaction token holders**: Any process that has been granted a transaction capability token and can enqueue operations against that transaction. Token holders may operate at different privilege levels.

### Technology Stack

- **Formal specification**: TLA+ (with potential TLAPS-powered proofs of formal properties) for modeling the filesystem, transaction mechanism, and security policies.
- **Implementation language**: F* (F-star), a proof-oriented programming language, for the server and reference client.
- **Integration testing**: Go with testcontainers for isolated behavioral tests and benchmarks, including demonstrations of security properties under concurrent multi-user scenarios (varying privilege levels, including root).

## Architectural Constraints

1. **No hardlinks**: The filesystem does not support hardlinks. Every file has exactly one parent directory. This simplifies reasoning about the directory tree as a true tree (not a DAG) and eliminates an entire class of access-control edge cases.

2. **No atime tracking**: Access time metadata is not maintained. This eliminates a source of write amplification on read-heavy workloads and removes a covert channel for information leakage about file access patterns.

3. **Capability-based transaction access**: Transaction participation is governed solely by possession of a capability token. There is no separate ACL or role-based check for transaction membership. Token secrecy is the security boundary; the server treats any bearer of a valid token as an authorized participant.

4. **Token non-dissemination over network**: Transaction tokens are assumed to be distributed only among local processes (e.g., via Unix domain sockets, pipes, or shared memory). The system does not provide mechanisms for secure token transport over a network and does not guarantee security properties if tokens traverse network boundaries.

5. **Server-side transaction state**: Transaction state (operation queues, isolation bookkeeping, conflict detection) is maintained on the server side. Clients are stateless with respect to transaction coordination; they submit operations and receive results, but do not hold authoritative transaction state.

6. **Isolation from privilege escalation during concurrent policy changes**: The filesystem must guarantee that concurrent modifications to security policies (e.g., permission changes, ownership transfers) cannot create transient windows where file content becomes accessible to unprivileged processes. Policy changes and data access must be ordered such that security invariants hold at every observable state.

7. **Serializable transaction isolation**: Concurrent transactions must produce outcomes equivalent to some serial execution order. The system may employ snapshot isolation, two-phase locking, or serializable snapshot isolation internally, but the externally observable behavior must be serializable.

8. **Crash recovery without data loss**: Committed transactions must survive crashes. The server must implement write-ahead logging or an equivalent mechanism to guarantee that committed data can be recovered after an unclean shutdown.

9. **Transaction liveness**: Every transaction must eventually either commit or abort. The system must prevent indefinite transaction stalls through timeout-based expiration, deadlock detection, or a combination of both. Orphaned tokens (from crashed processes) must not hold resources indefinitely.

10. **POSIX compliance (modulo stated omissions)**: All standard filesystem operations (open, read, write, close, mkdir, rmdir, rename, chmod, chown, stat, readdir, truncate, symlink, readlink, unlink, etc.) must behave according to POSIX semantics except where explicitly stated otherwise (hardlinks, atime).

## Feature Index

*No features designed yet. Use `/formspec.1.design <feature-name>` to add features.*

## Changelog

*No changes recorded yet.*
