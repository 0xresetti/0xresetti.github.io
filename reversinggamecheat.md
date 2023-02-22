# First blog: Reversing a "Game Cheat" ;)

### One day I was chilling on Telegram, when someone who shared a group with me decided to mass spread some leaked game cheats & other tools! Lets take a look and see if they are what they say they are... 

Now I've never spoken to this person in my life, and at first i just thought it was someone advertising their cheat software and that was all, so I initially responded with how you would respond to any random person who sends you something you dont need:

<img src="/images/message.png" alt= "The message in question" width="50%" height="50%">

However, after looking at the website, I recognized it from somewhere, I'm not sure where exactly, but something in my head told me that this was a template that is being used to serve malware instead of actual cheats/tools.

<img src="/images/website.png" alt= "The website" width="50%" height="50%">

This was when I messaged the guy back saying "wait this is malware isnt it LOL".

After browsing around on the website for a while, I ended up downloading the "COD Warzone: Wallhack / ESP / NoRecoil / Aimbot" which came in the form of a .RAR file, named "Gloader by Dv9.rar"

<img src="/images/gloader.png" alt= "Gloader" width="50%" height="50%">

Opening up the file and extracting it to my Live Malware folder, I saw we had a few files and folders:

<img src="/images/thefiles.png" alt= "The files" width="50%" height="50%">

The "Gloader.exe" is the only thing interesting, the folders are just there to make it look more "legit" and like it does something, and the Preferences file has nothing interesting or related to the malware either.

Running the Gloader.exe through DetectItEasy, I saw it was a .NET executable!

<img src="/images/die.png" alt= "The files" width="50%" height="50%">
