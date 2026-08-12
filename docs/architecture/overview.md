# Aegis Core — Architecture Overview

## Purpose

Aegis Core is the security-critical microkernel of Altis OS. It is a Rust-based `x86_64` microkernel designed around minimal privileged code, capability-based security, strong isolation, explicit IPC, and verification-oriented development.

Aegis Core is inspired by seL4's architectural principles, but it is an independent implementation and does **not** inherit seL4's formal verification guarantees.

## High-Level Architecture

```text
┌─────────────────────────────────────────────────────────┐
│                         ALTIS OS                        │
│                                                         │
│  Applications                                           │
│        │                                                │
│        ▼                                                │
│  Userspace Services                                     │
│  ├── Init                                              │
│  ├── Filesystem                                        │
│  ├── Network                                           │
│  ├── Device Manager                                    │
│  └── Drivers                                           │
│        │                                                │
│        │ IPC / Capabilities                             │
│        ▼                                                │
│  ┌───────────────────────────────────────────────────┐  │
│  │                  AEGIS CORE                       │  │
│  │                                                   │  │
│  │ Capabilities  IPC  Threads  Scheduler  Memory   │  │
│  │ Kernel Objects  Interrupts  Syscalls  CPU        │  │
│  └──────────────────────┬────────────────────────────┘  │
│                         │                               │
│                         ▼                               │
│                      Hardware                          │
└─────────────────────────────────────────────────────────┘
```

The kernel is the primary security boundary. Most operating-system functionality should remain outside the kernel.

## Core Responsibilities

Aegis Core should provide only mechanisms that require kernel privilege:

- CPU and architecture primitives
- Exceptions and interrupts
- Threads and context switching
- Scheduling
- Address spaces
- Memory primitives
- Kernel objects
- Capability management
- IPC
- System calls
- Security enforcement

High-level functionality such as filesystems, networking, GUI, audio, and most drivers should run in userspace whenever practical.

## Core Subsystems

```text
AEGIS CORE
├── arch/          CPU-specific implementation
├── boot/          Boot information and initialization
├── capability/    Capability spaces, rights, lookup, revoke
├── ipc/           Endpoints, messages, notifications, replies
├── task/          Threads, TCBs, contexts, scheduler
├── memory/        Frames, pages, address spaces, page tables
├── object/        Kernel object definitions and lifetimes
├── syscall/       Minimal userspace/kernel interface
├── interrupt/     IRQ handling and controlled delivery
├── security/      Isolation and security invariants
├── sync/          Kernel synchronization primitives
└── debug/         Serial/debug facilities
```

## Capability Architecture

Capabilities represent authority to operate on kernel objects.

```text
Process
   │
   ├── Capability → Endpoint
   ├── Capability → Frame
   ├── Capability → Thread
   └── Capability → IRQ
```

A process must not be able to manufacture authority that has not been explicitly granted.

Planned capability operations include:

```text
Create
Copy
Move
Mint
Lookup
Delete
Revoke
Derive
```

The exact semantics must be specified before implementation.

## Kernel Objects

Initial object types:

```text
TCB
CNode
Endpoint
Notification
Frame
VSpace
IRQ
Reply
Untyped / Memory Object
```

Every object requires defined:

- Lifetime
- Ownership
- Valid states
- Access rights
- Allowed operations
- Security invariants

## Memory Architecture

Each isolated process should have its own virtual address space.

```text
Physical RAM
     │
     ▼
Physical Frames
     │
     ▼
Page Tables
     │
     ├───────────────┐
     ▼               ▼
Address Space A   Address Space B
     │               │
Process A         Process B
```

Memory operations should include:

- Physical frame ownership
- Mapping
- Unmapping
- Permission control
- Address-space creation
- Address-space destruction
- Controlled memory sharing

High-level allocation policy can be implemented by userspace services.

## Threads and Scheduler

A thread is represented by a Thread Control Block (TCB).

```text
TCB
├── CPU context
├── Address-space reference
├── Scheduling state
├── Priority
└── IPC state
```

Initial states:

```text
Running
Ready
Blocked
Suspended
Inactive
Terminated
```

The scheduler should eventually support priorities, ready queues, blocking, unblocking, timer preemption, context switching, and eventually SMP.

## IPC

IPC is the primary communication mechanism between isolated components.

```text
Process A
    │
    │ Send
    ▼
 Endpoint
    │
    │ Receive
    ▼
Process B
```

Initial IPC primitives:

```text
Send
Receive
Reply
Notify
Block
Unblock
Capability transfer
```

The kernel provides the mechanism; userspace defines higher-level protocols.

## Interrupt Architecture

```text
Hardware
    │
    ▼
CPU Interrupt
    │
    ▼
Aegis Core
    │
    ▼
IRQ Object
    │
    ▼
Authorized Userspace Component
```

