# Aegis Core — Kernel Invariants

## 1. Purpose

This document defines the core invariants that Aegis Core must preserve.

An invariant is a property that must remain true whenever the kernel is in a valid state.

Aegis Core is designed as a small, capability-based microkernel inspired by the security architecture and verification principles of seL4.

These invariants are intended to guide:

- implementation
- code review
- testing
- debugging
- model checking
- formal specification
- eventual formal verification

An invariant must not be treated as merely a comment. Security-critical invariants should eventually be represented in machine-checkable specifications or assertions.

---

# 2. Invariant Categories

Aegis Core invariants are divided into:

```text
I — Global kernel state
C — Capabilities
M — Memory
T — Threads
S — Scheduler
Ipc — IPC
O — Objects
A — Architecture
R — Resources
X — Isolation
```

---

# 3. Global Kernel State

## I-001 — Kernel State Is Internally Consistent

At every valid kernel state:

```text
All kernel-owned data structures satisfy their defined invariants.
```

A corrupted internal structure must never be considered valid kernel state.

---

## I-002 — Kernel State Transitions Are Valid

Every state transition must be one of the transitions explicitly allowed by the corresponding subsystem.

```text
Valid state
    ↓
authorized operation
    ↓
valid new state
```

The kernel must never transition directly into an undefined state.

---

## I-003 — Kernel Memory Is Kernel-Owned

Kernel metadata and kernel execution state must reside in memory controlled by the kernel.

Userspace must not be able to modify kernel state directly.

---

# 4. Capability Invariants

Capabilities are the primary authority mechanism of Aegis Core.

## C-001 — Valid Capability References a Valid Object

Every valid capability must reference an existing kernel object.

```text
valid capability
        ↓
valid object
```

A capability must never reference freed or nonexistent kernel state.

---

## C-002 — Capability Rights Cannot Increase

Capability derivation must never create more authority than the source capability possesses.

Formally:

```text
rights(derived) ⊆ rights(source)
```

Example:

```text
READ | WRITE
      ↓
READ
```

is valid.

```text
READ
 ↓
READ | WRITE
```

is invalid.

---

## C-003 — Invalid Capabilities Have No Authority

An invalid, null, revoked, or otherwise unusable capability must not authorize an operation.

```text
invalid capability → no authority
```

---

## C-004 — Capability Checks Are Mandatory

Every operation requiring authority must perform the appropriate capability validation before accessing the protected object.

---

## C-005 — Capability Identity Is Not Authority

An identifier, handle, index, pointer, or numeric value must never grant authority merely because it matches an object identifier.

Authority comes from a valid capability.

---

## C-006 — Capability Revocation Removes Authority

When a capability is revoked, it must no longer authorize operations covered by the revoked authority.

---

## C-007 — Capability Transfer Cannot Increase Authority

IPC or other capability-transfer mechanisms must preserve the authority restrictions of the transferred capability.

---

## C-008 — Capability Destruction Is Safe

Destroying a capability must not corrupt the object it references or unrelated capabilities.

---

## C-009 — Capability Tables Remain Consistent

Capability tables must not contain:

```text
duplicate invalid entries
dangling object references
out-of-range indexes
invalid rights combinations
```

---

# 5. Memory Invariants

## M-001 — Address-Space Isolation

One address space must not access another address space's memory without explicit authority.

```text
Address space A
      ✕
Address space B
```

unless an authorized shared-memory mechanism exists.

---

## M-002 — Userspace Cannot Access Kernel Memory

Userspace must not be able to read or write protected kernel memory.

---

## M-003 — Memory Permissions Are Enforced

Mappings must respect their intended permissions.

Examples:

```text
read-only → cannot write
non-executable → cannot execute
unmapped → cannot access
```

---

## M-004 — Mapping Requires Authority

A process cannot create or modify a mapping unless it has the required memory authority.

---

## M-005 — Physical Memory Ownership Is Consistent

A physical memory frame must have a well-defined ownership state.

It must not simultaneously be treated as exclusively owned by two unrelated security domains.

---

## M-006 — No Unauthorized Aliasing

The kernel must prevent unauthorized mappings of the same physical memory into security domains.

Intentional shared memory must be explicitly authorized.

---

