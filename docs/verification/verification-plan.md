# Aegis Core — Verification Plan

## 1. Purpose

This document defines the verification strategy for Aegis Core.

Aegis Core is intended to be a small, capability-based microkernel inspired by the security and verification principles of systems such as seL4.

The goal is to make the most security-critical kernel properties explicit, testable, and eventually formally verified.

Verification should progress from ordinary testing to stronger forms of analysis and formal proof.

---

## 2. Verification Goals

The primary goals are:

```text
Memory isolation
Capability correctness
IPC correctness
Scheduler correctness
Object lifetime safety
Kernel/user isolation
Interrupt isolation
Architecture correctness
Concurrency correctness
System-call correctness
```

The highest priority is verifying properties that protect security boundaries.

---

## 3. Verification Levels

Aegis Core should use several verification levels:

```text
Level 1 — Compilation
Level 2 — Static analysis
Level 3 — Unit testing
Level 4 — Integration testing
Level 5 — Property-based testing
Level 6 — Fuzz testing
Level 7 — Runtime invariant checking
Level 8 — Model checking
Level 9 — Formal specification
Level 10 — Formal proof
```

Not every component requires the same level immediately.

---

## 4. Verification Philosophy

The project should follow:

```text
Specification
      ↓
Implementation
      ↓
Tests
      ↓
Invariant checks
      ↓
Formal model
      ↓
Proof
```

The implementation should not be considered secure merely because tests pass.

Tests provide evidence that specific cases work.

Formal verification aims to establish that specified properties hold for all cases covered by the model.

---

## 5. Trusted Computing Base

Verification effort should focus heavily on the trusted computing base.

Potential TCB:

```text
Boot code
Architecture-specific kernel code
Memory management
Capability enforcement
IPC primitives
Thread/context switching
Interrupt handling
Critical synchronization
```

Large userspace services should remain outside the kernel TCB whenever possible.

---

## 6. Security Properties

The following properties are high-priority verification targets.

### Capability authority

```text
A capability cannot grant more authority than intended.
Capability derivation cannot increase authority.
Invalid capabilities cannot authorize operations.
Revoked capabilities cannot continue granting authority.
```

### Memory isolation

```text
Userspace cannot access kernel memory.
One address space cannot access another without authority.
Unauthorized mappings are rejected.
Memory permissions are enforced.
```

### IPC

```text
Unauthorized endpoints cannot be accessed.
IPC capability transfers cannot create unauthorized authority.
Malformed messages cannot violate isolation.
Blocked and waking threads remain consistent.
```

### Scheduling

```text
Blocked threads are not scheduled.
Terminated threads are not scheduled.
A thread cannot execute on two CPUs simultaneously.
CPU affinity is respected.
Scheduler state remains consistent.
```

---

## 7. Kernel Invariants

The kernel should define explicit invariants.

Examples:

```text
I1: Every valid capability references a valid object.
I2: Capability rights cannot increase through derivation.
I3: No userspace mapping grants unauthorized kernel access.
I4: Every running thread belongs to exactly one CPU.
I5: A blocked thread is not runnable.
I6: A terminated object cannot be accessed through valid authority.
I7: Every kernel object has valid lifetime state.
I8: IPC operations validate all required authority.
I9: Scheduler queues contain no duplicate thread entries.
I10: Security checks cannot be bypassed through identifiers alone.
```

These invariants should eventually be represented in machine-checkable specifications where practical.

---

## 8. Specification Before Proof

Before formally proving a component, define its intended behavior.

For each critical subsystem document:

```text
Inputs
Outputs
State
Allowed transitions
Security assumptions
Invariants
Failure behavior
Concurrency assumptions
```

Example:

```text
Capability Derive

Input:
    Existing capability
    Requested rights

Precondition:
    Requested rights ⊆ existing rights

Result:
    New capability with requested rights

Security property:
    New authority ≤ source authority
```

---

## 9. Verification Layers

### Layer 1 — Compiler

Every commit should compile the relevant targets.

Example:

```powershell
cargo check --workspace
```

The exact CI commands should follow the project's supported Rust toolchain.

---

### Layer 2 — Formatting

Use automated formatting checks.

```powershell
cargo fmt --all -- --check
```

Formatting failures should fail CI.

---

### Layer 3 — Linting

Use Clippy where compatible with the kernel configuration.

```powershell
cargo clippy --workspace
```

Warnings that affect safety should be reviewed rather than blindly suppressed.

---

### Layer 4 — Unit Tests

Subsystems should have focused tests.

Priority targets:

```text
Capability tables
Capability rights
Run queues
Thread states
Memory-region calculations
IPC state machines
Object lifetime
```

---

