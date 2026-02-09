---
title: "The Jailbreak Process - Under the Hood"
date: 2026-02-08 00:00:00
classes: wide
header:
  teaser: /assets/images/Jailbreak-Proccess-P1/1.jpg
  overlay_image: /assets/images/Jailbreak-Proccess-P1/1.jpg
  overlay_filter: 0.5
ribbon: DarkSlateGray
excerpt: "Dissecting the internals of checkra1n: from checkm8 exploitation to PongoOS and kernel patching"
description: "A deep technical dive into the checkm8 exploit, PongoOS boot environment, and how checkra1n transforms a locked-down iPhone into a research platform"
categories:
  - Forensics
  - Mobile
tags:
  - iOS
  - Jailbreak
  - Checkm8
  - Checkra1n
  - Exploit
  - PongoOS
  - Bootrom
  - Kernel
toc: true
toc_sticky: true
toc_label: "On This Blog"
toc_icon: "biohazard"
---

# Introduction

In [Part 1](/forensics/mobile/iOS-Jailbreak/) we went through the history, the types of jailbreaks, the tools, and the notable exploits that shaped the scene. If you missed it, go read it first — what follows assumes you're comfortable with concepts like tethered vs untethered, bootrom vs userland, and DFU mode.

Now we're going deeper.

I remember the first time I ran checkra1n on a test device. The whole process took maybe 2 minutes. Plug in the phone, click Start, follow the DFU instructions, and... that's it. Cydia appeared on the home screen. It felt almost anticlimactic compared to the redsn0w days where you had to pray the process wouldn't fail halfway through.

But here's the thing — those 2 minutes involve one of the most elegant exploit chains ever developed for a consumer device. A hardware vulnerability that Apple literally cannot patch without redesigning silicon, combined with a custom pre-boot operating system that surgically modifies the iOS kernel before it even starts running.

This post breaks down exactly what happens during those 2 minutes.

# Why Should You Care About the Internals?

Before we dive in, let me address something I've seen too often in the security community: people who use jailbreak tools daily for research but have zero idea how they actually work. They treat checkra1n like a magic button.

If you're doing iOS forensics, offensive security, or vulnerability research, understanding jailbreak internals isn't just nice to have — it's essential:

- You **can't evaluate the forensic integrity** of an extraction if you don't know what the jailbreak modified on the device.
- You **can't assess risk** if you don't understand what security mechanisms were disabled and which ones remain active.
- You **can't troubleshoot** a failed jailbreak on a target device during a time-sensitive engagement.
- You **can't contribute** to the community that makes your research possible in the first place.

And there's a practical reality: stock iPhones are locked down hard. Even with a developer account, you can't fork processes, you can't attach debuggers to system apps, you can't inspect other applications' files, and you can't see what's really happening under the hood. Almost every significant iOS security discovery — from the "locationgate" GPS tracking scandal to modern zero-day chains — was either found on or validated using jailbroken hardware.

# Where the Vulnerability Lives Matters

In Part 1 we classified jailbreaks by **persistence** (tethered, untethered, etc.). But there's another dimension that directly determines how powerful a jailbreak can be: the **location of the vulnerability** in the boot chain.

## Bootrom Level — The Holy Grail

The bootrom (SecureROM) is the very first code that runs when you power on an iPhone. It's burned into the silicon during manufacturing. Read-only. Immutable. If you find a bug here, Apple's only option is to release new hardware.

A bootrom exploit gives you:

- **Control over the entire boot chain** — you can patch or replace LLB, iBoot, and the kernel before they even load.
- **Full hardware access** — including the GID key in the AES engine, which means you can decrypt firmware images (something impossible from any higher privilege level).
- **Permanence across iOS versions** — the same exploit works whether the device runs iOS 6 or iOS 14. Apple can't patch it away.

This is exactly what makes **checkm8** so devastating. It affects every Apple device from the A5 chip (iPhone 4S, 2011) through the A11 chip (iPhone X, 2017) — **six years** of hardware, hundreds of millions of devices, all permanently vulnerable.

## iBoot Level — Close, But Patchable

iBoot is the second-stage bootloader. It still runs early enough to give you kernel patching capability and hardware-level AES access. The catch: iBoot lives in flash, not in ROM. Apple can (and does) patch iBoot vulnerabilities with software updates, which makes these exploits valuable but short-lived.

## Userland Level — The Hard Way

