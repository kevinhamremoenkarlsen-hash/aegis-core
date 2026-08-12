Aegis Core — Inter-Process Communication (IPC)

1. Purpose

IPC is the primary communication mechanism between isolated components in Aegis Core and Altis OS.

Aegis Core should provide a small, deterministic IPC mechanism while leaving higher-level communication protocols to userspace.

The design is inspired by capability-oriented microkernels such as seL4.

Core principles:

Explicit communication authority

Capability-controlled endpoints

Strong isolation

Minimal kernel work

Deterministic behavior

Safe blocking and wake-up

Optional capability transfer

No implicit global communication channels

2. IPC Architecture

┌──────────────────┐
│   Process A      │
│                  │
│ Endpoint Cap     │
└────────┬─────────┘
         │
         │ IPC
         ▼
┌──────────────────┐
│    Aegis Core    │
│                  │
│ Validate Cap     │
│ Validate Rights  │
│ Transfer Data    │
│ Block / Wake     │
└────────┬─────────┘
         │
         │ IPC
         ▼
┌──────────────────┐
│   Process B      │
│                  │
│ Endpoint Cap     │
└──────────────────┘

The kernel provides the transport mechanism. Userspace services define what messages mean.

3. Security Model

IPC must be capability controlled.

A process cannot communicate with an endpoint merely because it knows an identifier.

Incorrect:

Endpoint ID
    │
    ▼
Access endpoint

Correct:

Endpoint Capability
        │
        ▼
Validate capability
        │
        ▼
Validate IPC rights
        │
        ▼
Access endpoint

The capability system therefore forms the security boundary for IPC.

4. IPC Primitives

Initial primitives:

Send
Receive
Call
Reply
Notify
Wait
Poll
Cancel

The exact syscall interface may change during implementation.

Conceptually:

Send    → transmit a message
Receive → wait for a message
Call    → send and wait for a reply
Reply   → answer a call
Notify  → signal an event
Wait    → block until an event/message
Poll    → check without blocking
Cancel  → cancel a supported blocked operation

The kernel should avoid adding high-level protocol semantics.

5. Endpoints

An endpoint is a kernel object used for synchronous message communication.

Endpoint
├── Waiting senders
├── Waiting receivers
├── State
└── Synchronization data

An endpoint is accessed through a capability.

Example:

Process A
   │
   └── Send Capability
             │
             ▼
          Endpoint
             ▲
             │
   ┌─────────┘
   │
Process B
   │
   └── Receive Capability

Different capabilities may provide different rights.

6. Endpoint Rights

Possible endpoint rights:

Send
Receive
Call
Grant
Control

Example:

Client:
Send + Call

Server:
Receive + Reply

The exact right model should be finalized together with the capability specification.

7. Synchronous IPC

Aegis Core should initially prioritize synchronous IPC.

Basic flow:

Sender
   │
   │ Send
   ▼
Endpoint
   │
   │ Receiver available?
   ├──── Yes ────► Transfer message
   │
   └──── No ─────► Block sender

Receiver:

Receiver
   │
   │ Receive
   ▼
Endpoint
   │
   │ Sender waiting?
   ├──── Yes ────► Transfer message
   │
   └──── No ─────► Block receiver

This minimizes kernel buffering and makes ownership explicit.

8. Direct Handoff

Where possible, IPC should support direct transfer between sender and receiver without unnecessary intermediate buffering.

Sender
   │
   │ Message
   ▼
Kernel
   │
   │ Direct transfer
   ▼
Receiver

This can reduce:

Memory copying

Kernel storage

Latency

Complexity

The implementation must still validate every boundary crossing.

9. Message Structure

A basic IPC message may contain:

Message
├── Message label
├── Payload words
├── Capability transfer metadata
└── Length

Conceptually:

┌──────────────┬──────────┬───────────────┬────────┐
│ Label        │ Length   │ Payload       │ Caps   │
└──────────────┴──────────┴───────────────┴────────┘

The kernel should keep the generic message format small.

Userspace protocols can define the meaning of labels and payloads.

10. Message Labels

A message label identifies the requested operation or protocol.

Example:

FILE_OPEN
FILE_READ
FILE_WRITE
PROCESS_CREATE
DEVICE_REQUEST
NETWORK_SEND

These labels should normally be defined by userspace protocols rather than hard-coded into the microkernel.

The kernel only transports the label.

11. Payload

The payload contains protocol data.

Conceptually:

Message
├── label
├── payload[0]
├── payload[1]
├── payload[2]
└── ...

The kernel must validate:

Message size

Address validity

Buffer accessibility

Alignment where required

Capability transfer metadata

The kernel must never trust userspace pointers.

12. Call and Reply

A call combines sending a request with waiting for a reply.

Client
   │
   │ Call
   ▼
Server
   │
   │ Reply
   ▼
Client

Typical service interaction:

Application
     │
     │ Call(FILE_OPEN)
     ▼
Filesystem Service
     │
     │ Reply(handle)
     ▼
Application

The kernel manages the synchronization mechanism; the service defines the protocol.

