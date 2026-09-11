# Contributing to Altis Kernel

Thank you for your interest in contributing to Altis Kernel.

Altis Kernel is an open-source Rust-based microkernel designed as the foundation of Altis OS. The goal is to create a secure, modular, scalable and high-performance kernel with a strong focus on reliability, isolation and long-term maintainability.

Everyone is welcome to contribute, whether through code, documentation, testing, research, design ideas or security reviews.

---

# Code of Conduct

All contributors are expected to:

* Be respectful and constructive.
* Discuss technical decisions professionally.
* Accept feedback and reviews.
* Focus on improving the project.

Harassment, personal attacks, or intentionally harmful contributions are not accepted.

---

# Ways to Contribute

You can contribute by:

* Writing Rust code.
* Improving kernel architecture.
* Adding hardware support.
* Finding and fixing bugs.
* Improving documentation.
* Creating tests.
* Reviewing security-related code.
* Researching operating system concepts.

---

# Before Contributing

Before making large changes:

1. Read the documentation in `/docs`.
2. Check existing Issues and Pull Requests.
3. Discuss major architectural changes before implementation.
4. Make sure the change fits the goals of Altis Kernel.

Large changes to core systems such as:

* Memory management
* Scheduler
* IPC system
* Capability system
* Security model
* Kernel architecture

should be discussed before development begins.

---

# Development Setup

## Requirements

You need:

* Rust toolchain
* Cargo
* Git
* QEMU (for testing)
* A supported compiler toolchain

Install Rust:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Clone the repository:

```bash
git clone https://github.com/AltisOS/altis-kernel.git
cd altis-kernel
```

Build the project:

```bash
cargo build
```

---

# Project Structure

The kernel is organized into separate modules:

```
altis-kernel/

├── arch/
│   ├── x86_64/
│   ├── aarch64/
│   └── riscv64/
│
├── memory/
├── scheduler/
├── ipc/
├── capability/
├── process/
├── vm/
├── hypervisor/
│
├── tests/
├── docs/
└── tools/
```

Each module should have a clear responsibility.

---

# Coding Guidelines

## Rust

Altis Kernel follows these principles:

* Prefer safe Rust whenever possible.
* Keep `unsafe` code minimal and documented.
* Write clear and maintainable code.
* Avoid unnecessary dependencies.
* Consider security implications of every change.

Example:

```rust
// SAFETY:
// Explain why this unsafe operation is required.
unsafe {
    operation();
}
```

---

# Commit Guidelines

Use clear commit messages.

Good:

```
Add physical memory allocator
Fix IPC capability validation
Improve scheduler documentation
```

Avoid:

```
update stuff
fix bug
changes
```

---

# Pull Requests

All changes must be submitted through Pull Requests.

A good Pull Request should:

* Explain what changed.
* Explain why the change is needed.
* Include testing information.
* Include documentation updates if needed.

Example:

```
Title:
Add initial memory allocator

Description:
This adds the first version of the physical memory allocator.

Testing:
Tested with QEMU x86_64.
```

---

# Testing

Before submitting a Pull Request:

Run:

```bash
cargo test
```

Test the kernel in an emulator when possible:

```bash
qemu-system-x86_64
```

New features should include tests whenever possible.

---

# Security Issues

Do not publicly report serious security vulnerabilities.

Instead, report them privately through the security reporting process described in `SECURITY.md`.

Security is a core priority of Altis Kernel.

---

# Design Philosophy

Altis Kernel follows these principles:

* Small and modular kernel.
* Strong isolation between components.
* Capability-based security.
* Minimal trusted computing base.
* Hardware independence.
* Long-term maintainability.
* Open collaboration.

---

# Becoming a Maintainer

Regular contributors may become maintainers after demonstrating:

* Good technical understanding.
* Responsible code reviews.
* Understanding of Altis Kernel architecture.
* Commitment to project goals.

Maintainers help review contributions but must follow the project's security and quality standards.

---

# Thank You

Every contribution helps build a more secure and capable foundation for Altis OS.

Thank you for helping develop Altis Kernel.
