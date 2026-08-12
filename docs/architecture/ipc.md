Aegis Core — Capability System

1. Purpose

The capability system is the primary authority and access-control mechanism of Aegis Core.

A capability represents explicit authority to operate on a specific kernel object.

The fundamental security principle is:

Possession of a valid capability is required to exercise the corresponding authority.

Aegis Core follows a capability-oriented architecture inspired by seL4. The implementation is independent and must establish its own security properties.

2. Security Goals

The capability system must provide:

Explicit authority

Least privilege

Strong isolation

No ambient authority

Controlled authority transfer

Controlled authority duplication

Capability revocation

Object lifetime safety

Separation between object identity and authority

Deterministic permission checking

A userspace process must not gain kernel-object authority merely by knowing an object identifier or memory address.

3. Basic Model

                 Capability
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       Object       Rights    Metadata
       reference              / state
          │
          ▼
    Kernel Object

A capability should conceptually contain or reference:

Capability
├── Object reference
├── Object type
├── Rights
├── Derivation information
└── Capability state

The exact internal representation is an implementation detail and must not expose privileged kernel information to userspace.

4. Authority Model

Authority flows explicitly.

Root / Initial Authority
          │
          ▼
      Capability
          │
       ┌──┴──┐
       ▼     ▼
     Copy   Mint
       │     │
       ▼     ▼
     Child  Restricted Child

Authority must never appear spontaneously.

The kernel is responsible for ensuring that every capability operation is authorized by an existing capability.

5. Capability Rights

Rights describe what operations a capability permits.

Example generic rights:

Read
Write
Execute
Map
Grant
Send
Receive
Reply
Control
Manage

Not every object type supports every right.

For example:

Frame
├── Read
├── Write
├── Execute
└── Map

An endpoint might instead support:

Endpoint
├── Send
├── Receive
└── Grant

The kernel must reject rights that are invalid for a particular object type.

6. Rights Reduction

A capability may be derived with fewer rights than its parent.

Example:

Original:
Read + Write + Execute

        │
        ▼

Derived:
Read + Write

A derived capability must never gain a right that was absent from the source capability.

Formally:

rights(child) ⊆ rights(parent)

unless an explicitly privileged kernel operation defines another safe rule.

7. Capability Spaces

Each protected execution environment should have an isolated capability space.

A capability space stores references to capabilities available to that execution environment.

Conceptually:

Process
   │
   ▼
CNode / Capability Space
   │
   ├── Slot 0 → Endpoint capability
   ├── Slot 1 → Frame capability
   ├── Slot 2 → Thread capability
   └── Slot 3 → VSpace capability

A process should normally interact with capabilities through handles or capability slots rather than raw kernel object addresses.

8. Capability Slots

A slot stores one capability.

CNode
├── Slot 0
├── Slot 1
├── Slot 2
├── Slot 3
└── ...

A slot should have a well-defined state:

Empty
Occupied

The kernel must validate the slot before performing an operation.

An invalid slot must produce a controlled failure rather than undefined behavior.

9. CNodes

A CNode is a kernel object used to organize capability slots.

Conceptually:

CNode
│
├── Slot 0
├── Slot 1
├── Slot 2
│
└── CNode
     ├── Slot 0
     └── Slot 1

Hierarchical CNodes can provide scalable capability addressing.

Aegis Core should define:

Slot indexing

CNode size

CNode creation

CNode destruction

Nested CNodes

Lookup rules

Authority required for CNode operations

10. Capability Lookup

A capability lookup should be deterministic.

Conceptual flow:

Userspace capability handle
            │
            ▼
       Capability lookup
            │
            ▼
       Validate slot
            │
            ▼
       Validate object
            │
            ▼
       Validate rights
            │
            ▼
       Perform operation

The kernel must never trust a userspace-provided pointer as proof of authority.

11. Capability Creation

Capabilities should normally originate from existing authority.

Conceptual operation:

Source capability
       │
       ▼
   Validate source
       │
       ▼
   Create/derive authority
       │
       ▼
Destination slot

The kernel must verify:

The source capability exists.

The source capability is valid.

The caller has the required authority.

The destination slot is valid.

The resulting rights are permitted.

The operation preserves capability invariants.

12. Copy

Copying creates another capability referring to the same underlying object.

Capability A
     │
     │ copy
     ▼
