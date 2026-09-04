# Aegis Core — Kernel Milestones

## M0 — Project Foundation
- [x] Rust workspace
- [x] `x86_64-unknown-none` target
- [x] `#![no_std]`
- [x] Kernel entry point
- [x] Panic handler
- [x] Basic serial debugging
- [x] Kernel builds successfully
- [ ] Reproducible kernel build
- [ ] Bootable kernel image
- [ ] QEMU boot test

## M1 — Architecture
- [ ] Define kernel/user-space boundary
- [ ] Define kernel object model
- [ ] Define capability model
- [ ] Define IPC model
- [ ] Define address-space model
- [ ] Define thread/TCB model
- [ ] Define scheduler model
- [ ] Define interrupt model
- [ ] Minimize the Trusted Computing Base (TCB)

## M2 — CPU & Architecture
- [ ] CPU initialization
- [ ] GDT
- [ ] IDT
- [ ] Exception handling
- [ ] Interrupt handling
- [ ] CPU context management
- [ ] Register abstraction
- [ ] Timer
- [ ] Multi-core initialization
- [ ] CPU-local state

## M3 — Memory Management
- [ ] Physical frame abstraction
- [ ] Page abstraction
- [ ] Page tables
- [ ] Virtual address spaces
- [ ] Kernel and user address spaces
- [ ] Memory regions
- [ ] Frame allocation
- [ ] Memory ownership
- [ ] Memory isolation
- [ ] Validate all mappings
- [ ] Prevent unauthorized memory access

## M4 — Kernel Objects
- [ ] Generic kernel object abstraction
- [ ] Thread / TCB
- [ ] CNode
- [ ] Endpoint
- [ ] Notification
- [ ] Frame
- [ ] VSpace
- [ ] IRQ object
- [ ] Object lifetime management

## M5 — Capability System
- [ ] Capability structure
- [ ] CSpace
- [ ] Capability lookup
- [ ] Capability rights
- [ ] Capability creation
- [ ] Capability transfer
- [ ] Capability derivation
- [ ] Capability revoke
- [ ] Capability deletion
- [ ] Capability validation
- [ ] Prevent unauthorized authority escalation

## M6 — IPC
- [ ] Endpoint implementation
- [ ] Message passing
- [ ] Synchronous IPC
- [ ] Notifications
- [ ] Reply objects
- [ ] Capability transfer
- [ ] IPC blocking/wakeup
- [ ] IPC permission checks
- [ ] IPC correctness tests

## M7 — Threads & Scheduler
- [ ] Thread states
- [ ] Thread creation/destruction
- [ ] TCB management
- [ ] CPU context switching
- [ ] Ready queues
- [ ] Scheduler
- [ ] Priorities
- [ ] Time slices
- [ ] Timer integration
- [ ] Multi-core scheduling
- [ ] Scheduler invariants

## M8 — System Calls
- [ ] Syscall entry
- [ ] Syscall dispatcher
- [ ] Capability syscalls
- [ ] IPC syscalls
- [ ] Thread syscalls
- [ ] Memory syscalls
- [ ] Address-space syscalls
- [ ] Interrupt syscalls
- [ ] Reply syscalls
- [ ] Validate syscall arguments
- [ ] Reject invalid capabilities
- [ ] Reject invalid memory references

## M9 — Interrupts
- [ ] Interrupt controller abstraction
- [ ] IRQ objects
- [ ] IRQ capabilities
- [ ] Interrupt routing
- [ ] Interrupt delivery
- [ ] Timer interrupts
- [ ] User-space interrupt handling
- [ ] Interrupt isolation

## M10 — User-Space Boundary
- [ ] Initial user-space task
- [ ] Root task
- [ ] Initial CSpace
- [ ] Initial VSpace
- [ ] User-space startup
- [ ] Kernel/user transitions
- [ ] User-space IPC
- [ ] User-space service model

## M11 — Driver Isolation
- [ ] Device capability model
- [ ] Driver process model
- [ ] Driver IPC interface
- [ ] Driver memory mapping
- [ ] Interrupt delivery to drivers
- [ ] DMA isolation
- [ ] Driver restart model
- [ ] Driver crash isolation

