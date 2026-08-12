# Aegis Core

> The security-critical microkernel core of Altis OS.

Aegis Core is a Rust-based, `x86_64` microkernel designed for **Altis OS**. Its architecture is inspired by the principles behind **seL4**: a minimal privileged kernel, capability-based access control, strong isolation, explicit IPC, and a design suitable for rigorous security analysis and future formal verification.

Aegis Core is **not seL4** and does not inherit seL4's formal verification guarantees. It is an independent kernel that uses similar architectural principles.

---

## Table of Contents

- [Goals](#goals)
- [Design Principles](#design-principles)
- [Architecture](#architecture)
- [Kernel Responsibilities](#kernel-responsibilities)
- [What Does Not Belong in the Kernel](#what-does-not-belong-in-the-kernel)
- [Project Structure](#project-structure)
- [Kernel Components](#kernel-components)
- [Capability System](#capability-system)
- [IPC](#ipc)
- [Memory Management](#memory-management)
- [Kernel Objects](#kernel-objects)
- [Threads and Scheduling](#threads-and-scheduling)
- [Interrupts](#interrupts)
- [System Calls](#system-calls)
- [Security Model](#security-model)
- [Verification](#verification)
- [Development Milestones](#development-milestones)
- [Building](#building)
- [Testing in QEMU](#testing-in-qemu)
- [Development Rules](#development-rules)
- [Current Status](#current-status)
- [Future Work](#future-work)
- [License](#license)

---

# Goals

Aegis Core is intended to provide the smallest practical privileged foundation for Altis OS.

The primary goals are:

- Strong process isolation
- Capability-based security
- Minimal kernel attack surface
- Explicit and controlled IPC
- Memory isolation
- Hardware abstraction where appropriate
- Deterministic and understandable kernel behavior
- Minimal privileged code
- Rust memory safety wherever possible
- Architecture suitable for formal verification
- Clear separation between kernel and userspace
- Security-first development
- Auditable source code

---

# Design Principles

### 1. Minimal kernel

Only functionality that requires kernel privilege should be implemented in the kernel.

```text
Kernel
├── Isolation
├── Capabilities
├── IPC
├── Threads
├── Scheduling
├── Memory primitives
├── Interrupt primitives
└── CPU/architecture primitives
```

Complex services should run outside the kernel.

### 2. Capability-based security

Access to kernel resources is controlled through capabilities.

```text
Process
   │
   ▼
Capability
   │
   ▼
Kernel Object
```

A process should only be able to operate on resources for which it possesses the appropriate capability.

### 3. Least privilege

Every component should receive only the capabilities it actually needs.

### 4. Isolation

Failure of one userspace component should not automatically compromise other components.

### 5. Explicit communication

Components should communicate through controlled IPC mechanisms rather than directly accessing each other's memory.

### 6. Verification-oriented design

Important kernel invariants should be documented from the beginning so that future formal verification is practical.

---

# Architecture

Aegis Core follows a microkernel architecture.

```text
┌──────────────────────────────────────────┐
│              Altis OS                    │
│                                          │
│  Applications                            │
│  ──────────────────────────────────────  │
│  Userspace Services                      │
│  ├── Filesystem                          │
│  ├── Network                             │
│  ├── Drivers                             │
│  ├── Device Manager                      │
│  ├── Process Services                    │
│  └── Other System Services               │
│                                          │
│  ──────────────────────────────────────  │
│              Aegis Core                  │
│                                          │
│  Capabilities                            │
│  IPC                                     │
│  Threads / Scheduler                     │
│  Address Spaces                           │
│  Memory Primitives                        │
│  Interrupts                               │
│  Kernel Objects                           │
│  CPU Architecture                         │
│                                          │
│  ──────────────────────────────────────  │
│              Hardware                    │
└──────────────────────────────────────────┘
```

The kernel should remain small while system functionality is implemented as isolated userspace services.

---

# Kernel Responsibilities

Aegis Core is responsible for:

- CPU initialization
- Exception handling
- Interrupt handling
- Thread management
- Context switching
- Scheduling
- Address-space management
- Memory mapping primitives
- Capability management
- Kernel objects
- IPC
- System calls
- Hardware resource isolation
- Security-critical enforcement

---

# What Does Not Belong in the Kernel

The following should normally be implemented outside Aegis Core:

```text
Filesystem
Network stack
TCP/IP
HTTP
GUI
Window manager
USB stack
Audio stack
GPU services
Bluetooth
Wi-Fi services
Authentication services
Shell
Package manager
Applications
```

Drivers should also be outside the kernel whenever the hardware and architecture allow it.

The goal is to keep the trusted computing base as small as practical.

---

# Project Structure

```text
Aegis-core/
│
├── .cargo/
│   └── config.toml
│
├── docs/
│   ├── architecture/
│   │   ├── overview.md
│   │   ├── capabilities.md
│   │   ├── ipc.md
│   │   ├── memory.md
│   │   ├── scheduling.md
│   │   └── security.md
│   │
│   ├── verification/
│   │   ├── verification-plan.md
│   │   ├── invariants.md
│   │   └── threat-model.md
│   │
│   └── milestones.md
│
├── kernel/
│   ├── Cargo.toml
│   │
│   └── src/
│       ├── main.rs
│       ├── arch/
│       │   ├── mod.rs
│       │   ├── cpu.rs
│       │   ├── gdt.rs
│       │   ├── idt.rs
│       │   ├── interrupts.rs
│       │   ├── timer.rs
│       │   └── x86_64/
│       │       ├── mod.rs
│       │       ├── cpu.rs
│       │       ├── paging.rs
│       │       └── registers.rs
│       ├── boot/
│       │   ├── mod.rs
│       │   └── boot_info.rs
│       ├── capability/
│       │   ├── mod.rs
│       │   ├── capability.rs
│       │   ├── cspace.rs
│       │   ├── rights.rs
│       │   ├── lookup.rs
│       │   └── revoke.rs
│       ├── ipc/
│       │   ├── mod.rs
│       │   ├── endpoint.rs
│       │   ├── message.rs
│       │   ├── notification.rs
│       │   └── reply.rs
│       ├── task/
│       │   ├── mod.rs
│       │   ├── thread.rs
│       │   ├── tcb.rs
│       │   ├── context.rs
│       │   ├── scheduler.rs
│       │   └── priority.rs
│       ├── memory/
│       │   ├── mod.rs
│       │   ├── boot_allocator.rs
│       │   ├── frame.rs
│       │   ├── page.rs
│       │   ├── address_space.rs
│       │   ├── page_table.rs
│       │   ├── allocator.rs
│       │   └── region.rs
│       ├── object/
│       │   ├── mod.rs
│       │   ├── object.rs
│       │   ├── endpoint.rs
│       │   ├── notification.rs
│       │   ├── thread.rs
│       │   ├── cnode.rs
│       │   ├── frame.rs
│       │   └── vspace.rs
│       ├── syscall/
│       │   ├── mod.rs
│       │   ├── dispatcher.rs
│       │   ├── capability.rs
│       │   ├── ipc.rs
│       │   ├── thread.rs
│       │   ├── memory.rs
│       │   └── interrupt.rs
│       ├── interrupt/
│       │   ├── mod.rs
│       │   ├── controller.rs
│       │   ├── irq.rs
│       │   └── handler.rs
│       ├── security/
│       │   ├── mod.rs
│       │   ├── isolation.rs
│       │   ├── permissions.rs
│       │   ├── invariants.rs
│       │   └── audit.rs
│       ├── sync/
│       │   ├── mod.rs
│       │   ├── spinlock.rs
│       │   ├── queue.rs
│       │   └── atomic.rs
│       ├── panic/
│       │   ├── mod.rs
│       │   └── handler.rs
│       └── debug/
│           ├── mod.rs
│           ├── serial.rs
│           └── logging.rs
│
├── src/
│   └── main.rs
├── build.rs
├── Cargo.toml
├── Cargo.lock
├── README.md
├── LICENSE
└── rust-toolchain.toml
```

Empty modules should not be created merely to match the planned structure. Add files as the implementation reaches each milestone.

---

# Kernel Components

## Architecture

`arch/` contains CPU-specific functionality:

- CPU initialization
- GDT
- IDT
- Exceptions
- Interrupt entry
- CPU registers
- Paging primitives
- Context switching
- Architecture-specific instructions

The architecture layer should be isolated so additional CPU architectures can eventually be supported.

## Boot

`boot/` handles information supplied during kernel startup and keeps bootloader-specific details isolated from the rest of the kernel.

## Capabilities

`capability/` implements:

- Capability representation
- Capability lookup
- Capability rights
- Capability spaces
- Capability creation
- Capability copying
- Capability movement
- Capability deletion
- Capability revocation
- Capability validation

## IPC

`ipc/` provides:

- Endpoints
- Notifications
- Messages
- Replies
- Blocking and unblocking
- Capability transfer

## Tasks

`task/` manages:

- Threads
- TCBs
- CPU contexts
- Thread states
- Priorities
- Scheduler
- Context switching

---

# Memory Management

The kernel should provide primitives for:

- Physical memory ownership
- Frames
- Pages
- Page tables
- Address spaces
- Mapping
- Unmapping
- Memory permissions
- Memory isolation

Conceptually:

```text
Physical Memory
       │
       ▼
Memory Capability
       │
       ├── Frame
       └── Mapping
              │
              ▼
          Address Space
```

A high-level memory-management service can run in userspace.

---

# Kernel Objects

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

Every kernel object must have clearly defined:

- Lifetime
- Ownership
- Access rights
- Valid states
- Allowed operations
- Security invariants

---

# Capability System

Capabilities are central to the Aegis security architecture.

A capability represents authority to operate on a kernel object.

Conceptually:

```text
Capability
├── Object reference
├── Object type
├── Rights
└── Validity information
```

Possible rights include:

```text
READ
WRITE
EXECUTE
GRANT
CONTROL
REVOKE
```

The exact rights model will be defined during implementation.

A process must not be able to manufacture a valid capability for an object to which it has not been granted authority.

---

# IPC

IPC is a primary mechanism for communication between isolated components.

The kernel should provide:

```text
Send
Receive
Reply
Notify
Block
Unblock
Transfer capability
```

Higher-level protocols should be implemented in userspace.

```text
Application
     │
     │ IPC
     ▼
File Service
     │
     │ IPC
     ▼
Storage Driver
```

The kernel enforces authority and communication mechanisms rather than understanding high-level filesystem or network protocols.

---

# Threads and Scheduling

The initial scheduler should provide:

- Ready queues
- Thread states
- Priorities
- Context switching
- Timer-driven preemption
- Blocking
- Unblocking

Initial thread states:

```text
Running
Ready
Blocked
Suspended
Inactive
Terminated
```

---

# Interrupts

The interrupt subsystem should provide:

- CPU exceptions
- Hardware IRQ handling
- Interrupt routing
- IRQ capabilities
- Timer interrupts
- Controlled delivery to authorized components

Where possible, hardware interrupts should be exposed to userspace through controlled capabilities instead of requiring drivers to execute in kernel mode.

---

# System Calls

The syscall interface should remain intentionally small.

Potential categories:

```text
Capability
IPC
Thread
Memory
Interrupt
Debug
```

High-level operating-system functionality should normally be implemented through userspace services.

For example, these should normally not be kernel syscalls:

```text
open_file()
create_socket()
draw_window()
send_http_request()
load_application()
```

---

# Security Model

Aegis Core is designed around:

### Least privilege

Components receive only the authority they require.

### Isolation

Separate address spaces prevent unauthorized direct memory access.

### Capability enforcement

Authority is explicitly represented and enforced through capabilities.

### Small trusted computing base

Security-critical privileged code should be minimized.

### No implicit trust

A userspace service should not automatically receive access to every system resource.

### Controlled IPC

Communication occurs through explicitly authorized channels.

### Fault isolation

A failure in a userspace component should be contained whenever possible.

---

# Security Invariants

Initial invariants include:

### Capability invariant

A process cannot obtain authority that has not been legitimately granted to it.

### Memory isolation invariant

A process cannot access memory outside the mappings and permissions granted to it.

### IPC invariant

A thread cannot communicate through an endpoint unless it possesses the necessary authority.

### Object lifetime invariant

A kernel object cannot be accessed after its authority and lifetime have ended.

### Scheduler invariant

Only valid runnable threads may be scheduled.

These are conceptual starting points and must be refined into precise, machine-checkable properties.

---

# Verification

A major long-term goal is rigorous formal verification.

The project should prioritize:

- Small components
- Explicit state
- Clear invariants
- Deterministic behavior where practical
- Minimal hidden state
- Restricted `unsafe` Rust
- Documented assumptions
- Separation of mechanisms and policies

Formal verification is a future goal and must not be claimed until actually completed.

Aegis Core must not claim seL4-level formal guarantees merely because its architecture is inspired by seL4.

---

# Development Milestones

## M0 — Boot

- [x] Rust kernel
- [x] `no_std`
- [x] `no_main`
- [x] x86_64 target
- [x] Bootloader integration
- [x] BIOS image generation
- [x] Serial output
- [x] Successful QEMU boot

## M1 — CPU Initialization

- [ ] GDT
- [ ] IDT
- [ ] CPU exceptions
- [ ] Basic interrupt foundation
- [ ] Improved panic handling

## M2 — Memory Foundation

- [ ] Boot memory map
- [ ] Physical frame management
- [ ] Page tables
- [ ] Virtual address spaces
- [ ] Memory permissions

## M3 — Kernel Objects

- [ ] TCB
- [ ] CNode
- [ ] Endpoint
- [ ] Notification
- [ ] Frame
- [ ] VSpace
- [ ] IRQ
- [ ] Reply
- [ ] Memory objects

## M4 — Capability System

- [ ] Capability handles
- [ ] Capability rights
- [ ] CSpaces
- [ ] Capability lookup
- [ ] Copy
- [ ] Move
- [ ] Mint
- [ ] Delete
- [ ] Revoke

## M5 — IPC

- [ ] Endpoints
- [ ] Send
- [ ] Receive
- [ ] Reply
- [ ] Notifications
- [ ] Blocking
- [ ] Unblocking
- [ ] Capability transfer

## M6 — Threads

- [ ] TCB implementation
- [ ] CPU context
- [ ] Context switching
- [ ] Thread states
- [ ] Priorities
- [ ] Scheduler
- [ ] Timer-driven scheduling

## M7 — Interrupts

- [ ] IRQ objects
- [ ] Interrupt controller
- [ ] IRQ routing
- [ ] Timer
- [ ] Userspace interrupt delivery

## M8 — System Calls

- [ ] Capability syscalls
- [ ] IPC syscalls
- [ ] Thread syscalls
- [ ] Memory syscalls
- [ ] Interrupt syscalls
- [ ] Syscall validation

## M9 — Isolation

- [ ] Kernel/user separation
- [ ] Address-space isolation
- [ ] Capability isolation
- [ ] Fault isolation
- [ ] Security invariant testing

## M10 — Minimal Userspace

- [ ] Initial userspace task
- [ ] Init
- [ ] Memory service
- [ ] Process service
- [ ] IPC services
- [ ] Device services

## M11 — Verification

- [ ] Formalized invariants
- [ ] Security model
- [ ] Verification plan
- [ ] Machine-checkable properties
- [ ] Formal verification of critical components

---

# Building

Target:

```text
x86_64-unknown-none
```

Build:

```powershell
cargo build
```

The BIOS image is generated under:

```text
target/debug/build/altis-os/*/out/altis-os-bios.img
```

The exact intermediate directory is generated by Cargo and may change between builds.

---

# Testing in QEMU

Run the generated BIOS image with:

```powershell
qemu-system-x86_64 -drive format=raw,file="PATH\TOltis-os-bios.img"
```

The current kernel should reach the Rust kernel entry point, initialize serial output, print the initial boot message, and halt.

---

# Development Rules

1. Keep the kernel small.
2. Prefer safe Rust.
3. Minimize `unsafe`.
4. Document every security-critical `unsafe` operation.
5. Do not bypass the capability model.
6. Keep mechanisms separate from policy.
7. Test every milestone.
8. Do not claim formal verification prematurely.
9. Do not add userspace functionality to the kernel merely for convenience.
10. Every new privileged feature must have a documented security rationale.

---

# Current Status

| Property | Status |
|---|---|
| Project | Altis OS |
| Kernel | Aegis Core |
| Architecture | x86_64 |
| Language | Rust |
| Kernel model | Microkernel |
| Security model | Capability-based |
| Boot mode | BIOS |
| Bootloader | bootloader 0.11.x |
| Target | `x86_64-unknown-none` |
| QEMU testing | Working |
| Current milestone | **M0 — Boot complete** |

The kernel currently:

1. Boots through the bootloader.
2. Enters the Rust kernel entry point.
3. Initializes serial output.
4. Prints the initial kernel boot message.
5. Halts safely.

Next milestone:

**M1 — CPU Initialization**

---

# Long-Term Architecture

```text
                    ALTIS OS
                       │
        ┌──────────────┴──────────────┐
        │                             │
   Applications                System Services
        │                             │
        └──────────────┬──────────────┘
                       │
                      IPC
                       │
              ┌────────▼────────┐
              │   AEGIS CORE    │
              │                 │
              │ Capabilities    │
              │ IPC             │
              │ Threads         │
              │ Scheduler       │
              │ Memory          │
              │ Interrupts      │
              │ Kernel Objects  │
              │ CPU primitives  │
              └────────┬────────┘
                       │
                    Hardware
```

The kernel is the security boundary while most operating-system functionality lives outside it.

---

# Relationship to seL4

Aegis Core takes architectural inspiration from seL4, particularly:

- Microkernel design
- Capability-based security
- Explicit IPC
- Strong isolation
- Small trusted computing base
- Minimal kernel responsibilities
- Security invariants
- Verification-oriented development

Aegis Core is independently implemented and is not a fork or implementation of seL4.

The project should only claim formal properties that have actually been demonstrated.

---

# Future Work

Long-term development may include:

- Additional CPU architectures
- SMP / multi-core support
- Advanced scheduling
- More complete capability derivation
- Capability revocation
- Advanced memory management
- Userspace driver framework
- Device services
- Networking services
- Storage services
- Secure boot integration
- Hardware security features
- Fault recovery
- Runtime isolation
- Formal verification
- Security auditing
- Reproducible builds

---

# Project Philosophy

Aegis Core is not intended to become a giant collection of operating-system functionality.

Its purpose is to provide a **small, secure and understandable foundation** on which the rest of Altis OS can be built.

```text
Small kernel
     +
Capabilities
     +
Isolation
     +
IPC
     +
Minimal privilege
     +
Verifiable invariants
     =
Aegis Core
```

**Aegis Core — the trusted foundation of Altis OS.**

---

## License

See [`LICENSE`](LICENSE).
