# KernelSU Security Audit Report

## Summary
- Total findings: 22
- Critical: 1 | High: 4 | Medium: 7 | Low: 5 | Informational: 5

---

## Critical Findings

### C-1: Integer overflow in `do_new_get_allow_list_common` allocation size (supercalls.c:166)

**File:** `kernel/supercalls.c:166`

**Description:** The `cmd.count` field is a user-controlled `u16` read via `copy_from_user`. The allocation `sizeof(int) * cmd.count` performs an implicit integer promotion. While a `u16` max of 65535 × 4 = 262140 bytes is within kernel allocation limits and won't overflow a `size_t`, the real issue is that `cmd.count` is trusted without validation against the actual allow-list length. The kernel allocates a buffer of user-specified size, then `ksu_get_allow_list()` writes into it bounded by the caller-supplied `cmd.count`. If a caller supplies a very large `cmd.count`, the kernel allocates a large buffer unnecessarily, but the data copied back via `copy_to_user` at line 187–188 uses the *updated* `cmd.count` (output from `ksu_get_allow_list`), which may differ from the originally allocated size. This is correct in practice because `ksu_get_allow_list` clamps `j < length`, but the pattern is fragile.

However, more critically, at line 187–188:
```c
copy_to_user(&((struct ksu_new_get_allow_list_cmd *)arg)->uids, arr,
             sizeof(int) * cmd.count))
```
The `cmd.count` here is the *output* count from `ksu_get_allow_list`, which is bounded by the input count (clamped via `j < length`). But the `arg` pointer is cast to `struct ksu_new_get_allow_list_cmd *` which has `uids[0]` (flexible array member). The copy writes to userspace past the end of the fixed struct fields. If the user didn't allocate enough space in their buffer, this writes to arbitrary user memory — but that's the user's own address space, so this is by design for flexible array members.

**Severity:** Critical (downgraded to Medium on further analysis — see note below)

**Note:** On re-analysis, this is actually safe because: (1) the allocation is bounded by u16 max × 4 = ~256KB, (2) `ksu_get_allow_list` clamps the output, and (3) writing to user-space with `copy_to_user` cannot corrupt kernel memory. Reclassifying as **Medium** — the risk is a DoS via large allocation from manager/root processes only.

**Exploitability:** Low — requires `manager_or_root` permission. An attacker with manager access could cause kernel to allocate up to 256KB temporarily.

**Suggested fix:**
```c
if (cmd.count > 4096) { // reasonable upper bound
    return -EINVAL;
}
```

---

## High Findings

### H-1: Race condition on `post_fs_data_lock` / `boot_complete_lock` static bools (supercalls.c:97–113)

**File:** `kernel/supercalls.c:97-113`

**Description:** The `do_report_event` handler uses `static bool` variables (`post_fs_data_lock`, `boot_complete_lock`) without any synchronization primitive. These are read-then-write patterns:
```c
static bool post_fs_data_lock = false;
if (!post_fs_data_lock) {
    post_fs_data_lock = true;
    on_post_fs_data();
}
```
If two threads call `do_report_event(EVENT_POST_FS_DATA)` simultaneously, both could see `false`, enter the block, and call `on_post_fs_data()` twice. This is a classic TOCTOU race.

**Severity:** High — `on_post_fs_data()` loads the allowlist and initializes the observer. Double-initialization could corrupt internal state.

**Exploitability:** Medium — requires root permission (`only_root` check), and the window is narrow during boot. But if ksud is compromised or buggy, it could trigger this.

**Suggested fix:**
```c
static atomic_t post_fs_data_lock = ATOMIC_INIT(0);
if (atomic_cmpxchg(&post_fs_data_lock, 0, 1) == 0) {
    on_post_fs_data();
}
```

### H-2: Race condition on `ksu_rc_pos` and global `fops_proxy` in init.rc hook (ksud.c:280,413)

**File:** `kernel/ksud.c:280,413-423`

**Description:** The `ksu_rc_pos` global variable and `fops_proxy` struct are accessed without locking:
- `ksu_rc_pos` is read/written in `read_proxy` and `read_iter_proxy` without synchronization
- `fops_proxy` is populated in `ksu_handle_sys_read` and the `orig_read`/`orig_read_iter` function pointers are globals
- The `static bool rc_hooked` flag has the same TOCTOU issue as H-1

