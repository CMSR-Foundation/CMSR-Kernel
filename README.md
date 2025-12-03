# 📘 **CMSR Kernel**

## *Capsuled MicroService Runner — Kernel Architecture & Implementation Overview*

**Version:** 0.1-pre
**Component:** `cmsr-kernel`
**Language:** Rust (no `unsafe` outside of mandatory HAL components)

---

# 🔷 Introduction

The **CMSR Kernel** is a post-UNIX microkernel designed around four fundamental primitives:

* **Capsules** — isolated execution units
* **Capabilities** — unforgeable tokens that gate all access
* **Objects** — everything in the system is an object
* **Messages** — the only way to interact with objects

CMSR eliminates global namespaces, ambient authority, and shared mutable state.
The kernel is minimal, deterministic, audited, and fully capability-driven.

This README provides a complete technical overview of the kernel’s architecture, behavior, and guarantees.

---

# 🔷 Goals

The CMSR kernel is built with these goals:

1. **Strong Isolation**
   Each capsule receives its own address space, capabilities, and object handles.

2. **Capability-Based Security**
   No UIDs, no file permissions, no global handles.
   All access must be explicitly granted via cryptographic capability tokens.

3. **Minimal Kernel Surface**
   All higher-level services (networking, storage, shell, router, etc.) live in system capsules, not in the kernel.

4. **Deterministic, Auditable Behavior**
   Every security-relevant operation is checked and optionally audited.

5. **Message Passing as the Only Mechanism**
   No syscalls that mutate global state.
   All operations are performed by sending messages to system capsules.

---

# 🔷 High-Level Architecture

The CMSR kernel consists of the following major subsystems:

* **Scheduler (`sched`)**
* **Memory Manager (`mm`)**
* **Capability Engine (`caps`)**
* **IPC & Endpoint Router (`ipc`)**
* **Audit Core (`audit`)**
* **Policy Hooks (`policy`)**
* **Boot & Initialization Logic (`boot`)**

The kernel does **not** contain:

* networking stacks
* filesystems
* shell logic
* user management
* device drivers above the HAL layer
* cryptographic libraries (those exist in system capsules)

This separation keeps the kernel small, secure, and verifiable.

---

# 🔷 Capsules

Capsules are the only execution units the kernel recognizes.

### Capsule Properties

* Private virtual address space
* Isolated page tables
* Dedicated capability set
* One or more execution contexts (threads)
* Explicit lifecycle: *Created → Running → Frozen → Failed → Stopped*

### What the Kernel Guarantees

* A capsule cannot access another capsule’s memory
* A capsule cannot send or receive messages without a capability
* A capsule cannot enumerate other capsules
* A capsule cannot discover system structure without explicit permissions
* A capsule cannot obtain CPU, memory, or IO access beyond quotas

### Capsule Lifecycle (Kernel Side)

1. Capsule created (uninitialized address space allocated)
2. Capability set assigned
3. First execution context created
4. Capsule scheduled
5. Halt/freeze/stop based on capsule or admin events
6. Teardown: memory unmapped, capabilities revoked, resources freed

---

# 🔷 Capability Engine (`caps`)

Capabilities are cryptographically strong tokens stored in kernel space.

A capability defines:

```
Capability {
    token: [u8; 32],          // unguessable random
    object_id: ObjectID,      // kernel-managed
    ops: HashSet<Operation>,  // allowed operations
    limits: CapabilityLimits, // TTL, quotas, rate, max_uses
    delegation_depth: u8,
    labels: BTreeMap<String, String> // metadata
}
```

Capabilities are required for **every** operation, including:

* sending messages
* receiving messages
* opening storage objects
* accessing timers
* using network objects
* creating new capsules (admin only)
* modifying graphs (admin only)

### Kernel Responsibilities

* Validate tokens on every operation
* Enforce TTL expiry
* Enforce quotas (IO, memory, bandwidth)
* Enforce delegation rules
* Perform atomic revocation
* Notify audit core of relevant actions

### Non-Enumerability

The kernel provides **no API** to list capabilities.
A capsule only sees the tokens it currently owns.

---

# 🔷 Memory Manager (`mm`)

The memory manager provides:

### 1. Physical Frame Management

* frame allocator
* NUMA-aware allocation (if supported)
* physical isolation guarantees

### 2. Virtual Memory

Each capsule has:

* unique page tables
* isolated higher-half kernel mapping (read-only or fully absent)
* configurable virtual address space limits

### 3. Quotas & Enforcement

Capsules can be restricted via:

* maximum virtual size
* maximum physical usage
* allocation rate-limits

If limits are exceeded:

* allocation fails
* capsule may be frozen or restarted by policy

### 4. Kernel Memory Protection

Capsules cannot map kernel memory.
All syscalls are message-like, mediated by object endpoints.

---

# 🔷 Scheduler (`sched`)

CMSR uses a preemptive scheduler designed for fairness and isolation.

### Scheduling Model

Each capsule has:

* a scheduling class (interactive, batch, realtime, system)
* CPU shares
* optional priority hints
* optional latency targets

The scheduler decides:

* which execution context runs
* how long it runs
* when preemption occurs

