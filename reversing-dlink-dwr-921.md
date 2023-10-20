# Reverse Engineering the D-Link DWR-921 Router

### Recently with me learning reverse engineering and also getting my hands on lots of IoT Exploitation and Hardware Hacking courses, I wanted to go out and buy some shitty router for $30 and see what I could find inside.

### The Router:

The Router is a **D-Link DWR-921** Router, upon plugging it into my system via ethernet and browsing to the IP address I found in the manual, I logged in with the default credentials ```admin:(nothing)```.

The firmware was out of date, and so I went online to find an updated version that I could unpack with **binwalk** and then update the router to that latest version.

Funnily enough, the D-Link website didn't have any firmware for this device, since its pretty old and i bought it second hand from an electronics store, the only thing i could find was manuals.

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/e5e597f6-e657-451b-8a4f-dbd24478509a)

But with a simple Google search i found something kinda suspicious lol

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/a205e1ba-b932-47f4-8046-c78289317915)

The firmware was there...

![Screenshot_2](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/5087aae9-fe10-451d-93dd-48a7c0ca544e)

It was the legitimate D-Link Russian site...

So i said fuck it and downloaded it to be unpacked, its going in a VM anyways.

