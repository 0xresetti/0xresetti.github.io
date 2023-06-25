# WhiteSnake Stealer Analysis

### I'm drunk and this is my analysis on WhiteSnake Stealer. My friend Eliyax wouldn't stop bothering me to analyse it with him so here it is.

### Shoutout to my cat Vince for giving me the emotional support while reversing this shit

First off we got a fuckin .NET binary !!!

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/68fa4e0c-6bcb-48fb-9fac-5c4fd0d6a44a)

How fun and enjoyable, no Ghidra required.

After initally opening the binary, everything is obfuscated and shit, so i used ```de4dot``` to unobfuscate it and decode the strings:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/e686eef4-1bc5-4d11-8336-5b3e1c6905ac)

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/7f24a3c2-27f2-405e-ba41-360b7e04cf35)

After "cleaning", the file is now decoded and we can view the strings:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/93128590-da74-46d8-89a8-3c15f58a8059)

A lot of the function names or whatever the fuck the shit in the left side of DnSpy is called is encoded/obfuscated with random names, so anything that i mention will come from this list:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/6b763c0f-6236-43cd-8348-8eade4b282f4)

btw, you will see a lot of dissing to ESET in this malware, lol

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/f16df5b2-87a1-4810-94ca-c5cb0b4aeba3)

The malware seems to create a ```schtask``` for something, i would assume its for persistence.

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/99b9d22d-b277-4e4f-abf2-9bf3debebd5a)

Also, the malware seems to make a bunch of random requests to random websites like ```google.kz```, ```blog.cyble.com```, and ```cyware.com```

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/63c6dfe4-bb28-4e0c-82b2-dd88ad5ca489)

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/67e0310a-c6f6-4ce5-a19a-6a2fd6401131)

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/602c9d54-f9e9-4fe4-bd29-6479c06af4f8)

I haven't made a direct connection yet, but i believe the parameters used on the end of these requests are encoded variables or code used by the malware

For example, here is some encoded strings that ```Process.Start``` calls:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/333ed992-8706-4dee-a9b1-232bc0b74f9a)

Then here is one of the ```webClient.DownloadString``` functions:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/2a6fd033-2068-47ca-b168-6b59c55a4ad2)

See how the ```1Mxm1dRnMr``` is similar to the ```Process.Start``` strings? I cant find a connection, but i assume this is some sort of evasion method which is unique to me, i havent seen it before

The malware also downloads TOR for some reason? I messaged the WhiteSnake developer himself and i asked if it uses TOR, and he confirmed it does! What a nice threat actor. I assume TOR is used for uploading the data:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/0027fee7-506f-4bb1-b384-59cf5f385734)

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/eb161d95-7510-4a40-9708-b928346fff22)

```mA2Z``` seems to be where most of the functionality calls for the stealer are located, for example:

Here is the malware compress function, it compresses data into a ZIP file, and again we see it using the ```webClient.DownloadString``` "obfuscation" that we saw earlier:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/dd606743-9ca8-4c0d-80a9-48644952d06f)

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/4c7ea9eb-417c-4ff8-a36c-46cea43adf7d)

We also have a ```WEBCAM``` call function, i assume to record the webcam or screenshot it:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/b15cde53-9b23-4deb-ad20-3ddd19df14ee)

Then of course, we have a ```KEYLOGGER``` function.

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/f64f4692-d647-4138-96e4-8e410ca2aca9)

We also have a ```LIST_PROCESSES``` function.

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/e170f092-d424-4916-94d6-948dddaca84b)

Overall, from what ive seen, ```mA2Z``` is where a lot of the malicious functionality lies.

The malware also uses SQL Queries! What for? I have no fucking idea, this shit is Windows malware, and as much as WhiteSnakes dev apparently has a Linux stub, i dont see ```Win32_ComputerSystem``` working on Linux lol:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/d65fc91b-b94e-47a8-9189-1640838a0456)

Moving on, the ```quq``` function is sending some data regarding the infected machine to the CNC server:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/1834a3d0-c236-448b-8670-206a8750b520)

I assume this is the inital infection data, that the threat actor could possible use to filter their infections out via Country, OS, Report Size, etc, based on the strings seen above.

