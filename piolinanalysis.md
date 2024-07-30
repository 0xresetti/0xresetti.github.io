# Piolin ATM Malware Analysis

### Once again, while gathering new ATM Malware resources for some study time, I came across a (kind-of) undocumented sample of Piolin, which I once again got from the [Global ATM Malware Wall](https://atm.cybercrime-tracker.net/index.php?x=stats). (file hash: `5f4215368817570e7a390c9f6e265a7db343c9664d22008d5971dac707751524`)

### What is Piolin?

Piolin is a modified version of [Ploutus](https://malpedia.caad.fkie.fraunhofer.de/details/win.ploutus_atm), first detected in 2013 and has had neumerous updates since then. It is designed to dispense cash directly from the ATM instead of stealing card information.

The reason why I say this is a (kind-of) undocumented sample, is because the only documentation/analysis report I could find on Piolin was [a post from Zingbox](https://www.zingbox.com/wp-content/uploads/2018/03/Meet-Piolin.pdf), which as you can see, since Zingbox was aquired by Palo Alto Networks, no longer exists.

I was also unfortunately unable to find any archived version of this PDF:

![image](https://github.com/user-attachments/assets/4791839e-860b-4da6-9f34-525aac71ce73)

However, that is where I come in!

Since the only information I could find about this malware was from a [Malwarebytes Report](https://www.malwarebytes.com/blog/news/2019/08/atm-attacks-and-fraud-part-2) talking about ATM malware, which was "Daniel Regalado, principal security researcher for Zingbox, noted in a (lost) blog post that a modified Ploutus variant called Piolin was used in the first ATM jackpotting crimes in the North America, and that the actors behind these attacks are not the same actors behind the jackpotting incidents in Latin America". I decided to take it upon myself to document this malware publicly, so that more people/companies can understand it.

### With that, let's get started.

