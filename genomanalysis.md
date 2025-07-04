# Genom Crypto Traffer Malware Analysis

### Recently, I found someone within the JH Discord (John Hammonds Discord Server), who said they had been infected by some sort of malware that stole all the crypto from his wallet.

![image](https://github.com/user-attachments/assets/a7153624-7b9d-46de-b717-09286c32f4dc)

![image](https://github.com/user-attachments/assets/f6e2e39e-21fa-4ada-8d69-9206ef19ec3d)

### What is a "traffer"?

Before we start, we have to understand the type of actors we are dealing with here, they are known as "traffers":

- *"Traffer" - refers to an individual who specializes in trafficking, managing and monetizing malicious online traffic, often involving stealer malware, crypto drainers, and more. Primarily for financial gain.*

For more information on these types of teams that work within this area of crypto fraud, see this great article from [@g0njxa on Twitter.](https://trac-labs.com/hearts-stolen-wallets-emptied-insights-into-cryptolove-traffers-team-3f65e84ccebe)

### How did this happen?

After I messaged the victim ("_why_so_serious___"), he sent me the link to the "Genom.exe" file he downloaded, this file could also be downloaded from the Discord server that he joined for a supposed "job interview". The websites landing page the traffers were hosting was inside the "#links" channel on the Discord server ("genomgame.com")

![genomdiscord](https://github.com/user-attachments/assets/2ddf518c-2a68-4e94-a9c1-069081089566)

![Screenshot_8](https://github.com/user-attachments/assets/9051d7f2-743b-4b12-89bb-f01945f50e7c)

It's worthy to note that he was "interviewed" for this job as a "beta tester/moderation job"

![joboffer](https://github.com/user-attachments/assets/93d66661-f9c8-4e34-b244-b7560043de0a)

This adds up with the key takeaways from @gonjxa's article, mentioning the use of "job listings" to entice crypto users into downloading malware.

![image](https://github.com/user-attachments/assets/a60ca456-a5a3-411a-9bff-658e6c587d57)

The Discord server itself is a clone of a real crypto NFT collection known by the same name as "Genom", this is their *real* LinkedIn page:

![image](https://github.com/user-attachments/assets/c4ae1164-bb48-462d-bf87-ef03f9fd63ae)

Their website @ "genom2.com" is offline, their GitBook has been suspended, their Rarible account has been removed and their Twitter account has been suspended.

![image](https://github.com/user-attachments/assets/3ed1513d-628f-447d-a537-4ea0ab90d2a1)

![image](https://github.com/user-attachments/assets/d5dfa34d-5a47-44ec-97ae-b3375ecb1cd2)

![image](https://github.com/user-attachments/assets/4a1c9699-a3f1-47ec-9e52-41a904616e42)

![image](https://github.com/user-attachments/assets/a14cbfae-a04e-4e6f-8772-140e821f5192)

However, if we go back to the original Discord server screenshot seen above at the start of this article, you will see a different Twitter account ("@genomthegame"), a different website ("genomgame.com"), and a different GitBook page ("genom-1.gitbook.io/genom" - which has been suspended with the message: "This content has been detected as spam and suspended", I'm not sure of the difference between this message and the legitimate Genom GitBook being suspended with the message "The organization has been suspended", but one of them was suspended, and the other, ***the fake one*** was suspended for spam. **¯\\___(ツ)___/¯** )

![image](https://github.com/user-attachments/assets/f9313e97-bd89-46d2-86f6-e17cc7574783)

![Screenshot_8](https://github.com/user-attachments/assets/9051d7f2-743b-4b12-89bb-f01945f50e7c)

![image](https://github.com/user-attachments/assets/1b4b7af3-e879-4786-9686-8a3287aaa10c)

### Overall, the current chain of events is the following:

- The victim was looking for a job, and unfortunately found a fake crypto NFT collection that was advertising job listings for "beta tester/moderation job" roles.
- The victim was interviewed in a seemingly "legitimate" interview for the job position. *(After all, who would do a "legitimate" job interview just to steal crypto?)*.. ***sarcasm***
- The victim was directed to their fake website (genomgame.com), which allowed him to sign up and create an account, then allowed him to download the Genom game
- The site redirected to a direct download link which downloaded "Genom.exe"

![image](https://github.com/user-attachments/assets/019db15a-6a90-4cec-b717-7fd708999aa9)

***(Yes, I know Dropbox link in the screenshot downloads a file called "genom.exe" with no capital "G", this is because the traffers have updated the file since I analysed it)***

### Analysing Genom.exe

Now for the fun part.

Based on my analysis, Genom.exe initiates a multi-stage infection chain that leverages a legitimate-looking update process to deploy its payload. It extracts "SmartUpdateHelper.exe" alongside a malicious Electron app.asar file, which triggers a GET request to an external server to download and execute additional malware. The infection proceeds by dropping DLLs, memory injection, and syscall manipulation, ultimately injecting code into the legitimate explorer.exe process to establish communication with a C2 server. The final payload is likely REMCOS and/or a stealer malware such as LummaC2 or Rhadamanthys, enabling remote access and data exfiltration.

![flow](https://github.com/user-attachments/assets/5c12fa03-18c8-455b-87f7-6613dda04df6)

<p align="center"><strong>Genom.exe Infection Chain</strong></p>



After getting the file and throwing it into DiE, I got this output:

![image](https://github.com/user-attachments/assets/fea72f76-7c3b-4f2a-9889-f01cfc46ce89)

Since the file is a **Nullsoft Scriptable Install System** file, we can use Universal Extractor to extract the files within.

Once extracted, we can see 4 folders with some strange names:

![image](https://github.com/user-attachments/assets/62088ba8-5c9d-448a-8171-44780c06b261)

3 out of 4 of these folders include some useless stuff, mainly random DLLs and an uninstall.exe for the legitimate software this malware pretends to, but one of the folders, specifically `_ﾕ`, contains another folder called `_ﾚ`, which contains "app-64.7z"

Inside this 7z archive is a couple files and folders:

![image](https://github.com/user-attachments/assets/98c49756-95ad-4149-86a6-ddea8d221b3b)

Now, from my research, the "SmartUpdateHelper.exe" seems to be legitimate, and the malware is actually inside of the "resources" folder. Here we have the following two files:

![image](https://github.com/user-attachments/assets/08427570-4f33-4a56-b63f-3488dd3ee78f)

Regarding "app.asar", my Electron nerds will know about that and we'll get into it in a moment. However, if you don't know about "elevate.exe", this is a third-party tool commonly used by malware to execute other processes such as CMD with administrator privileges. Basic stuff, overall, we are interested in the "app.asar" file.

Extracting this "app.asar" file gives us a few JS files of interest, one of them specifically is named `main.js`, looking into this file we can see a simple download and execution script:

![image](https://github.com/user-attachments/assets/013576b3-7ce6-4d49-8118-0e164b040169)

The script also brings up a fake Cloudflare captcha page, most likely used as a distraction:

![image](https://github.com/user-attachments/assets/f984a40f-daef-4641-9666-a20e816866d2)

After sending a GET request to the `https://smartcloudnetwork.com/request` link, we get a downloaded file called `calc.exe`.

### Analysing calc.exe

![image](https://github.com/user-attachments/assets/1a25503f-933f-4367-a3ba-9918684abd46)

This is the DetectItEasy report for `calc.exe`, clearly it is an MSI installer file, therefore, using [lessmsi](https://lessmsi.activescott.com/) we can analyse and extract the files within th installer:

![image](https://github.com/user-attachments/assets/532df752-fdaf-4d53-9470-7d1b2755726d)

Here we can see multiple files, including one "ISDbg.exe" which is particularly interesting. There are also a few DLLs here that were also dropped, primarily `ISUIServices.dll` and `MSIMG32.dll`

Both of these DLLs are malware, but are used later in the chain. For now, the "ISDbg.exe"/"calc.exe" injects itself into the `explorer.exe` process, and communicates with a new C2.

```C2 = cloudhostcdn.net (IP: 185.193.126.198)```

**Previous fingerprints have indicated this IP address has been associated with other stealer campaigns such as Lumma Stealer, PureLog Stealer, DBatLoader, Masslogger RAT and Raccoon Stealer**

![image](https://github.com/user-attachments/assets/b49def14-8ed4-418b-bf47-bab9bc5f5bbf)

Futhermore, dropped files previously indicate that Remcos RAT was used in this campaign or a previous campaign:

![image](https://github.com/user-attachments/assets/7a1046b8-39a0-4256-8e76-5956dc97af39)

After executing, the malware establishes admin permissions by using a CMSTPLUA COM exploit: [POC & more information here](https://gist.github.com/api0cradle/d4aaef39db0d845627d819b2b6b30512) & [here](https://g3tsyst3m.github.io/privilege%20escalation/Creative-UAC-Bypass-Methods-for-the-Modern-Era/)