Below, we see the CNC choice for this malware, which is Telegram, as you can see the malware is using Telegrams API to communicate the stolen data, and we also get some information on the ```chat_id```

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/52614c93-6182-490f-9134-707f09d94036)

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/a9020673-ed4d-484b-b666-028d6f1ff31f)

If we look into the ```wRFiM.g_zc``` function seen below:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/4ae80e88-85be-49f7-ac3c-f08de557e380)

We can see the Telegram channel ID and the HTTP API token:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/2492e8f9-51b3-4010-bc5a-564f8dc3242a)

For anyone who wants to fuck around (since the CNC we found this binary on is still up and running + we believe this campaign is still ongoing), here is the API token & channel ID:

```
Channel ID: 5668321496

Token API: 5805920195:AAHrkiYfOXg55Cncdj5wUj0Ov4rUYjQg7iU
```

Moving onto the ```ra``` function, we also have have some XML related functions which relate to the XML written stealer variables/targets which i will talk about later:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/d47b2411-c541-4c2a-abd9-9596344afc78)

Continuing on, I eventually found a function called ```rkbzL```, which im pretty sure was being used as the keylogger function to capture keys inputted into the victim computer:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/1c717adb-deb1-4b79-bdbc-8c72c9a021fd)

As you can see from above, the function is noting any notable keys like ```ESC```, ```Space``` and ```LWin``` and i would assume sending them to the Telegram CNC

Furthermore, the ```wCbjr``` function is more than likely noting down and stealing the storage drives that are connected to the computer:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/0b137ea0-91ad-4120-9195-f211fce1470e)

It also seems to be creating some sort of persistence module within ```%SystemDrive%\\Users\\{0}\\AppData\\Roaming\\Microsoft\\Windows\\Start Menu\\Programs\\Startup```

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/a7db732f-cb48-4f1c-b3fc-984303f9ae80)

Finally, probably the only interesting thing about the malware, ***the configuration***. While looking at the ```wRFiM``` function, we have a bunch of random IP addresses:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/4f4eea69-2ab0-4f3d-bb79-2887905a0eb5)

Browsing to these IPs results in either a ```transfer.sh``` link or a timeout:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/37f2c03f-0e39-4352-905c-b4d9e85b62c0)

Interestingly, these transfer.sh links had their upload links, so when browsing to them, a lot of them had 404s, like below:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/15163a02-d382-4996-b320-765c3bd9d2ee)

Some of them timed out:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/3494ea69-3851-46f6-8c48-d7ba78f7046d)

And rarely, i would get some error messages, which could be being used by the malware as a way of error reporting, but im not 100% sure:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/b3356b78-1a90-4db3-9b89-707d04902bf7)

Apart from that, we also have the variable names of the Channel ID and Token API key which i mentioned earlier:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/b70a1743-a930-434e-b0ba-e030b2492685)

And we also have a big XML "encoded"/"written" string of all the data that the stealer should steal and send to the Telegram CNC:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/ec89d75c-b969-47f5-a49d-b1d8daa09449)

This includes:

- Firefox Browser
- Vivaldi Browser
- CocCoc Browser
- CentBrowser
- Opera Browser
- OperaGX Browser
- CoreFTP
- Windscribe VPN
- Authy
- WinAuth
- OBS
- FileZilla
- AzireVPN
- Snowflake Client
- Steam
- Discord
- The Bat! Email Client
- Outlook
- Signal
- Pidgin
- Telegram
- Atomic Wallet
- Wasabi Wallet
- Binance
- Guarda Wallet
- Coinomi Wallet
- Bitcoin Core Wallet
- Electrum Wallet
- Exodus Wallet
- JaxxLiberty Wallet
- Metamask Wallet
- Ronin Wallet
- BinanceChain
- TronLink Wallet
- Phantom Wallet
- ```*.txt;*.doc*;*.xls*;*.kbd*;*.pdf``` Files

The stealer sends all of this data to the Telegram channel

```
Channel ID: 5668321496

Token API: 5805920195:AAHrkiYfOXg55Cncdj5wUj0Ov4rUYjQg7iU
```

Overall, a fun 3 hour reversing session, thanks to Eliyax for the laughs and enjoyment while reversing this shit.