The kernel controls which component is authorized to receive an interrupt. Drivers should run in userspace whenever the platform permits it.

## System Calls

The syscall surface should remain small.

```text
Syscalls
├── Capability
├── IPC
├── Thread
├── Memory
├── Interrupt
└── Debug
```

Every syscall must validate both inputs and authority.

```text
Userspace
   │
   ▼
Syscall Entry
   │
   ▼
Dispatcher
   │
   ▼
Capability Validation
   │
   ▼
Kernel Operation
   │
   ▼
Return
```

High-level operations such as opening files or making network requests should normally be userspace operations.

## Architecture Layer

The current architecture target is:

```text
x86_64
```

Architecture-specific code should be isolated from generic kernel mechanisms.

It handles:

- CPU setup
- Registers
- GDT/IDT
- Exceptions
- Interrupt entry
- Paging primitives
- Context switching
- CPU-specific instructions

This separation leaves room for future architectures.

## Boot Flow

```text
Firmware
   │
   ▼
Bootloader
   │
   ▼
Kernel Image
   │
   ▼
Rust Kernel Entry
   │
   ▼
Architecture Initialization
   │
   ▼
Kernel Initialization
   │
   ▼
Initial Userspace
   │
   ▼
Scheduler
```

The current project generates a BIOS image and successfully boots in QEMU.

## Planned Initialization Order

```text
1. Enter kernel
2. Validate boot information
3. Initialize architecture
4. Initialize CPU exception handling
5. Initialize interrupts
6. Initialize physical memory
7. Initialize kernel objects
8. Initialize capabilities
9. Initialize address spaces
10. Initialize scheduler
11. Initialize IPC
12. Initialize syscalls
13. Create initial userspace
14. Start initial userspace
15. Enter scheduler
```

The exact order may change during implementation.

## Security Model

The central security rule is:

> Authority must be explicit.

Important properties:

### Least privilege

Components receive only the authority they require.

### Memory isolation

A process cannot access memory outside its authorized mappings.

### Capability integrity

A process cannot fabricate valid authority for an object it has not been granted.

### IPC authority

Communication requires the necessary capability.

### Fault isolation

A userspace failure should not automatically compromise unrelated components.

### Small trusted computing base

Privileged code should remain as small and auditable as practical.

## Verification-Oriented Design

The architecture should make future formal verification practical.

Important invariants include:

- Memory isolation
- Capability integrity
- IPC authorization
- Valid thread scheduling
- Kernel object lifetime
- Preservation of kernel state invariants

These are design goals until they are converted into precise specifications and actually verified.

## Userspace Architecture

A future minimal userspace environment may look like:

```text
                    Aegis Core
                         │
                         ▼
                       Init
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       Memory         Process         Device
       Service        Service         Manager
          │              │              │
          └──────────────┼──────────────┘
                         │
                         ▼
                    Applications
```

Services communicate using IPC and capabilities rather than unrestricted shared kernel state.

## Dependency Direction

The preferred dependency direction is:

```text
Application
     ↓
Userspace Service
     ↓
IPC / Syscall
     ↓
Aegis Core
     ↓
Architecture Layer
     ↓
Hardware
```

Higher layers should not bypass lower-layer security mechanisms.

## Development Strategy

Implementation should proceed in small milestones:

```text
Boot
  ↓
CPU initialization
  ↓
Exceptions
  ↓
Memory
  ↓
Kernel objects
  ↓
Capabilities
  ↓
Threads
  ↓
Scheduler
  ↓
IPC
  ↓
System calls
  ↓
Isolation
  ↓
Initial userspace
  ↓
Userspace services
```

Each stage should be tested before substantial complexity is added.

## Current State

```text
Kernel language       Rust
Kernel type           Microkernel
Architecture          x86_64
Target                x86_64-unknown-none
Boot mode             BIOS
Bootloader             bootloader 0.11.x
no_std                Yes
QEMU boot             Working
Serial output         Working
Kernel entry          Working
```

Current milestone:

**M0 — Boot complete**

Next milestone:

**M1 — CPU Initialization**

## Relationship to seL4

Aegis Core is architecturally inspired by seL4, especially:

- Capability-based security
- Microkernel design
- Explicit IPC
- Strong isolation
- Small trusted computing base
- Verification-oriented development

However:

```text
Aegis Core ≠ seL4
```

Aegis Core is independently implemented. No seL4 formal-security guarantee should be attributed to Aegis Core unless it has independently been demonstrated.

## Architectural Rule

> Keep the kernel small, keep authority explicit, and move everything that does not require kernel privilege into isolated userspace.

This rule should guide future architecture and implementation decisions.