### Starvation Prevention

The scheduler tracks runnable queues and ensures no capsule is indefinitely blocked unless a policy explicitly freezes it.

### CPU Isolation

Capsules cannot see thread IDs or cores.
The kernel may pin real-time capsules or system capsules depending on policy.

---

# 🔷 IPC & Endpoint Router (`ipc`)

**Messages** are the only way to interact with the kernel or other capsules.

### Endpoints

Endpoints are kernel objects representing:

* message queues
* ingress or egress points for capsules
* control interfaces for system capsules

### Message Format (Abstract)

```
Header {
    label: u64,
    len: u32,
    flags: u32
}
Payload [len bytes]
```

### Routing Rules

* Sender must present a valid Send capability
* Receiver must present a valid Recv capability
* Messages follow the static graph topology
* Backpressure is enforced (bounded queues)
* Drops and errors follow policy configuration

### Ordering

* Optional FIFO per producer→consumer
* No global ordering guarantees

### Backpressure

* `WouldBlock` returned if queue full
* Policy may drop low-priority messages
* Capsules expected to retry or degrade gracefully

---

# 🔷 Audit Core (`audit`)

The kernel emits audit events for all security-relevant actions.

### Event Types

* Capsule lifecycle events
* Capability issuance, revocation, delegation
* Messaging failures (unauthorized, backpressure, drops)
* Object operations (storage, network, control)
* Policy actions (denied/throttled)

### Guarantees

* Audit logs are append-only
* Logs are not modifiable by capsules
* Only authorized readers receive audit streams

---

# 🔷 Policy Hooks (`policy`)

The kernel does not interpret policy files.
Instead, it exposes hooks that system capsules (e.g., `sys.policy`) implement.

Policy hooks include:

* capability issuance pre-check
* capability delegation validation
* object operation validation
* runtime violation callbacks
* resource quota enforcement

The kernel trusts policy capsules only through special system capabilities.

---

# 🔷 Objects

Everything the kernel manages is modeled as an object:

### Object Classes

* **Endpoint**
* **Network Object (abstract until sys.net config)**
* **Timer**
* **Control Object**
* **Storage Handle (thin wrapper for sys.storage)**

Objects have unique, kernel-generated ObjectIDs.

Capsules cannot enumerate objects unless explicitly authorized.

---

# 🔷 Timers

Timers generate scheduled message events.

Types:

* one-shot
* periodic

Timer events are delivered through a capsule’s endpoint.
Timer capabilities define:

* allowed intervals
* max frequency
* optional quota
* TTL

---

# 🔷 Boot Process

1. Firmware loads kernel
2. Kernel initializes:

   * physical memory
   * interrupt controllers
   * capability engine root
   * scheduler
   * audit core
3. Load boot policy configuration
4. Create initial system capsules
5. Launch router, storage, auth, shell daemon, etc.
6. Enter fully operational state

---

# 🔷 Error Handling & Failure Model

### Capsule Crash

* kernel wipes address space
* capabilities are revoked
* policy determines restart/quarantine

### Unauthorized Operation

* operation denied
* audit event recorded
* policy may throttle or revoke caps

### Memory / CPU Pressure

* allocations fail
* capsule throttled or frozen

### Graph Violation

* invalid endpoint results in immediate error

---

# 🔷 Syscall Model (Message-Oriented)

CMSR intentionally **does not expose POSIX-style syscalls**.

All privileged actions follow one pattern:

1. Capsule sends a message to an object endpoint
2. Kernel validates capability
3. Kernel performs minimal operation or forwards to system capsule
4. Response is delivered as a message

There is no shared memory, no global state mutation, no escape path.

---

# 🔷 Kernel Code Structure (Suggested)

```
cmsr-kernel/
 ├─ src/
 │   ├─ boot/
 │   ├─ mm/
 │   ├─ sched/
 │   ├─ caps/
 │   ├─ ipc/
 │   ├─ audit/
 │   ├─ policy/
 │   ├─ objects/
 │   ├─ traps/
 │   ├─ arch/
 │   │   ├─ x86_64/
 │   │   └─ aarch64/
 │   ├─ util/
 │   └─ kernel.rs
 ├─ docs/
 ├─ tests/
 ├─ Cargo.toml
 └─ README.md
```

---

# 🔷 Current Development Status

CMSR Kernel is currently in **active development**, targeting:

* x86_64
* aarch64
* deterministic scheduler v1
* capability engine MVP
* message routing MVP
* memory isolation & basic page tables

Upcoming milestones:

* IPC backpressure policies
* audit streaming
* initial policy hooks
* first system capsule boot

---

# 🔷 Contributing

The kernel accepts contributions from experienced:

* OS developers
* Rust systems programmers
* formal verification researchers
* security engineers
* microkernel experts

The contribution process requires:

1. Code review by two maintainers
2. Audit log for all unsafe blocks
3. Tests for all kernel subsystems
4. Design documentation for new features

---

# 🔷 License

CMSR Kernel is released under the **Apache 2.0 License** to encourage research, adoption, and commercial use while maintaining strong guarantees of openness and patent protection.
