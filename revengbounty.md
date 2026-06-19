# Executing a VM-Hostile Crackme *inside* a VM (and why patching was the wrong move)

### So I picked up a reversing bounty and the challenge was beautifully simple to read and absolutely cursed to actually do:

*"Successfully execute this sample inside a VM and publish a technical blog explaining how you did it."*

That's it. Run the thing in a VM. Except the second you drop it in any VM, it instantly closes on you. No crash dialog, no error, no nothing, it just dies.

[📷 PASTE screenshot_234 HERE — the crackmes.one page for kaganisildak / Malwation "$1K AntiVM": difficulty **Insane**, the $1000 VM-execution bounty + the $250 password bonus]

The interesting part, and the whole reason I'm writing this, is that the winning move here is **not** patching the binary. The challenge literally says "execute *this* sample", and you'll see later the binary self-hashes itself, so the moment you touch a single byte it bails. The actual win is making the **VM lie convincingly enough that the genuine, unmodified binary runs itself.** Hide the VM, don't crack the binary.

The sample:
- `crackme_NoVM.exe`, 64-bit MSVC console app, 1,597,073 bytes
- SHA256 `5E3ABC2D38A41E0DF846026C5E41690EDFF0498258D4594355CC633A062679B1`
- Image base `0x140000000` (it's ASLR'd at runtime so I'll quote everything as `base + RVA`)

Keep that SHA in mind, because at the end the file we successfully run inside a VM has the **exact same hash**. No patch.

### First contact: IDA won't even open it

Loading it into IDA, immediately:

```
Cannot read 8388607 bytes, starting at offset 134217727
```

[📷 PASTE screenshot_216 HERE — the IDA "Cannot read 8388607 bytes, starting at offset 134217727" error dialog on load]

`8388607 = 0x7FFFFF`, `134217727 = 0x7FFFFFF`. Suspiciously power-of-two-adjacent, right? That's not a coincidence, that's someone messing with me on purpose.

So instead of trusting the tool I parsed the PE **by hand**. The PE header itself is clean, 6 normal sections, entry point `0x1229CC`, no section runs past end of file. So it's not the section table lying.

The landmine is in the **Debug Directory**. The first entry is a forged `IMAGE_DEBUG_TYPE_CODEVIEW` (Type=2) with:
- `SizeOfData = 0x7FFFFF`
- `AddressOfRawData = 0x7FFFFFF`
- `PointerToRawData = 0x7FFFFFF`

[📷 PASTE screenshot_217 HERE — PE-bear/CFF Explorer Debug Directory view — the forged Type=2 CODEVIEW entry with SizeOfData=0x7FFFFF and the bogus pointers, alongside the two legit MSVC entries (Type 12 VC_FEATURE, Type 13 POGO)]

IDA, being eager, follows the debug dir to load symbols, tries to read a `0x7FFFFF` blob at file offset `0x7FFFFFF`, and chokes. **Windows ignores the debug directory at load time**, so the sample still runs perfectly fine. The malformation is aimed squarely at tooling, not the OS. This becomes a recurring theme, by the way: *the binary is broken on purpose to break your tools, never the loader.*

Fun detail, the other two debug entries are actually legit MSVC (Type 12 `VC_FEATURE`, Type 13 `POGO`). That's how you fingerprint the toolchain.

The fix is trivial. Make an analysis-only copy, zero the Debug data-directory entry, 8 bytes at file offset **`0x1C8`**, reload. (Niche bit for the curious: data dir #6 is Debug, `dir_start = 0x198`, `+6*8 = 0x1C8`.) IDA opens it happily after that.

P.S. don't zero it on your run copy, only the analysis copy. We're not patching the real one, remember.

### Recon: what are we even dealing with

[📷 PASTE screenshot_221 HERE — DetectItEasy on crackme_NoVM.exe — PE32+ console, MSVC, statically-linked CRT]

- Only **KERNEL32** imported. That's it.

[📷 PASTE screenshot_218 HERE — the imports view (IDA/CFF) showing KERNEL32 as the only imported DLL]
- 428 strings, and they're *all* CRT boilerplate. **Zero** VM-vendor strings. I grepped `vmware`, `vbox`, `qemu`, all of it, nothing. And that absence is itself a clue, more on that in a sec.

[📷 PASTE screenshot_219 + screenshot_220 HERE — strings vendor-absence: empty filter for vmware (219) / vbox (220); the list is all-CRT boilerplate]

- Anti-debug imports are sitting right there though: `IsDebuggerPresent`, `Get/SetThreadContext`, a pile of timing APIs (`GetTickCount64`, `QueryPerformanceCounter`, `GetSystemTimeAsFileTime`), plus `TerminateProcess`/`ExitProcess`.

[📷 PASTE screenshot_218 HERE (same image as above) — imports list with the anti-debug / timing APIs highlighted (IsDebuggerPresent, Get/SetThreadContext, GetTickCount64, QueryPerformanceCounter, GetSystemTimeAsFileTime)]

Here's the deduction I want to spell out because it ends up being the whole ballgame: with **no registry, file, device, firmware, or network APIs** and **no vendor strings**, any environment sensing this thing does has to be **CPU-level or timing based.** There's literally nothing else for it to use.

### The obfuscation

Whoever wrote this knew what they were doing. The fingerprints:

- **Control-flow flattening.** There's a dispatcher at RVA `0x1190F`:
  ```
  mov r8,[r12+r9]
  add r8,r15
  jmp r8
  ```
  That's `jmp stateTable[state] + base`. Classic flattening, the real control flow lives in a state variable.

[📷 PASTE screenshot_222 HERE — IDA disasm of the flattening dispatcher at RVA 0x1190F — the `mov r8,[r12+r9]; add r8,r15; jmp r8` block]

- **Opaque predicates** everywhere, e.g. `(x*(x+1)) & 1` which is *always* 0 (product of two consecutive ints is always even). Fake branches to make you waste your life.
- **Pointer encryption.** Real pointers are recovered as `*(global) - 0x4E0253A3BAFBADE9` and `global - 0x688D3FBC21B48D4D` (64-bit additive keys). Decompilers render these as absolutely nonsense huge offsets, which is the point.

- **Anti-disassembly junk bytes** and a **decoy `main`.** IDA's `main` symbol points at `0x14007A9D0`, which is 55 junk bytes starting with `0x3F` (invalid in x64) followed by `CC` padding.

[📷 PASTE screenshot_223 HERE — the decoy `main` at 0x14007A9D0 — hex/disasm view showing the 0x3F junk bytes + CC padding that IDA mislabels as main]

  Confession time: I briefly mis-computed the CRT's `call main` target and went chasing a phantom function for a bit. The lesson I took away, and I'm leaving this in because it's honest, is **trust the actual call bytes over the tool's heuristic symbol.** The CRT actually calls main at `0x140122957`, the bytes are `E8 74 80 F5 FF`. The decoy symbol was a trap and I walked right into it.
- **Encrypted strings**, decrypted on demand. That's why static string search came up blind earlier. I later confirmed the cipher is **ChaCha20**, the `expand 32-byte k` constant shows up in memory.

- Two real hash primitives baked in:
  - **XXH3** at `sub_140093230` (constants `0x9E3779B185EBCA87`, `0xC2B2AE3D27D4EB4F`, `0x61C8864E7A143579`)
  - a **MurmurHash2** PRNG seed at `sub_140096800` (`0x5BD1E995` / `0x6C1B4395`, mixing `rdtsc ^ GetTickCount64 ^ rand`). Note the `rdtsc` showing up in the seed, foreshadowing.

- **Code self-hash / anti-tamper.** I proved this later but I'll spoil it now: edit *any* byte and the process exits `10002` (`0x2712`). This is what makes patching a losing game.

Lovely stuff. Genuinely one of the nicer protected binaries I've poked at.

### Ruling out the usual anti-VM

Before I went off on a timing tangent I wanted to eliminate the boring stuff, because credibility:

- No **VMware backdoor**, the `VMXh` magic (`68 58 4D 56`) is absent.

- No **descriptor red-pills**, no `sidt`/`sgdt`/`sldt`/`smsw` anywhere.
- The only real **CPUID** routine is MSVC's `__isa_available_init` (vendor "GenuineIntel", ISA feature leaves). It never tests the hypervisor bit (ECX[31]) or leaf `0x40000000`. So it's not sniffing the hypervisor via CPUID either.

- No active **TLS callbacks**, the callback array starts with a NULL.
- No custom kill-switch. `ExitProcess`/`TerminateProcess` are CRT-only, so the anti-VM **branches** somewhere, it doesn't cleanly bail with a dedicated "die" function. Which means I have to find the branch.

So all the off-the-shelf detections are out. It's timing. It has to be timing. I just didn't have proof yet.

### Going dynamic

Setup, because this part is genuinely useful for anyone wiring up a similar rig:
- **IDA Pro MCP + Ghidra MCP** for static, driven from the host.
- **x64dbg + an MCP plugin inside FlareVM** for dynamic, also driven from the host.

First problem, networking. VirtualBox NAT means the guest IP (`10.0.2.15`) isn't reachable from the host, so the host can't talk to the plugin. Fix is a NAT port-forward:

```
VBoxManage controlvm "FlareVM" natpf1 "x64dbgmcp,tcp,127.0.0.1,3000,,3000"
```

Then the host just hits `127.0.0.1:3000`. Bind the plugin to `0.0.0.0` and allow inbound 3000 in the guest firewall and you're good.

Second problem, version mismatch:

```
The procedure entry point DbgUpdateGui could not be located...
```

That's the MCP plugin being built against a newer x64dbg SDK than the one FlareVM ships with. Fix: update x64dbg to the latest snapshot. Easy.

Then I tried **ScyllaHide** with everything turned on, all hooks + timing-API hooks + KiUserExceptionDispatcher. Didn't help even slightly.

And honestly that null result is *important*, because here's the reasoning: ScyllaHide hooks *APIs*. It cannot touch the `rdtsc` **instruction** itself. So the fact that it did nothing basically confirmed for me that the timing check is **rdtsc-based**, not API-based. Sometimes the thing that doesn't work tells you the most.

### The crash, and the rdtsc discovery

Under x64dbg it dies with:

```
C000001D  EXCEPTION_ILLEGAL_INSTRUCTION
```

[📷 PASTE screenshot_226 HERE — x64dbg first-chance `C000001D EXCEPTION_ILLEGAL_INSTRUCTION`, RIP parked on the `9A 1D 1A 24…` decoy blob at RVA 0x8ED10]

at a junk blob, RVA `0x8ED10`, where the byte is `9A` (far-call, which is illegal in x64). Those same bytes exist in the file, so it's a **decoy** that the code deliberately jumps into when something goes wrong. It's not a bug, it's a trap door.

Now for the fun part. The **determinism test**:
- Run it straight through, it always crashes at `0x8ED10`.
- But the **moment I pause in the debugger**, the crash *moves*, e.g. into an XXH3 hash that overruns the image and access-violates somewhere else.

Inserting delay changes *where* it dies. That's timing-sensitivity, full stop. The control flow is a function of *how long things took*.

The smoking gun: I scanned `.text` for `rdtsc` (`0F 31`). Only a handful in the whole binary, and the two that matter are at RVA **`0x11912`** and **`0x11946`** — right next to the flattening dispatcher from earlier. The math is worth quoting because it's gorgeous and evil:

```
rdx = 2*(TSC>>8) + (TSC & 0xFF)
```

A hash-mix of TSC bits, stored straight into the dispatcher's state array. In plain English: **the timestamp is woven directly into the control flow.** The CPU's clock literally decides which "state" the flattened dispatcher jumps to next. There's no `if (vm) die()` to find because the timing *is* the branch.

[📷 PASTE screenshot_224 HERE — byte-search results for `0F 31` (rdtsc) showing only a handful of hits, with the two at RVA 0x11912 / 0x11946 highlighted]

[📷 PASTE screenshot_225 HERE — IDA disasm of the rdtsc-fold block at RVA 0x11912 — `rdtsc; ... shr rax,1; mul rsi; shr rdx,5` feeding the dispatcher state array]

Here's the truth table I built:

| Environment | Result |
|---|---|
| Host, no debugger | **Runs** (reaches the password prompt) |
| VM, no debugger | Instant close |
| Host **or** VM, under a debugger | Crash `0x8ED10` |

[📷 PASTE composite HERE — three states side by side: VM instant-close = screenshot_241 (gif) ✅; under-debugger crash = screenshot_226 ✅; host-no-debugger reaching the prompt = still to grab (quick host run)]

Interpretation: in a stock VM, `rdtsc` gets trapped/emulated, so it comes back "wrong" (too slow, or inconsistent), the dispatcher's state computes wrong, and it jumps into the decoy. A debugger adds the same kind of overhead and gets the same punishment. **Bare-metal-with-no-debugger is the only happy path.** So to run it in a VM, I need the VM's `rdtsc` to behave exactly like bare metal.

### Dead ends

Not everything I tried worked, and I think hiding that makes writeups worse, so:

- **The AVX hypothesis.** CRT `__isa_available` reads `2` (SSE4.2, no AVX) inside the VM. So I thought maybe it's an AVX check. Set `VBoxInternal/CPUM/IsaExts/AVX(2)=1`... and `coreinfo` *still* showed `AVX -`. Why? The host **Hyper-V backend** was masking it. Wrong theory, but it taught me the host backend was in the way, which mattered later.

[📷 PASTE screenshot_240 HERE — `coreinfo` in the VM still showing `AVX -` even after setting the IsaExts/AVX extradata — i.e. the Hyper-V backend masking it]
- **TSCMode.** Set `VBoxInternal/TM/TSCMode RealTSCOffset`, no change *at that point*, also because Hyper-V was overriding it. The right idea at the wrong time.
- **The "entropy must cancel" reasoning.** Bare metal runs deterministically every single time *despite* `rdtsc` returning a different value on every run. So I (briefly, wrongly) reasoned that the TSC-entropy must be cosmetic, and therefore there must be one single discrete check I could patch out. Nope. The determinism comes from the timing *deltas* being consistent on real hardware, not from the entropy being fake. Led me down the patching path, which leads us to...

### The self-hash wall

I did actually build a patch. Neutralized both `rdtsc` blocks in the file (`mov rcx,0` + NOPs at file offsets **`0x10D12`** and **`0x10D46`**, using `file = RVA - 0xC00`). Result: `crackme_cracked.exe` runs but immediately exits `0x2712` (10002).

[📷 PASTE screenshot_230 HERE — terminal showing `crackme_cracked.exe` exiting with code 0x2712 / 10002 instead of reaching the prompt (e.g. `echo %errorlevel%`)]

So then I ran the clean experiment to prove what was happening. I flipped **one single byte** in the *never-executed decoy* (RVA `0x8ED10` / file `0x8E110`). That decoy is not on the execution path, nothing ever runs it. And yet: **same `0x2712` exit.**

[📷 PASTE screenshot_231 + screenshot_232 + screenshot_233 HERE — at file offset 0x8E110 the byte is `9A` in the original (231) vs `65` in selfhash_test.exe (232); running it still exits 10002 (233) — proving the broad self-hash catches even a dead-byte edit]

That's the proof. The only way flipping a dead byte can be noticed is a **broad code self-hash** over the whole code section. So **any** static byte-patch, anywhere, gets detected. To patch this thing for real you'd have to defeat the self-hash *and* the TSC-keyed dispatch *and* the decoys, all at once. 

Which is exactly why the cleaner answer is to never touch the binary at all. Fix the environment, not the file.

### The environment defeat

Root cause, one more time: `rdtsc` emulation. The whole game is giving the guest **bare-metal `rdtsc` passthrough** so the timing math comes out identical to real hardware.

**Step 1, find what's actually pinning the hypervisor.** Turns out it wasn't "Hyper-V the role", it was **VBS / Memory Integrity (HVCI)** quietly forcing everything through the Hyper-V backend. Diagnostics you can run to confirm:
- `(Get-CimInstance Win32_ComputerSystem).HypervisorPresent` → `True`
- `Win32_DeviceGuard.VirtualizationBasedSecurityStatus` → `2` (running)
- `Win32_DeviceGuard.SecurityServicesRunning` → `2` (Memory Integrity)

[📷 PASTE screenshot_239 HERE — PowerShell output of the three diagnostics — HypervisorPresent=True, VirtualizationBasedSecurityStatus=2, SecurityServicesRunning=2]

**Step 2, kill it (host, as Administrator), then reboot:**

```
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\HypervisorEnforcedCodeIntegrity" /v Enabled /t REG_DWORD /d 0 /f
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v EnableVirtualizationBasedSecurity /t REG_DWORD /d 0 /f
bcdedit /set hypervisorlaunchtype off
```

What each line is actually doing — this trio **is** the host-side half of why the binary runs in a VM:
- `HypervisorEnforcedCodeIntegrity\Enabled = 0` → turns **Memory Integrity (HVCI)** off.
- `EnableVirtualizationBasedSecurity = 0` → turns off **VBS** — the thing silently force-loading the Hyper-V hypervisor in the first place.
- `bcdedit … hypervisorlaunchtype off` → stops Windows launching its *own* hypervisor at boot, so **VirtualBox gets native VT-x and a real `rdtsc`** instead of the trapped/emulated one. This is the whole point: as long as Windows owns the hypervisor, every VM's `rdtsc` is second-hand and the crackme smells it.

After reboot, `HypervisorPresent` → **False**. VirtualBox now uses native VT-x instead of WHPX/Hyper-V.

[📷 PASTE screenshot_242 HERE — post-reboot PowerShell `HypervisorPresent` → False]

**Step 3... the first VM test still failed.** Yeah. After the reboot `HypervisorPresent` read `False` — the host half was done, Windows had handed the CPU back to VirtualBox — and I fully expected the sample to run. It didn't. The unmodified binary *still* instant-closed in the VM. Native VT-x alone was not enough, and I want that in here because I genuinely thought I was done and I wasn't. **The host fix and the VM fix are two halves of the same lock — `HypervisorPresent=False` is necessary but not sufficient; on its own the sample still dies.**

[📷 PASTE screenshot_243 (gif) HERE — the unmodified sample STILL instant-closing in the VM (cpus=8, Hyper-V already removed) — the "I thought I was done" failure. Pairs with screenshot_244 (gif) below, where the same binary runs fine once cpus=1 + TSC passthrough are applied → a clean before/after of the vCPU fix.]

**Step 4, the decisive piece, vCPU count.** The VM was set to `cpus=8`. Here's the kicker: with multiple vCPUs, VirtualBox **traps `rdtsc` to keep the cores' TSCs in sync.** So every single `rdtsc` becomes a slow VM-exit, which is exactly the overhead the crackme is sniffing for. Drop it to one core:

```
VBoxManage modifyvm "FlareVM" --cpus 1 --paravirtprovider none
VBoxManage setextradata "FlareVM" VBoxInternal/TM/TSCMode RealTSCOffset
```

- `--cpus 1` → no TSC-sync trapping
- `--paravirtprovider none` → no paravirt-clock skew
- `RealTSCOffset` → host-TSC passthrough

[📷 PASTE screenshot_243 (png) HERE — `restore_fixed_VM.cmd` executing the `VBoxManage modifyvm --cpus 1 --paravirtprovider none` + `TSCMode RealTSCOffset` commands]

**Result.** In FlareVM, `coreinfo` now shows `AVX *` / `AVX2 *` (the masking is gone), and the **unmodified** `crackme_NoVM.exe` finally prints:

```
Hold on, getting ready...
You have 10 seconds between keystrokes to enter the password: **********
Let me check........
Incorrect password
```

[📷 PASTE screenshot_228 HERE — `coreinfo` in the VM now showing `AVX *` / `AVX2 *`]

[📷 PASTE screenshot_227 HERE — THE MONEY SHOT: the unmodified crackme_NoVM.exe reaching the password prompt inside FlareVM, with the VM chrome/window title clearly visible to prove it's in a VM]

[📷 PASTE screenshot_229 HERE — HashMyFiles showing crackme_NoVM.exe (the one run in the VM) SHA256 `5E3ABC2D…79B1` matching the original — proving no patch]

That's it. That's the win. The genuine sample, self-hash intact, **same SHA256 as the file I started with**, running inside a VM, reaching its password prompt. Challenge met. No patch, no crack, just a VM convinced it's bare metal.

### The lesson

The anti-VM here was never a string check or a CPUID flag. It was **timing woven into a control-flow-flattening obfuscator**, and the real fight wasn't in the binary at all, it was at the **hypervisor layer** — Hyper-V/WHPX vs native VT-x, and that sneaky multi-vCPU TSC-sync trapping. "Execute *this* sample" is best honored by making the environment honest, and the code self-hash makes that the *only* clean option anyway, so it works out.

Reversible cleanup for anyone following along (this restores WSL2 / Docker / Sandbox, you'll want it back):

```
reg ... Enabled = 1
reg ... EnableVirtualizationBasedSecurity = 1
bcdedit /set hypervisorlaunchtype auto
```
then reboot.

### Appendix — quick reference

<details>

- Image base `0x140000000`; EP `start` `0x1229CC`; CRT→main call `0x140122957` → `0x14007A9D0` (decoy/junk).
- Debug-dir landmine: Type=2 CODEVIEW, `SizeOfData=0x7FFFFF`, ptrs `0x7FFFFFF`. Neutralize by zeroing data dir #6 (file offset `0x1C8`).
- Dispatcher `jmp [r12+r9]+r15` @ `0x1190F`; `rdtsc` dispatch blocks @ `0x11912` / `0x11946`; decoy/crash @ `0x8ED10`; self-hash bail exit code `0x2712` (10002).
- Pointer-encryption keys: `0x4E0253A3BAFBADE9`, `0x688D3FBC21B48D4D`.
- Ciphers/hashes: ChaCha20 (`expand 32-byte k`), XXH3 (`0x9E3779B185EBCA87`, `0xC2B2AE3D27D4EB4F`, `0x61C8864E7A143579`), MurmurHash2 (`0x5BD1E995` / `0x6C1B4395`).
- Patch offset mapping (for the failed patch): `file = RVA - 0xC00`; rdtsc blocks at file `0x10D12` / `0x10D46`; decoy byte at file `0x8E110`.
- Host fix: disable HVCI/VBS + `hypervisorlaunchtype off`. VM fix: `--cpus 1 --paravirtprovider none` + `TSCMode RealTSCOffset` (+ AVX/AVX2 isaexts).

</details>

### #UPDATE — going after the password

Right, so the bounty itself ("execute it in a VM") is **done and dusted**, this whole post is that win. For the record the challenge is **kaganisildak / Malwation's "$1K AntiVM"** off crackmes.one, and it's rated **Insane**, which, having now lived in it for a while, yeah, that's fair. The VM-execution was the $1000; the actual **password** is a separate **+$250 bonus**, and at the time of writing nobody's published a solution to it. So that `Incorrect password` taunt has been living rent-free in my head and i went back in for the bonus.

What follows is the honest state of it: exactly how far i got and the wall i hit. No "trust me bro", only stuff i actually confirmed. I have **not** cracked the password yet, and i'd rather show you the wall than fake a win.

### The target
```
Hold on, getting ready...
→ You have 10 seconds between keystrokes to enter the password: **********
→ Let me check........
→ Incorrect password
```

**10 chars**, it **auto-checks the instant you hit the 10th key**, and there's a **10-second idle timeout per keystroke** (too slow → `No password entered??` and it bails). That timeout is itself an anti-analysis measure, it hard-caps how long you get with a live process.

### Reading the check without a debugger
You can't debug this thing, the same `rdtsc` timing wall + the code self-hash from the main analysis guard the check too (debugger contact → it faults, any byte-patch → exit `10002`). So i went fully **passive** instead, and this part i'm actually proud of:

- `SuspendThread` + `GetThreadContext` + `ReadProcessMemory` is **not** a debugger, no debug port, no DR registers, so it sails straight past the timing trap and lets me read the live process while it runs.
- To actually drive the prompt i inject keystrokes with `WriteConsoleInput` using real VK + scan codes, that's the bit that makes the CRT `_getch` path actually swallow the input. Once that worked the process eats all 10 chars, runs the check, prints `Incorrect password`, and exits clean ~2.85s later. (Quick honesty beat: my first injection harness was silently broken, the input wasn't being consumed at all, so an early "i found hash bytes next to my input" read was garbage. I rebuilt it and re-ran everything, so the findings below are on a check that genuinely ran.)

[📷 PASTE screenshot_235 + screenshot_236 HERE — `condrv.py` running (235) + `condrv_test.log` output (236): `inject ok=True written=20 | EXITED after ~2.85s exitcode=0x0`, i.e. injection works and the check runs to completion]

### What the check actually is
It is **not** a plaintext compare. Driving a known input in and diffing process memory across the check, the expected password is **never** in RAM as plaintext, the only thing that shows up is my own input plus constant decrypted-string noise. So there's nothing to just lift out of memory.

It's also not some simple hash-equals-a-constant. The strings and resources sit under a proper **authenticated-crypto stack** that i pulled apart in IDA:
- **XChaCha20-Poly1305** (HChaCha20 subkey → ChaCha20 → Poly1305 tag verify),
- **XXH3** with a 192-byte custom secret,
- and a chunky (**~88 KB**) **key-derivation routine**,

all sitting on the same ChaCha20 from the main analysis. The input gets run **through** that one-way pipeline, the check is "did this transform/decrypt validate", not "does this string equal that string". I confirmed the input genuinely drives a transform too (different inputs → different internal hash values). In short the password behaves like a **key**, not a stored secret, so there's no plaintext to recover.

[📷 PASTE screenshot_238 HERE — IDA pseudocode of `sub_14010B6F0` (XChaCha20-Poly1305 AEAD): derives the XChaCha20 keystream, verifies the Poly1305 tag (returns `0xFFFFFFFF` on mismatch), then decrypts via `sub_1400A7330` — the authenticated-crypto wall]

### The wall
Here's the part that makes this genuinely Insane-tier and not a one-afternoon job: **the control flow is exception-driven.** Instead of normal `call`/`jmp`, the obfuscator **software-raises a `C000001D` exception** (via `RtlRaiseException`) and a handler decides which block runs next. I caught it red-handed by freezing the process mid-check and reading the exception record it was building, code `0xC000001D`, raised straight out of the dispatcher. That single design choice is *why* IDA xrefs dead-end and *why* debuggers die, and it's the same `C000001D` fingerprint from the main analysis, just software-raised here instead of a decoy fault.

[📷 PASTE screenshot_237 HERE — (paired with the emu shot) `snapshot.py` freezing the process + the `EXCEPTION_RECORD` with `ExceptionCode = 0xC000001D`]

[📷 PASTE screenshot_237 HERE — `emu_check.py` offline Unicorn replay stopping at `*** EXCEPTION RAISED: ZwRaiseException ExceptionCode=0xc000001d`; the `stubs called` line shows `RtlGetExtendedContextLength2` / `RtlInitializeExtendedContext2` / `ZwRaiseException` (the software-raise fingerprint)]

To actually read the password comparison you'd have to **devirtualize that exception dispatcher**, recover its block table, rebuild the real control flow, then walk to the compare. On a pro-built VM-obfuscator that's a multi-day project, not one-more-script, so i'm calling it here for now.

So: the machinery's fully mapped, the password isn't cracked, and that's the honest state of it. Shoutout to kaganisildak, genuinely one of the meanest, best-built things i've taken apart. The $250's still on the table, if/when i devirt this thing i'll update. (Unfinished, stay tuned.)