13. Reply Objects

A reply object can represent authority to respond to a specific call.

Conceptually:

Client
   │
   │ Call
   ▼
Server
   │
   └── Reply capability
            │
            ▼
          Client

Reply authority must be isolated so that one service cannot arbitrarily reply to another process's outstanding call.

14. Notifications

Notifications provide lightweight event signaling.

Producer
   │
   │ Notify
   ▼
Notification Object
   │
   │ Wake
   ▼
Consumer

Notifications are useful for:

Interrupt delivery

Timers

Device events

Scheduler events

Lightweight synchronization

A notification normally carries less information than a full IPC message.

15. Interrupt + IPC

Hardware interrupts can eventually be converted into userspace-visible notifications.

Hardware
   │
   ▼
CPU IRQ
   │
   ▼
Aegis Core
   │
   ▼
IRQ Capability
   │
   ▼
Notification
   │
   ▼
Userspace Driver

This allows many drivers to remain outside the kernel.

16. Blocking

IPC operations may block when the requested counterpart is unavailable.

Example:

Receive
   │
   ├── Message available
   │       │
   │       ▼
   │     Return
   │
   └── No message
           │
           ▼
        Block thread

A blocked thread should not consume CPU time.

The scheduler moves it out of the runnable state until the required event occurs.

17. Wake-Up

When the required IPC event occurs:

Blocked Thread
      │
      │ IPC event
      ▼
Wake-up
      │
      ▼
Ready Queue
      │
      ▼
Scheduler

Wake-up operations must be race-safe.

The implementation must prevent:

Lost wake-ups

Double wake-ups

Invalid thread state transitions

Use-after-free

18. IPC State Machine

A thread performing IPC may have states such as:

Running
   │
   ▼
IPC Operation
   │
   ├── Completed ──► Running
   │
   └── Blocked ────► Blocked on IPC
                         │
                         ▼
                       Wake
                         │
                         ▼
                       Ready
                         │
                         ▼
                      Running

Only valid transitions should be permitted.

19. Capability Transfer

IPC may optionally transfer capabilities.

Process A
   │
   │ Message + Capability
   ▼
Aegis Core
   │
   │ Validate transfer
   ▼
Process B

The sender must possess the necessary transfer/grant authority.

The destination must have a valid capability slot.

The kernel must ensure that the transferred authority does not exceed the sender's allowed authority.

20. Capability Transfer Example

Server
   │
   ├── Endpoint capability
   └── Frame capability
          │
          │ IPC transfer
          ▼
Client
   │
   └── Receives restricted Frame capability

Example:

Server rights:
Read + Write

Transferred rights:
Read

The receiving process must not receive more authority than allowed by the transfer operation.

21. Shared Memory IPC

Large data should not necessarily be copied through IPC messages.

Instead:

Process A
   │
   │ Frame capability
   ▼
Shared Frame
   ▲
   │ Frame capability
   │
Process B

The IPC message can carry a capability or reference to an explicitly shared memory object.

This can support efficient:

File transfers

Network buffers

Graphics buffers

Large data structures

Device buffers

Shared memory must remain explicitly authorized.

22. Copying vs Shared Memory

Small messages:

Process A
    │
    │ small IPC message
    ▼
Process B

Large data:

Process A ──► Shared Memory ◄── Process B
       │                         │
       └──── IPC metadata ───────┘

The kernel should not unnecessarily copy large buffers.

23. IPC and Memory Safety

Userspace pointers must be treated as untrusted.

Before accessing userspace memory, the kernel must ensure that:

The address belongs to the caller's address space.

The memory is mapped.

Required permissions exist.

The range is valid.

Integer overflow cannot bypass bounds checks.

The operation cannot access kernel memory.

Invalid buffers must result in controlled errors.

24. IPC Errors

Possible errors:

InvalidEndpoint
InvalidCapability
InsufficientRights
InvalidMessage
InvalidBuffer
MessageTooLarge
InvalidCapabilityTransfer
InvalidSlot
EndpointClosed
ThreadStateError
OperationCancelled
Timeout

Errors must not expose privileged kernel state.

25. Timeouts

Timeouts may be added after the basic IPC model is stable.

Conceptually:

Receive(timeout)
      │
      ├── Message arrives
      │       └── Success
      │
      └── Timeout expires
              └── Timeout error

Timeout implementation should integrate with the kernel timer and scheduler rather than creating an independent timing system.

26. Cancellation

Blocked IPC operations may eventually support cancellation.

Thread
   │
   ▼
Blocked on IPC
   │
   │ Cancel
   ▼
Ready / Error

Cancellation semantics must be carefully specified to prevent races between:

Message arrival
Wake-up
Cancellation
Thread destruction
Endpoint destruction

27. Endpoint Lifetime

Endpoints are kernel objects protected by capabilities.

Example:

Capability A ─┐
Capability B ─┼──► Endpoint
Capability C ─┘

Deleting one capability must not destroy the endpoint while valid capabilities remain.

Endpoint destruction must safely handle waiting threads.

Possible policy:

Endpoint destroyed
       │
       ├── Wake waiting senders
       └── Wake waiting receivers

The exact error returned must be defined by the IPC specification.

28. Concurrency

IPC must work correctly on multi-core systems.

Important synchronization requirements:

Endpoint queue consistency

Atomic state transitions

Safe thread wake-up

Capability transfer consistency

Object lifetime safety

No lost wake-ups

No double enqueue

No use-after-free

The implementation should avoid unnecessary global locks.

29. Priority and IPC

IPC interacts with scheduling.

A future implementation may need to account for priority inversion.

Example:

High-priority client
        │
        ▼
Low-priority server
        │
        ▼
High-priority client blocked

Possible future mechanisms include priority inheritance or priority donation.

These should only be added after the basic scheduler and IPC model are stable and formally specified.

30. IPC Security Rules

Rule 1 — No capability, no IPC

A process requires valid endpoint authority.

Rule 2 — No implicit channels

Communication channels must be explicitly established.

Rule 3 — Validate every message

All userspace input is untrusted.

Rule 4 — Validate transferred capabilities

Capability transfer must be authorized.

Rule 5 — Preserve isolation

IPC must not provide unintended access to another process's memory.

Rule 6 — Protect kernel memory

Userspace IPC cannot directly expose kernel memory.

Rule 7 — Preserve object lifetime

Waiting threads and endpoints must remain valid during IPC operations.

Rule 8 — Atomic state transitions

Concurrent IPC operations must preserve kernel invariants.

31. IPC Protocol Example

A userspace filesystem service could define:

Client
   │
   │ Call(FILE_OPEN, "/file")
   ▼
Filesystem Service
   │
   │ Reply(file_cap)
   ▼
Client

Later:

Client
   │
   │ Call(FILE_READ, file_cap, buffer)
   ▼
Filesystem Service
   │
   │ IPC with storage service
   ▼
Storage Driver
   │
   │ Reply(data)
   ▼
Filesystem Service
   │
   │ Reply(bytes_read)
   ▼
Client

Aegis Core only provides the IPC mechanism and authority checks.

32. Suggested Rust Module Structure

kernel/src/
└── ipc/
    ├── mod.rs
    ├── message.rs
    ├── endpoint.rs
    ├── notification.rs
    ├── reply.rs
    ├── send.rs
    ├── receive.rs
    ├── call.rs
    ├── transfer.rs
    ├── blocking.rs
    ├── wake.rs
    ├── error.rs
    └── tests.rs

Possible future modules:

    ├── timeout.rs
    ├── cancel.rs
    ├── shared_memory.rs
    └── priority.rs

The exact structure may change as implementation develops.

33. Syscall Layer

The syscall layer should expose a minimal IPC interface.

Conceptual interface:

sys_send()
sys_receive()
sys_call()
sys_reply()
sys_notify()
sys_wait()

Capability handles should be passed as arguments.

Example:

sys_call(endpoint_cap, message)

The syscall layer validates the capability before entering the IPC subsystem.

34. Testing Strategy

Endpoint tests

Create endpoint

Send

Receive

Call

Reply

Destroy endpoint

Blocking tests

Block sender

Block receiver

Wake sender

Wake receiver

Prevent lost wake-ups

Capability tests

Valid endpoint capability

Invalid capability

Missing Send right

Missing Receive right

Unauthorized transfer

Message tests

Empty message

Normal message

Maximum message

Oversized message

Invalid buffer

Invalid length

Concurrency tests

Multiple senders

Multiple receivers

Simultaneous send/receive

Endpoint destruction during wait

Capability transfer during concurrent IPC

Security tests

Forged capability

Invalid pointer

Kernel-memory pointer

Rights escalation

Unauthorized endpoint access

35. Verification Goals

The IPC subsystem should eventually have precise specifications for:

Endpoint state
Message state
Thread IPC state
Blocking
Wake-up
Send
Receive
Call
Reply
Notification
Capability transfer
Object lifetime
Concurrency

Important properties include:

No unauthorized communication
No memory isolation violation
No lost wake-up
No invalid thread state
No use-after-free
No capability escalation

These properties should be treated as specifications to verify, not assumptions.

36. Implementation Order

IPC should be implemented in controlled stages:

1. Define IPC message format
2. Define endpoint object
3. Define endpoint rights
4. Implement send
5. Implement receive
6. Implement blocking
7. Implement wake-up
8. Implement call
9. Implement reply
10. Implement notifications
11. Integrate capability lookup
12. Implement capability transfer
13. Integrate memory validation
14. Integrate scheduler
15. Add timeouts
16. Add cancellation
17. Add shared-memory mechanisms
18. Add SMP support
19. Add security tests
20. Define formal IPC invariants
21. Begin verification work

Do not add advanced features before the basic IPC state machine is stable.

37. Design Rule

The central IPC rule of Aegis Core is:

Communication is an explicit capability-controlled operation between isolated components.

The kernel should provide the smallest practical IPC mechanism while keeping protocol logic, services, and application-level communication in userspace.
