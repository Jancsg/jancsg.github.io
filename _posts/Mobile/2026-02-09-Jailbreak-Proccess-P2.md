---
title: "Kernel Payloads & Patches - Breaking the Jail"
date: 2026-02-09 00:00:00
classes: wide
header:
  teaser: /assets/images/Jailbreak-Proccess-P1/1.jpg
  overlay_image: /assets/images/Jailbreak-Proccess-P1/1.jpg
  overlay_filter: 0.5
ribbon: DarkSlateGray
excerpt: "What checkra1n's KPF actually patches — based on the real PongoOS source code"
description: "A source-code-level analysis of checkra1n's Kernel Patch Finder: AMFI trustcache bypass, rootfs remount, W^X removal, sandbox shellcode, and tfp0 access — all in AArch64"
categories:
  - Forensics
  - Mobile
tags:
  - iOS
  - Jailbreak
  - Kernel
  - Checkra1n
  - KPF
  - AArch64
  - AMFI
  - Sandbox
  - PongoOS
toc: true
toc_sticky: true
toc_label: "On This Blog"
toc_icon: "biohazard"
---

# Introduction

In [the introduction](/forensics/mobile/iOS-Jailbreak/) we mapped out the jailbreak landscape. In [Part 1](/forensics/mobile/Jailbreak-Proccess-P1/) we tore apart checkra1n's chain — from checkm8's use-after-free in SecureROM, through PongoOS, to the moment the Kernel Patch Finder (KPF) runs.

We stopped right at the edge: KPF scans the XNU kernel and applies surgical patches. But what does it *actually* patch?

I'm going to be transparent here: most jailbreak documentation online (including older versions of this post) describes the **comex-era datautils0 patches** — ARM32 Thumb byte patterns from the redsn0w days targeting iOS 4-5 on A4/A5 chips. Those are historically important, but they're **not what checkra1n does**. checkra1n targets A7-A11 devices (all **AArch64/ARM64**), and its Kernel Patch Finder uses a completely different approach.

