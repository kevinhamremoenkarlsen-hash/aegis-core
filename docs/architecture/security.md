# Aegis Core — Security

## 1. Purpose

Security is a fundamental part of Aegis Core.

Aegis Core is designed as a small, capability-based microkernel inspired by the security principles of systems such as seL4.

The security architecture should prioritize:

- Strong isolation
- Least privilege
- Capability-based authority
- Minimal kernel attack surface
- Memory safety
- Controlled IPC
- Explicit resource ownership
- Formal verification where practical
- Deterministic and auditable behavior

The kernel should provide mechanisms, while higher-level policy should remain outside the kernel whenever possible.

---

## 2. Security Model

The basic security model is:

```text
             Aegis Core
                 │
       ┌─────────┴─────────┐
       │                   │
   Mechanisms           Isolation
       │                   │
       └─────────┬─────────┘
                 │
          Capability System
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
    Threads    Memory      IPC
       │         │         │
       └─────────┼─────────┘
                 ▼
             User space
```

The kernel should not trust userspace merely because it is part of the operating system.

---

## 3. Core Security Principle

The central security rule is:

> Possession of a capability grants authority; absence of a capability means no authority.

A thread should not gain access merely because it knows an object ID, memory address, handle, or name.

Authorization must be based on valid kernel-managed authority.

---

## 4. Least Privilege

Every component should receive only the authority it needs.

For example:

```text
Network service
 ├── Network capabilities
 └── No filesystem capability

Filesystem service
 ├── Storage capabilities
 └── No network capability

Application
 ├── Specific service capabilities
 └── No direct hardware access
```

Avoid giving large global privileges when a smaller capability is sufficient.

---

## 5. Capability System

Aegis Core should use capabilities as the primary kernel authorization mechanism.

A capability represents authority over a kernel object or resource.

Conceptually:

```text
Capability
├── Object reference
├── Rights
└── Capability metadata
```

Possible rights include:

```text
Read
Write
Execute
Grant
Manage
Send
Receive
Map
Control
```

Rights should be explicit.

---

## 6. Capability Derivation

Capabilities may be derived from existing authority.

Example:

```text
Root capability
      │
      ├── Read-only capability
      │
      └── Limited management capability
```

A derived capability must never gain more authority than its parent.

This follows the principle:

```text
Derived authority ≤ Source authority
```

---

## 7. Capability Transfer

Capabilities may be transferred through controlled IPC.

Conceptually:

```text
Process A
   │
   │ capability transfer
   ▼
IPC
   │
   ▼
Process B
```

The kernel must validate the transfer.

A sender must possess sufficient rights to grant the capability.

---

## 8. Capability Revocation

Aegis Core must eventually support safe capability revocation.

Revocation must account for:

- Derived capabilities
- Transferred capabilities
- Object lifetime
- Outstanding IPC
- Mapped resources
- Cached references

Revocation must never leave an invalid capability usable.

A future implementation may use capability derivation trees or another explicit revocation structure.

---

## 9. Capability Handles

Userspace should not directly manipulate raw kernel pointers.

A capability handle should be an opaque userspace value.

Example:

```text
Userspace:
    CapabilityHandle(42)

Kernel:
    handle → capability → object
```

The handle must not expose the physical or virtual address of the kernel object.

---

## 10. Object Isolation

Kernel objects must have explicit ownership and lifetime rules.

Examples:

```text
Thread
Address Space
IPC Endpoint
Notification
Memory Object
Capability Space
Interrupt Object
Device Object
```

A userspace process must not access another process's kernel objects without an appropriate capability.

---

## 11. Memory Isolation

Each address space should be isolated.

Conceptually:

```text
Process A
┌──────────────────┐
│ Private memory   │
└──────────────────┘

Process B
┌──────────────────┐
│ Private memory   │
└──────────────────┘
```

Process A must not access Process B's memory unless an explicit shared-memory capability permits it.

---

## 12. Kernel Memory Protection

Kernel memory must remain inaccessible to ordinary userspace.

The memory subsystem must enforce separation between:

```text
Kernel memory
User memory
Device memory
Shared memory
```

User-controlled pointers must never be trusted without validation.

