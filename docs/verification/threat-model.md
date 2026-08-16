# Aegis Core — Threat Model

## 1. Purpose

This document defines the security threat model for Aegis Core.

Aegis Core is designed as a small, capability-based microkernel inspired by the security architecture and verification goals of systems such as seL4.

The purpose of this document is to identify:

- assets that must be protected
- trust boundaries
- attackers and their capabilities
- attack surfaces
- security threats
- security assumptions
- required mitigations
- properties that should eventually be formally verified

This document should be updated whenever the kernel architecture changes.

---

# 2. Security Objectives

The primary security objectives are:

```text
O1 — Kernel integrity
O2 — Kernel confidentiality
O3 — Address-space isolation
O4 — Capability integrity
O5 — IPC authorization
O6 — Thread isolation
O7 — Resource isolation
O8 — Controlled hardware access
O9 — Secure privilege transitions
O10 — Verifiable security properties
```

The fundamental principle is:

```text
No authority without an explicit security mechanism.
```

---

# 3. Security Model

Aegis Core should follow a least-authority model.

```text
Kernel
  │
  ├── Capability system
  │
  ├── Memory protection
  │
  ├── IPC
  │
  ├── Scheduling
  │
  └── Hardware mediation
       │
       ▼
    Userspace
```

Userspace components should not receive authority merely because they are userspace components.

Authority should be explicitly represented and controlled.

---

# 4. Assets

## 4.1 Kernel Code

The kernel's executable code must be protected from unauthorized modification.

Threats include:

```text
Userspace modification
DMA-based modification
Memory corruption
Unauthorized kernel mappings
Boot-time modification
```

---

## 4.2 Kernel Memory

Kernel memory contains:

```text
Kernel stacks
Scheduler state
Capability tables
Memory-management structures
IPC state
Object metadata
Security-critical state
```

Unauthorized access could compromise the entire system.

---

## 4.3 Capability State

Capability metadata represents authority.

Protection requirements:

```text
Capabilities cannot be forged.
Rights cannot be increased.
Revoked authority cannot remain usable.
Capability references cannot become dangling.
```

---

## 4.4 User Memory

User processes must be isolated from each other.

A process should only access:

```text
its own memory
```

or memory explicitly shared through authorized mechanisms.

---

## 4.5 CPU State

CPU state includes:

```text
Registers
Page-table state
Privilege level
Interrupt state
Thread context
Control registers
```

Incorrect handling can break isolation or privilege boundaries.

---

## 4.6 Hardware Resources

Potential protected resources include:

```text
Physical memory
MMIO regions
Interrupts
Timers
DMA-capable devices
Storage
Network devices
CPU features
```

Access should be mediated by explicit authority.

---

# 5. Trust Boundaries

The main trust boundaries are:

```text
Boot environment
       │
       ▼
Aegis Core
       │
       ├──────────────┐
       ▼              ▼
 Userspace A      Userspace B
       │              │
       └──────┬───────┘
              ▼
             IPC
```

Additional boundaries include:

```text
Kernel ↔ Hardware
Kernel ↔ Firmware
Kernel ↔ Bootloader
Userspace ↔ Kernel
Userspace ↔ Userspace
Device ↔ Memory
CPU ↔ Address Space
```

Every boundary must have an explicit security mechanism.

---

# 6. Threat Actors

## 6.1 Malicious Userspace Process

This is the primary threat actor.

Assumptions:

```text
The process may be fully compromised.
The process may intentionally call syscalls incorrectly.
The process may send malformed IPC.
The process may attempt unauthorized memory access.
The process may attempt resource exhaustion.
```

The kernel must not trust userspace input.

---

## 6.2 Compromised Userspace Service

A legitimate service may become compromised.

Examples:

```text
Filesystem service
Network service
Device service
Display service
Authentication service
```

A compromised service should have only the capabilities it actually requires.

---

## 6.3 Malicious or Compromised Device

A device may potentially:

```text
Generate unexpected interrupts
Perform DMA
Return malformed data
Access memory
Trigger unusual hardware states
```

Hardware trust assumptions must be explicitly documented.

---

## 6.4 Malicious Boot Environment

Depending on the threat model, the bootloader or firmware may be considered trusted, partially trusted, or outside the kernel's security guarantee.

Aegis Core should clearly state which assumptions apply.

---

## 6.5 Physical Attacker

A physical attacker may potentially:

```text
Modify storage
Replace boot media
Access hardware
Reset the machine
Inspect memory
Modify firmware
```

Physical attacks are outside the basic kernel threat model unless additional platform-security mechanisms are implemented.

---

# 7. Attacker Capabilities

The default untrusted userspace attacker may:

```text
Execute arbitrary userspace code
Issue arbitrary system calls
Provide malformed syscall arguments
Attempt invalid memory accesses
Attempt unauthorized IPC
Attempt to obtain capabilities
Attempt resource exhaustion
Crash its own process
Interact with authorized services
```

The attacker must not be assumed to have:

```text
Kernel memory access
Kernel execution
Arbitrary physical memory access
Forged capabilities
Hardware privilege
```

Those are precisely the properties the kernel must protect.

---

# 8. Attack Surface

The primary attack surfaces are:

```text
System calls
IPC
Capability operations
Memory mapping
Thread creation
Thread control
Scheduler interfaces
Interrupt interfaces
Device interfaces
Shared memory
Userspace pointers
Boot information
Architecture-specific entry/exit code
```

The kernel should minimize these interfaces.

---

# 9. System Call Threats

## T-001 — Invalid Arguments

An attacker may supply:

```text
Null pointers
Invalid addresses
Out-of-range values
Invalid object IDs
Malformed structures
Unexpected flags
```

### Mitigation

Every syscall must validate all attacker-controlled inputs.

---

## T-002 — Pointer Confusion

Userspace pointers must never automatically be treated as trusted kernel pointers.

### Mitigation

Use explicit safe user-memory access mechanisms and validate:

```text
address
length
permissions
overflow
mapping
ownership
```

---

## T-003 — Integer Overflow

Attackers may attempt to exploit arithmetic overflow in:

```text
sizes
addresses
offsets
page counts
buffer lengths
```

### Mitigation

Use checked arithmetic and explicit bounds validation for security-critical calculations.

---

# 10. Capability Threats

## T-004 — Capability Forgery

An attacker may attempt to construct a value that looks like a valid capability.

### Mitigation

Capability validity must depend on protected kernel state, not merely an attacker-controlled identifier.

---

## T-005 — Rights Escalation

An attacker may attempt:

```text
READ → WRITE
USER → KERNEL
LIMITED → FULL
```

### Mitigation

Enforce:

```text
derived_rights ⊆ source_rights
```

and verify this invariant throughout capability operations.

---

## T-006 — Stale Capability

An attacker may attempt to reuse a capability after the referenced object was destroyed.

### Mitigation

Use a capability/object lifetime design that prevents stale authority from becoming valid again.

---

## T-007 — Confused Deputy

A privileged service may accidentally use its own authority on behalf of an untrusted process.

### Mitigation

Services should:

```text
Validate caller authority
Use explicit capability passing
Avoid ambient authority
Minimize privileged capabilities
```

---

# 11. Memory Threats

## T-008 — Unauthorized Memory Access

A process may attempt to access another process's memory.

### Mitigation

Hardware page protection plus kernel-controlled address-space management.

---

## T-009 — Kernel Memory Mapping

A process may attempt to map protected kernel memory.

### Mitigation

Kernel mappings must be protected and unavailable to untrusted domains.

---

## T-010 — Unauthorized Writable Mapping

A read-only or protected page may be mapped as writable.

### Mitigation

The kernel must verify requested permissions against the authority represented by capabilities.

---

## T-011 — Use-After-Free

A process or kernel subsystem may retain a reference to an object after its lifetime ends.

### Mitigation

Use explicit object lifetime and reference rules.

---

## T-012 — Double Free

An object or physical page may be released twice.

### Mitigation

Resource ownership and object lifecycle invariants.

---

## T-013 — Memory Aliasing

Unauthorized aliases may allow one security domain to modify memory belonging to another.

### Mitigation

All shared mappings must be explicitly authorized.

---

# 12. IPC Threats

## T-014 — Unauthorized IPC

A process may attempt to communicate with an endpoint it does not have authority to access.

### Mitigation

Endpoint access requires an appropriate capability.

---

## T-015 — Malformed Message

A malicious process may send invalid or unexpected message data.

### Mitigation

IPC interfaces must validate:

```text
message size
message structure
capabilities
buffer references
rights
```

---

## T-016 — Capability Transfer Escalation

A malicious sender may attempt to transfer more authority than it possesses.

### Mitigation

Transferred rights must never exceed source authority.

---

## T-017 — IPC Denial of Service

A process may attempt to consume IPC resources indefinitely.

### Mitigation

Use:

```text
resource limits
bounded queues where appropriate
timeouts where appropriate
scheduler controls
```

---

# 13. Scheduler Threats

## T-018 — Scheduler Manipulation

A process may attempt to obtain unauthorized CPU time.

### Mitigation

Scheduler policy must be enforced by the kernel.

---

## T-019 — Priority Abuse

A process may attempt to manipulate priority to starve other tasks.

### Mitigation

Priority changes require appropriate authority.

---

## T-020 — Invalid Thread State

Malformed operations may attempt to place a thread into contradictory states.

Example:

```text
RUNNING + BLOCKED
```

