# Running a VM-Hostile Crackme Inside a VM

### So I picked up a reversing bounty, and the challenge sounded simple enough:

*"Successfully execute this sample inside a VM and publish a technical blog explaining how you did it."*

That's it. Run it in a VM. Except the second you drop it in a VM, it instantly closes on you. No crash dialog, no error, nothing, it just dies.

<img width="2148" height="1224" alt="Screenshot_234" src="https://github.com/user-attachments/assets/870ba811-afc7-42f6-b070-36ee1dbd43be" />

The interesting part is that the way to beat this isn't to patch the binary. The challenge says "execute *this* sample", and the binary self-hashes itself anyway (more on that later), so the second you change a byte it dies. Instead, I had to make the VM look enough like real hardware that the original, unmodified binary runs on its own.

The sample:
- `crackme_NoVM.exe`, 64-bit MSVC console app, 1.5MB
- SHA256 `5E3ABC2D38A41E0DF846026C5E41690EDFF0498258D4594355CC633A062679B1`
- Image base `0x140000000` (it's ASLR'd at runtime so I'll quote everything as `base + RVA`)

Keep that SHA in mind, because at the end the file we successfully run inside a VM has the **exact same hash**. No patch.

### First contact: IDA won't even open it

Loading it into IDA, immediately:

<img width="439" height="164" alt="Screenshot_216" src="https://github.com/user-attachments/assets/ee0a00f5-8052-456d-a3d3-ab92ac45a3bd" />

`8388607 = 0x7FFFFF`, `134217727 = 0x7FFFFFF`

Hmmmmm...

Instead of trusting the tool I parsed the PE **by hand**. The PE header itself is clean, 6 normal sections, entry point `0x1229CC`, no section runs past end of file.

The landmine is in the **Debug Directory**. The first entry is a forged `IMAGE_DEBUG_TYPE_CODEVIEW` (Type=2) with:
- `SizeOfData = 0x7FFFFF`
- `AddressOfRawData = 0x7FFFFFF`
- `PointerToRawData = 0x7FFFFFF`

<img width="1259" height="220" alt="Screenshot_217" src="https://github.com/user-attachments/assets/1e5809d5-0f38-48f5-b304-27b02fa72479" />

IDA follows the debug dir to load symbols, tries to read a `0x7FFFFF` blob at file offset `0x7FFFFFF`, and it doesnt work. Windows ignores the debug directory at load time, so the sample still runs fine. It's malformed on purpose to break RE tools, not Windows, which is a theme throughout this whole sample.

### Recon: what are we even dealing with

<img width="725" height="554" alt="Screenshot_221" src="https://github.com/user-attachments/assets/c6d9077c-3a43-4084-a915-45fec7d95399" />

- Only **KERNEL32** imported. That's it.

- 428 strings, and they're *all* CRT boilerplate. Zero VM-vendor strings. I searched for `vmware`, `vbox`, `qemu`, all of it, nothing. And that absence is itself a clue, more on that in a sec.

<img width="731" height="203" alt="Screenshot_219" src="https://github.com/user-attachments/assets/51fd8f49-c455-46d4-9a2c-8a0845ffd915" />

<img width="667" height="207" alt="Screenshot_220" src="https://github.com/user-attachments/assets/06d0309a-3054-44a3-8fe2-d65bb5b013f6" />

- Anti-debug imports are sitting right there though: `IsDebuggerPresent`, `Get/SetThreadContext`, a pile of timing APIs (`GetTickCount64`, `QueryPerformanceCounter`, `GetSystemTimeAsFileTime`), plus `TerminateProcess`/`ExitProcess`.

<img width="687" height="760" alt="Screenshot_218" src="https://github.com/user-attachments/assets/c1589382-de02-464c-827b-256ab6a23218" />

So with **no registry, file, device, firmware or network APIs**, and **no vendor strings**, in my mind whatever environment check this thing is doing has to be **CPU-level or timing based**.

### The obfuscation

Mr kaganisildak knows what he's doing.

- **Control-flow flattening.** There's a dispatcher at RVA `0x1190F`:
  ```
  mov r8,[r12+r9]
  add r8,r15
  jmp r8
  ```
  That's `jmp stateTable[state] + base`. The real control flow lives in a state variable.

<img width="308" height="627" alt="Screenshot_222" src="https://github.com/user-attachments/assets/4ead6913-ca5d-4a2e-93d9-b502e2d12acc" />

- **Opaque predicates** everywhere, e.g. `(x*(x+1)) & 1` which is *always* 0 (product of two consecutive ints is always even). Fake branches.
- **Pointer encryption.** Real pointers are recovered as `*(global) - 0x4E0253A3BAFBADE9` and `global - 0x688D3FBC21B48D4D` (64-bit additive keys). Decompilers render these as nonsense huge offsets.
- **Anti-disassembly junk bytes** and a **decoy `main`.** IDA's `main` symbol points at `0x14007A9D0`, which is 55 junk bytes starting with `0x3F` (invalid in x64) followed by `CC` padding.

<img width="962" height="176" alt="Screenshot_223" src="https://github.com/user-attachments/assets/f184f098-46b9-4cfe-b5ab-5e0b64fd0900" />

I briefly mis-computed the CRT's `call main` target and went chasing a fake function for a bit. The lesson I took away is **trust the actual call bytes over the tool's heuristic symbol.** The CRT actually calls main at `0x140122957`, the bytes are `E8 74 80 F5 FF`. The decoy symbol was a trap that i walked into.

- **Encrypted strings**, decrypted on demand. That's why static string search came up blind earlier. I later confirmed the cipher is **ChaCha20**, the `expand 32-byte k` constant shows up in memory.
- Two real hash primitives baked in:
- **XXH3** at `sub_140093230` (constants `0x9E3779B185EBCA87`, `0xC2B2AE3D27D4EB4F`, `0x61C8864E7A143579`)
- a **MurmurHash2** PRNG seed at `sub_140096800` (`0x5BD1E995` / `0x6C1B4395`, mixing `rdtsc ^ GetTickCount64 ^ rand`). Note the `rdtsc` showing up in the seed... (foreshadowing)
- **Code self-hash / anti-tamper.** I proved this later but I'll spoil it now: edit *any* byte and the process exits `10002`, which isn't a Windows error, its the value this crackme spits out when its own integrity check fails. Making patching useless.

### Ruling out the usual anti-VM

Before going down the timing rabbit hole I wanted to rule out the usual stuff:

- No **VMware check**, the `VMXh` magic (`68 58 4D 56`) is absent.
- No **descriptor red-pills**, no `sidt`/`sgdt`/`sldt`/`smsw` anywhere.
- The only real **CPUID** routine is MSVC's `__isa_available_init` (vendor "GenuineIntel"). It never tests the hypervisor bit (`ECX[31]`) or leaf `0x40000000`. So it's not sniffing the hypervisor via CPUID either.
- No active **TLS callbacks**, the callback array starts with a NULL.
- No custom kill-switch. `ExitProcess`/`TerminateProcess` are CRT-only, so the anti-VM **branches** somewhere, it doesn't cleanly bail with a dedicated "die" function. Which means I have to find the branch.

So all the off-the-shelf detections are out. It's timing. It has to be timing. I just didn't have proof yet.

### Dynamic analysis

I tried **ScyllaHide** with everything turned on, all hooks + timing-API hooks + KiUserExceptionDispatcher. Didn't help even slightly.

That null result actually told me something useful: ScyllaHide hooks *APIs*, it can't touch the `rdtsc` **instruction** itself. So the fact that it changed nothing pointed me at the timing check being `rdtsc`-based rather than something hookable.

### The crash, and the rdtsc discovery

Under x64dbg it dies with:

```
C000001D  EXCEPTION_ILLEGAL_INSTRUCTION
```

<img width="736" height="830" alt="Screenshot_226" src="https://github.com/user-attachments/assets/9ac7b65c-35de-44d2-8809-ad8ebc2b648b" />

at a junk blob, RVA `0x8ED10`, where the byte is `9A` (far-call, illegal in x64). Those same bytes exist in the file, so it's a **decoy** the code jumps into on purpose when something goes wrong.

Now for the fun part:
- Run it straight through, it always crashes at `0x8ED10`.
- But the **moment I pause in the debugger**, the crash *moves*, e.g. into an XXH3 hash that overruns the image and access-violates somewhere else.

Inserting delay changes *where* it dies, so the control flow depends on how long things take. It's timing-sensitive.

So I scanned `.text` for `rdtsc` (`0F 31`). Only a handful in the whole binary, and the two that matter are at RVA **`0x11912`** and **`0x11946`**, right next to the flattening dispatcher from earlier. The math behind it:

```
rdx = 2*(TSC>>8) + (TSC & 0xFF)
```

It mixes the TSC bits and stores the result straight into the dispatcher's state array. So the timestamp is feeding directly into the control flow, the clock value decides which block the flattened dispatcher jumps to next. There's no `if (vm) = die()` to find anywhere, the timing itself is the check.

<img width="976" height="189" alt="Screenshot_224" src="https://github.com/user-attachments/assets/185cdf6c-bd9c-4054-b44d-d5a95f2834a3" />

<img width="884" height="672" alt="Screenshot_225" src="https://github.com/user-attachments/assets/78712367-6b79-45f0-8ad7-ab527c1aadc9" />

So, here's the current situation:

| Environment | Result |
|---|---|
| Host (No VM), no debugger | **Runs** (reaches the password prompt) |
| VM, no debugger | Instant close |
| Host **or** VM, under a debugger | Crash `0x8ED10` |

<img width="800" height="495" alt="Screenshot_241" src="https://github.com/user-attachments/assets/f782967d-5b40-4c6f-b9fa-0f5b561cf934" />

In a stock VM, `rdtsc` gets trapped/emulated, so it comes back wrong (too slow, or inconsistent), the dispatcher's state computes wrong, and it crashes. A debugger adds the same kind of overhead so it breaks the same way. Bare metal with no debugger is the only setup that works. So to run it in a VM, I need the VM's `rdtsc` to behave exactly like bare metal.

### Dead ends

Not everything I tried worked, but I'll leave these in anyway:

- **The AVX hypothesis.** CRT `__isa_available` reads `2` (SSE4.2, no AVX) inside the VM. So I thought maybe it's an AVX check. Set `VBoxInternal/CPUM/IsaExts/AVX(2)=1`... and `coreinfo` *still* showed `AVX -`. Why? The host **Hyper-V backend** was masking it. Wrong theory, but it taught me the host backend was in the way, which mattered later.

<img width="762" height="433" alt="Screenshot_240" src="https://github.com/user-attachments/assets/96eb9ece-d6b5-4071-8223-ca25db89ca8e" />

- **TSCMode.** Set `VBoxInternal/TM/TSCMode RealTSCOffset`, no change *at that point*, also because Hyper-V was overriding it. The right idea at the wrong time.
- Bare metal runs deterministically every single time *despite* `rdtsc` returning a different value on every run. So I (briefly, wrongly) reasoned that the TSC-entropy must be cosmetic, and therefore there must be one single discrete check I could patch out. This didn't work. The determinism comes from the timing *deltas* being consistent on real hardware, not from the entropy being fake. Led me down the patching path, which leads us to...

### The self-hash wall

I did actually build a patch. Neutralized both `rdtsc` blocks in the file (`mov rcx,0` + NOPs at file offsets **`0x10D12`** and **`0x10D46`**, using `file = RVA - 0xC00`). The result is: `crackme_cracked.exe`, which runs, but immediately exits with that `10002` code again.

<img width="977" height="150" alt="Screenshot_230" src="https://github.com/user-attachments/assets/d947badd-15f1-4117-8c0f-dbfebc6acf96" />

So then I ran the clean experiment to prove what was happening. I flipped **one single byte** in the *never-executed decoy* (RVA `0x8ED10` / file `0x8E110`). That decoy is not on the execution path, nothing ever runs it. And yet: **same `10002` exit.**

<img width="160" height="62" alt="Screenshot_231" src="https://github.com/user-attachments/assets/ec021282-58ed-4d55-81df-f4f9e63d0ae2" />

<img width="139" height="68" alt="Screenshot_232" src="https://github.com/user-attachments/assets/1a302994-8093-4be6-9a32-b01919f0f70b" />

<img width="901" height="150" alt="Screenshot_233" src="https://github.com/user-attachments/assets/e0c0e97b-5b62-4843-9952-54367930dae4" />

That's the proof. The only way flipping a dead byte can be noticed is a **broad code self-hash** over the whole code section. So **any** static byte-patch, anywhere, gets detected. To patch this thing for real you'd have to defeat the self-hash *and* the TSC-keyed dispatch *and* the decoys, all at once. 

Which is exactly why the cleaner answer is to never touch the binary at all. Patch the environment, not the file.

### The environment defeat

Root cause again: `rdtsc` emulation. The goal is to give the guest **bare-metal `rdtsc` passthrough** so the timing comes out the same as real hardware.

**Step 1, find what's actually pinning the hypervisor.** Turns out it wasn't "Hyper-V the role", it was **VBS / Memory Integrity (HVCI)** quietly forcing everything through the Hyper-V backend:

<img width="966" height="265" alt="Screenshot_239" src="https://github.com/user-attachments/assets/d2f26132-9dea-4142-a747-e812f93f658f" />

**Step 2, kill it (host, as Administrator), then reboot:**

```
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\HypervisorEnforcedCodeIntegrity" /v Enabled /t REG_DWORD /d 0 /f
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v EnableVirtualizationBasedSecurity /t REG_DWORD /d 0 /f
bcdedit /set hypervisorlaunchtype off
```

What each line is actually doing — this trio **is** the host-side half of why the binary runs in a VM:
- `HypervisorEnforcedCodeIntegrity\Enabled = 0` > turns **Memory Integrity (HVCI)** off.
- `EnableVirtualizationBasedSecurity = 0` > turns off **VBS** — the thing silently force-loading the Hyper-V hypervisor in the first place.
- `bcdedit … hypervisorlaunchtype off` > stops Windows launching its *own* hypervisor at boot, so **VirtualBox gets native VT-x and a real `rdtsc`** instead of the trapped/emulated one. As long as Windows owns the hypervisor, every VM's `rdtsc` is second-hand, and the crackme picks up on it.

After reboot, `HypervisorPresent` > **False**. VirtualBox now uses native VT-x instead of WHPX/Hyper-V.

<img width="758" height="230" alt="Screenshot_242" src="https://github.com/user-attachments/assets/b45e82ae-4c1c-405b-914d-6fc92da1c6d6" />

**Step 3... the first VM test still failed.** After the reboot `HypervisorPresent` read `False`, so the host side was done and Windows had handed the CPU back to VirtualBox. I expected the sample to run at this point. It didn't, the unmodified binary *still* instant-closed in the VM. So turning off Hyper-V on its own wasn't enough, you also need the VM-side fix, which I figured out next.

<img width="800" height="572" alt="Screenshot_243" src="https://github.com/user-attachments/assets/412da909-c2db-42e6-bea7-47b7b1d3232f" />

**Step 4, the decisive piece, vCPU count.** The VM was set to `cpus=8`. Here's the kicker: with multiple vCPUs, VirtualBox **traps `rdtsc` to keep the cores' TSCs in sync.** So every single `rdtsc` becomes a slow VM-exit, which is exactly the overhead the crackme is sniffing for. To fix, simply drop it to one core:

```
VBoxManage modifyvm "FlareVM" --cpus 1 --paravirtprovider none
VBoxManage setextradata "FlareVM" VBoxInternal/TM/TSCMode RealTSCOffset
```

- `--cpus 1` > no TSC-sync trapping
- `--paravirtprovider none` > no paravirt-clock skew
- `RealTSCOffset` > host-TSC passthrough

<img width="513" height="234" alt="Screenshot_243" src="https://github.com/user-attachments/assets/b8c1ab57-9ebf-4e2d-8806-c247deadfca2" />

**Result.** In FlareVM, `coreinfo` now shows `AVX *` / `AVX2 *` (the masking is gone), and the **unmodified** `crackme_NoVM.exe` finally prints:

```
Hold on, getting ready...
You have 10 seconds between keystrokes to enter the password: **********
Let me check........
Incorrect password
```

<img width="758" height="243" alt="Screenshot_228" src="https://github.com/user-attachments/assets/3886041e-5922-45bd-abc4-34e9d273ec6e" />

<img width="800" height="540" alt="Screenshot_244" src="https://github.com/user-attachments/assets/d6d9d33a-ba18-4ffc-b57f-9040293be77a" />

<img width="830" height="169" alt="Screenshot_229" src="https://github.com/user-attachments/assets/10d1d550-625d-4a0f-b3de-0e47fe88fe09" />

**That's the win.** The original sample, self-hash intact, **same SHA256 as the file I started with**, running inside a VM and reaching its password prompt. No patching, no cracking, I just made the VM look like bare metal.

### The lesson

The anti-VM here wasn't a string check or a CPUID flag, it was **timing folded into a control-flow-flattening obfuscator**, and the actual problem was at the **hypervisor layer** instead of in the binary, Hyper-V/WHPX vs native VT-x plus the multi-vCPU TSC-sync trapping. And since the binary self-hashes itself, fixing the environment instead of the file is the only clean way to run it anyway.

### Going after the password

The bounty itself ("execute it in a VM") is all done, this whole writeup is that win. The VM-execution was the $1000; the actual **password** is a separate **+$250 bonus** (which kaganisildak kindly offered to me after I emailed him a screenshot of his binary running inside my VM).

What follows is the honest state of it: I didn't manage to get the password, but i'll show how far i got and the wall i hit.

### The target
```
Hold on, getting ready...
> You have 10 seconds between keystrokes to enter the password: **********
> Let me check........
> Incorrect password
```

The interesting part is the password, which is **max 10 chars**, it **auto-checks the instant you hit the 10th key**, and there's a **10-second idle timeout per keystroke** (too slow > `No password entered??` and it dies). That timeout is itself an anti-analysis measure, it hard-caps how long you get with a live process.

### Reading the check without a debugger
You can't debug this thing, the same `rdtsc` timing wall + the code self-hash from the main analysis guard the check too (debugger contact > it faults, any byte-patch > exit `10002`). So i went fully **passive** instead, and this part i'm actually proud of:

- `SuspendThread` + `GetThreadContext` + `ReadProcessMemory` is **not** a debugger, no debug port, no DR registers, so it sails straight past the timing trap and lets me read the live process while it runs.
- To actually drive the prompt i inject keystrokes with `WriteConsoleInput` using real VK + scan codes, that's the bit that makes the CRT `_getch` path actually swallow the input. Once that worked the process eats all 10 chars, runs the check, prints `Incorrect password`, and exits clean roughly 3s later.

<img width="1668" height="223" alt="Screenshot_235" src="https://github.com/user-attachments/assets/cee5471e-db28-4dcd-a207-31d34fe54eb7" />

<img width="999" height="173" alt="Screenshot_236" src="https://github.com/user-attachments/assets/d5e1c97c-27d2-42ff-9c5f-ba42156dc38f" />

### What the check actually is
It is **not** a plaintext compare. Driving a known input in and diffing process memory across the check, the expected password is **never** in RAM as plaintext, the only thing that shows up is my own input plus constant decrypted-string noise. So there's nothing to just lift out of memory.

It's also not some simple hash-equals-a-constant. The strings and resources sit under a proper **authenticated-crypto stack** that i pulled apart in IDA:
- **XChaCha20-Poly1305** (HChaCha20 subkey > ChaCha20 > Poly1305 tag verify),
- **XXH3** with a 192-byte custom secret,
- and a chunky (**88 KB**) **key-derivation routine**

The input gets run **through** that one-way pipeline, the check is "did this transform/decrypt validate", not "does this string equal the correct string". I confirmed the input genuinely drives a transform too (different inputs > different internal hash values). In short the password behaves like a **key**, not a stored secret, so there's no plaintext to recover.

<img width="1273" height="862" alt="Screenshot_238" src="https://github.com/user-attachments/assets/4221c6d2-a1db-49cf-8c3f-1853d73f7f77" />

### The wall
This is the part that makes it difficult, the control flow is exception-driven. So instead of normal `call`/`jmp`, the obfuscator **software-raises a `C000001D` exception** (via `RtlRaiseException`) and a handler decides which block runs next. I caught it red-handed by freezing the process mid-check and reading the exception record it was building, code `0xC000001D`, raised straight out of the dispatcher. That single design choice is *why* IDA xrefs dead-end and *why* debuggers die, and it's the same `C000001D` fingerprint from the main analysis, just software-raised here instead of a decoy fault.

<img width="1098" height="372" alt="Screenshot_237" src="https://github.com/user-attachments/assets/f50f431f-7909-4d7a-9365-6a03d2ba4236" />

To actually read the password comparison you'd have to **devirtualize that exception dispatcher**, recover its block table, rebuild the real control flow, then walk to the compare.. Which I might look into in the future, but I was quite excited to get this running in a VM in the first place, so I'll take my cake while its on the table :)

Thanks for reading, and thanks to [@kaganisildak](https://x.com/kaganisildak) for the challenge!