Everything in this post comes from reading the actual [PongoOS KPF source code](https://github.com/checkra1n/PongoOS/blob/master/checkra1n/kpf/main.c). No guessing, no relabeling old patches with new names.

# How KPF Works: Mask-Based Pattern Matching

Before we look at individual patches, let's understand KPF's engine. Unlike the old static byte-pattern scanning, KPF uses a function called `xnu_pf_maskmatch()` that takes:

- An **array of instruction patterns** (what the target code looks like)
- An **array of masks** (which bits matter and which can vary)
- A **callback function** (what to do when a match is found)

This is critical because Apple's compiler produces different register allocations, instruction ordering, and offsets across iOS versions and chip architectures. The masks let KPF match the *structure* of the code while ignoring the parts that change.

For example, to match any `ldr w*, [x*, #offset]` instruction, KPF uses:

```
pattern: 0xB9400000
mask:    0xFFC00000
```

The mask says "I care about the top 10 bits (which identify this as an LDR with a specific addressing mode) but not the bottom 22 (which encode the registers and offset)."

Every patch below uses this mechanism. Let's go through them.

# Patch 1: AMFI Trustcache Bypass

**What it targets**: Apple Mobile File Integrity (AMFI) maintains a **trust cache** — a built-in list of SHA-256 hashes of Apple binaries. When a binary is about to execute, AMFI checks if its hash is in the trust cache. If it is, the binary is considered trusted and signature validation is skipped.

**What KPF does**: It finds the trust cache lookup function and patches it to **always return 1** (trusted). This makes AMFI believe every binary on the system is in the trust cache, effectively disabling code signature validation.

**How it finds it**: KPF searches for this AArch64 instruction pattern — the inner loop of the trustcache binary search:

```asm
ADD  X9, X9, #0x18      ; entry size (24 bytes per hash entry)
MOVZ W10, #0x16          ; constant used in address calculation
LSR  X11, X8, #1         ; divide index by 2 (binary search)
MADD X12, X11, X10, X9   ; calculate entry address
```

In `xnu_pf_maskmatch` terms:

```c
uint64_t matches[] = {
    0x91000000, // ADD x*
    0x52800200, // MOVZ w*, #0x16
    0xD3000000, // LSR *
    0x9B000000  // MADD *
};
uint64_t masks[] = {
    0xFF000000,
    0xFFFFFF00,
    0xFF000000,
    0xFF000000
};
```

**The callback** (`kpf_amfi_callback`) then determines if it found a leaf function or a full routine:

- **Leaf function** (no stack frame): Overwrite the first two instructions with `MOVZ X0, #1; RET` — function returns 1 immediately.
- **Full routine** (has prolog/epilog): Scan backward for `STP`/`LDP` instructions indicating a stack frame, then find all `MOV X0, X*` or `MOV W0, #0` instructions near the return point and replace them with `MOVZ X0, #1`.

```c
// Leaf version — simple override
opcode_stream[0] = 0xD2800020; // MOVZ X0, #1
opcode_stream[1] = 0xD65F03C0; // RET

// Routine version — patch the return value
patchpoint[0] = 0xD2800020;    // MOVZ X0, #1 (replaces MOV X0, X19 or MOV W0, #0)
```

This is the single most important patch — without it, no unsigned binary can execute.

# Patch 2: Rootfs Remount (mac_mount)

**What it targets**: On stock iOS, the root filesystem is mounted read-only, and `mount()` syscall checks prevent remounting it as read-write or creating union mounts. Two specific flags are enforced:
- **`MNT_ROOTFS`** (bit 6 at offset 0x71): Prevents modification operations on the root filesystem.
- **`MNT_UNION`** (bit 5): Prevents union mounts entirely.

**What KPF does**: Two sub-patches:

1. **NOP out the `MNT_UNION` check**: Finds a `TBNZ W*, #5, *` (test bit 5, branch if not zero) and replaces it with `NOP`.
2. **Bypass the `MNT_ROOTFS` check**: Finds `LDRB W8, [X8, #0x71]` (load the flags byte) and replaces it with `MOV X8, XZR` (load zero instead). Now the subsequent bit-test always reads 0.

**How it finds it**: KPF searches for the unique constant `0x1FFE` which appears in `mac_mount`:

```c
uint64_t matches[] = { 0x321F2FE9 }; // ORR W9, WZR, #0x1FFE
// or alternatively:
uint64_t matches[] = { 0x5283FFC9 }; // MOVZ W9, #0x1FFE
```

This constant is distinctive enough that it only matches inside `mac_mount`. From there, the callback scans nearby for the TBNZ and LDRB instructions to patch.

**Why this matters**: Without this patch, you can't remount `/` as read-write. No file modifications to the root filesystem. No installing Cydia, no SSH, nothing.

# Patch 3: Unmounting the Rootfs (dounmount)

**What it targets**: Even after remounting the rootfs as read-write, iOS protects against unmounting it. The `dounmount` function contains a call to `vnode_rele_internal` that will crash the system if you try to unmount the root filesystem.

**What KPF does**: NOPs out a specific call instruction inside `dounmount` to prevent the crash.

**How it finds it**: KPF searches for a distinctive sequence:

```c
uint64_t matches[] = {
    0x52800001, // MOVZ W1, #0
    0x52800002, // MOVZ W2, #0
    0x52800003, // MOVZ W3, #0
};
```

This three-register zeroing pattern is rare enough to narrow the search. The callback then performs detailed flow analysis — checking for `MOV X0, Xn` followed by `BL` (function call) patterns — to confirm it found `dounmount` and identify the exact `BL` instruction to NOP out.

# Patch 4: W^X Bypass (vm_map_protect)

**What it targets**: iOS enforces **W^X** (Write XOR Execute) — memory pages cannot be simultaneously writable and executable. This is checked in `vm_map_protect()` when you call `mprotect()`:

```c
if ((new_prot & VM_PROT_WRITE) &&
    (new_prot & VM_PROT_EXECUTE) &&
    !(current->used_for_jit)) {
    new_prot &= ~VM_PROT_EXECUTE;
}
```

And further down, there's a second check:

```c
if (map->map_disallow_new_exec == TRUE) {
    if ((new_prot & VM_PROT_EXECUTE) || ...) {
        return KERN_PROTECTION_FAILURE;
    }
}
```

**What KPF does**: It doesn't just NOP out one check — it **inserts a branch** that skips both the W^X enforcement and the `map_disallow_new_exec` check entirely.

**How it finds it**: Two pattern variants are searched:

```c
// Variant 1: AND + SUBS pattern
uint64_t matches[] = {
    0x121F0600, // AND W*, W*, #6     (isolate WRITE|EXECUTE bits)
    0x71001900  // SUBS W*, W*, #6    (check if both are set)
};

// Variant 2: MVN + TST pattern
uint64_t i_matches[] = {
    0x2A2003E0, // MVN W*, W*
    0x721F041F, // TST W*, #6
    0x54000000  // B.EQ *
};
```

Notice these are **AArch64 encodings** — `0x121F0600` is `AND` in A64, completely different from the ARM32 Thumb `AND.W R0, R1, #6` that older documentation shows.

The callback finds the match, then scans forward for a `TBNZ W*, #9, *` (the `map_disallow_new_exec` check), and **rewrites the branch** to jump past both checks:

```c
uint32_t delta = first_ldr - (&opcode_stream[2]);
delta &= 0x03FFFFFF;
delta |= 0x14000000;        // B (unconditional branch)
opcode_stream[2] = delta;   // Skip everything
```

**Why it matters**: Without this, MobileSubstrate (Cydia Substrate) can't hook functions at runtime, emulators can't generate native code, and most tweaks simply don't work. But it does weaken security — a remote exploit only needs a tiny ROP chain to make shellcode executable via `mprotect()`.

# Patch 5: Code Signing in the Page Fault Handler (vm_fault_enter)

**What it targets**: When the CPU accesses a memory page that isn't currently loaded, the kernel's page fault handler (`vm_fault_enter()`) checks the page's code signing status. If an unsigned page is being mapped as executable, it triggers a `KERN_CODESIGN_ERROR`.

**What KPF does**: The KPF source has a `vm_fault_enter_callback` that patches the code signing validation logic inside the page fault path. The exact implementation is complex (the function is large and the compiler generates different code per iOS version), but the result is the same: unsigned pages can be mapped executable without triggering a codesign error.

This works in tandem with the AMFI trustcache patch — AMFI handles the signature check at exec time, while `vm_fault_enter` handles it at the memory mapping level. Both need to be disabled for unsigned code to actually run.

# Patch 6: tfp0 Access (task_conversion_eval)

**What it targets**: `task_for_pid(0)` gives you the kernel's task port, which means arbitrary kernel read/write from userspace. Apple added protection in `task_conversion_eval()` that checks if the caller is a "platform binary" (Apple-signed). If the caller isn't platform-signed but the target is, the conversion is denied.

The check boils down to:

```c
if ((victim->t_flags & TF_PLATFORM) && !(caller->t_flags & TF_PLATFORM)) {
    return KERN_DENIED;
}
```

But there's a shortcut: if `caller == victim`, it returns SUCCESS immediately.

**What KPF does**: It finds the `caller == victim` comparison and patches it to **always evaluate as true** by replacing `CMP X28, X22` with `CMP XZR, XZR` (comparing zero to zero always succeeds). This makes the function return SUCCESS immediately, bypassing the platform binary check.

**How it finds it**: KPF searches for the TF_PLATFORM flag check pattern:

```c
uint64_t matches[] = {
    0xB9400000, // LDR W*, [X*, #offset]    (load victim->t_flags)
    0x36500000, // TBZ W*, #0xA, *          (test TF_PLATFORM bit)
    0xB9400000, // LDR W*, [X*, #offset]    (load caller->t_flags)
    0x36500000, // TBZ W*, #0xA, *          (test TF_PLATFORM bit)
};
```

The callback verifies that both LDR instructions use the same offset (both are reading `t_flags` from different task structs), then walks backward to find the `CMP` instruction that compares caller to victim, including handling `MOV` instructions that might have shuffled registers around.

This is much more sophisticated than the old `task_for_pid(0)` branch patch from the comex era — instead of just removing a PID check, KPF defeats the platform binary verification that Apple added as a deeper defense.

# Patch 7: Kernel Map Access (convert_port_to_map)

**What it targets**: Starting in iOS 14 betas, Apple added a panic check in `convert_port_to_map_with_flavor()`. When userspace requests write access to a `vm_map_t`, the function checks if the map is backed by `kernel_pmap`. If it is, the kernel panics:

```
panic: "userspace has control access to a kernel map through task"
```

This was specifically designed to kill tfp0-based kernel memory access.

**What KPF does**: Finds the `B.EQ` branch that leads to the panic and replaces it with `NOP`.

**How it finds it**: Searches for the entry pattern:

```c
uint64_t matches[] = {
    0xAA0103F0, // MOV X[16-31], X1
    0x7100083F, // CMP W1, #2
    0x54000000, // B.EQ *
    0x7100061F, // CMP W[16-31], #1
    0x54000000, // B.EQ *
};
```

The callback verifies the register numbers match, then finds the next `B.EQ` after the `kernel_pmap` comparison and NOPs it.

# Patch 8: Custom dyld Path

**What it targets**: When iOS loads a binary, the kernel function `load_dylinker()` verifies that the dynamic linker path matches `"/usr/lib/dyld"`. If it doesn't, the load fails.

**What KPF does**: Replaces the hardcoded string comparison with a hook that allows using either the default dyld path or a custom one provided by the jailbreak.

**How it finds it**: Searches for the inline `strcmp` pattern that compares against `"/usr/lib/dyld"`:

```c
uint64_t matches[] = {
    0x54000002, // B.CS *          (bounds check)
    0x90000000, // ADRP *          (load page of "/usr/lib/dyld")
    0x91000000, // ADD X*, X*, #*  (add page offset)
    0xAA0003E0, // MOV X*, X*      (set up comparison)
    0x39400000, // LDRB W*, [X*]   (load byte from input path)
    0x39400000, // LDRB W*, [X*]   (load byte from "/usr/lib/dyld")
    0x6B00001F  // CMP W*, W*      (compare bytes)
};
```

The callback replaces the comparison with a `BL` to the jailbreak's dyld hook function.

# Patch 9: Sandbox Shellcode

**What it targets**: The sandbox evaluator, which blocks access to paths that jailbreak operations require (stashed apps, modified system directories).

**What KPF does**: Unlike the other patches that modify existing code, this one **injects new code**. KPF finds an area of unused memory (NOP padding or zero-filled space) inside the kernel's executable region and copies custom shellcode into it. This shellcode hooks into the sandbox evaluation path and implements custom rules for path-based access decisions.

**How it finds the space**: Searches for a large block of consecutive zeros in `__TEXT_EXEC`:

```c
// find a place inside of the executable region that has
// no opcodes in it (just zeros/padding)
extern uint32_t sandbox_shellcode, sandbox_shellcode_end;
uint32_t count = &sandbox_shellcode_end - &sandbox_shellcode;
// ... generates a match pattern of all zeros
```

The actual shellcode is defined in `shellcode.S` — an AArch64 assembly file that implements the custom sandbox evaluation logic. It's linked into KPF and copied into the kernel at runtime.

# What About the "Classic" Patches?

If you've read other jailbreak documentation, you might be wondering about some patches that are conspicuously absent from KPF:

| "Classic" Patch | Status in checkra1n's KPF |
|---|---|
| `security.mac.proc_enforce = 0` | **Not present**. KPF uses the AMFI trustcache bypass and individual function patches instead of flipping a global variable. |
| `PE_i_can_has_debugger` | **Not present**. checkra1n doesn't enable KDP by default. |
| `cs_enforcement_disable` variable | **Not directly patched**. Instead, `vm_fault_enter` is patched at the code level and AMFI's trustcache is bypassed. |
| Old `task_for_pid(0)` PID check | **Replaced** by the `task_conversion_eval` and `convert_port_to_map` patches, which defeat Apple's newer defenses. |
| comex's `sb_evaluate` hook (ARM32 ASM) | **Replaced** by AArch64 shellcode injected into kernel memory. |

The evolution makes sense. Apple's security model got deeper with each iOS version — simple variable patches stopped being sufficient. KPF's approach is to patch the actual enforcement code, not the global switches that *used to* control it.

# Context: Why This Matters for Older vs Newer Jailbreaks

checkra1n (iOS 12-14, A7-A11) operates in the **rootful** paradigm — it modifies the root filesystem directly. The `mac_mount` and `dounmount` patches are essential for this.

For **iOS 15+ rootless jailbreaks** (palera1n, Dopamine), the game changes again. Signed System Volume (SSV) means even with these patches, you can't write to the real root filesystem. Those tools use:
- **fakefs** (a writable copy of the root filesystem) or
- **rootless** installation into `/var/jb/` on a separate APFS volume

The kernel patches are similar in concept but adapted for the rootless architecture — the sandbox shellcode targets different paths, and new patches handle SSV-specific checks.

# Summary: checkra1n's KPF Patch Map

```
KPF starts → scans kernel binary in memory
     │
     ├── AMFI trustcache → always return "trusted"
     │   (allows all binaries to execute)
     │
     ├── mac_mount → NOP MNT_UNION check + zero MNT_ROOTFS flag
     │   (allows remounting / as read-write)
     │
     ├── dounmount → NOP vnode_rele call
     │   (allows unmounting rootfs without crash)
     │
     ├── vm_map_protect → branch over W^X + map_disallow_new_exec
     │   (allows writable+executable memory)
     │
     ├── vm_fault_enter → patch CS enforcement logic
     │   (allows unsigned pages to be executable)
     │
     ├── task_conversion_eval → CMP XZR, XZR
     │   (allows tfp0 from non-platform binaries)
     │
     ├── convert_port_to_map → NOP kernel_pmap panic
     │   (allows kernel map access via tfp0, iOS 14+)
     │
     ├── dyld → hook load_dylinker string comparison
     │   (allows custom dynamic linker)
     │
     └── sandbox → inject shellcode into unused kernel space
         (custom path-based access rules)
```

Nine patches. Each one targets a specific enforcement point. Together, they transform a cryptographically locked-down kernel into an open research platform.

# Wrapping Up the Series

This series went from "what is a jailbreak?" to reading actual KPF source code:

- **The Introduction** covered the landscape — history, types, tools, notable exploits.
- **Part 1** walked through checkra1n's full chain — checkm8 → PongoOS → KPF.
- **Part 2** (this post) showed what KPF actually patches, based on the real source code — not recycled documentation from a different era.

The lesson I keep coming back to: **read the source**. Most of the jailbreak knowledge floating around online describes patches from 2011-2012. The fundamentals are related, but the actual implementation in modern tools is completely different. If you want to understand what's really happening on your device, [the code](https://github.com/checkra1n/PongoOS/blob/master/checkra1n/kpf/main.c) is right there.

# References

- [PongoOS KPF Source Code — `main.c`](https://github.com/checkra1n/PongoOS/blob/master/checkra1n/kpf/main.c)
- [PongoOS Sandbox Shellcode — `shellcode.S`](https://github.com/checkra1n/PongoOS/blob/master/checkra1n/kpf/shellcode.S)
- [checkra1n Official](https://checkra.in/)
- [The Apple Wiki — AppleMobileFileIntegrity](https://theapplewiki.com/wiki/AppleMobileFileIntegrity)
- [The Apple Wiki — Jailbreak Exploits](https://theapplewiki.com/wiki/Jailbreak_Exploits)
- [Brandon Azad — Bypassing Platform Binary Task Threads](https://bazad.github.io/2018/10/bypassing-platform-binary-task-threads/)
- [comex — datautils0 (Historical Reference)](https://github.com/comex/datautils0)