---

## 13. Pointer Safety

Kernel interfaces must treat all userspace addresses as untrusted.

Before accessing userspace memory, the kernel must verify:

```text
Address is valid
Address belongs to permitted address space
Operation is permitted
Range is valid
No integer overflow occurred
```

The kernel should avoid dereferencing unchecked userspace pointers.

---

## 14. Rust Memory Safety

Aegis Core should use Rust's type and ownership system to reduce memory-safety vulnerabilities.

Unsafe Rust should be restricted to small, well-defined areas such as:

```text
Architecture-specific code
Context switching
Interrupt entry
MMU/page-table operations
Hardware access
Boot code
Low-level synchronization
```

Unsafe code should be documented and reviewed.

The project should use:

```rust
#![deny(unsafe_op_in_unsafe_fn)]
```

where appropriate.

---

## 15. Unsafe Code Policy

Every unsafe block should have a clear safety justification.

Example:

```rust
// SAFETY:
// The pointer was created from a validated kernel-owned address
// and is aligned and valid for the required operation.
unsafe {
    ...
}
```

Unsafe code should not be used merely for convenience.

---

## 16. IPC Security

IPC is a security boundary.

The kernel must validate:

```text
Sender
Receiver
Endpoint capability
Message permissions
Capability transfers
Memory mappings
Object references
```

A process should not be able to send to an endpoint without appropriate authority.

---

## 17. Shared Memory

Shared memory must be explicitly authorized.

Example:

```text
Process A
   │
   │ shared-memory capability
   ▼
Kernel
   │
   ▼
Process B
```

Shared memory should never appear accidentally because two address spaces use the same physical memory.

---

## 18. System Calls

Every system call must validate its arguments.

Example:

```text
Userspace
   │
   │ syscall
   ▼
Kernel
   │
   ├── Validate syscall number
   ├── Validate capability
   ├── Validate arguments
   ├── Validate memory
   └── Perform operation
```

Invalid requests should fail safely.

---

## 19. Syscall Surface

The kernel should expose as few system calls as practical.

A smaller syscall interface provides:

```text
Smaller attack surface
Easier auditing
Simpler verification
Clearer security boundaries
```

Complex functionality should preferably be implemented by userspace services.

---

## 20. Privilege Separation

Aegis Core should avoid a traditional model where every system service runs with unrestricted kernel privileges.

Instead:

```text
Kernel
  │
  ├── Scheduler
  ├── Memory
  ├── IPC
  └── Capability enforcement
        │
        ▼
     Services
        │
        ▼
    Applications
```

Services should receive only the capabilities they require.

---

## 21. Device Access

Applications should not directly access hardware.

A typical architecture is:

```text
Application
     │
     ▼
Device Service
     │
     ▼
Capability-controlled kernel interface
     │
     ▼
Hardware
```

Hardware access should require explicit authority.

---

## 22. Interrupt Security

Interrupt objects should be protected by capabilities.

A userspace component should only receive interrupt notifications that it has been explicitly authorized to handle.

The kernel must prevent arbitrary userspace code from modifying interrupt-controller state.

---

## 23. DMA Security

DMA-capable devices can access memory independently of the CPU.

A future IOMMU-based design should restrict DMA access to authorized memory regions.

Conceptually:

```text
Device
  │
  ▼
IOMMU
  │
  ├── Allowed memory
  └── Blocked memory
```

DMA isolation is required for strong device isolation.

---

## 24. CPU Security

The scheduler must preserve security boundaries.

Scheduling priority must not grant:

```text
Memory access
Capability authority
Device access
IPC authority
```

The scheduler controls CPU allocation only.

---

## 25. Address-Space Security

Every address-space transition must be controlled by the memory subsystem.

The kernel must prevent:

```text
User → Kernel memory
Process A → Process B memory
Unprivileged code → unauthorized device memory
```

Address-space permissions should follow least privilege.

---

## 26. Executable Memory

Memory permissions should distinguish between:

```text
Read
Write
Execute
```

Where supported, writable memory should not automatically be executable.

The kernel should avoid unnecessary executable mappings.

---

## 27. Stack Protection

