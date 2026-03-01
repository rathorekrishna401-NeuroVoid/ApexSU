# ApexSU

A hardened, Rust-focused Android root solution forked from [KernelSU](https://github.com/tiann/KernelSU), built for security researchers and power users.

[![CI](https://github.com/qrjhamron/ApexSU/actions/workflows/ci.yml/badge.svg)](https://github.com/qrjhamron/ApexSU/actions/workflows/ci.yml)
[![License: GPL v2](https://img.shields.io/badge/License-GPL_v2-blue.svg)](LICENSE)

## What is ApexSU?

ApexSU provides kernel-level root access for Android devices using a loadable kernel module (LKM). It intercepts system calls to grant root privileges to allowed applications while maintaining a minimal footprint.

The project focuses on code quality and security hardening. The userspace daemon and JNI bridge are written in Rust with strict clippy linting, documented unsafe blocks, and comprehensive input validation. The kernel module remains in C as required by the Linux kernel API.

## Key Differences from KernelSU

| Feature | KernelSU | ApexSU |
|---------|----------|--------|
| JNI bridge | C++ | Rust (zero C++) |
| Module validation | Basic | Comprehensive (path traversal, size limits, field validation) |
| System diagnostics | None | Built-in `ksud diagnose` command |
| Error handling | Silent suppression | Structured logging for all IOCTL errors |
| Kernel hardening | Standard | Atomic operations, NULL checks, write-to-temp persistence |
| Clippy policy | Not enforced | Zero warnings (`clippy::all` + `clippy::pedantic`) |

## Supported Devices

- GKI 2.0 devices (Android 12+, kernel 5.10+)
- LKM mode: Android 13+
- See [KernelSU device support](https://kernelsu.org/guide/installation.html) for full compatibility list

Requires an unlocked bootloader.

## Installation

1. Check your device is supported (GKI 2.0, unlocked bootloader)
2. Download the latest APK from [Releases](https://github.com/qrjhamron/ApexSU/releases)
3. Install the APK
4. Follow in-app instructions to patch your boot image

## Building from Source

### Prerequisites

- Rust (stable, 1.82+)
- [cargo-ndk](https://github.com/nickelc/cargo-ndk): `cargo install cargo-ndk`
- Android NDK r29
- JDK 17
- Android SDK with build-tools 35.0.0

### Build Commands

```bash
# Build userspace daemon
cd userspace/ksud
cargo ndk -t arm64-v8a build --release

# Build JNI bridge
cd manager/app/src/main/rust/kernelsu-jni
cargo ndk -t arm64-v8a build --release

# Copy native libraries
mkdir -p manager/app/src/main/jniLibs/arm64-v8a
cp userspace/ksud/target/aarch64-linux-android/release/ksud \
   manager/app/src/main/jniLibs/arm64-v8a/libksud.so
cp manager/app/src/main/rust/kernelsu-jni/target/aarch64-linux-android/release/libkernelsu.so \
   manager/app/src/main/jniLibs/arm64-v8a/libkernelsu.so

# Build APK
cd manager
./gradlew assembleRelease
```

## Architecture

```
Manager App (Kotlin UI)  ──►  JNI Bridge (Rust)
                                    │
                               IOCTL (anon inode)
                                    │
ksud Daemon (Rust)  ◄──────────────►│
                                    │
                          Kernel Module (C)
```

The kernel module hooks `execve` and `setresuid` syscalls to enforce an allowlist of root-granted UIDs. Communication uses IOCTLs on an anonymous inode — no `/proc`, `/sys`, or `/dev` entries.

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for full details.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for build instructions, code standards, and the Rust-First Law.

## Security

To report security vulnerabilities, please use private disclosure. See [SECURITY.md](SECURITY.md).

## License

GPL-2.0 — inherited from KernelSU and the Linux kernel.

## Acknowledgments

- [KernelSU](https://github.com/tiann/KernelSU) project and contributors
- [topjohnwu](https://github.com/topjohnwu) (Magisk) for magiskboot
