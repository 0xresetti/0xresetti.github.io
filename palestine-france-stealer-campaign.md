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

### Regarding The File, named ```321chat.exe``` has the following characteristics:

- **32-bit Executable**
- **MD5 Hash:** ```4aec2a150e4135f61dff2e55de07b9e9```
- **File Description: "Usefull Application"** (??? I thought this was funny)
- **File is a Nullsoft Scriptable Install System (NSIS) type file**

I don't really know where to start with this malware, it is a stealer malware first of all, but not one that I have seen before. The hash has never been uploaded to VirusTotal and it is Fully Undetected: https://www.virustotal.com/gui/file/c7b11a2b9719590183eb1985f61b7b5b11b8bdead56003c65279d9a7c47b728a

Some new, unique malware! Just asking to be analysed. Let's get started.

I used ProcDot to visualize the malware layout and how it executes, and from my analysis, the file executes multiple child processes of itself, the most interesting one being ```PID: 4320```

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/3ea431b5-8be5-4df3-9def-7d147368fd1b)

This ```PID: 4320``` directly creates a thread with ```ID: 6184```, this thread is used for creating the persistence registry keys, and creating other threads that are used for stealing the users information and creating the final ZIP file which is eventually sent to the C2.

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/680da865-b93d-46fc-9572-2c7e441204b3)

^^^ ```TID: 6184``` creating a persistent registry key ^^^

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/cc2eaa4c-aae0-48d4-95f6-b6c7979801cb)

^^^ ```TID: 6184``` creating 4 separate threads ready for writing the stolen information into a ZIP file that will be sent to the C2 ^^^

Using ProcDot, we can actually target specific child processes and see what they were doing, since we are focusing on the stealer malware, I will change the render configuration to only show threads and processes created by ```PID: 4320```

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/4f51a841-ca9e-45c8-88a4-b1627767cea3)

After loading this specific PID, we can see some DLLs being loaded, some communications being made to some servers, one of them specifically stood out to me, ```185.199.117.133:443```. The ```TIDs: 7068, 9708, 2984, and 7952``` were being used to download a zip file called ```"extensions.zip"```. Eventually, ```TIDs: 7068 and 7952``` both start reading data from the ZIP file then start writing data to the following folders:

```C:\ProgramData\ChromeExtensionsNova\extension-cookies```

```C:\ProgramData\ChromeExtensionsNova\extension-tokens```

These folders both contain .js files which are used to steal Discord tokens and information:

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/bf171b43-d4ff-4826-a578-e3d87403a07e)

^^^ ```background.js``` script found inside ```C:\ProgramData\ChromeExtensionsNova\extension-tokens\js\``` ^^^

The script also shows some information about the type of stealer that is being used:

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/7a4c33f5-f22f-4073-ae29-28072983e74f)

As you can see, they have a Telegram: [https://t.me/Sordeal](https://t.me/Sordeal)

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/3a8fd3b8-259c-42c6-9c75-cd2d0dacd5ad)

Looks like this one specifically is "Hawkish Grabber" considering the ```webhook_url``` variable. A French stealer.

The other folder, ```C:\ProgramData\ChromeExtensionsNova\extension-cookies``` is a folder specifically targeting Roblox accounts and cookies, it then sends the data to the C2:

![image](https://github.com/0xresetti/0xresetti.github.io/assets/114181159/76a298ee-25cd-4f7d-9a04-61c75b61f391)

^^^ ```background.js``` script found inside ```C:\ProgramData\ChromeExtensionsNova\extension-cookies\scripts\``` ^^^