Drivers should run outside the microkernel whenever practical.

## M12 — Virtualization
- [ ] VM object model
- [ ] VM creation/destruction
- [ ] Virtual CPU
- [ ] Guest memory isolation
- [ ] Guest address spaces
- [ ] Virtual interrupts
- [ ] VM scheduling
- [ ] VM capabilities
- [ ] VM lifecycle management
- [ ] Multiple concurrent VMs
- [ ] VM isolation testing

Target architecture:

```text
                    Aegis Core
                 Microkernel / TCB
                         │
          ┌──────────────┼──────────────┐
          │              │              │
         VM 1           VM 2           VM 3
          │              │              │
     Guest Kernel   Guest Kernel   Guest Kernel
          │              │              │
      Services       Services       Services
```

## M13 — Security Hardening
- [ ] Least-privilege enforcement
- [ ] Capability isolation
- [ ] Memory isolation
- [ ] IPC isolation
- [ ] VM isolation
- [ ] Driver isolation
- [ ] Kernel input validation
- [ ] Object lifetime validation
- [ ] Integer overflow review
- [ ] Concurrency review
- [ ] Undefined-behavior review
- [ ] Kernel attack-surface reduction
- [ ] Security audit

## M14 — Testing
- [ ] Unit tests
- [ ] Integration tests
- [ ] Capability tests
- [ ] IPC tests
- [ ] Memory tests
- [ ] Scheduler tests
- [ ] Interrupt tests
- [ ] VM tests
- [ ] Isolation tests
- [ ] Regression tests
- [ ] Fuzz testing
- [ ] Syscall fuzzing
- [ ] IPC fuzzing
- [ ] Capability fuzzing
- [ ] Memory-management fuzzing

## M15 — Verification
- [ ] Define kernel invariants
- [ ] Define security properties
- [ ] Define system assumptions
- [ ] Formalize capability rules
- [ ] Formalize memory-isolation rules
- [ ] Formalize IPC rules
- [ ] Formalize scheduler properties
- [ ] Verify critical kernel components
- [ ] Verify capability isolation
- [ ] Verify memory isolation
- [ ] Verify IPC correctness
- [ ] Document proof assumptions

Aegis Core is inspired by the security and verification philosophy of seL4, but it is its own implementation. It should not claim seL4-level formal verification until such verification has actually been completed.

# Release Milestones

## R0 — Bootable Kernel
- [ ] Kernel builds
- [ ] Bootloader works
- [ ] QEMU boots
- [ ] Serial output works
- [ ] Panic handling works
- [ ] CPU initialization works

## R1 — Minimal Microkernel
- [ ] Kernel objects
- [ ] Threads
- [ ] TCB
- [ ] Address spaces
- [ ] Capabilities
- [ ] IPC
- [ ] System calls
- [ ] Interrupts
- [ ] Scheduler

## R2 — Isolated System
- [ ] User-space execution
- [ ] Capability-controlled resources
- [ ] Memory isolation
- [ ] IPC isolation
- [ ] Driver isolation
- [ ] User-space services

## R3 — Hypervisor / VM Core
- [ ] VM creation
- [ ] Virtual CPUs
- [ ] Guest memory isolation
- [ ] Virtual interrupts
- [ ] VM scheduling
- [ ] Multiple simultaneous VMs

## R4 — Security Hardened
- [ ] Completed threat model
- [ ] Security invariants enforced
- [ ] Extensive fuzzing
- [ ] Security audit
- [ ] Reproducible builds
- [ ] Secure boot architecture

## R5 — Verified Core
- [ ] Formal kernel model
- [ ] Formal invariants
- [ ] Capability verification
- [ ] Memory-isolation verification
- [ ] IPC verification
- [ ] Critical kernel components formally verified

# Long-Term Kernel Goal

Aegis Core should remain a small, capability-based microkernel providing only the mechanisms necessary for:

1. Isolation
2. Capabilities
3. IPC
4. Memory management
5. Scheduling
6. Interrupt management
7. Virtualization

Complex functionality should remain outside the kernel whenever possible.

The primary design objective is to minimize the Trusted Computing Base while maintaining strong isolation between processes, services, drivers, and virtual machines.