### Mitigation

Explicit thread-state machine and invariant checks.

---

## T-021 — Scheduler Data Corruption

A corrupted run queue could result in:

```text
execution of invalid threads
double scheduling
lost threads
security-boundary violations
```

### Mitigation

Maintain scheduler invariants and validate queue operations.

---

# 14. Interrupt Threats

## T-022 — Unauthorized Interrupt Control

Userspace may attempt to redirect protected interrupts.

### Mitigation

Interrupt configuration requires explicit authority.

---

## T-023 — Interrupt Flooding

A device or malicious component may generate excessive interrupts.

### Mitigation

Use interrupt moderation, rate limiting, device isolation, or appropriate scheduling policies where required.

---

## T-024 — Interrupt State Corruption

Incorrect interrupt state may break kernel synchronization or privilege transitions.

### Mitigation

Architecture-specific interrupt code must maintain strict invariants and be heavily tested.

---

# 15. DMA Threats

DMA-capable hardware can potentially access physical memory without ordinary CPU page-table checks.

## T-025 — Unauthorized DMA

A device may modify memory outside its authorized region.

### Mitigation

Where supported, use an IOMMU or equivalent hardware isolation mechanism.

DMA ownership and mappings must be explicitly controlled.

---

# 16. Resource Exhaustion

## T-026 — Memory Exhaustion

A process may allocate excessive memory.

### Mitigation

Use:

```text
Capability-controlled allocation
Resource quotas
Per-domain limits
Safe allocation failure
```

---

## T-027 — Object Exhaustion

An attacker may create excessive:

```text
Threads
Endpoints
Capabilities
Address spaces
Kernel objects
```

### Mitigation

Resource accounting and explicit limits.

---

## T-028 — CPU Exhaustion

A process may attempt to monopolize CPU resources.

### Mitigation

Scheduler policy and resource controls.

---

# 17. Concurrency Threats

## T-029 — Race Conditions

Concurrent operations may create inconsistent security state.

High-risk areas:

```text
Capability revocation
Object destruction
IPC
Scheduler queues
Memory mappings
Reference counting
```

### Mitigation

Use carefully defined synchronization and eventually model-check concurrency-sensitive state machines.

---

## T-030 — Deadlock

Kernel locks may become permanently blocked.

### Mitigation

Define lock ordering and minimize lock scope.

---

## T-031 — Lost Wake-Up

A thread may remain blocked because a wake-up event was incorrectly handled.

### Mitigation

Treat scheduler and IPC state transitions as atomic operations where required.

---

# 18. Privilege Transition Threats

## T-032 — Userspace-to-Kernel Transition

Malformed syscall entry state may attempt to exploit privilege transitions.

### Mitigation

Validate architecture-defined entry state and construct a controlled kernel execution context.

---

## T-033 — Kernel-to-Userspace Return

Incorrect return state could potentially grant unintended privilege.

### Mitigation

Validate:

```text
instruction pointer
stack pointer
page tables
privilege level
segment state where applicable
CPU flags
```

before returning to userspace.

---

# 19. Boot Threats

## T-034 — Modified Kernel Image

An attacker with control of the boot environment may replace the kernel image.

### Mitigation

Possible future mechanisms:

```text
Secure Boot
Measured Boot
Signed kernel images
Verified boot chain
```

These mechanisms are platform-level security features and should be clearly separated from the microkernel's own guarantees.

---

## T-035 — Malformed Boot Information

Bootloader-provided information may be invalid or unexpected.

### Mitigation

Validate all boot information before using it to establish kernel state.

---

# 20. Firmware Threats

Firmware may have privileged access below the kernel.

Potential threats include:

```text
Malicious firmware
Firmware vulnerabilities
Unauthorized device configuration
SMM-level attacks
```

The base Aegis Core threat model should document whether firmware is trusted.

---

# 21. Side-Channel Threats

Potential side channels include:

```text
Timing
Caches
Branch predictors
Speculative execution
Shared CPU resources
Memory contention
```

These are advanced threats.

Initial Aegis Core verification should prioritize direct isolation failures first.

Future designs may add explicit mitigations where required.

---

# 22. Denial of Service

Aegis Core should distinguish:

```text
Security violation
```

from:

```text
Availability attack
```

A malicious userspace process may be able to consume its own resources or intentionally terminate itself.

It should not be able to corrupt the kernel or permanently prevent unrelated security domains from functioning merely through ordinary unprivileged operations.

---

# 23. Threats Outside the Base Model

The following may require separate platform-security models:

```text
Physical attacks
Compromised firmware
Malicious CPU hardware
Compromised boot ROM
Advanced side-channel attacks
Hardware fault injection
Supply-chain attacks
Malicious peripherals
```