## M-007 — Unmapping Does Not Create Dangling Kernel State

Removing a mapping must leave page-table and memory-management structures consistent.

---

## M-008 — Page-Table Structures Are Valid

All active page-table structures must satisfy the architecture's required alignment, hierarchy, and permission rules.

---

# 6. Thread Invariants

## T-001 — Every Thread Has One Valid State

A thread must always be in a valid state such as:

```text
Created
Ready
Running
Blocked
Sleeping
Terminated
```

The exact state machine is defined by the scheduler design.

---

## T-002 — Terminated Threads Cannot Run

A terminated thread must never be placed into the runnable set.

---

## T-003 — Blocked Threads Cannot Run

A blocked thread must not execute until the event that caused the block has been resolved.

---

## T-004 — Every Running Thread Has a CPU

A running thread must be associated with a CPU on which it is executing.

---

## T-005 — One Thread Cannot Run Twice

A thread must not simultaneously occupy multiple execution slots unless the architecture explicitly defines such behavior.

For normal kernel threads:

```text
one thread → at most one CPU at a time
```

---

## T-006 — Thread Context Is Valid

A saved CPU context must satisfy the requirements of the architecture before it is restored.

---

## T-007 — Thread Ownership Is Defined

Every thread must belong to a defined scheduling and security domain.

---

# 7. Scheduler Invariants

## S-001 — Runnable Threads Are Valid

Every entry in a scheduler run queue must reference a valid runnable thread.

---

## S-002 — No Duplicate Run-Queue Entries

A thread must not appear more than once in a run queue unless explicitly required by the scheduler design.

---

## S-003 — Blocked Threads Are Not Runnable

```text
blocked ∩ runnable = ∅
```

---

## S-004 — Terminated Threads Are Not Runnable

```text
terminated ∩ runnable = ∅
```

---

## S-005 — Scheduler Respects Priority

If the scheduling policy requires priority ordering, scheduler decisions must respect the defined priority rules.

---

## S-006 — CPU Affinity Is Respected

A thread must only execute on CPUs permitted by its affinity configuration.

---

## S-007 — Scheduler State Survives Context Switches

A context switch must not leave the scheduler in a state where:

```text
two threads appear running on one CPU
a thread appears running and blocked simultaneously
a thread disappears from all valid scheduler states
```

---

# 8. IPC Invariants

## Ipc-001 — IPC Requires Authority

A thread must have the required capability to access an IPC endpoint.

---

## Ipc-002 — Endpoint State Is Consistent

An IPC endpoint must accurately represent its current state.

For example:

```text
idle
sender waiting
receiver waiting
message transfer
```

---

## Ipc-003 — Message Transfer Preserves Isolation

IPC must not allow a sender to directly modify arbitrary memory belonging to a receiver.

---

## Ipc-004 — IPC Capability Transfer Preserves Rights

Transferred capabilities must not gain rights during IPC.

```text
transferred_rights ⊆ original_rights
```

---

## Ipc-005 — Invalid IPC Arguments Are Rejected

Malformed or unauthorized IPC operations must fail safely.

---

## Ipc-006 — Wake-Up State Is Consistent

A thread awakened by IPC must enter an appropriate scheduler state.

It must not become both:

```text
blocked
```

and:

```text
runnable
```

at the same time.

---

## Ipc-007 — Endpoint Destruction Is Safe

Destroying an IPC endpoint must not leave dangling references or permanently inconsistent blocked-thread state.

---

# 9. Object Lifetime Invariants

## O-001 — Objects Have Valid Lifetime States

Every kernel object must have a valid lifecycle.

Example:

```text
Allocated
    ↓
Initialized
    ↓
Active
    ↓
Destroying
    ↓
Destroyed
```

---

## O-002 — Destroyed Objects Cannot Be Used

Once an object has been destroyed, no valid authority may access it.

---

## O-003 — References Are Tracked

Objects must not be freed while valid kernel references still require them, unless the reference model explicitly permits this.

---

## O-004 — No Use-After-Free

The kernel must never access an object after its lifetime has ended.

---

## O-005 — No Double Destruction

An object must not be destroyed more than once.

---

# 10. Resource Invariants

## R-001 — Resource Ownership Is Explicit

