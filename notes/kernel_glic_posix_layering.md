# Kernel / glibc / POSIX Layering — Notes

## Why the glibc manual is the reference for learning signals (or POSIX topics) on Linux

- **POSIX is a spec, not code.** It defines rules (e.g., "SIGKILL must be uncatchable," "a `sigaction()` function must exist") but provides no implementation and no in-depth behavioral documentation.
- **glibc is Linux's concrete implementation of that spec.** It implements the standard C library functions (`printf`, `malloc`, `open`) plus the POSIX system call wrappers (`signal()`, `sigaction()`, `kill()`, `sigprocmask()`), sitting between user programs and the kernel's raw syscalls.
- There is no separate "Linux signals" manual — the kernel's own documentation is sparse and low-level (source comments, terse man pages). glibc's manual is the most complete, accurate writeup of how signals actually behave on Linux, because glibc is the layer that implements them.
- **Key implication:** reading the glibc manual on signals *is* reading the practical, working documentation of POSIX signal behavior as implemented on Linux. This holds regardless of what language a program is written in (C, Python, Bash) — all of them ultimately go through libc (or the equivalent syscall) to install handlers, block signals, or send them.

## The three-layer model

1. **Kernel** — does the real work: delivers signals, tracks pending/blocked masks, manages the filesystem, drivers, scheduling, memory. Exposes this only through raw, low-level syscalls (not friendly APIs).
2. **glibc** — wraps those raw syscalls into the standardized C functions defined by POSIX. It's not a generic "helper" — it's the concrete realization of the POSIX contract on top of a specific kernel.
3. **Programs (bash, Python, etc.)** — call into glibc's API. E.g., `trap "echo caught TERM" TERM` in bash → bash calls `sigaction()` (a glibc function) → glibc calls into the kernel → kernel tracks/delivers the signal → glibc's machinery invokes the handler.

## Origin of glibc

- Developed by the **GNU Project** (Richard Stallman / FSF), started mid-1980s, as part of building a complete free Unix-like OS ("GNU's Not Unix").
- Needed because existing C libraries were tied to proprietary Unix vendors with licensing restrictions; GNU needed a freely licensed C library since virtually everything (compiler, utilities, any C program) depends on one.
- GNU had built most of the OS (GCC, bash, coreutils, glibc) by the early 1990s but lacked a kernel. Linus Torvalds released the Linux kernel in 1991. Linux kernel + GNU userland (incl. glibc) = what's commonly called "Linux" (more precisely GNU/Linux). glibc became the default C library on essentially all major Linux distributions.

## Why the kernel is large and constantly changing, but glibc isn't

**Kernel:**
- The *conceptual categories* are a short, stable list: process scheduling, memory management, filesystem layer (VFS), networking stack, device drivers, IPC, security modules, syscall interface.
- The size/churn comes from the *combinatorial* explosion **within** each category — especially device drivers, which make up roughly 60–70% of kernel source. Every piece of hardware (network card, GPU, storage controller, USB device, sensor) needs its own driver, and new hardware ships constantly.
- Other growing areas: filesystems (ext4, btrfs, XFS, NFS, overlayfs...), networking protocols, virtualization/containers (KVM, namespaces, cgroups), new CPU architectures.
- Result: short conceptual list, but code volume keeps expanding because hardware and use cases keep expanding. New kernel release roughly every 9–10 weeks, thousands of commits per release, mostly driver/hardware support rather than core logic changes.

**glibc:**
- Implements a spec (POSIX/ISO C) — specs change slowly, unlike hardware.
- Extreme backward-compatibility requirements (symbol versioning keeps 20-year-old binaries running unmodified) — maintainers are structurally conservative; breaking changes are near-unacceptable.
- No hardware-driver-style bottomless growth category — string functions, malloc, pthreads, locale data don't multiply the way hardware does.
- Much smaller, tightly-controlled maintainer group compared to the kernel's thousands of contributors (many corporate, each pushing their own hardware support).

## Why new hardware requires new kernel code

- Every device has its own protocol for how the OS configures it, sends commands, and reads data (register layouts, command sets, interrupt behavior, timing). The kernel doesn't inherently know this.
- This applies even to newer/faster versions of existing hardware categories, not just entirely new categories: e.g., a new NVMe SSD controller adds a new command (new power-management state, new error reporting) that an existing driver doesn't recognize until code is added to use it.
- Without added support, older/basic commands still work (usually standardized/backward-compatible), but new capabilities aren't available until someone writes the code.
- **Who writes it:** largely hardware vendors themselves (Intel, AMD, NVIDIA, Qualcomm, Realtek, etc.), who employ engineers to write and upstream Linux drivers — self-interest, since unsupported hardware under Linux loses sales across servers, cloud, Android, embedded systems. Cooperation varies by vendor; some drivers are community-reverse-engineered when vendors are uncooperative. All driver code goes through public kernel review (mailing lists, maintainers) before merging.
- **Backward compatibility** (old hardware still working on new kernels) is a separate concern from new-hardware support — maintained by keeping old driver code in-tree and keeping the syscall/ABI stable, not by manufacturers doing ongoing work on old devices.

## Kernel's internal structure: not one giant if/else

- The kernel is **not** a large branching chain across core logic (`if device == X ... else if device == Y ...` scattered everywhere).
- It uses a **driver model**: each device driver is a self-contained module registering with the kernel and exposing a standard set of function pointers (open, read, write, ioctl, etc.).
- At boot/hotplug time, the kernel matches a physical device (by vendor ID / device ID) to the correct driver; from then on it calls that driver's functions through a common interface.
- Branching/complexity is concentrated in:
  - **Device matching** (vendor/device ID tables — genuinely large lookup tables).
  - **Inside each individual driver**, for that driver's own hardware quirks/revisions.
- Core subsystems (scheduler, VFS, memory manager) stay generic and largely branch-free — they call into whichever driver was matched, rather than branching on device type themselves. This containment is the actual purpose of the driver abstraction.


```
// ===== LAYER 1: Your program =====
main() {
    kill(pid=1234, sig=SIGTERM)      // just looks like a normal C function call
}


// ===== LAYER 2: glibc (libc.so) — this is what "kill" actually is =====
function kill(pid, sig) {
    // translate the friendly call into what the kernel understands
    syscall_number = 62              // "62" means "kill" to the kernel, on this CPU arch
    register[RAX] = syscall_number
    register[RDI] = pid
    register[RSI] = sig

    trigger_cpu_instruction("syscall")   // <-- this is the actual hop into kernel mode
                                          //     CPU privilege level switches: user -> kernel

    result = register[RAX]           // kernel puts its return value back here
    if (result < 0) {
        errno = -result               // glibc translates raw error code into errno
        return -1
    }
    return result
}


// ===== LAYER 3: Kernel — runs after the "syscall" instruction traps in =====
function handle_syscall(number, arg1, arg2) {
    switch (number) {
        case 62:                      // matched by number, not by name "kill"
            return sys_kill(arg1, arg2)
        // ... hundreds of other numbered cases ...
    }
}

function sys_kill(target_pid, sig) {
    target_process = find_process(target_pid)
    target_process.pending_signals.add(sig)   // just flips a bit — doesn't run anything yet
    return 0
}


// ===== LATER: kernel delivers the signal, asynchronously =====
// next time target_process is scheduled to run:
function before_resuming_process(proc) {
    if (proc.pending_signals.has(SIGTERM)) {
        if (proc.has_custom_handler(SIGTERM)) {
            jump_to_user_handler(proc, SIGTERM)   // hands control back to userspace code
        } else {
            default_action(SIGTERM)   // e.g. terminate the process
        }
    }
}
```