In practice, this code runs only during early init from a single process (`init` reading `init.rc`), so the race window is extremely narrow.

**Severity:** High (theoretical) — could cause init.rc injection to be applied partially or duplicated if somehow triggered from multiple threads.

**Exploitability:** Very Low — init is single-threaded during this phase. The code self-unregisters the kprobe after first use.

**Suggested fix:** Use `atomic_cmpxchg` for the `rc_hooked` flag to guarantee at-most-once semantics.

### H-3: `strncpy` without guaranteed NUL-termination (throne_tracker.c:305)

**File:** `kernel/throne_tracker.c:305`

**Description:**
```c
strncpy(data->package, package, KSU_MAX_PACKAGE_NAME);
```
`strncpy` does **not** guarantee NUL-termination if `package` is ≥ `KSU_MAX_PACKAGE_NAME` bytes. Later code uses `strncmp(np->package, package, KSU_MAX_PACKAGE_NAME)` which is bounded but `pr_info("prune uid: %d, package: %s\n", uid, package)` and similar `%s` uses assume NUL-termination.

**Severity:** High — potential kernel info-leak or crash via `%s` format reading past buffer.

**Exploitability:** Medium — `package` comes from parsing `/data/system/packages.list` using `strsep`, which produces NUL-terminated tokens. But if the file is corrupted or crafted, a line without proper delimiters could yield a token exactly `KSU_MAX_PACKAGE_NAME` bytes without NUL.

**Suggested fix:**
```c
strscpy(data->package, package, KSU_MAX_PACKAGE_NAME);
```
(Note: `strscpy` always NUL-terminates and is preferred in modern kernels.)

### H-4: Unchecked `kernel_read` return values in APK signature verification (apk_sign.c:76-89)

**File:** `kernel/apk_sign.c:76-89`

**Description:** In `check_block()`, multiple `kernel_read` calls are made without checking their return values:
```c
kernel_read(fp, size4, 0x4, pos); // no return check
kernel_read(fp, size4, 0x4, pos); // no return check
kernel_read(fp, size4, 0x4, pos); // no return check
```
If any of these reads fail (truncated file, I/O error), `size4` retains its previous value, leading to use of stale/uninitialized data for subsequent offset calculations and potential out-of-bounds reads.

Similarly in `check_v2_signature()` at lines 196-238, multiple `kernel_read` calls go unchecked.

**Severity:** High — this is the APK signature verification path that determines whether an app becomes the KernelSU manager. A malformed APK could cause incorrect verification results.

**Exploitability:** Medium — requires placing a crafted APK on the device. The code has other guards (size comparisons, hash matching), but the unchecked reads could lead to bypassing verification in edge cases.

**Suggested fix:** Check every `kernel_read` return value and bail on short reads.

---

## Medium Findings

### M-1: `strcpy` usage with compile-time constant strings (allowlist.c:69,110-111)

**File:** `kernel/allowlist.c:69,110-111`

**Description:** Three uses of `strcpy` exist:
```c
strcpy(default_root_profile.selinux_domain, KSU_DEFAULT_SELINUX_DOMAIN);
strcpy(profile.key, "com.android.shell");
strcpy(profile.rp_config.profile.selinux_domain, KSU_DEFAULT_SELINUX_DOMAIN);
```
All source strings are compile-time constants that fit within their destination buffers (`KSU_SELINUX_DOMAIN` = 64 bytes, `KSU_MAX_PACKAGE_NAME` = 256 bytes). There is no overflow risk in practice.

**Severity:** Medium — no actual vulnerability, but `strcpy` in kernel code is a code smell and can become a problem if constants change.

**Suggested fix:** Replace with `strscpy(dst, src, sizeof(dst))` for defense-in-depth.

### M-2: Integer cast `u32 as usize` in module_config deserialization (module_config.rs:161,174)

**File:** `userspace/ksud/src/module_config.rs:161,174`

**Description:**
```rust
let key_len = u32::from_le_bytes(key_len_buf) as usize;
let mut key_buf = vec![0u8; key_len];
```
A malformed config file could specify `key_len` up to `u32::MAX` (4GB), causing an allocation attempt that would either OOM-kill the process or panic. There's no upper bound validation on `key_len` or `value_len`.

