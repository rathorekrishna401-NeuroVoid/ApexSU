# ApexSU

> A hardened Android root solution built on KernelSU.
> Rust-first userspace. Stealth hardening. Built for precision.

[![CI](https://github.com/qrjhamron/ApexSU/actions/workflows/ci.yml/badge.svg)](https://github.com/qrjhamron/ApexSU/actions/workflows/ci.yml)
[![Version](https://img.shields.io/badge/version-0.1.0--beta-blue)](https://github.com/qrjhamron/ApexSU/releases)
[![License: GPL v2](https://img.shields.io/badge/License-GPL_v2-blue.svg)](LICENSE)
[![Android](https://img.shields.io/badge/Android-12%2B-green)](https://developer.android.com)

---

## What is ApexSU?

ApexSU is a fork of [KernelSU](https://kernelsu.org) focused on three things:
security hardening, Rust migration, and stealth improvements.

It is not a replacement for KernelSU. It is KernelSU with a different philosophy:
less C, more Rust, smaller attack surface, harder to detect.

---

## How ApexSU differs from KernelSU

| | KernelSU | ApexSU |
|---|---|---|
| JNI bridge | C++ | Rust |
| Anon inode name | `[ksu_driver]` | `[io_uring]` |
| Module validation | Basic | Path traversal + size + field validation |
| Diagnostics | None | Built-in health check (`ksud diagnose`) |
| Dead code tolerance | — | Zero `#[allow(dead_code)]` |
| Rust codebase % | ~15% | ~24% |
| Clippy policy | Not enforced | Zero warnings (`clippy::all` + `clippy::pedantic`) |

---

## Requirements

- Android 12 or higher
- Kernel 5.10+ (GKI 2.0)
- Unlocked bootloader

For full device compatibility list:
→ [kernelsu.org/guide/installation](https://kernelsu.org/guide/installation.html)

---

## Installation

1. Download the latest APK from [Releases](https://github.com/qrjhamron/ApexSU/releases)
2. Install the APK
3. Follow the in-app instructions
4. For flashing guide: [kernelsu.org](https://kernelsu.org/guide/installation.html)

---

## Building from source

Requirements:
- Rust stable (1.82+)
- Android NDK r29
- JDK 21
- Android SDK with build-tools 35.0.0

```bash
# Clone
git clone https://github.com/qrjhamron/ApexSU.git
cd ApexSU

# Build userspace daemon
cd userspace/ksud
cargo ndk -t arm64-v8a build --release
cd ../..

# Build manager APK
cd manager
./gradlew assembleRelease
```

---

## Architecture

```
┌─────────────────┐     ioctl      ┌──────────────────┐
│  Manager App    │ ────────────→  │  Kernel Module   │
│  (Kotlin + Rust)│                │  (C, kernel-space)│
└────────┬────────┘                └──────────────────┘
         │                                  ↑
         ↓                                  │
┌─────────────────┐     ioctl               │
│      ksud       │ ────────────────────────┘
│  (Rust daemon)  │
└─────────────────┘
```

The kernel module hooks syscalls to enforce a UID allowlist for root access.
Communication uses IOCTLs on an anonymous inode — no `/proc`, `/sys`, or `/dev` entries.

---

## Security

To report a vulnerability: open a private GitHub Security Advisory.
Do not open public issues for security bugs. See [SECURITY.md](SECURITY.md).

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for build instructions and code standards.

---

## Acknowledgments

- [KernelSU](https://github.com/tiann/KernelSU) — upstream project
- [topjohnwu](https://github.com/topjohnwu) — Magisk and magiskboot
- [Rust for Linux](https://rust-for-linux.com) — inspiration for Rust kernel work

---

## License

GPL-2.0 — inherited from KernelSU and the Linux kernel.