These should not silently be considered protected unless corresponding mechanisms exist.

---

# 24. Security Assumptions

The initial model assumes:

```text
A1 — CPU hardware follows its documented architecture.
A2 — Memory-protection hardware operates correctly.
A3 — The compiler/toolchain satisfies documented assumptions.
A4 — The boot environment provides the guarantees explicitly claimed by Aegis Core.
A5 — Cryptographic mechanisms, if used, satisfy their documented assumptions.
A6 — Hardware outside the trusted model cannot arbitrarily modify protected memory.
A7 — Verified kernel code is executed as specified.
```

Assumptions should be reviewed whenever the threat model changes.

---

# 25. Security Boundaries and Required Properties

| Boundary | Required Property |
|---|---|
| Userspace → Kernel | Syscalls cannot bypass authorization |
| Process → Process | Memory isolation |
| Process → Process | IPC requires authority |
| Userspace → Hardware | Explicit capability |
| Kernel → Userspace | Controlled privilege transition |
| Capability → Object | Valid authority required |
| Device → Memory | DMA isolation |
| Bootloader → Kernel | Boot information validation |
| CPU → Address Space | Correct page-table enforcement |

---

# 26. High-Priority Threats

The initial highest-priority threats are:

```text
T-004  Capability forgery
T-005  Capability rights escalation
T-007  Confused deputy
T-008  Unauthorized memory access
T-009  Kernel memory mapping
T-010  Unauthorized writable mapping
T-011  Use-after-free
T-014  Unauthorized IPC
T-016  Capability transfer escalation
T-020  Invalid thread state
T-025  Unauthorized DMA
T-029  Race conditions
T-032  Privilege-transition bugs
T-033  Kernel-to-userspace return bugs
T-034  Modified kernel image
```

These should receive priority in design reviews and verification.

---

# 27. Threat-to-Invariant Mapping

Threats should map to the invariants defined in `invariants.md`.

Examples:

```text
T-004 → C-001, C-003, C-005
T-005 → C-002
T-006 → C-001, O-002
T-008 → M-001
T-009 → M-002
T-010 → M-003, M-004
T-011 → O-003, O-004
T-014 → Ipc-001
T-016 → Ipc-004
T-020 → T-001, T-002, T-003
T-025 → M-005, R-001
T-029 → CC-001, CC-002, CC-005
T-032 → A-architecture invariants
T-033 → A-architecture invariants, X-001
```

The mapping should be expanded as the kernel develops.

---

# 28. Verification Strategy

Security threats should be connected to verification methods.

```text
Threat
  ↓
Security property
  ↓
Invariant
  ↓
Test
  ↓
Model
  ↓
Formal proof
```

Example:

```text
Capability escalation
        ↓
Derived rights cannot increase
        ↓
C-002
        ↓
Property-based tests
        ↓
Capability state model
        ↓
Formal proof
```

---

# 29. Security Review Checklist

For every security-sensitive change:

```text
[ ] Threat model reviewed
[ ] Trust boundary identified
[ ] Attacker capabilities considered
[ ] Affected assets identified
[ ] Relevant invariant identified
[ ] Capability implications reviewed
[ ] Memory implications reviewed
[ ] IPC implications reviewed
[ ] Scheduler implications reviewed
[ ] Concurrency implications reviewed
[ ] Unsafe code reviewed
[ ] Negative tests added
[ ] Regression tests added
[ ] Formal specification updated if applicable
```

---

# 30. Threat Model Maintenance

This document must evolve with the kernel.

Update it when:

```text
A new syscall is added
A new capability type is added
A new hardware interface is added
SMP is introduced
DMA support is introduced
New userspace services are introduced
Memory-management architecture changes
IPC semantics change
Privilege transitions change
Formal verification scope changes
```

---

# 31. Security Principle

The central Aegis Core security principle is:

```text
Minimize authority.
Make authority explicit.
Validate every boundary.
Preserve invariants.
Keep the kernel small.
Verify the critical properties.
```

The long-term objective is to make the most important claims machine-checkable rather than relying solely on testing or code review.

---

# 32. Status

Current status:

```text
[✓] Assets identified
[✓] Trust boundaries identified
[✓] Primary attacker model
[✓] Attack surfaces
[✓] Capability threats
[✓] Memory threats
[✓] IPC threats
[✓] Scheduler threats
[✓] Interrupt threats
[✓] DMA threats
[✓] Concurrency threats
[✓] Resource-exhaustion threats
[✓] Boot threats
[✓] Threat-to-invariant mapping
[ ] Automated threat regression tests
[ ] Formal threat model
[ ] End-to-end security proof
```

This threat model should remain aligned with `invariants.md`, `verification-plan.md`, `capabilities.md`, `memory.md`, `ipc.md`, and `scheduling.md`.