**Severity:** Medium — ksud runs as root; a corrupted config file in `/data/adb/` could cause the ksud daemon to crash.

**Exploitability:** Low — requires write access to `/data/adb/ksu/` which is root-only.

**Suggested fix:**
```rust
const MAX_ENTRY_LEN: usize = 64 * 1024; // 64KB reasonable max
let key_len = u32::from_le_bytes(key_len_buf) as usize;
if key_len > MAX_ENTRY_LEN {
    bail!("key length {key_len} exceeds maximum");
}
```

### M-3: Potential allocation size from user-controlled count (supercalls.c:166)

**File:** `kernel/supercalls.c:166`

**Description:** (Reclassified from C-1.) The `cmd.count` from userspace directly controls the kernel allocation size: `kmalloc(sizeof(int) * cmd.count, GFP_KERNEL)`. While bounded by `u16` (max 256KB), there is no tighter validation. A manager/root process could request allocations that, under memory pressure, could cause OOM or stall.

**Severity:** Medium — requires manager_or_root permissions, bounded by u16.

**Suggested fix:** Add a reasonable upper limit check.

### M-4: `unwrap()` calls in non-test Rust code (metamodule.rs:228-229,252, module.rs:218, utils.rs:126)

**File:** `userspace/ksud/src/metamodule.rs:228-229,252`, `userspace/ksud/src/module.rs:218`, `userspace/ksud/src/utils.rs:126`

**Description:** Several `.unwrap()` calls in production code:
- `metauninstall_path.to_str().unwrap()` — panics on non-UTF8 paths
- `metauninstall_path.parent().unwrap()` — panics if path has no parent
- `path.as_ref().parent().unwrap()` — same
- `zip.by_index(i).unwrap().size()` — panics on invalid zip index

While these are unlikely to fail in normal operation (Android paths are UTF-8, paths always have parents in practice), a panic in ksud could prevent module operations.

**Severity:** Medium — ksud crash during module installation/uninstallation could leave the system in an inconsistent state.

**Suggested fix:** Replace with `.context("...")?` or `.unwrap_or_default()` as appropriate.

### M-5: `unsafe { env::set_var(...) }` usage (su.rs:296)

**File:** `userspace/ksud/src/su.rs:296`

**Description:** `env::set_var` is marked unsafe in Rust 2024 edition because it is not thread-safe. If `ksud` ever becomes multi-threaded, this could cause data races on the environment block.

**Severity:** Medium — currently ksud appears single-threaded in the su path, so not exploitable now.

**Suggested fix:** Accept this if single-threaded invariant is maintained. Document the requirement.

### M-6: `do_report_event` EVENT_MODULE_MOUNTED has no once-guard (supercalls.c:114-118)

**File:** `kernel/supercalls.c:114-118`

**Description:** Unlike `EVENT_POST_FS_DATA` and `EVENT_BOOT_COMPLETED`, the `EVENT_MODULE_MOUNTED` case has no lock/guard:
```c
case EVENT_MODULE_MOUNTED: {
    pr_info("module mounted!\n");
    on_module_mounted();
    break;
}
```
`on_module_mounted()` sets `ksu_module_mounted = true`. This is idempotent (writing `true` multiple times is harmless), but the pattern inconsistency could mask issues if the handler gains side effects in the future.

**Severity:** Medium — no current vulnerability, but inconsistent pattern is a maintenance risk.

### M-7: Preempt toggle in execve su-compat path (sucompat.c:143-149)

**File:** `kernel/sucompat.c:143-149`

**Description:**
```c
if (ret < 0 && preempt_count()) {
    preempt_enable_no_resched_notrace();
    ret = strncpy_from_user(path, fn, sizeof(path));
    preempt_disable_notrace();
}
```
The code explicitly toggles preemption to handle page faults during `strncpy_from_user`. While the comment acknowledges this is "crazy", it could theoretically cause issues: if a scheduler event occurs during the enabled window, the kprobe handler could be rescheduled, and the subsequent `preempt_disable` might not properly restore the preempt count if nested.

**Severity:** Medium — this is a last-resort fallback that handles an edge case. The code is well-commented and seems intentional.

**Exploitability:** Very Low — requires a specific kernel configuration where the filename page is not resident during the kprobe handler.

---

## Low Findings

### L-1: Missing `app_profile.key` NUL-termination enforcement on load (allowlist.c:488-501)

