# Contributing to ApexSU (KernelSU Fork)

## Architecture Overview

ApexSU is a kernel-based root solution for Android with three main components:

```
┌─────────────────────────────────────────────────┐
│              Android Manager App                │
│         (Kotlin/Jetpack Compose UI)             │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ Screens  │  │ViewModels│  │  JNI Bridge   │  │
│  │(Compose) │  │ (Kotlin) │  │   (Rust)      │  │
│  └──────────┘  └──────────┘  └──────┬───────┘  │
└─────────────────────────────────────┼───────────┘
                                      │ IOCTL
┌─────────────────────────────────────┼───────────┐
│              Userspace Daemon (ksud)             │
│                    (Rust)                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ Modules  │  │ SEPolicy │  │  Boot Patch   │  │
│  │ Manager  │  │ Patcher  │  │   Engine      │  │
│  └──────────┘  └──────────┘  └──────────────┘  │
└─────────────────────────────────────┼───────────┘
                                      │ IOCTL (anon inode FD)
┌─────────────────────────────────────┼───────────┐
│              Kernel Module (LKM)                │
│                    (C)                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ Syscall  │  │ Allowlist│  │   SELinux     │  │
│  │  Hooks   │  │ Manager  │  │   Hooks       │  │
│  └──────────┘  └──────────┘  └──────────────┘  │
└─────────────────────────────────────────────────┘
```

Communication between layers uses IOCTLs on an anonymous inode file descriptor,
delivered via a reboot syscall kprobe hook (magic values `0xDEADBEEF`, `0xCAFEBABE`).

## The Rust-First Law

**All new userspace code must be written in Rust.**

- No new `.c` files outside `kernel/`
- No new Python or shell scripts unless interfacing with something that cannot be Rust
- New kernel-space code may use C only when required by the Linux kernel API

## Development Setup

### Required Tools

| Tool | Version | Purpose |
|------|---------|---------|
| Rust | 1.93+ | Userspace daemon and JNI bridge |
| Android NDK | 29.0.13846066 | Cross-compilation for Android |
| Android SDK | API 36 | Manager app build |
| Java | OpenJDK 21 | Gradle build |
| cargo-ndk | latest | `cargo install cargo-ndk` |

### Building

#### Userspace Daemon (ksud)

```bash
cd userspace/ksud
cargo ndk -t arm64-v8a build --release
```

#### JNI Bridge

```bash
cd manager/app/src/main/rust/kernelsu-jni
cargo ndk -t arm64-v8a build --release
```

#### Manager APK

The APK requires native libraries to be present before building:

```bash
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

### Running Tests

```bash
# Rust unit tests (host)
cd userspace/ksud
cargo test

# Rust linting
cargo ndk -t arm64-v8a clippy -- -D warnings

# Rust formatting
cargo fmt --check
```

## Code Standards

### Rust

- Edition: 2024 (ksud), 2021 (JNI bridge)
- Error handling: `anyhow` for applications, `thiserror` for libraries
- No `unwrap()` in library code — use `?` operator or proper error handling
- No `expect()` without a good reason documented in the message
- Logging: `log` crate (not `println!` or `eprintln!`)
- Zero clippy warnings (`clippy::all` + `clippy::pedantic` enabled)
- Every `unsafe` block must have a `// SAFETY:` comment explaining why it is sound

### Kotlin

- Kotlin files should only contain UI code (Composables, ViewModels, Activities)
- Business logic should be extracted to Rust and called via JNI
- Every Kotlin file touched must end up smaller or equal in size

### C (kernel-space only)

- All kernel C files must have SPDX license headers
- Check `kmalloc`/`kzalloc` return values for NULL
- Use `atomic_t` for shared state, not plain `bool`/`int`
- Every early return must clean up allocated resources

### Commit Messages

Format: `<scope>: <summary>`

Scopes: `kernel`, `ksud`, `manager`, `meta-overlayfs`, `docs`, `scripts`

Examples:
```
ksud: add module validation before installation
kernel: fix NULL dereference in throne_tracker
manager: update superuser screen layout
docs: add contributing guidelines
```

Keep subject lines under 72 characters. Use sentence case. No trailing period.

## Adding a New Feature

### Checklist

- [ ] Write the implementation in Rust (userspace) or C (kernel-only)
- [ ] Add unit tests
- [ ] Run `cargo ndk -t arm64-v8a clippy -- -D warnings` — zero warnings
- [ ] Run `cargo fmt` — properly formatted
- [ ] Run `cargo test` — all tests pass
- [ ] Add `///` doc comments for public functions
- [ ] Add `// SAFETY:` comment for any `unsafe` block
- [ ] Full APK build succeeds

### Where to Add Code

| Feature Type | Location | Language |
|-------------|----------|----------|
| Kernel hooks | `kernel/` | C |
| Userspace logic | `userspace/ksud/src/` | Rust |
| Android UI | `manager/app/src/main/java/` | Kotlin |
| JNI bridge | `manager/app/src/main/rust/` | Rust |
| Build scripts | `scripts/` | Rust (preferred) or Python |

## Security Policy

See [SECURITY.md](SECURITY.md) for vulnerability reporting.

### In Scope for Security Review

- Kernel module (all files in `kernel/`)
- IOCTL interface (`supercalls.c`, `ksucalls.rs`)
- Allowlist persistence (`allowlist.c`)
- Root grant flow (`setuid_hook.c`, `su.rs`)
- SELinux policy injection (`selinux/`)
- Module installation and validation (`module.rs`, `module_validator.rs`)

### Known Limitations

- Kernel module requires matching kernel version for kprobe hooks
- SELinux policy changes are applied at boot and cannot be dynamically reverted
- Module overlay uses bind mounts which may be detectable by some apps