## 10. Host-Side Tests

Logic that does not require actual hardware should preferably be testable outside the booted kernel.

Examples:

```text
Capability derivation
Priority ordering
Queue operations
Address-range calculations
State-machine transitions
Identifier validation
```

This makes testing faster and easier.

---

## 11. Kernel Integration Tests

Integration tests should execute real kernel components together.

Example:

```text
Boot
 ↓
Memory initialization
 ↓
Capability initialization
 ↓
Scheduler initialization
 ↓
IPC initialization
 ↓
Create initial thread
 ↓
Start userspace
```

Failures should identify the subsystem and invariant involved.

---

## 12. Boot Verification

The boot process should verify:

```text
Kernel image is correctly constructed.
Entry point is valid.
Boot information is interpreted safely.
Memory regions are valid.
Required CPU state is initialized.
Kernel invariants are established before userspace starts.
```

The bootloader itself should remain outside Aegis Core's own kernel logic where appropriate, while the interface between bootloader and kernel must be carefully validated.

---

## 13. Memory Verification

Memory management is one of the highest-priority verification areas.

Verify:

```text
Page allocation
Page ownership
Mapping
Unmapping
Permissions
Address-space isolation
Kernel mappings
User mappings
Shared memory
Page-table consistency
Physical/virtual address conversions
```

Important invariant:

```text
No unauthorized mapping can be created.
```

---

## 14. Capability Verification

Test:

```text
Capability creation
Capability lookup
Capability rights
Capability derivation
Capability transfer
Capability revocation
Capability destruction
Invalid handles
Stale handles
Object lifetime
```

Example property:

```text
derive(source, requested_rights)

must satisfy:

requested_rights ⊆ source.rights
```

No operation should create authority that was not already authorized.

---

## 15. IPC Verification

Verify the complete IPC state machine.

Example:

```text
Endpoint
   │
   ├── Sender waits
   ├── Receiver waits
   ├── Message transfer
   └── Wake-up
```

Test:

```text
Normal messages
Empty messages
Maximum-size messages
Invalid messages
Blocked senders
Blocked receivers
Timeouts
Capability transfer
Concurrent send/receive
Endpoint destruction
```

Security property:

```text
No unauthorized IPC communication.
```

---

## 16. Scheduler Verification

Verify:

```text
Thread creation
Thread destruction
Ready state
Running state
Blocked state
Priority
Round robin
Yield
Preemption
Wake-up
CPU affinity
Context switching
SMP behavior
```

Important properties:

```text
Blocked threads never run.
Terminated threads never run.
A thread cannot run simultaneously on multiple CPUs.
```

---

## 17. Concurrency Verification

Concurrency is a major verification target once SMP support exists.

Check:

```text
Lock correctness
Atomic operations
Reference counting
Object lifetime
Scheduler queues
Capability tables
IPC queues
Memory mappings
Interrupt state
```

Look specifically for:

```text
Data races
Deadlocks
Livelocks
Use-after-free
Double free
Lost wake-ups
Double scheduling
```

---

## 18. Interrupt Verification

Verify:

```text
Interrupt registration
Interrupt routing
Interrupt acknowledgement
Interrupt masking
Interrupt capabilities
Interrupt-to-thread notification
```

Security property:

```text
Untrusted code cannot arbitrarily control interrupt routing.
```

---

## 19. Architecture Verification

Architecture-specific code should be isolated.

For x86_64:

```text
kernel/src/arch/x86_64/
```

Important verification targets:

```text
CPU initialization
GDT/segment state
Interrupt entry
Interrupt return
Context switching
Page tables
TLB management
Timer setup
CPU feature handling
```

Architecture-specific unsafe code requires particularly careful review.

---

## 20. Unsafe Code Audit

Every unsafe block should answer:

```text
Why is unsafe required?
What invariant makes it safe?
Who establishes that invariant?
How is it maintained?
What happens if the assumption is violated?
```

Use explicit safety comments.

Example:

```rust
// SAFETY:
// The address has been validated as a kernel-owned,
// correctly aligned mapping before this operation.
unsafe {
    ...
}
```

The project should maintain an inventory of unsafe code.

---

## 21. Static Analysis

Use available static-analysis tools where they support the kernel configuration.

Potential checks include:

```text
cargo check
cargo clippy
rustfmt
dependency auditing
unsafe-code review
architecture-specific linting
```

Tools should be pinned or documented in CI where reproducibility matters.

---

## 22. Dependency Verification

Dependencies should be minimized.

For each dependency document:

```text
Purpose
Version
License
Security relevance
Whether it enters the TCB
Whether it is required at runtime
```

The kernel should avoid unnecessary dependencies.

