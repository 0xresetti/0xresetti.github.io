![Alt Text](https://c.tenor.com/cJRcMyUAiMcAAAAd/tenor.gif)

# Piolin ATM Malware Analysis

### Once again, while gathering new ATM Malware resources for some study time, I came across a (kind-of) undocumented sample of Piolin, which I once again got from the [Global ATM Malware Wall](https://atm.cybercrime-tracker.net/index.php?x=stats). (file hash: `5f4215368817570e7a390c9f6e265a7db343c9664d22008d5971dac707751524`)

### What is Piolin?

Piolin is a modified version of [Ploutus](https://malpedia.caad.fkie.fraunhofer.de/details/win.ploutus_atm), first detected in 2013 and has had neumerous updates since then. It is designed to dispense cash directly from the ATM instead of stealing card information.

The reason why I say this is a (kind-of) undocumented sample, is because the only documentation/analysis report I could find on Piolin was [a post from Zingbox](https://www.zingbox.com/wp-content/uploads/2018/03/Meet-Piolin.pdf), which as you can see, since Zingbox was aquired by Palo Alto Networks, no longer exists.

I was also unfortunately unable to find any archived version of this PDF:

![image](https://github.com/user-attachments/assets/4791839e-860b-4da6-9f34-525aac71ce73)

However, that is where I come in!

Since the only information I could find about this malware was from a [Malwarebytes Report](https://www.malwarebytes.com/blog/news/2019/08/atm-attacks-and-fraud-part-2) talking about ATM malware, which was "Daniel Regalado, principal security researcher for Zingbox, noted in a (lost) blog post that a modified Ploutus variant called Piolin was used in the first ATM jackpotting crimes in the North America, and that the actors behind these attacks are **not** (!!) the same actors behind the jackpotting incidents in Latin America". 

**The key sentence here is: "the actors behind these attacks are not the same actors behind the jackpotting incidents in Latin America"**

So, I decided to take it upon myself to document this malware publicly, so that more people/companies can understand it.

### With that, let's get started.

I initially used CFF Explorer to get some initial information about the binary, and we can see the original file name for this sample was ```CalcAgilis.exe```. Nothing really special here, we could have seen this from the original [ATM Malware Wall Post](https://atm.cybercrime-tracker.net/index.php?x=threat&hash=5f4215368817570e7a390c9f6e265a7db343c9664d22008d5971dac707751524), however it's always good to check these things.

![image](https://github.com/user-attachments/assets/aded5128-c8fe-4c19-9a96-e2a0814c80a0)

We can also use pestudio to check for embedded files, it's my favorite for looking for that type of stuff, and it looks like we got a bit lucky:

![image](https://github.com/user-attachments/assets/d9fdaec1-3c51-4612-a508-072c502d1523)

I dumped these two files into another folder by double clicking on the red address you see on the right side in the screenshot above

![image](https://github.com/user-attachments/assets/6249e397-a746-474a-ba71-083530c81c3f)

However, on an intial quick-look with HxD, I saw that neither of these files have any MZ header indicating that it is an executable, or anything else, however that also doesn't mean that they arent used by the malware.

![image](https://github.com/user-attachments/assets/e2eee27c-dee0-44b5-9c11-506f867b65bc)

I'll probably come back to this later if I find something in the decompiled .NET code that indicates the decoding of embedded resources/files.

For now, let's use DetectItEasy to see what we are working with, considering it is a modified Ploutus variant, I would assume it is .NET

![image](https://github.com/user-attachments/assets/3d7d2bfe-d3eb-47de-9b1c-daf1a4b51aa7)

Lovely stuff, and a pretty simple obfuscator to deobfuscate too, .NET Reactor has had many deobfuscators made for it, including the [.NETReactorSlayer](https://github.com/SychicBoy/NETReactorSlayer), and [de4dot](https://github.com/de4dot/de4dot). For this deobfuscation I will just be using .NETReactorSlayer since it is newer and has a lot more custom options to choose from.

![image](https://github.com/user-attachments/assets/57b7a71c-0b92-475c-bcf6-adafd03814f4)

We got some error about decrypting resources, but I'm sure it wont be a problem.

After deobfuscating, we get a file with "_Slayed" appended to the end of it, and look at that file size! Dramatically decreased, this is a good sign. We can also see in DetectItEasy that the signature for .NETReactor has disappeared!

![image](https://github.com/user-attachments/assets/808baedc-e46c-4de2-b769-c13737a7b7b5)

![image](https://github.com/user-attachments/assets/929015b5-6d30-478c-bfb7-b74bd1c3e5e0)

**(NOTE: I also tried using de4dot to deobfuscate and clean the file, and it gave me a file with an even smaller deobfuscated file size... (de4dot = 123kb | NETReactorSlayer = 143kb)**

The difference between the two deobfuscated files seems to be in the resources, looking at both of them in pestudio reveals the total filesize for the resources of the .NETReactorSlayer deobfuscated binary is **5068 bytes**, whereas the de4dot deobfuscated binary has a total resource file size of **12408 bytes**

![image](https://github.com/user-attachments/assets/f0015b99-5036-4e89-bca8-d0470aef7e14)

![image](https://github.com/user-attachments/assets/bff16482-c3be-410b-8136-f4ab381ccb6b)

The Relative Virtual Address (RVA) for both of the deobfuscated binaries also differ, with the one for .NETReactorSlayer being ```0x000151F8```, and de4dot being ```0x0000F470```

I'm not 100% sure why this might be, however I think im going to look at the de4dot binary in DNSpy first, since that seems to have more resources, and from looking over both binaries initially in DNSpy, I can see that both binaries have pretty much the same code. Though I could be wrong regarding the resources, perhaps de4dot thinks there are more resources than there actually are, I'm sure I'll find out soon.

![image](https://github.com/user-attachments/assets/69dcdd32-b111-4b66-9ec3-b2e79150a6b2)

And yes, I checked each method and could see the code was exactly the same for each deobfuscated binary.

Anyways, enough about conspiracy theories, let's move on.

### Dynamic FlareVM Analysis

Now, there are two things I could do from here on out, I could load the deobfuscated binary into DNSpy and start looking at the code, or I could have a look at the original executable in a Virtual Machine.

The VM sounds more fun for now, so let's boot up FlareVM.