**File:** `kernel/allowlist.c:488-501`

**Description:** When loading profiles from the `.allowlist` file, the code reads `sizeof(profile)` bytes directly with `kernel_read`. The `profile.key` field (256 bytes) is not explicitly NUL-terminated after read. If the on-disk file is corrupted and `key` has no NUL byte, string operations on it (e.g., `strcmp` in `ksu_set_app_profile`) could read out of bounds within the struct (bounded by the struct size, not unbounded).

**Severity:** Low — the struct is stack-allocated at 256 bytes, so reads are bounded. But `pr_info("load_allow_uid, name: %s", profile.key)` could print garbage.

**Suggested fix:** Add `profile.key[KSU_MAX_PACKAGE_NAME - 1] = '\0';` after loading.

### L-2: `allowlist` file permissions are world-readable (allowlist.c:396)

**File:** `kernel/allowlist.c:396`

**Description:** The allowlist file is created with mode `0644`:
```c
filp_open(KERNEL_SU_ALLOWLIST, O_WRONLY | O_CREAT | O_TRUNC, 0644);
```
This means any process on the device can read the list of apps granted root access. While the data isn't secret (it's UIDs and package names), exposing it could help attackers enumerate targets.

**Severity:** Low — the file is in `/data/adb/ksu/` which has restricted directory permissions.

**Suggested fix:** Use `0600` for the file permissions.

### L-3: No bounds check on `cert_len` before stack allocation (apk_sign.c:94-99)

**File:** `kernel/apk_sign.c:94-99`

**Description:**
```c
#define CERT_MAX_LENGTH 1024
char cert[CERT_MAX_LENGTH];
if (*size4 > CERT_MAX_LENGTH) {
    return false;
}
kernel_read(fp, cert, *size4, pos);
```
The 1024-byte `cert` buffer is allocated on the kernel stack. While there is a bounds check, 1KB on the kernel stack is large. Combined with other stack frames, this could approach the 8KB kernel stack limit on some configurations.

**Severity:** Low — the check prevents overflow, but the stack usage is high.

**Suggested fix:** Consider using `kmalloc` for the cert buffer or reducing `CERT_MAX_LENGTH`.

### L-4: `unsafe` blocks in ksud su.rs for libc FFI calls (su.rs:28,63,67,68,72,216,236,239,240,264,296)

**File:** `userspace/ksud/src/su.rs`

**Description:** Multiple `unsafe` blocks for libc interop (`isatty`, `dup2`, `close`, `getpwuid`, `getpwnam`, `CStr::from_ptr`). Each is necessary for FFI and follows standard patterns. The `CStr::from_ptr` calls at lines 239-240 assume the libc `getpwuid` result has valid NUL-terminated strings, which is guaranteed by POSIX but not by Rust's type system.

**Severity:** Low — all unsafe usage is justified and follows idiomatic Rust FFI patterns.

### L-5: `getpwuid` is not thread-safe (JNI bridge lib.rs:641, su.rs:236)

**File:** `manager/app/src/main/rust/kernelsu-jni/src/lib.rs:641`, `userspace/ksud/src/su.rs:236`

**Description:** Both the JNI bridge and ksud use `libc::getpwuid()` which returns a pointer to a static buffer. If called from multiple threads (e.g., in the manager app), the returned data could be corrupted.

**Severity:** Low — the JNI bridge comment acknowledges "single-threaded JNI call" assumption. In Android, JNI calls from the same native method are typically serialized by the Java thread.

---

## Informational

### I-1: JNI bridge struct layouts match kernel definitions

**File:** `manager/app/src/main/rust/kernelsu-jni/src/lib.rs`

**Description:** The JNI bridge's `#[repr(C)]` struct definitions (`AppProfile`, `RootProfile`, `Capabilities`, etc.) must exactly match the kernel's C struct layouts. The current definitions appear correct:
- Field types and sizes match (`i32` ↔ `int32_t`, `u64` ↔ `u64`, etc.)
- Array sizes match (`KSU_MAX_GROUPS = 32`, `KSU_MAX_PACKAGE_NAME = 256`, `KSU_SELINUX_DOMAIN = 64`)
- The union handling via `nrp_config()` is sound (documented SAFETY comment, `NrpConfig` is smaller than `RpConfig`)