Stacks should have explicit bounds.

A future implementation may support:

```text
Guard pages
Stack bounds checking
Separate kernel/user stacks
Stack overflow detection
```

The kernel must avoid silently continuing after a stack-corruption condition.

---

## 28. Integer and Bounds Safety

Security-sensitive code must check for:

```text
Integer overflow
Integer underflow
Length overflow
Pointer arithmetic overflow
Array bounds
Page alignment
Address-space limits
```

Conversions between integer types should be explicit where they can affect memory safety.

---

## 29. Initialization Security

The boot process should initialize security-critical structures before exposing kernel services.

Conceptually:

```text
Boot
 │
 ├── CPU initialization
 ├── Memory initialization
 ├── Interrupt initialization
 ├── Capability system
 ├── Scheduler
 ├── IPC
 └── Start trusted userspace
```

Uninitialized security state must never be exposed to userspace.

---

## 30. Trusted Computing Base

The trusted computing base should be kept small.

Potential TCB components:

```text
Kernel
Architecture-specific low-level code
Critical boot code
Capability enforcement
Memory isolation
IPC enforcement
Context switching
```

Large userspace services should remain outside the kernel TCB whenever possible.

---

## 31. Attack Surface Reduction

Avoid placing unnecessary functionality in the kernel.

Prefer:

```text
Kernel:
  Mechanisms

Userspace:
  Policy
  Drivers where practical
  Filesystems
  Networking
  User services
```

A small kernel is easier to audit and verify.

---

## 32. Fault Isolation

A failure in a userspace service should not automatically compromise the kernel.

Example:

```text
Network service crashes
        │
        ▼
Other services continue
        │
        ▼
Kernel remains isolated
```

Critical services should be restartable where practical.

---

## 33. Kernel Panic Policy

A kernel panic indicates that a critical kernel invariant has failed.

The kernel should not attempt unsafe recovery from severe corruption.

The panic path should:

```text
Stop normal execution
Record minimal diagnostic information
Stop or safely halt the affected CPU/system
```

The exact production behavior can be defined later.

---

## 34. Information Disclosure

Kernel interfaces should avoid exposing unnecessary information.

Avoid leaking:

```text
Kernel addresses
Uninitialized memory
Other processes' data
Internal capability structures
Sensitive hardware state
```

Diagnostic interfaces should be deliberately designed.

---

## 35. Randomness

Security-sensitive random values should eventually use a properly initialized cryptographic random source.

Randomness may be required for:

```text
Security tokens
Memory randomization
Identifiers
Cryptographic services
```

The kernel should not treat predictable counters as cryptographic randomness.

---

## 36. Cryptography

Aegis Core should avoid implementing complex cryptographic primitives inside the kernel unless there is a strong architectural reason.

Prefer a small, auditable cryptographic service in userspace.

If cryptographic primitives are required in the kernel, they must have:

```text
Well-defined APIs
Known algorithms
Careful constant-time considerations
Independent review
Tests
```

---

## 37. Secure Identifiers

Kernel object identifiers should not be treated as authorization.

For example:

```text
Object ID = 1234
```

does not itself grant access.

Access requires:

```text
Valid capability
+
Required rights
```

---

## 38. Capability Confused-Deputy Protection

A service must not accidentally use its authority on behalf of an untrusted caller.

Example problem:

```text
Application
   │
   │ malicious request
   ▼
Privileged service
   │
   │ uses its own capability
   ▼
Protected resource
```

Services should verify that the caller is authorized for the requested operation.

---

## 39. TOCTOU Protection

Security-sensitive checks must account for time-of-check/time-of-use problems.

The kernel should avoid designs where:

```text
Check object
     │
     ▼
Object changes
     │
     ▼
Use object
```

Kernel object references and capabilities should provide stable authority semantics.

---

## 40. Concurrency Security

SMP concurrency must not create security bypasses.

Security-sensitive structures require correct synchronization.

Important targets:

```text
Capability tables
Object lifetime
Memory mappings
IPC endpoints
Scheduler state
Reference counts
Interrupt state
```

Race conditions must be treated as potential security vulnerabilities.

---

## 41. Reference and Lifetime Safety

