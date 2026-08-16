# Aegis Core — Scheduling

## 1. Purpose

The scheduler controls which runnable thread receives CPU time.

Aegis Core uses a small, capability-aware scheduler inspired by microkernel principles.

Goals:

- Strong thread isolation
- Predictable scheduling
- Efficient context switching
- Safe blocking and wake-up
- Multi-core support
- IPC and timer integration
- Minimal kernel policy

Scheduling authority and capability authority remain separate: the scheduler controls CPU allocation; capabilities control access to resources.

## 2. Thread States

Initial states:

```text
New
Ready
Running
Blocked
Suspended
Terminated
```

State flow:

```text
New → Ready → Running
             ├→ Ready
             ├→ Blocked → Ready
             └→ Terminated
```

Every transition must preserve scheduler invariants.

## 3. Thread Control Block

Each thread requires a kernel-side control structure:

```text
Thread
├── Thread ID
├── State
├── Priority
├── Scheduling policy
├── CPU affinity
├── Time slice
├── CPU context
├── Address-space reference
├── IPC state
├── Wait state
└── Architecture state
```

The structure should remain as small as practical.

## 4. Scheduling Policy

The first scheduler should use:

```text
Priority-based round robin
```

Higher-priority runnable threads are selected first. Threads at the same priority rotate fairly:

```text
A → B → C → A
```

Advanced real-time policies should wait until the basic scheduler is stable.

## 5. Ready Queues

Runnable threads are stored in ready queues.

Example:

```text
Priority 3: A → B
Priority 2: C → D
Priority 1: E → F
```

The queue must prevent:

- Duplicate entries
- Invalid states
- Removing the wrong thread
- Use-after-free
- Concurrent corruption

## 6. Time Slices

A time slice limits how long a preemptible thread runs before the scheduler reevaluates the runnable set.

```text
Thread A  ████████
Thread B          ████████
Thread C                  ████████
```

Timer handling should be separate from scheduling policy.

## 7. Preemption

A timer interrupt can cause the scheduler to reconsider the running thread:

```text
Running
   │
   │ timer interrupt
   ▼
Scheduler
   ├── same thread → continue
   └── another thread → context switch
```

The interrupted CPU context must be saved before switching.

## 8. Yield

A thread may voluntarily release the CPU:

```text
Thread → yield → Scheduler → another runnable thread
```

Yield must not be required for normal preemptive scheduling.

## 9. Blocking

Threads waiting for an unavailable resource should block rather than consume CPU.

Typical reasons:

```text
IPC
Notification
Timer
Resource
```

Flow:

```text
Running → Blocked → Ready → Running
```

The scheduler owns the state transition; IPC and other subsystems request blocking or wake-up.

## 10. Wake-Up

When the required event occurs:

```text
Blocked thread
      │
      ▼
    Wake
      │
      ▼
Ready queue
      │
      ▼
Scheduler
```

Wake-up must prevent lost wake-ups and double queue insertion.

## 11. IPC Integration

IPC and scheduling must remain separate:

```text
IPC
 │
 └── request block/wake
          │
          ▼
      Scheduler
```

Example:

```text
Client → Call → Blocked
                  │
             Server Reply
                  │
                  ▼
                Ready
                  │
                  ▼
               Running
```

## 12. Timer Integration

The timer subsystem provides time-based events:

```text
Hardware Timer
      │
      ▼
Timer Interrupt
      │
      ├── update time
      ├── process timers
      └── request scheduling
```

The timer must not contain scheduler policy.

## 13. Idle Thread

Each CPU should have an idle thread.

```text
Runnable work? ── Yes → run selected thread
       │
       No
       ▼
   idle thread
```

On x86_64 the idle path may eventually use `hlt`.

## 14. Context Switching

Context switching transfers execution between threads:

```text
Thread A
   │
   │ save context
   ▼
Scheduler
   │
   │ load context
   ▼
Thread B
```

Architecture-specific code belongs under:

```text
kernel/src/arch/x86_64/
```

The generic scheduler should not manipulate x86_64 registers directly.

## 15. Address-Space Switching

A thread may belong to a different address space:

```text
Thread A → Address Space A
      │
      ▼ context switch
      │
Thread B → Address Space B
```

Memory-management code should own architecture-specific address-space switching.

## 16. CPU Affinity

Threads may have an allowed CPU mask:

```text
Thread A → CPU 0
Thread B → CPU 1
Thread C → CPU 0 or CPU 1
```

The scheduler must never execute a thread outside its affinity.

## 17. SMP Scheduling

For multiple CPUs:

```text
Scheduler
 ├── CPU 0 queue
 ├── CPU 1 queue
 └── CPU 2 queue
```

The first SMP implementation may use a global scheduler. Later, per-CPU run queues can improve scalability.

SMP scheduling must safely handle:

- Thread state
- Wake-ups
- Migration
- CPU affinity
- Queue ownership
- Context switching

## 18. Per-CPU State

Possible per-CPU scheduler data:

```text
Current thread
Idle thread
Ready queue
CPU ID
Timer state
Scheduler-local data
```

Per-CPU state should reduce unnecessary global locking.

## 19. Thread Migration

A thread can eventually migrate between CPUs:

```text
CPU 0
  │
  │ migrate
  ▼
CPU 1
```

Migration must prevent:

- Running the same thread on two CPUs
- Duplicate queue entries
- Lost wake-ups
- Invalid affinity
- Concurrent destruction

## 20. Priority Inversion

Example:

```text
High-priority thread
        │
        ▼
      waits
        │
        ▼
Low-priority thread
```

A future scheduler may support priority inheritance or donation.

These mechanisms should be introduced only after the basic model is stable.

## 21. Scheduling and Capabilities

Priority does not grant authority.

A high-priority thread does not automatically receive:

```text
More memory
More capabilities
More IPC rights
More device access
```

Capabilities determine authority; scheduling determines CPU allocation.

## 22. Scheduler Syscalls

A minimal future interface may contain:

```text
thread_yield
thread_set_priority
thread_get_priority
thread_set_affinity
thread_get_state
```

Privileged scheduler operations must be capability controlled.

## 23. Thread Lifecycle

```text
Create
  ↓
New
  ↓
Ready
  ↓
Running
  ├── Yield → Ready
  ├── Block → Blocked → Ready
  └── Exit → Terminated
```

A terminated thread must never return to a ready queue.

## 24. Scheduler Invariants

The scheduler must preserve:

```text
A thread is on at most one ready queue.
A Running thread belongs to exactly one CPU.
A Blocked thread is not runnable.
A Terminated thread is never scheduled.
A thread cannot run on a disallowed CPU.
A thread cannot run simultaneously on two CPUs.
```

These invariants should eventually become formal verification targets.

## 25. Rust Module Structure

Suggested structure:

```text
kernel/src/
├── scheduler/
│   ├── mod.rs
│   ├── scheduler.rs
│   ├── thread.rs
│   ├── state.rs
│   ├── priority.rs
│   ├── run_queue.rs
│   ├── cpu.rs
│   ├── affinity.rs
│   ├── timer.rs
│   ├── idle.rs
│   ├── context.rs
│   ├── error.rs
│   └── tests.rs
│
└── arch/
    └── x86_64/
        ├── context.rs
        ├── interrupts.rs
        └── timer.rs
```

## 26. Testing Strategy

### Thread states

- New → Ready
- Ready → Running
- Running → Ready
- Running → Blocked
- Blocked → Ready
- Running → Terminated
- Reject invalid transitions

### Ready queues

- Insert
- Remove
- Peek
- Priority ordering
- Round-robin ordering
- Duplicate prevention

### Scheduling

- Highest-priority selection
- Same-priority rotation
- Yield
- Preemption
- Idle scheduling

### Blocking

- Block thread
- Wake thread
- Multiple waiters
- Wake exactly once
- Prevent lost wake-ups

### SMP

- Per-CPU queues
- CPU affinity
- Migration
- Concurrent wake-up
- Prevent double execution

### Security

- Unauthorized priority changes
- Invalid thread capability
- Invalid CPU affinity
- Unauthorized scheduling-state access

## 27. Verification Goals

Important properties:

```text
No thread runs on two CPUs simultaneously
No terminated thread is scheduled
No blocked thread is scheduled
No invalid state transition
No duplicate ready-queue membership
No unauthorized scheduler operation
No lost wake-up
No scheduler data-structure corruption
```

Any real-time guarantees must be formally specified rather than assumed.

## 28. Implementation Order

```text
1. Define Thread object
2. Define thread states
3. Define scheduler interface
4. Implement ready queue
5. Implement priority selection
6. Implement round robin
7. Implement yield
8. Implement blocking
9. Implement wake-up
10. Integrate timers
11. Implement preemption
12. Implement context switching
13. Integrate address-space switching
14. Implement idle thread
15. Add CPU affinity
16. Add SMP support
17. Add per-CPU queues
18. Add migration
19. Add scheduler accounting
20. Add advanced priority mechanisms
21. Define formal scheduler invariants
22. Begin verification
```

Advanced SMP and real-time features should not be added before the single-core scheduler is stable.

## 29. Relationship With Other Subsystems

```text
Scheduler
   ├── Threads
   ├── IPC → block / wake
   ├── Memory → address-space switching
   ├── Timer → preemption
   ├── Interrupts → scheduling events
   └── Capabilities → scheduling authority
```

Subsystem boundaries should remain explicit.

## 30. Design Rule

> The scheduler controls CPU allocation; capabilities control authority.

The first scheduler should be small, deterministic, testable, and easy to reason about before advanced scheduling policies are introduced.
