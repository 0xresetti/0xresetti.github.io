# ATMWizX Undocumented Sample Analysis

### While gathering new ATM Malware resources for some study time, I came across an undocumented sample of ATMWizX which I got from the [Global ATM Malware Wall](https://atm.cybercrime-tracker.net/index.php?x=stats). (file hash: `7bd2c97ac5027c360011dc5aa8f2371cd934f73e885e41f7e80152332b3af1db`) Shoutout those guys, they got some real exotic shit.

### What is ATMWizX?

As described by [SecureList](https://securelist.com/atm-pos-malware-landscape-2017-2019/96750/), ATMWizX was discovered in the fall of 2018 and dispenses all cash automatically, starting with the most valuable cassettes.

Considering this sample is "undocumented", I will be looking into ATMWizX and how it works. Lets get started.

### Initial looks:

First off, as always, let's [die!](https://github.com/horsicq/Detect-It-Easy)

![image](https://github.com/user-attachments/assets/fd660d2e-cbe3-419d-9b7f-cef9b879259a)

Loading the binary into IDA, I am instantly bought to the `DllMain` function:

![image](https://github.com/user-attachments/assets/97ad2c59-343b-4029-9acd-df5f8a67d46f)

Though I can also see a `DllEntryPoint` function, which in turn calls DllMain:

![image](https://github.com/user-attachments/assets/15197ca7-8c1a-4984-a001-7942bfc3aadc)

Overall, I tried both options, and we can load it using a simple DLL loader which executes the `DllEntryPoint` OR `DllMain` function, let's see how that goes:

![image](https://github.com/user-attachments/assets/a0f0f242-fe55-449f-92da-25a211025c28)

Absolutely insane hacks. Just as a note, yes, all we can see is "No driver found!", which is unforunate, I'd love to showcase the binary acting how it would if connected to an actual ATM, however 1. I spent a good couple hours trying to do that before writing this blog, and didn't really get far, and 2. I will still talk about those functions and strings in the code later on.