Kernel objects must not be used after destruction.

A future object model should clearly define:

```text
Creation
Ownership
References
Capability references
Destruction
Revocation
Reclamation
```

Memory reclamation must occur only after all valid references are gone.

---

## 42. Logging and Diagnostics

Security diagnostics should provide useful information without unnecessarily exposing sensitive data.

Production logging should avoid leaking:

```text
Secrets
Raw capability contents
User data
Sensitive memory
Private keys
```

Debug builds may expose additional diagnostics under explicit developer controls.

---

## 43. Secure Defaults

Security-sensitive features should default to the safer behavior.

Examples:

```text
No capability → deny
Invalid handle → deny
Invalid address → reject
Unknown syscall → reject
Unknown object → reject
Missing permission → reject
```

Fail closed rather than fail open.

---

## 44. Security Testing

Testing should include:

### Capability tests

- Invalid capability
- Revoked capability
- Insufficient rights
- Capability transfer
- Capability derivation
- Unauthorized object access

### Memory tests

- Invalid address
- Out-of-bounds mapping
- Unauthorized mapping
- Cross-process access
- Writable/executable mappings

### IPC tests

- Unauthorized endpoint access
- Invalid message
- Invalid capability transfer
- Malformed userspace buffers
- Race conditions

### Scheduler tests

- Unauthorized priority changes
- Invalid thread handles
- CPU-affinity violations
- Concurrent thread operations

### Concurrency tests

- Double free
- Use-after-free
- Race conditions
- Lock-order problems
- Object lifetime races

---

## 45. Formal Verification Goals

Aegis Core should eventually use formal methods for the most security-critical components.

Potential properties:

```text
Capability authority cannot increase
Unauthorized memory access is impossible
Unauthorized IPC is impossible
Kernel object isolation is preserved
Invalid capabilities cannot authorize operations
Critical scheduler invariants hold
Critical memory invariants hold
```

Formal verification should begin with small, clearly specified components.

---

## 46. Security Invariants

The following invariants should guide implementation:

```text
No capability grants more authority than its rights.
No userspace thread can directly access kernel memory.
No process can access another process's memory without authority.
No invalid capability can authorize an operation.
No terminated object remains usable.
No blocked thread is scheduled.
No unauthorized device access is permitted.
No unchecked userspace pointer is dereferenced.
No security boundary depends only on a userspace identifier.
```

---

## 47. Security Review Requirements

Changes to security-critical components should receive additional review.

Security-critical areas include:

```text
kernel/src/capability/
kernel/src/memory/
kernel/src/ipc/
kernel/src/scheduler/
kernel/src/arch/
boot code
syscall interface
```

Changes should document:

```text
Threat model
Security assumptions
New authority
Changed invariants
Unsafe code
Testing performed
```

---

## 48. Recommended Security Development Order

```text
1. Define threat model
2. Define security invariants
3. Define kernel/user boundary
4. Implement capability primitives
5. Implement object isolation
6. Implement memory isolation
7. Implement secure IPC
8. Validate syscall arguments
9. Restrict unsafe code
10. Add interrupt/device isolation
11. Add SMP synchronization
12. Add DMA/IOMMU isolation
13. Add security testing
14. Add fuzzing
15. Formally specify critical invariants
16. Begin formal verification
```

---

## 49. Threat Model

The initial threat model should assume:

```text
Userspace applications are untrusted.
Userspace services may contain bugs.
IPC messages may be malicious.
Userspace pointers may be invalid.
Concurrent operations may race.
A compromised userspace service must not automatically compromise the kernel.
```

Hardware and firmware trust assumptions should be documented separately.

---

## 50. Design Rule

Aegis Core security is based on:

```text
Minimal kernel
      +
Memory isolation
      +
Capability-based authority
      +
Controlled IPC
      +
Least privilege
      +
Memory-safe implementation
      +
Explicit invariants
      +
Formal verification where practical
```

The kernel should enforce mechanisms that make unauthorized operations impossible or reject them safely.

The ultimate objective is not merely to make attacks difficult, but to construct strong, explicit security boundaries that can be tested, audited, and eventually formally verified.