Dependency updates should be reviewed rather than automatically accepted.

---

## 23. Fuzz Testing

Fuzzing should target interfaces that process attacker-controlled input.

High-value targets:

```text
Syscall arguments
IPC messages
Capability handles
Memory-map requests
Object identifiers
Boot information
Userspace buffers
Protocol parsers
```

Fuzzers should look for:

```text
Panics
Memory corruption
Invariant violations
Unexpected acceptance
Incorrect authorization
Infinite loops
```

---

## 24. Property-Based Testing

Property-based tests should test general rules instead of only fixed examples.

Example:

```text
For every valid capability:
    derived rights ⊆ source rights
```

Example:

```text
For every valid scheduler state:
    a blocked thread is never runnable
```

Example:

```text
For every valid memory mapping:
    permissions match the authorized mapping
```

---

## 25. Model Checking

Small state machines should eventually be model checked.

Good candidates:

```text
Capability derivation
Capability revocation
IPC
Thread state transitions
Scheduler queues
Object lifetime
```

Model checking is especially useful for discovering concurrency and state-transition bugs before attempting full formal proofs.

---

## 26. Formal Specification

The most critical components should eventually have formal specifications.

Potential specification layers:

```text
Abstract security model
        ↓
Kernel object model
        ↓
Capability model
        ↓
Memory model
        ↓
IPC model
        ↓
Scheduler model
        ↓
Concrete implementation
```

The abstract model should define what the kernel is allowed to do.

---

## 27. Refinement

A major long-term goal is refinement:

```text
Abstract specification
        ↓
Refined kernel model
        ↓
Concrete implementation
```

The purpose is to show that the implementation behaves according to the security properties of the abstract model.

Aegis Core should avoid attempting a complete proof before its architecture and invariants are stable.

---

## 28. Formal Verification Scope

Initial formal verification should focus on small, security-critical mechanisms.

Recommended order:

```text
1. Capability rights
2. Capability derivation
3. Capability lookup
4. Object lifetime
5. Memory isolation
6. IPC authorization
7. Thread-state invariants
8. Scheduler invariants
9. Context-switch assumptions
10. Full kernel refinement
```

The exact proof technology can be selected once the kernel architecture is mature enough.

---

## 29. Verification of Security Boundaries

Every boundary should have an explicit verification target.

```text
Userspace ↔ Kernel
Process ↔ Process
Thread ↔ Thread
Service ↔ Service
Device ↔ Memory
CPU ↔ Address Space
Capability ↔ Object
```

For each boundary ask:

```text
What authority crosses the boundary?
How is it validated?
What prevents unauthorized access?
What invariant guarantees isolation?
```

---

## 30. Negative Testing

Verification must test rejected operations as heavily as successful operations.

Examples:

```text
Invalid capability → reject
Insufficient rights → reject
Invalid pointer → reject
Invalid address range → reject
Unauthorized IPC → reject
Invalid thread state → reject
Invalid CPU affinity → reject
Destroyed object → reject
```

Security systems must verify both:

```text
What is allowed
and
What must never be allowed
```

---

## 31. Fault Injection

Introduce controlled failures into tests.

Examples:

```text
Allocation failure
Invalid userspace pointer
Unexpected interrupt
IPC endpoint disappearance
Thread termination during IPC
Memory exhaustion
Concurrent object destruction
```

The expected result should be safe failure rather than corrupted kernel state.

---

## 32. Runtime Assertions

During development builds, important invariants may be checked at runtime.

Examples:

```text
assert thread state is valid
assert queue membership is consistent
assert capability rights are valid
assert object lifetime is valid
assert CPU ownership is correct
```

Production builds may reduce diagnostic checks only after the underlying invariants are well established.

---

## 33. Debug Kernel

A debug configuration should provide additional verification.

Potential features:

```text
Invariant checking
Scheduler diagnostics
Capability diagnostics
Memory mapping checks
Object lifetime checks
Verbose panic information
Test-only assertions
```

Debug-only features must not silently change security semantics.

---

## 34. CI Verification

A future CI pipeline should approximately follow:

```text
Checkout
   ↓
Toolchain verification
   ↓
Formatting
   ↓
Compilation
   ↓
Clippy/static analysis
   ↓
Unit tests
   ↓
Integration tests
   ↓
Property tests
   ↓
Security checks
   ↓
Kernel image build
   ↓
Boot test in emulator
   ↓
Verification artifacts
```

Every stage should produce reproducible results.

---

## 35. Emulator Testing

QEMU should be used for repeatable kernel boot and integration testing.

Example conceptual pipeline:

