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

Booting into FlareVM and running the original binary sample prompts an error about a missing library, ```Interop.CASHDISPENSER3Lib```:

![image](https://github.com/user-attachments/assets/6f80d2f1-680b-4f25-b920-e7a71aa7c257)

I attempted to create a fake version of this library however I ran into an error stating the library was compiled for a different .NET runtime version, the malware uses version 2.0, I installed this, but it still didn't work.

Not to worry though, it's in .NET so we can just read the decompiled code in DNSpy.

Since I saw that there was multiple resources in the executable, and the file is .NET, we can use ExtremeDumper to dump the embedded assemblies.

![image](https://github.com/user-attachments/assets/4c852155-b7bb-4372-86b3-356d625edaca)

The embedded assemblies are the following:

![image](https://github.com/user-attachments/assets/47d5b547-1344-457d-9620-3833243aa5c1)

```
- "_.dll"
- "anub3tlf.dll"
- "CalcAgilis.exe"
```

(The "anub3tlf.dll" filename seems to be randomly generated, since when I did this initially I got a different filename being "ruhbxcrx.dll")

We can safely assume that the CalcAgilis.exe is the malware, but what are the other two DLLs? I took a quick look in DNSpy since they were both .NET assemblies.

The "anub3tlf.dll" looks to be for reading and writing data to and from the MandeB.bin file, since the ```ConfigPlus``` reference is a class in the deobfuscated sample

![image](https://github.com/user-attachments/assets/92b1d893-c96e-4f03-a28b-804c23f46e2f)

![image](https://github.com/user-attachments/assets/6b2987ab-069a-4c17-a11a-ada1d70efaaa)

![image](https://github.com/user-attachments/assets/0244e426-0eb5-47ec-9935-ea89a4e32069)

The "_.dll" assembly simply has the "CalcAgilis.exe" malware embedded in the resources, which it reads, loads, and executes. The resource name is ```"_"```

![image](https://github.com/user-attachments/assets/78787429-170e-4f51-9263-7346a9de5c32)

Now we know what both of these DLLs do, let's look at the actual malware itself.

Initial looks at the references shows the library we saw in the error before and another one seemingly for interacting with the ATM Pin Pad

![image](https://github.com/user-attachments/assets/d1cd7e52-4cf0-43a3-b65a-7d862ccdeb22)

We can also see some interesting Type References used, from "Dispense" on the CashDispenser type reference, to "ReadData(PINReadData)" on the PinPad type reference, this is probably for controlling the ATM instead of actually stealing PINs, there are also some other interesting ones for getting information such as the status of the ATM and other information in the list.

![image](https://github.com/user-attachments/assets/97fb7c37-246c-491e-bb44-3b773a5b0fed)

Moving onto the actual code, first up we got ```B77Dw5684tb4mZjTIr.YAMXsbHMSTujjYPE1P```, which seems to be used for loading some sort of fake "Diebold.gif" image inside a WinForm, then bringing it to the front and controlling it, using ```base.Visible``` to make it visible and invisible when required. The "Diebold.gif" is not in the samples resources (unless it is in some encrypted resource, more on that later), however it could be on all Diebold Nixdorf ATMs by default, though that is too niche of a question to find the answer on Google lol.

![image](https://github.com/user-attachments/assets/4ce5d07f-836f-46f4-a602-f95a5e6a4194)

```Class0``` seems to be empty, not sure if this is an issue with the deobfuscation by de4dot or NETReactorSlayer, but both of the binaries' ```Class0``` are empty:

![image](https://github.com/user-attachments/assets/b914a2c5-bee7-49a2-ade0-5befb58d7951)

```Class1``` is where it gets a bit more interesting, it looks like this is where the malware is using the WinAPI functions called ```CreateDC``` and ```ReleaseDC``` for seemingly creating a Device Context (DC) for drawing and writing information on the ATM's screen.

Information about ```CreateDC``` from MSDN here: [https://learn.microsoft.com/en-us/previous-versions/ms959931(v=msdn.10)](https://learn.microsoft.com/en-us/previous-versions/ms959931(v=msdn.10))

![image](https://github.com/user-attachments/assets/9622a940-70e9-4f75-9f39-90c6d48e7eb7)

These ```DrawString``` calls for the "C1", "C2" and "C3" status information continue up to "C18" (I think the "C" stands for "Cassette", since the later code for selecting cassettes only goes up to 18), after that, the following code is executed:

![image](https://github.com/user-attachments/assets/bd4d156f-f755-44db-81c4-3de9e9e8bda1)

Which writes the ATM information on the screen, specifically:

- HWID
- ATMID
- Counter Information
- Total number of Cassettes
- "Codigo" (meaning "Code" in Portuguese) for **something???**
- Code1 (again, for **something???**)
- Code2 (again, for **something???**)

The other letters in the list, being the "S:", "D:" and "CV:" I don't know what they are, CV could mean "Current Value" of a specific cassette, but I'm not 100% sure, their string reference values/variables are empty and are probably filled at runtime

![image](https://github.com/user-attachments/assets/b7ecba06-060e-4756-bf75-a959dfc0701b)

Once finished, the final block of code looks to get the Device Context information using ```GetDC```, create a graphics object from that Device Context, clear the previously displayed information on the screen with a black rectangle, then write ```string_0``` (whatever that may be) to the screen and then clear it again with a black rectangle. It then finally uses ```ReleaseDC``` to release the Device Context.

![image](https://github.com/user-attachments/assets/33c733a2-5f3a-4511-9db8-b677524f8304)

That seems to be all regarding ```Class1```.

Moving on to ```Class2``` this primarily checks the current OS version, then check to see if the letter "P" is present as an input parameter, and if it is, it logs ```[APP]Modo Test``` and seemingly puts the malware into some sort of "Test Mode". It also logs ```[XFS]Windows 7 Detected.``` if Windows 7 is detected. 

Finally, at the end it executes ```RocHkU0iSSGhso0QcG.ef1ZVbjbWw1MAC5lYX```, which seems to be the actual "cash dispensing" and "PinPad reading" part (which as I mentioned before, is probably actually for controlling the ATM via the PinPad, instead of stealing PINs from victims)

![image](https://github.com/user-attachments/assets/17561040-3728-43f5-853f-cb0a57c3bdcb)

Lets take a look at ```RocHkU0iSSGhso0QcG.ef1ZVbjbWw1MAC5lYX```, before I get fully started though, since this part of the code has a lot of logging output, I thought I would mention about ```Class5```, which has a function that I renamed to "LogThis" (it was previously called ```viDavtxfHg``` as seen in the "Test mode" logs), and it does what it says on the tin. It is called whenever there is text or data that needs to be logged to a file, it is called "Log.txt" and is created on execution of the malware

![image](https://github.com/user-attachments/assets/f4b409c8-86cc-42d8-ad39-c78eae62f94c)

```string_0``` is defined as "Log.txt", seen below

![image](https://github.com/user-attachments/assets/8f801302-8907-4ab5-ae5c-d878d77c4bcf)

So whenever you see something like ```"Class5.LogThis("Hello this is a thing to log");``` you now know that "Hello this is a thing to log" is being logged to the Log.txt file.

Now, back to ```RocHkU0iSSGhso0QcG.ef1ZVbjbWw1MAC5lYX```, the first bit of code just looks to be initializing the PinPad input

![image](https://github.com/user-attachments/assets/ccbd0a33-0205-46f5-9abc-6bd01ef6c274)

The next main bit is where the malware seems to initially start and create controls for the ```axAXFS3Pinpad``` and ```axAXFS3CashDispenser1``` ActiveX controls.

![image](https://github.com/user-attachments/assets/e503fd50-5d98-49f3-bff7-c79c263168b9)

It also seems like the malware is utilizing time a lot, probably for checking if the status of the current license is active or not, I've not heard anything about this malware being sold to other criminals, its fairly old, so I doubt it, but still, that seems to be what the code is doing, it then as i said before, starts to create controls and logs "Inicializando..." (Initialising...) to the Log.txt file.

![Screenshot_45](https://github.com/user-attachments/assets/e2daa4de-3378-4f24-9501-1ce8c7837a34)

After that, it begins to setup various event handlers from the ```axAXFS3CashDispenser1``` component.

![image](https://github.com/user-attachments/assets/69e7add5-b805-4721-8b8c-ccc4976c375b)

And the same for the ```axAXFS3Pinpad``` component, however it also sets the logical service name to ```DBD_EPP4```, and initialises it after it sets up the handlers, logging "Inicializacion PinPad Completada" (PinPad Initialisation Complete) when done.

![image](https://github.com/user-attachments/assets/fb3cf6b3-af43-41f0-b30b-6bbc2e898781)

Just below that as well we can see the code to initialise the cash dispenser, this code sets the name of the device to ```DBD_AdvFuncDisp``` this time, which if you didn't know, is the logical service name for DieBold ATMs, I learnt this info from [this tweet](https://x.com/r3c0nst/status/1234494497486233607), shoutout that guy. Finally it logs "Inicializacion Dispenser Completada" when done.

![image](https://github.com/user-attachments/assets/5397516c-2601-43df-ab96-a8fa6c5c26f7)

Towards the end, you can see two new threads are created and started, running the two methods ```method_7``` and ```method_8```, further down in this massive function, or by just clicking on either one of them, we can see they are just executing the "OpenSession" function from the ```axAXFS3CashDispenser1``` and ```axAXFS3Pinpad``` components.

![image](https://github.com/user-attachments/assets/94f734e4-2de3-464a-9123-bbc75c79dee8)

Next up in the code seems to just be a bunch of functions for information logging, getting device statuses, and error logging.

![image](https://github.com/user-attachments/assets/d8dadb8e-6f17-48bc-a062-1f07a6c1cf3d)

After that, we see a bunch of what looks like code to handle the keypresses on the PinPad itself, for example, if a certain key on the PinPad is pressed, that key will corrospond to a function key like F1 or F2

![image](https://github.com/user-attachments/assets/5417aa9e-fe2d-4da3-82a5-ca5b1e4db003)

These function keys are fed into ```method_6``` further down in the code, which manages what each function key should actually do

![Screenshot_48](https://github.com/user-attachments/assets/6d07257c-eb88-4ee9-aca5-7491cd454595)

For example, if the key that corrosponds to "F2" is pressed, then the code will feed that into ```method_6``` and log "Pinpad:Activate Receive" to the Log.txt file, then call ```Class6.smethod_0``` with the "2" integer, you can see the code for ```Class6.smethod_0``` below:

![image](https://github.com/user-attachments/assets/71c80e49-7576-471a-9c34-5ba2ea83335b)

This code activates the ATM, I assume to get it "ready" for cash to be dispensed. Since the code just below it is for if "F3" was pressed, which instead feeds the "3" integer into ```Class6.smethod_0```, which is the code to actually dispense money from the ATM

![image](https://github.com/user-attachments/assets/3ff4e6bf-88eb-46c5-9b9a-b325da8fe4e2)

![image](https://github.com/user-attachments/assets/977c7673-9561-4c1f-bbb1-c7953956ff62)

The image above, with the one line of code squared in red is the final call to ```Class2.method_2```, after the criminal has finished selecting the cassette to dispense from and how much to dispense... This function is called, below is the code of ```Class2.method_2```, which as you can see, finally calls ```axAXFS3CashDispenser1.Dispense``` to dispense cash.

![image](https://github.com/user-attachments/assets/d3641572-4263-42d0-8db0-a0a03bd08520)

List of valid function keys that actually do stuff (so far):

```
F2 = Activate ATM
F3 = Dispense Cash
F4 = Change some values (not sure what)
```

(This post is unfinished, stay tuned for more!)
