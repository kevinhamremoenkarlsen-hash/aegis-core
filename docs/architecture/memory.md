# Aegis Core — Memory Management

## 1. Purpose

Aegis Core uses capability-controlled memory management to provide strong isolation between the kernel, processes, threads, and userspace services.

The memory subsystem is designed around microkernel principles inspired by seL4:

- Minimal kernel memory management
- Explicit authority through capabilities
- Strong address-space isolation
- No implicit sharing
- Deterministic behavior where practical
- Userspace memory managers where possible
- Hardware-enforced page permissions
- Clear ownership and lifetime rules

The kernel should manage the mechanisms required for safe memory isolation while higher-level memory policies can remain in userspace.

---

## 2. Memory Architecture

```text
┌─────────────────────────────────────────┐
│              Userspace                  │
│                                         │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│ │ Process A│ │ Process B│ │ Service  │ │
│ └────┬─────┘ └────┬─────┘ └────┬─────┘ │
│      │             │             │       │
└──────┼─────────────┼─────────────┼───────┘
       │             │             │
       ▼             ▼             ▼
┌─────────────────────────────────────────┐
│              Aegis Core                 │
│                                         │
│ Address Spaces                          │
│ Page Tables                             │
│ Frame Capabilities                      │
│ Mapping / Unmapping                     │
│ Protection                              │
│ Memory Fault Handling                   │
└────────────────────┬────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────┐
│                 Hardware                │
│                                         │
│ Physical RAM                            │
│ MMU / Paging                            │
│ TLB                                     │
└─────────────────────────────────────────┘
```

---

## 3. Core Memory Objects

The initial memory model should contain explicit kernel objects for:

```text
Physical Frame
Page Table
Page Directory / Higher-Level Table
Address Space
Virtual Memory Region
Memory Mapping
```

The exact architecture-specific objects depend on x86_64 and later supported architectures.

---

## 4. Physical Memory

Physical memory is divided into frames.

For x86_64, the initial implementation can use a standard page size of:

```text
4 KiB
```

Future support may include:

```text
2 MiB huge pages
1 GiB huge pages
```

Large pages should only be introduced after the normal paging implementation is stable.

---

## 5. Frame Capabilities

A physical frame is represented by a capability.

Conceptually:

```text
Frame Capability
├── Physical address
├── Size
└── Rights
```

A process cannot use arbitrary physical memory merely by knowing its physical address.

Instead:

```text
Process
   │
   └── Frame Capability
             │
             ▼
        Physical Frame
```

This preserves explicit authority.

---

## 6. Frame Rights

Possible rights:

```text
Read
Write
Execute
Map
Grant
```

Example:

```text
Frame:
Read + Write

Mapped page:
Read only
```

A mapping must never grant more authority than the underlying capability permits.

---

## 7. Address Spaces

Each isolated process should have its own virtual address space.

```text
Process A
   │
   └── Address Space A
          │
          ├── Code
          ├── Data
          ├── Stack
          └── Heap

Process B
   │
   └── Address Space B
          │
          ├── Code
          ├── Data
          ├── Stack
          └── Heap
```

Process A must not access Process B's private mappings unless explicit shared-memory authority exists.

---

## 8. Virtual Memory

Virtual addresses are translated into physical addresses through architecture-specific page tables.

```text
Virtual Address
      │
      ▼
   Page Table
      │
      ▼
Physical Frame
```

The MMU performs the translation after the kernel establishes the mapping.

---

## 9. Page Mapping

A mapping connects a virtual page to an authorized physical frame.

```text
Virtual Page
      │
      │ Mapping
      ▼
Physical Frame
```

The mapping operation must validate:

- Frame capability
- Mapping rights
- Virtual address
- Page size
- Alignment
- Address-space capability
- Existing mapping
- Architecture constraints

---

## 10. Page Permissions

Memory pages should support hardware protection where available.

Typical permissions:

```text
R = Read
W = Write
X = Execute
```

Examples:

```text
Code:
R-X

Read-only data:
R--

Writable data:
RW-

Stack:
RW-
```

Writable executable memory should normally be avoided unless explicitly required.

---

## 11. W^X

Aegis Core should support a W^X-oriented memory policy:

```text
Writable OR Executable
```

rather than allowing arbitrary pages to be both writable and executable.

The policy should be enforced through the architecture's page permissions and userspace memory-management rules.

---

## 12. Kernel Memory

Kernel memory must be isolated from untrusted userspace.

Userspace must not be able to:

```text
Read kernel memory
Write kernel memory
Execute arbitrary kernel memory
Modify kernel page tables
```

Kernel mappings should use appropriate privilege-level protections.

---

## 13. Kernel Heap

The kernel may require a small internal allocator for kernel objects.

Potential uses:

```text
Thread objects
Endpoint objects
Capability structures
Scheduler structures
Page-table metadata
IPC structures
```

The kernel allocator must not become an uncontrolled general-purpose memory manager.

Where practical, fixed-size object allocators or typed allocators should be preferred for kernel objects.

---

## 14. Kernel Stack

Each kernel-executing thread requires safe stack handling.

Requirements:

- Separate stack per kernel thread where required
- Guard pages where practical
- Correct alignment
- Controlled size
- No userspace-controlled kernel stack pointers

The kernel must detect stack exhaustion as early as practical.

---

## 15. Userspace Memory Manager

Aegis Core should keep higher-level memory policy outside the kernel.

A future userspace memory manager can handle:

```text
Virtual memory allocation
Heap management
Memory reclamation
Process memory policies
Memory quotas
Shared-memory policies
```

The kernel provides the mechanisms required to safely create and modify mappings.

---

## 16. Untyped / Reclaimable Memory

A capability-oriented design may represent currently unallocated physical memory as a resource from which kernel objects or frames can be created.

Conceptually:

```text
Physical Memory
      │
      ▼
Memory Resource
      │
      ├── Frame
      ├── Page Table
      ├── Kernel Object
      └── Other Object
```

This resembles the resource-retyping model used by capability-oriented microkernels.

The exact Aegis Core object-retype model must be specified before implementation.

---

## 17. Memory Ownership

Every physical resource should have clear ownership or authority.

The system must avoid ambiguous ownership such as:

```text
Who owns this frame?
Who may map it?
Who may revoke it?
Who may reclaim it?
```

Capabilities should answer these questions.

---

## 18. Shared Memory

Memory sharing must be explicit.

```text
Process A
   │
   │ Frame Capability
   ▼
Shared Frame
   ▲
   │ Frame Capability
   │
Process B
```

No process should automatically gain access to another process's memory.

Shared memory can be used for:

- IPC buffers
- Network buffers
- Graphics
- File caches
- Device buffers
- Large data transfers

---

## 19. Copy-on-Write

Copy-on-write can be added later.

Concept:

```text
Process A ──┐
            ├── Shared Read-only Frame
Process B ──┘
```

When one process writes:

```text
Write fault
    │
    ▼
Allocate new frame
    │
    ▼
Copy data
    │
    ▼
Remap writable
```

This is useful for process creation and efficient memory sharing.

---

## 20. Memory Faults

Invalid memory accesses should generate architecture-specific faults.

Example:

```text
Userspace
   │
   │ Invalid access
   ▼
MMU
   │
   ▼
Page Fault
   │
   ▼
Aegis Core
   │
   ▼
Fault IPC
   │
   ▼
Userspace Fault Handler
```

The kernel should provide the mechanism for delivering faults without embedding application-specific recovery policies.

---

## 21. Page Fault Information

A fault message may contain information such as:

```text
Fault type
Fault address
Instruction address
Access type
Architecture-specific status
```

The kernel must avoid exposing unnecessary privileged information.

---

## 22. Fault Handling

A userspace memory manager may handle faults by:

```text
1. Receive fault
2. Determine required mapping
3. Obtain frame capability
4. Map frame
5. Resume faulting thread
```

Alternatively, an unrecoverable fault may terminate or isolate the affected process according to the system's process policy.

---

## 23. TLB

The Translation Lookaside Buffer caches virtual-to-physical translations.

When mappings change, the kernel must ensure that stale translations cannot violate isolation.

The implementation must correctly handle:

```text
Mapping
Unmapping
Permission changes
Address-space switches
Process destruction
SMP TLB invalidation
```

---

## 24. Address-Space Switching

When switching between processes:

```text
Process A
   │
   ▼
Scheduler
   │
   ▼
Address Space A
   │
   │ switch
   ▼
Address Space B
   │
   ▼
Process B
```

Architecture-specific mechanisms such as x86_64 CR3 and TLB behavior must be handled in the architecture layer.

---

## 25. Memory Regions

A process can conceptually contain regions such as:

```text
┌─────────────────────┐
│ Kernel / reserved   │
├─────────────────────┤
│ User code           │
├─────────────────────┤
│ Read-only data      │
├─────────────────────┤
│ Writable data       │
├─────────────────────┤
│ Heap                │
├─────────────────────┤
│ Shared memory       │
├─────────────────────┤
│ Memory-mapped data  │
├─────────────────────┤
│ Stack               │
└─────────────────────┘
```

The exact virtual-address layout should be defined in the architecture and process specifications.

---

## 26. Guard Pages

Guard pages can be placed around sensitive regions such as stacks.

```text
┌───────────────┐
│ Guard Page    │ ← inaccessible
├───────────────┤
│ Stack         │
├───────────────┤
│ Guard Page    │ ← inaccessible
└───────────────┘
```

This helps detect certain memory overflows.

---

## 27. DMA Memory

Devices may access physical memory directly through DMA.

DMA-capable memory must be treated as a security-sensitive resource.

Future support should consider:

```text
IOMMU
DMA isolation
Device memory capabilities
DMA buffer ownership
Device address spaces
```

A device must not automatically gain unrestricted access to system RAM.

---

## 28. IOMMU

On supported systems, an IOMMU can provide device-side memory isolation.

Conceptually:

```text
Device
   │
   ▼
IOMMU
   │
   ▼
Authorized DMA memory
```

This should eventually integrate with Aegis Core's capability model.

---

## 29. Memory Zeroing

When a physical frame changes security ownership, its previous contents must not become visible to the new owner.

Example:

```text
Process A
   │
   ▼
Frame released
   │
   ▼
Clear / securely reinitialize
   │
   ▼
Process B
```

This prevents information leakage between security domains.

---

## 30. Memory Reclamation

Memory can only be reclaimed when the kernel can prove that no valid authority or active mapping still requires it.

Potential sequence:

```text
Capability removed
      │
      ▼
Mappings removed
      │
      ▼
References released
      │
      ▼
Frame becomes reclaimable
```

Object lifetime and capability lifetime must be coordinated.

---

## 31. Memory Quotas

A future resource-management system may assign memory quotas to processes or services.

Example:

```text
Service A
Quota: 128 MiB

Service B
Quota: 512 MiB
```

Quota policy should preferably remain outside the minimal kernel where possible.

The kernel should enforce only the authority and resource constraints it must enforce for isolation.

---

## 32. Memory Security Rules

### Rule 1 — No arbitrary physical memory access

A physical address alone is not authority.

### Rule 2 — No unauthorized mapping

A process requires the appropriate frame and address-space capabilities.

### Rule 3 — Never trust userspace addresses

All userspace addresses must be validated.

### Rule 4 — Preserve page permissions

Mappings cannot exceed authorized rights.

### Rule 5 — Prevent kernel-memory exposure

Userspace cannot read or modify protected kernel memory.

### Rule 6 — Explicit shared memory

Memory sharing must be authorized.

### Rule 7 — Clear security-sensitive memory

Reclaimed memory must not leak previous contents.