When there's no bootrom or iBoot vulnerability available, the entry point has to be a user-facing application — usually the browser (MobileSafari) or a system service. This is how jailbreaks like **JailbreakMe 3.0** worked: a crafted PDF exploited a vulnerability in the font parser, achieving code execution inside MobileSafari's sandbox.

But that's just step one. From inside a sandboxed process, you still need a **separate kernel exploit** to escape the sandbox, disable code signing, and actually jailbreak the device. That's at minimum two exploit chains, both of which Apple can patch in the next iOS update.

# checkm8: The Exploit That Changed Everything

Now let's get into the specifics. The checkm8 exploit (CVE-2019-8900), released by **axi0mX** in September 2019, is arguably the most impactful iOS security finding ever published. Let me explain why.

## The Vulnerability

checkm8 targets a **use-after-free vulnerability** in the USB stack of Apple's SecureROM. Specifically, it lives in the DFU (Device Firmware Update) mode code — the minimal USB protocol handler that allows restoring a bricked device.

Here's what happens at a high level:

1. When a device enters DFU mode, the SecureROM allocates a global I/O buffer (~2KB) and creates a USB interface to handle incoming data transfers.
2. The USB control transfer handler processes `GET_DESCRIPTOR` requests to exchange device information with the host.
3. The vulnerability: when you initiate a USB transfer and then **abort it at precisely the right moment** (by triggering a USB reset mid-transaction), the DFU code frees its internal request structures... but **keeps pointers to the freed memory**. Classic use-after-free.

The result: the next time DFU processes a USB request, it follows those dangling pointers into memory that you now control.

## Heap Feng Shui

Finding a use-after-free is one thing. Reliably exploiting it in a bootrom with no ASLR bypass and minimal heap metadata is another. This is where checkm8 gets elegant.

The technique is called **heap feng shui** (or heap grooming):

1. **Create memory pressure**: Send a series of USB control transfers with sizes that aren't multiples of the USB packet size (64 bytes). Each one forces the DFU code to allocate request objects that never get properly freed — they pile up on the heap.
2. **Create predictable holes**: By carefully controlling the size and order of these allocations, you shape the heap layout so that the freed object's memory will be reused by your next allocation — one that you control the contents of.
3. **Trigger the use-after-free**: Send a stall-inducing setup packet, then abort. The DFU state machine enters an inconsistent state where it still references the freed structures.
4. **Place the payload**: The next allocation fills the freed slot with your carefully crafted data structure, which the DFU code then dereferences as if it were a legitimate request — executing your code.

On older devices (pre-A9), there's an even simpler path: a secondary DFU abort bug allows direct code execution without the full heap grooming sequence.

## What Makes It Unpatchable

Here's the key detail: the vulnerable code is in **SecureROM**, which is mask ROM — baked into the chip during the lithography process. It's not flash memory. There's no firmware update mechanism for it. The only fix is to manufacture new silicon with corrected code, which is exactly what Apple did starting with the A12 chip (iPhone XS, 2018).

Every A5 through A11 device ever manufactured — and every one that will ever exist — carries this vulnerability. Forever.

# checkra1n: From Exploit to Jailbreak

checkm8 is just the exploit — raw code execution in the bootrom. **checkra1n** is the jailbreak tool built on top of it, developed by a team led by **Luca Todesco** (qwertyoruiop) and first released in November 2019.

Here's what actually happens when you click "Start" in checkra1n.

## Phase 1: DFU Entry and Exploitation

checkra1n guides you through putting the device into DFU mode (the specific button combination depends on the device model — Home + Power for older devices, Volume Down + Power for iPhone 8/X).

Once in DFU, checkra1n sends the crafted USB payloads to trigger checkm8:

1. Heap grooming packets shape the DFU allocator's state.
2. The use-after-free is triggered via the stalled transfer + USB reset technique.
3. A small first-stage payload executes in the bootrom context — this patches the signature verification and hands control to the next stage.

At this point, the device will accept **any** image you feed it, regardless of Apple's signature. The boot chain's trust model is broken at its root.

## Phase 2: PongoOS — The Pre-Boot Environment

This is where checkra1n differs fundamentally from older jailbreaks like redsn0w. Instead of booting a one-off ramdisk with a hardcoded jailbreak script, checkra1n loads a **full custom pre-boot operating system** called **PongoOS**.

PongoOS runs in the gap between iBoot and the iOS kernel. Think of it as a tiny OS that takes over before iOS gets a chance to start. It includes:

- **A task scheduler** with preemption and interrupt handling
- **A virtual memory system** with physical memory management
- **Device drivers** for USB, display (framebuffer), UART/serial, and the AES crypto engine
- **An interactive shell** accessible via USB (using the `pongoterm` client)
- **A module loading system** for custom extensions
- **The Kernel Patch Finder (KPF)** — the component that actually patches the iOS kernel

This architecture is far more flexible than the old ramdisk approach. Researchers can load custom modules, interact with the device through a shell, and even boot alternative operating systems (including Linux) through PongoOS.

## Phase 3: Kernel Patching via KPF

Before handing off to the iOS kernel, PongoOS runs the **Kernel Patch Finder (KPF)**. This is the brain of the jailbreak — it scans the kernel binary in memory, locates specific security-critical code patterns, and applies surgical patches.

The KPF has to be adaptive because:
- Different iOS versions compile the kernel differently (different instruction sequences, register allocations, offsets).
- Different chip architectures (A7 vs A11) produce different binary patterns from the same source code.
- Apple introduces new security mechanisms with each iOS release.

What KPF patches (at a high level — we'll go deep on each in Part 2):

| Target | What Gets Disabled |
|--------|-------------------|
| **Code signing enforcement** | Allows unsigned binaries to execute |
| **MAC policy enforcement** | Disables Mandatory Access Control process restrictions |
| **AMFI checks** | Neutralizes Apple Mobile File Integrity validation |
| **Sandbox evaluation** | Modifies path-based sandbox rules for stashed apps |
| **Memory protection (W^X)** | Allows memory to be writable and executable simultaneously |
| **Debugging restrictions** | Enables kernel debugging and process introspection |

Each patch is typically just a few bytes — changing a conditional branch to an unconditional one, replacing a memory load with an immediate value, or NOPing out a check entirely. But finding *where* to apply those few bytes in a multi-megabyte kernel binary that changes with every iOS update — that's the hard part.

## Phase 4: iOS Boot with Modified Kernel

With patches applied, PongoOS hands control to the now-modified XNU kernel, which boots iOS as normal — except the security restrictions that normally prevent unsigned code execution are gone.

The boot continues through `launchd`, SpringBoard loads, and the home screen appears. But now there's a new app: **checkra1n loader**.

## Phase 5: Bootstrap Installation

The checkra1n loader is a minimal app that handles the userspace setup. When you tap "Install Cydia," it:

1. **Downloads and installs the Elucubratus bootstrap** — a collection of core UNIX utilities (`bash`, `apt`, `dpkg`, `ssh`, etc.) and library dependencies.
2. **Sets up Cydia** (or optionally Sileo) as the package manager.
3. **Installs Dropbear SSH server** — giving you remote shell access over USB or WiFi.
4. **Configures dpkg/apt** repositories for package management.

After this step, the device has a functional package management system, SSH access, and the ability to install tweaks and tools from repositories.

# The Modern Landscape: Rootful vs Rootless

If you're working with checkra1n on iOS 12-14, you're dealing with a **rootful** jailbreak — modifications go directly onto the root filesystem, just like the old days.

But the landscape has shifted dramatically for iOS 15+, and understanding this matters even if your current target is an older device.

## The SSV Problem

Starting with iOS 15, Apple introduced the **Signed System Volume (SSV)**. The entire system partition is cryptographically hashed, and iBoot verifies this hash at boot. If a single byte on the system volume has been modified, the device refuses to boot and drops into Recovery Mode.

This effectively killed traditional rootful jailbreaking. You can't just remount `/` as read-write and drop files onto it anymore.

## Rootful Workaround: fakefs

Tools like **palera1n** (the spiritual successor to checkra1n for iOS 15+) work around SSV by creating a **copy** of the root filesystem — a "fake filesystem" — and redirecting the boot process to use that copy instead of the signed original. This preserves the original system volume's hash while giving you a writable root.

The trade-off: it consumes significant storage space (the entire system partition is duplicated), and it's inherently less stable.

## Rootless: The New Standard

The modern approach is **rootless jailbreaking**: instead of touching the system volume at all, jailbreak files go into `/var/jb/` (which symlinks to a path on the `/private/preboot` APFS volume — a separate volume that SSV doesn't protect).

This means:
- The system partition stays untouched and cryptographically valid.
- Jailbreak components live on a separate volume that persists across reboots.
- Removing the jailbreak is as simple as deleting the bootstrap — no restore needed.
- But **every tweak needs to be updated** to look for files in `/var/jb/` instead of hardcoded system paths.

For checkra1n on iOS 12-14, you don't have to worry about this — but if you're moving to palera1n or Dopamine on newer devices, this distinction is critical.

# What About the Filesystem and AFC2?

If you've read older jailbreak documentation, you've seen references to modifications like replacing `/etc/fstab`, installing the AFC2 service, and application stashing. Let me quickly address how these concepts map to checkra1n.

## Filesystem Access

checkra1n's kernel patches handle the root filesystem mounting at the kernel level — the KPF patches allow the system to boot with a writable root without needing to modify `fstab` on disk. This is cleaner than the old approach and leaves fewer forensic artifacts.

## AFC2 and File Access

The old AFC2 service (which gave full root filesystem access over USB via `lockdownd`) is **no longer installed by default** in checkra1n. Instead, SSH via Dropbear is the standard method for filesystem access. This is both more flexible (you get a full shell, not just file transfer) and slightly more secure (SSH requires authentication).

If you need AFC2 specifically (some forensic tools require it), you can install `Apple File Conduit "2"` from Cydia — but for most research workflows, SSH is superior.

## Application Stashing

Modern iOS versions allocate significantly more space to the system partition than the old 1GB days. Combined with the rootless approach on newer jailbreaks, application stashing is largely obsolete. checkra1n on iOS 12-14 still uses the traditional `/Applications` path, but storage constraints are rarely an issue on modern hardware.

# The Semi-Tethered Reality

One thing to internalize about checkra1n: it's **semi-tethered**. The checkm8 exploit only executes in RAM — nothing persists to storage. Every time your device reboots, it returns to a stock, non-jailbroken state.

To re-jailbreak, you need to:
1. Connect the device to a computer running checkra1n.
2. Put it into DFU mode.
3. Run the exploit again.

This means your device is always one reboot away from being "clean" — which is actually useful in certain forensic and research scenarios where you need to toggle between jailbroken and stock states.

The old redsn0w-era distinction between tethered and untethered still applies, but with checkra1n you at least get a **bootable device** after reboot (your apps and data work fine, you just lose jailbreak functionality until you re-exploit). With the old tethered jailbreaks, your device was a brick until you plugged it back in.

# Summary

Let's recap the full checkra1n flow:

```
[DFU Mode] → checkm8 exploit (USB use-after-free + heap feng shui)
     ↓
[SecureROM] → Signature verification patched, boot chain trust broken
     ↓
[PongoOS loads] → Custom pre-boot OS with kernel access
     ↓
[KPF runs] → Kernel scanned and patched (code signing, AMFI, sandbox...)
     ↓
[XNU boots] → Modified kernel loads iOS normally
     ↓
[checkra1n loader] → Installs Cydia/Sileo, SSH, bootstrap utilities
     ↓
[Jailbroken device] → Full root access, unsigned code execution enabled
```

The entire chain is triggered by a hardware vulnerability that Apple cannot patch. The exploitation technique — heap feng shui on a bootrom's USB stack — is a masterclass in low-level exploitation. And the PongoOS architecture represents a generational leap from the one-off ramdisk scripts of the redsn0w era.

In [Part 2](/forensics/mobile/Jailbreak-Proccess-P2/) we'll go even deeper: the actual kernel patches that KPF applies, the specific security mechanisms each one disables, and the ARM assembly-level details of how a few changed bytes can dismantle iOS security.

# References

- [checkm8 Exploit — CVE-2019-8900](https://theapplewiki.com/wiki/Checkm8_Exploit)
- [A Comprehensive Write-up of the checkm8 BootROM Exploit — Alfie CG](https://alfiecg.uk/2023/07/21/A-comprehensive-write-up-of-the-checkm8-BootROM-exploit.html)
- [Technical Analysis of the checkm8 Exploit — Habr/DSec](https://habr.com/en/companies/dsec/articles/472762/)
- [PongoOS — GitHub (checkra1n)](https://github.com/checkra1n/pongoOS)
- [checkra1n Official](https://checkra.in/)
- [Signed System Volume Security — Apple](https://support.apple.com/guide/security/signed-system-volume-security-secd698747c9/web)
- [What is a Rootless Jailbreak? — iDevice Central](https://idevicecentral.com/jailbreak-tools/what-is-a-rootless-jailbreak-and-is-rootless-less-powerful/)
