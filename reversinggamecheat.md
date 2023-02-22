# First blog: Reversing a "Game Cheat" ;)

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

First binary: **```LEMON.exe```**
- Microsoft Visual C/C++ Compiled.
- MD5: ```6d5f74f263d5ab9b0e3315b495eb72d5```
- Heavily Obfuscated, lots of Anti-Analysis & Anti-Debugging techniques.

Unfortunately, I don't have IDA Pro and I'm still learning when it comes to going deep into malware with reverse engineering and I wasn't able to understand much to do with this file, however, thankfully websites like ```app.any.run``` exist! This website does live dynamic analysis on files, making it easy for anyone to understand what is going on.

Turns out the file was a build of **Rhadamanthys Stealer**, a new Stealer malware designed to steal Crypto Coins, System Information, and execute separate processes such as Powershell.

[You can take a look at the Any.Run scan here (click me!)](https://app.any.run/tasks/2e3aea94-e1a3-4dab-95fd-1ec803aae2ef)

Second binary: **```LEM.exe```**
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

Third binary: **```LicGet.exe```**
- .NET v4.0 Executable
- MD5: ```2b125292307de39b8be71d73a8eb2f8f```
- Requires Admin privileges to execute.

Opening the file in dnSpy, it is a fairly simple and small file with not much happening. First, the file checks to see if it is running as Administrator or not, given that the file prompts UAC when opened, most unknowing people would click Yes. This will give the process Administrator privileges, and then move to the ```jumptoSys``` function. If you click No on the UAC prompt, the file moves to the ```Disable``` function. 

<img src="/images/checkadmin.png" alt= "checkadmin" width="70%" height="70%">

Looking at the ```jumptoSys``` function, it's concept is to use the WinAPI's ```GetProcessByName``` method to get the Process Token of the ```winlogon``` process, which runs with Administrator privileges. It then duplicates this token and creates a new process using the duplicated token, this is probably to migrate the **LicGet.exe** process to a less obvious one, since ```winlogon``` is known to run with Administrator privileges.

<img src="/images/jumptosysfunc.png" alt= "jump2sys" width="70%" height="70%">

On the other hand, looking at the ```Disable``` function, it's concept is slightly different, it again uses the WinAPI's ```GetProcessByName``` method to get the Process Token, but instead of targeting the ```winlogon``` process, it targets the ```MsMpEng``` process, this process is the core process of the Windows Defender Anti-malware Application, and also runs with Administrator privileges. 

It looks like the ```Disable``` functions job is to emulate the ```MsMpEng``` process token, then allocate new memory and set the process token information to the same process token as the ```MsMpEng``` process, giving the **LicGet.exe** process Administrator privileges.

<img src="/images/disablefunc.png" alt= "jump2sys" width="70%" height="70%">

By the way, I could be completely wrong with this process, but thats what it looks like haha, as I say I'm still learning.

Both functions use the ```advapi32.dll``` DLL to perform these token impersonation actions.

Apart from that, there isnt much else, I think this is just used for privilege escalation. There is also a ```GetProcessOwner``` function, which from what I can see, it's in the name lol, it just gets the owner of the processes that it's targeting. 

Fourth binary: **```ezzzzzzz.exe```**

**Stay tuned! I'm still working on this blog!**