```text
cargo build
     ↓
Aegis BIOS image
     ↓
QEMU
     ↓
Boot kernel
     ↓
Serial output
     ↓
Test result
```

The kernel should provide a deterministic serial/debug interface for automated tests.

---

## 36. Reproducible Builds

Verification is stronger when builds are reproducible.

Record:

```text
Rust version
Target
Bootloader version
Dependency versions
Build configuration
Compiler flags
Verification tool versions
```

A release should be traceable to its exact source revision and toolchain.

---

## 37. Verification Artifacts

The project should store or generate:

```text
Test results
Coverage reports
Static-analysis results
Fuzzing results
Model-checking results
Formal specifications
Proof artifacts
Build metadata
Security-review notes
```

Verification artifacts should be versioned where practical.

---

## 38. Coverage

Coverage should be tracked for ordinary tests.

Coverage should include:

```text
Normal paths
Error paths
Security checks
State transitions
Concurrency-sensitive paths
Capability operations
Memory operations
IPC operations
```

High code coverage does not prove correctness, but low coverage identifies areas needing more testing.

---

## 39. Verification Matrix

| Component | Unit | Integration | Fuzz | Model | Formal |
|---|---|---|---|---|---|
| Capabilities | Required | Required | Required | Required | High priority |
| Memory | Required | Required | Required | Required | High priority |
| IPC | Required | Required | Required | Required | High priority |
| Scheduler | Required | Required | Useful | Required | High priority |
| Interrupts | Required | Required | Useful | Useful | Later |
| Architecture | Required | Required | Useful | Useful | Later |
| Boot | Useful | Required | Useful | Useful | Later |
| Userspace services | Required | Required | Service-dependent | Optional | Usually outside kernel proof |

---

## 40. Verification Milestones

### Milestone 1 — Bootable kernel

Requirements:

```text
Kernel builds
Kernel boots in QEMU
Serial output works
Basic panic handler works
```

### Milestone 2 — Stable kernel primitives

Requirements:

```text
Thread model defined
Memory model defined
Capability model defined
IPC model defined
Scheduler model defined
```

### Milestone 3 — Security invariants

Requirements:

```text
Core invariants documented
Negative tests implemented
Unsafe-code inventory created
Security boundaries documented
```

### Milestone 4 — Strong testing

Requirements:

```text
Unit tests
Integration tests
Property tests
Fuzz tests
QEMU automated tests
```

### Milestone 5 — Formal models

Requirements:

```text
Capability model
Memory model
IPC model
Scheduler model
State-transition specifications
```

### Milestone 6 — Formal proofs

Requirements:

```text
Critical capability properties proven
Critical memory-isolation properties proven
Critical IPC properties proven
Critical scheduler properties proven
```

### Milestone 7 — Refinement

Long-term goal:

```text
Abstract security model
        ↓
Formal kernel model
        ↓
Concrete Aegis Core implementation
```

---

## 41. Verification Priorities

Priority should be:

```text
P0 — Security boundary
P1 — Memory safety
P2 — Capability correctness
P3 — IPC correctness
P4 — Scheduler correctness
P5 — Concurrency
P6 — Architecture correctness
P7 — Performance
```

Performance should not be optimized at the expense of established security invariants.

---

## 42. What Counts as Verified?

A component should not be described as formally verified merely because:

```text
It compiles
It passes unit tests
It boots
It has high coverage
It works in QEMU
```

Those are testing and engineering evidence.

A component should be described as formally verified only when a documented formal specification and proof establish the claimed property under stated assumptions.

---

## 43. Assumptions

Formal results are only as strong as their assumptions.

Document assumptions such as:

```text
CPU behaves according to specified architecture semantics
Boot environment provides required guarantees
Hardware is not malicious
Cryptographic assumptions hold
Compiler/toolchain assumptions hold
Verified components are used exactly as specified
```

Assumptions must be explicit rather than hidden.

---

## 44. Security Review Checklist

Before merging security-critical changes:

```text
[ ] Threat model considered
[ ] Security boundary identified
[ ] Capability implications reviewed
[ ] Memory implications reviewed
[ ] Concurrency implications reviewed
[ ] Unsafe code reviewed
[ ] Invariants updated if necessary
[ ] Negative tests added
[ ] Regression tests added
[ ] Documentation updated
[ ] Formal specification updated if applicable
```

---

## 45. Development Rule

The verification strategy for Aegis Core is:

```text
Build it
   ↓
Test it
   ↓
Break it
   ↓
Check its invariants
   ↓
Model it
   ↓
Specify it
   ↓
Prove the critical properties
```

The objective is to gradually move security guarantees from assumptions and tests toward machine-checked properties.

Aegis Core should remain small enough that this verification strategy is realistically achievable.