### Rule 8 — Protect page-table authority

Userspace cannot arbitrarily modify page tables.

### Rule 9 — Protect DMA

Devices must not receive unrestricted memory access.

### Rule 10 — Preserve object lifetime

Memory objects cannot be reclaimed while valid references remain.

---

## 33. Rust Module Structure

Suggested kernel structure:

```text
kernel/src/
├── memory/
│   ├── mod.rs
│   ├── frame.rs
│   ├── page.rs
│   ├── address_space.rs
│   ├── mapping.rs
│   ├── page_table.rs
│   ├── allocator.rs
│   ├── fault.rs
│   ├── rights.rs
│   ├── region.rs
│   ├── reclaim.rs
│   └── tests.rs
│
└── arch/
    └── x86_64/
        ├── paging.rs
        ├── tlb.rs
        └── registers.rs
```

The architecture-independent memory subsystem should remain separate from x86_64-specific page-table code.

---

## 34. Syscalls

The memory subsystem may eventually expose minimal system calls such as:

```text
map
unmap
protect
create_address_space
create_frame
share
```

The exact syscall ABI must be specified together with the capability system.

A syscall must never allow a process to bypass capability checks.

---

## 35. Testing Strategy

### Frame tests

- Create frame
- Allocate frame
- Free frame
- Reclaim frame
- Verify ownership

### Mapping tests

- Map valid frame
- Unmap frame
- Invalid address
- Invalid alignment
- Unauthorized mapping
- Permission reduction
- Permission escalation attempt

### Isolation tests

- Process A cannot read Process B
- Process A cannot write Process B
- Userspace cannot access kernel memory
- Invalid capability cannot map memory

### Fault tests

- Unmapped read
- Unmapped write
- Execute violation
- Write to read-only page
- Fault delivery
- Fault recovery

### Reclamation tests

- Frame reuse
- Memory zeroing
- Reference counting
- Capability revocation
- Mapping destruction

### SMP tests

- Concurrent mapping
- Concurrent unmapping
- TLB invalidation
- Address-space switching
- Cross-core memory isolation

---

## 36. Verification Goals

The memory subsystem should eventually have formally specified invariants.

Important properties:

```text
No unauthorized physical access
No unauthorized virtual mapping
No capability escalation
No kernel-memory access from userspace
No stale mapping after required invalidation
No use-after-free
No memory ownership violation
No information leak through reclaimed frames
```

These properties should become part of the formal security model.

---

## 37. Implementation Order

Recommended implementation sequence:

```text
1. Define physical-address types
2. Define virtual-address types
3. Define page/frame types
4. Implement architecture-specific page tables
5. Implement basic address spaces
6. Implement frame capabilities
7. Implement mapping
8. Implement unmapping
9. Implement page permissions
10. Implement address-space switching
11. Implement TLB handling
12. Implement page faults
13. Connect faults to IPC
14. Implement memory reclamation
15. Implement memory zeroing
16. Add shared memory
17. Add copy-on-write
18. Add SMP support
19. Add IOMMU/DMA isolation
20. Define formal memory invariants
21. Begin verification
```

Advanced memory management should not be implemented before the basic isolation model is stable.

---

## 38. Relationship With Other Aegis Core Subsystems

Memory management is tightly connected to:

```text
Capabilities
     │
     ├── Frame authority
     ├── Address-space authority
     └── Mapping authority

IPC
     │
     ├── Shared memory
     └── Fault delivery

Scheduler
     │
     └── Address-space switching

Architecture
     │
     ├── Page tables
     ├── MMU
     └── TLB

Devices
     │
     └── DMA / IOMMU
```

The interfaces between these components should remain small and explicitly specified.

---

## 39. Design Rule

The central memory-management rule of Aegis Core is:

> Memory access is an explicit capability-controlled right, never an implicit privilege.

The kernel should provide the minimum mechanisms required to enforce isolation, while memory policies and higher-level memory management should remain outside the trusted computing base whenever practical.