Kernel resources must have a defined owner or ownership model.

Resources include:

```text
Physical memory
Kernel objects
Capabilities
IPC endpoints
Interrupts
CPU time
Address spaces
```

---

## R-002 — Resource Accounting Is Consistent

The kernel's accounting information must match the actual resource state.

---

## R-003 — Resource Exhaustion Fails Safely

If a resource cannot be allocated, the kernel must return a defined failure rather than entering an invalid state.

---

## R-004 — Unprivileged Code Cannot Exhaust Protected Resources Without Policy

Resource limits must prevent an untrusted domain from corrupting kernel operation through uncontrolled resource consumption.

---

# 11. Isolation Invariants

## X-001 — Kernel/User Isolation

Userspace cannot directly modify:

```text
kernel code
kernel stacks
kernel metadata
kernel capability structures
kernel scheduler state
```

---

## X-002 — Process Isolation

A process cannot access another process's private resources without authority.

---

## X-003 — Capability Isolation

Possessing one capability must not implicitly grant unrelated authority.

---

## X-004 — Identifier Isolation

Knowing an object's identifier must not be sufficient to access the object.

---

## X-005 — Privilege Separation

Security-sensitive kernel operations must only be available through explicitly authorized mechanisms.

---

# 12. Interrupt Invariants

## A-001 — Interrupt Ownership Is Defined

Every delivered interrupt must have a defined kernel handling path.

---

## A-002 — Interrupt Configuration Requires Authority

Untrusted code must not arbitrarily reconfigure protected interrupt routing.

---

## A-003 — Interrupt State Is Consistent

Interrupt enable/disable state must satisfy architecture and kernel requirements at every privileged transition.

---

## A-004 — Interrupt Handling Cannot Bypass Isolation

An interrupt handler must not create unauthorized access to protected memory or objects.

---

# 13. Architecture Invariants

Architecture-specific implementations must preserve the abstract kernel invariants.

For x86_64, this includes:

```text
Valid CPU context
Valid interrupt frame
Valid page-table state
Correct privilege transitions
Correct stack state
Correct register state
Correct interrupt return state
```

Architecture-specific code must not weaken capability or isolation guarantees.

---

# 14. Unsafe Code Invariants

Every `unsafe` operation must have a documented safety argument.

A safety argument should explain:

```text
Precondition
Invariant
Why the operation is valid
Who establishes the invariant
How the invariant is maintained
```

Example:

```rust
// SAFETY:
// The mapping has been validated as kernel-owned,
// correctly aligned, and writable before this operation.
unsafe {
    ...
}
```

---

# 15. Syscall Invariants

Every system call must:

```text
validate its arguments
validate capability authority
validate memory references
preserve kernel invariants
return a defined result
```

A userspace process must never be able to bypass a security check by supplying specially crafted syscall arguments.

---

# 16. Failure Invariants

## F-001 — Invalid Input Does Not Corrupt Kernel State

Invalid userspace input must result in a controlled failure.

---

## F-002 — Kernel Panic Does Not Continue Normally

If the kernel enters an unrecoverable panic state, it must not continue executing as though the state were valid.

---

## F-003 — Partial Operations Are Consistent

If an operation fails halfway through, the kernel must either:

```text
roll back safely
```

or:

```text
leave a documented valid partial state
```

It must never leave corrupted state.

---

# 17. Concurrency Invariants

## CC-001 — Shared State Is Synchronised

Concurrent access to mutable shared kernel state must use an appropriate synchronization mechanism.

---

## CC-002 — Atomic State Transitions

State transitions that must be atomic cannot be observed halfway through.

---

## CC-003 — No Data Races

Kernel shared state must not experience undefined concurrent mutation.

---

## CC-004 — Lock Ownership Is Consistent

Locks must not be simultaneously owned by incompatible execution contexts.

---

## CC-005 — Scheduler and IPC Remain Consistent Under Concurrency

Concurrent wake-up, blocking, scheduling, and IPC operations must not create contradictory states.

---

# 18. Formal Verification Invariants

The long-term goal is to make the most important invariants formally specified.

Highest-priority candidates:

```text
C-002  Capability rights cannot increase
M-001  Address-space isolation
M-002  Userspace cannot access kernel memory
Ipc-001 IPC requires authority
T-003  Blocked threads cannot run
S-003  Blocked threads are not runnable
O-002  Destroyed objects cannot be used
X-001  Kernel/user isolation
```

---

# 19. Invariant Testing

Each invariant should eventually have one or more verification mechanisms.

| Invariant | Assertion | Unit Test | Integration | Model | Formal |
|---|---|---|---|---|---|
| Capability rights | Yes | Yes | Yes | Yes | High |
| Memory isolation | Yes | Yes | Yes | Yes | High |
| IPC authority | Yes | Yes | Yes | Yes | High |
| Thread state | Yes | Yes | Yes | Yes | High |
| Scheduler | Yes | Yes | Yes | Yes | High |
| Object lifetime | Yes | Yes | Yes | Yes | High |
| Interrupts | Yes | Yes | Yes | Optional | Later |
| Architecture | Yes | Yes | Yes | Optional | Later |

---

# 20. Invariant Violation Handling

During development, invariant violations should provide enough information to identify:

```text
Invariant ID
Subsystem
Current state
Expected state
Relevant object/thread/capability
CPU
Execution context
```

Example:

```text
INVARIANT VIOLATION

ID: S-003
Subsystem: Scheduler
Property: Blocked threads cannot be runnable

Thread: 42
State: Blocked
Run queue: CPU 0
```

The exact diagnostic implementation may change between debug and release builds.

---

# 21. Invariant Review Rules

When modifying a security-critical subsystem:

```text
1. Identify affected invariants.
2. Determine whether any invariant changes.
3. Update this document if necessary.
4. Add or update tests.
5. Review unsafe code.
6. Review capability implications.
7. Review concurrency implications.
8. Update formal specifications when applicable.
```

---

# 22. Invariant Naming

Invariant IDs should remain stable.

Recommended format:

```text
C-xxx  Capability
M-xxx  Memory
T-xxx  Thread
S-xxx  Scheduler
Ipc-xxx IPC
O-xxx  Object
R-xxx  Resource
A-xxx  Architecture
X-xxx  Isolation
CC-xxx Concurrency
F-xxx  Failure
```

Do not casually renumber existing invariants because tests, specifications, and verification artifacts may reference them.

---

# 23. Core Security Invariant Set

The following properties form the initial security core of Aegis Core:

```text
1. Authority cannot be created from nothing.
2. Capability derivation cannot increase authority.
3. Invalid capabilities grant no authority.
4. Userspace cannot directly modify kernel memory.
5. Address spaces remain isolated without explicit authority.
6. Unauthorized IPC is rejected.
7. Blocked threads cannot execute.
8. Terminated threads cannot execute.
9. Destroyed objects cannot be accessed.
10. Resource failures cannot corrupt kernel state.
11. Kernel state transitions must preserve all invariants.
12. Architecture-specific mechanisms must preserve the abstract security model.
```

These should be treated as fundamental design constraints.

---

# 24. Relationship to Verification

The invariant hierarchy should eventually become:

```text
Design requirement
        ↓
Invariant
        ↓
Implementation assertion
        ↓
Automated test
        ↓
Formal model
        ↓
Formal proof
```

The strongest claims should be reserved for properties that are actually established by the corresponding verification method.

---

# 25. Long-Term Goal

Aegis Core should aim for a verification architecture where:

```text
Security requirements
        ↓
Formal invariants
        ↓
Abstract kernel model
        ↓
Concrete kernel implementation
        ↓
Machine-checked verification
```

The purpose is not to prove every line of code immediately.

The priority is to make the kernel's most important security guarantees precise enough that they can eventually be checked mathematically.

---

# 26. Status

Current status:

```text
[✓] Initial invariant catalogue
[✓] Capability invariants
[✓] Memory invariants
[✓] Thread invariants
[✓] Scheduler invariants
[✓] IPC invariants
[✓] Object lifetime invariants
[✓] Isolation invariants
[✓] Resource invariants
[✓] Concurrency invariants
[ ] Runtime invariant framework
[ ] Automated invariant tests
[ ] Formal specification
[ ] Model checking
[ ] Formal proof
[ ] End-to-end refinement proof
```

This document should evolve together with the Aegis Core architecture.