Capability B
     │
     └──────► Same object

The copy should preserve only rights that the operation is allowed to preserve.

Example:

A: Read + Write

copy(A) → B: Read + Write

Copying does not duplicate the underlying kernel object.

13. Mint

Minting creates a derived capability with explicitly restricted authority.

Parent Capability
       │
       ▼
      Mint
       │
       ▼
Restricted Capability

Example:

Parent:
Read + Write + Execute

Mint:
Read + Write

The child must not exceed the authority available from the parent and the minting operation.

14. Move

Moving transfers a capability from one slot to another.

Slot A
  │
  │ move
  ▼
Slot B

After a successful move:

Slot A → Empty
Slot B → Capability

The operation must be atomic from the perspective of concurrent kernel execution.

15. Delete

Deleting removes a capability from a capability slot.

Before:

Slot 4 → Capability

Delete

After:

Slot 4 → Empty

Deleting a capability does not necessarily destroy the underlying object.

Other capabilities may still reference that object.

16. Revocation

Revocation removes authority derived from a particular capability.

This requires capability derivation information or another formally defined mechanism.

Conceptually:

                 Root Capability
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
            Child A  Child B  Child C
              │
              ▼
           Child A2

Revoking the appropriate authority should invalidate the intended descendants according to the capability derivation rules.

The exact revocation algorithm must be specified and tested carefully.

17. Capability Derivation Tree

Aegis Core should maintain a derivation relationship between capabilities when required for safe revocation.

Example:

Root
├── A
│   ├── A1
│   └── A2
└── B
    ├── B1
    └── B2

This allows the kernel to reason about where authority originated.

The derivation structure must not allow a child capability to become more powerful than its ancestors.

18. Object Lifetime

Capabilities and kernel objects have different lifetimes.

Example:

Capability A ─┐
Capability B ─┼──► Object
Capability C ─┘

Deleting Capability A must not destroy the object while B or C still holds valid authority.

The object may become reclaimable when no valid references remain and the kernel determines that destruction is safe.

Object lifetime rules must prevent:

Use-after-free

Dangling capabilities

Stale references

Double destruction

Unauthorized resurrection

19. Capability and Object Identity

Userspace must not be able to turn an arbitrary object identifier into authority.

Incorrect model:

Object ID
   │
   ▼
Access object

Correct model:

Capability
   │
   ▼
Validated authority
   │
   ▼
Access object

Object identifiers may exist internally, but knowledge of an identifier alone must not grant access.

20. IPC Capabilities

IPC endpoints should be protected by capabilities.

Process A
   │
   └── Send capability
           │
           ▼
        Endpoint
           ▲
           │
   ┌───────┘
   │
Process B
   └── Receive capability

The kernel checks the appropriate capability before allowing communication.

Capabilities may also be transferred through IPC where the protocol and rights permit it.

21. Capability Transfer

Capability transfer should be explicit.

Process A
   │
   │ IPC + capability
   ▼
Kernel
   │
   ▼
Process B

The kernel must validate:

Sender authority

Destination

Capability validity

Transfer rights

Destination slot

Rights being transferred

A sender must not be able to transfer authority it does not possess.

22. Memory Capabilities

Physical memory should be represented by capabilities.

Conceptually:

Memory Authority
      │
      ▼
    Frame
      │
      ├── Read
      ├── Write
      ├── Execute
      └── Map

A process can map memory only when it possesses the necessary authority.

This enables explicit memory sharing:

Process A
   │
   │ Frame capability
   ▼
Shared Frame
   ▲
   │ Frame capability
   │
Process B

23. Thread Capabilities

Threads should also be represented by capabilities.

Possible rights:

Thread
├── Inspect
├── Control
├── Suspend
├── Resume
└── Configure

A process should not be able to control another thread without the corresponding authority.

24. Address-Space Capabilities

An address space can be represented by a capability.

Possible authority:

VSpace
├── Read metadata
├── Map
├── Unmap
├── Configure
└── Destroy

The exact rights must be constrained to prevent privilege escalation.

25. IRQ Capabilities

Interrupt authority should also be explicit.

IRQ capability
      │
      ▼
IRQ object
      │
      ▼
Authorized handler

A userspace driver must not receive arbitrary hardware interrupts without the corresponding capability.

26. Capability Errors

Capability operations should return controlled error results.

Possible errors:

InvalidCapability
InvalidSlot
InvalidObject
InsufficientRights
InvalidOperation
SlotOccupied
SlotEmpty
Revoked
ObjectDestroyed
InvalidTransfer

Errors must never expose privileged memory or kernel implementation details unnecessarily.

27. Concurrency

Capability operations may occur concurrently on multi-core systems.

The implementation must preserve:

Slot consistency

Object lifetime safety

Derivation-tree integrity

Revocation correctness

Atomic transfer semantics

Capability-right invariants

Synchronization mechanisms must themselves be designed to avoid deadlocks and races.

28. Security Invariants

The following invariants are fundamental.

Invariant 1 — No fabricated authority

A process cannot create arbitrary valid capabilities without authorized kernel operations.

Invariant 2 — Rights cannot increase

A derived capability cannot have more authority than permitted by its source.

child_rights ⊆ permitted(parent_rights)

Invariant 3 — Isolation

A capability belonging to one protection domain must not automatically become available to another.

Invariant 4 — Valid object reference

Every live capability must reference a valid kernel object.

Invariant 5 — Safe destruction

Destroying an object must not leave usable capabilities referencing reclaimed memory.

Invariant 6 — Revocation correctness

Revocation must remove exactly the authority defined by the revocation semantics.

Invariant 7 — Explicit transfer

Authority must cross protection boundaries only through an authorized mechanism.

29. Threat Model

The capability system must assume that userspace code may be malicious or compromised.

Userspace may attempt:

Forged handles
Invalid object IDs
Invalid pointers
Unauthorized capability operations
Rights escalation
Capability reuse
Race conditions
Double deletion
Invalid IPC transfers

The kernel must treat all userspace input as untrusted.

30. Rust Safety Requirements

The capability implementation should use Rust's type and ownership system wherever practical.

Security-critical unsafe code should be minimized.

Every unsafe block must have:

A documented safety invariant

A clear reason for requiring unsafe

Validity assumptions

Tests covering important boundary cases

The capability implementation must not rely on undefined behavior for security.

31. Suggested Module Structure

kernel/src/
└── capability/
    ├── mod.rs
    ├── cap.rs
    ├── rights.rs
    ├── cnode.rs
    ├── slot.rs
    ├── lookup.rs
    ├── derive.rs
    ├── revoke.rs
    ├── transfer.rs
    └── error.rs

Possible future modules:

    ├── tree.rs
    ├── object.rs
    ├── authority.rs
    └── tests.rs

The exact structure may change as the implementation develops.

32. Testing Strategy

Capability testing should include:

Basic tests

Create capability

Copy capability

Move capability

Delete capability

Lookup capability

Rights tests

Valid operation

Missing right

Reduced rights

Invalid right/object combinations

Revocation tests

Revoke parent

Revoke descendants

Preserve unrelated branches

Verify invalidated capabilities cannot be used

Lifetime tests

Destroy object with no capabilities

Attempt access through deleted capability

Multiple references

Object reclamation

Security tests

Forged handles

Invalid slots

Invalid object types

Rights escalation attempts

Unauthorized transfer

Concurrent operations

33. Formal Specification Goals

Before claiming formal security properties, Aegis Core should define precise specifications for:

Capability state
Capability derivation
Rights
Object lifetime
CNode behavior
Lookup
Copy
Move
Mint
Delete
Revoke
Transfer

These specifications can later become the basis for formal verification.

34. Implementation Order

The capability subsystem should be implemented in stages:

1. Define object identifiers
2. Define capability rights
3. Define capability representation
4. Implement capability slots
5. Implement CNode
6. Implement lookup
7. Implement copy
8. Implement move
9. Implement delete
10. Implement restricted derivation/mint
11. Define derivation tree
12. Implement revocation
13. Implement capability transfer
14. Integrate with IPC
15. Integrate with memory
16. Integrate with threads
17. Add concurrency support
18. Add extensive security tests
19. Write formal invariants
20. Begin verification work

No advanced revocation or SMP mechanism should be considered complete until its invariants are clearly defined and tested.

35. Design Rule

The central rule of the Aegis Core capability system is:

No authority without an explicit capability.

Capabilities should be the foundation connecting:

Objects
   │
   ▼
Authority
   │
   ▼
Rights
   │
   ▼
Operations

This provides the security foundation required for strong isolation throughout Altis OS.
