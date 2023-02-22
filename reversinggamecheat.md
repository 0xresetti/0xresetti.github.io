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

After the fake message box, the malware adds itself to the Windows Defender Exclusion list, pretty normal evasion technique here. Following that, the malware downloads and executes files from ```https://rentry[.]org/ebg7e/raw```, this is more likely the actual malware which will be dropped onto the device. Lets take a browse to that link:

<img src="/images/reentry.png" alt= "reentry" width="70%" height="70%">

Not gonna lie, i wasn't expecting **5** different exe files, looks like I've got a lot more analysing to do...

## Analysing the real malware.