No issues found. The `BUILD_BUG_ON(sizeof(profile.capabilities.effective) != sizeof(kernel_cap_t))` check in `app_profile.c:137` adds compile-time verification.

### I-2: check_symbol_rs tool has no unsafe code

**File:** `kernel/tools/check_symbol_rs/src/main.rs`

**Description:** The check_symbol_rs tool:
- Contains zero `unsafe` blocks
- Reads ELF files entirely into memory with `fs::read()` (safe, no mmap)
- Uses the `object` crate for ELF parsing, which handles malformed/truncated files gracefully via `Result` types
- All error paths return proper `Result<>` with context
- No command injection, no path traversal risks

No security issues found.

### I-3: Module ID validation prevents path traversal

**File:** `userspace/ksud/src/module.rs:48-57`

**Description:** Module IDs are validated against `^[a-zA-Z][a-zA-Z0-9._-]+$`, which prevents:
- Path traversal (`../` characters are rejected)
- Shell metacharacters (`;`, `|`, `&`, etc.)
- Empty or single-character IDs

This is a strong defense against directory traversal and command injection when module IDs are used in path construction (e.g., `/data/adb/modules/{id}/`).

### I-4: Icon path traversal protection in module listing

**File:** `userspace/ksud/src/module.rs:666-711`

**Description:** The `resolve_module_icon_path` function explicitly checks for and rejects:
- Absolute paths (line 676-684)
- Parent directory traversal (`..` components, line 685-696)

This prevents a malicious module from referencing arbitrary files via `actionIcon` or `webuiIcon` properties.

### I-5: IOCTL permission model is well-structured

**File:** `kernel/supercalls.c:615-696`

**Description:** The IOCTL handler table implements a clean permission model:
- `always_allow`: GET_INFO, CHECK_SAFEMODE (read-only, non-sensitive)
- `allowed_for_su`: GRANT_ROOT (manager or allow-listed UIDs)
- `only_root`: REPORT_EVENT, SET_SEPOLICY (boot-time operations)
- `only_manager`: GET/SET_APP_PROFILE (sensitive configuration)
- `manager_or_root`: all other operations

Permission checks run before handlers, preventing unauthorized access. The table is terminated by a sentinel entry.

---

## Positive Findings

1. **RCU-protected allowlist reads**: The allowlist uses `rcu_read_lock`/`list_for_each_entry_rcu` for lock-free reads and `mutex` + `list_replace_rcu`/`kfree_rcu` for safe updates. This is correct kernel RCU usage.

2. **Consistent NULL-checking of `kmalloc`/`kzalloc` returns**: Every allocation in the codebase is NULL-checked with proper error handling (`-ENOMEM` return or graceful fallback).

3. **Proper `copy_from_user`/`copy_to_user` error handling**: All copy operations check return values and return `-EFAULT` on failure.

4. **`strncpy_from_user` with size limits**: Userspace string copies use bounded variants (`strncpy_from_user`, `strncpy_from_user_nofault`) with explicit size limits, and the `do_nuke_ext4_sysfs` handler adds a specific `ENAMETOOLONG` check.

5. **Mount list protected by rwsem**: The `mount_list` in supercalls.c is protected by `mount_list_lock` (a `DECLARE_RWSEM`), with `down_write` for mutations and `down_read` for iterations. The duplicate check prevents unbounded list growth.

6. **Module script execution uses busybox, not shell interpolation**: Module scripts are executed via `Command::new(BUSYBOX_PATH).args(["sh", path])`, not via shell string interpolation, preventing command injection.

7. **Rust error handling via `anyhow::Result`**: The ksud codebase consistently uses `Result<>` with `?` operator and `.context()` for error propagation, avoiding silent failures.

8. **SELinux context enforcement**: The kernel module enforces SELinux domain transitions (`setup_selinux`, `is_ksu_domain`, `is_zygote`) for privilege boundaries, and the file wrapper uses `ksu_file_sid` to maintain proper security labeling.

9. **Kernel stack size is managed**: The `userspace_stack_buffer` function in sucompat.c writes below the userspace stack pointer for temporary path storage, avoiding kernel stack bloat for path manipulation.

10. **APK v1 signature check as defense-in-depth**: The APK verification (`is_manager_apk`) not only validates v2 signatures but also checks that no v1 or v3 signatures exist, preventing downgrade attacks.
