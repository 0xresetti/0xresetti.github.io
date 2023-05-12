# Binary Exploitation Notes

**Just a bunch of notes for me to remember when learning Binary Exploitation, in no particular order, I just write as I go**

- Sometimes, the ```0x``` address you see in the decompiled code will not be the same as the assembly code

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/f87cb7a5-0dc6-4940-b6d7-e9b4be90f00c)

- If you click on a function OR VARIABLE IN THE VARIABLE DECLARATIONS, you will see The Stack Layout in the disassembly

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/4b918cfe-bc21-4888-af81-984492765b8f)

- Stuff like ```Stack[-0x43]   input``` means the **offset** of the ```input``` variable is ```0x43```

- Hovering over ```0x``` addresses shows more information like the Hex & Decimal value

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/8d78e5df-e5a6-4f22-ad64-30b8149b5fb8)

- Always need to make sure architecture/endianess is correct when writing exploits. Refer to my [Assembly Notes](./assemblynotes.md)

- Sometimes, when calculating how many bytes of data are needed to fill up a buffer (or for padding), decimal values may need to be used

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/677256a5-d04b-43ef-9ea8-6326b2d42a5c)

**(14 was the wrong value, 20 was the correct amount of bytes)**

- Relabelling variables/functions/etc is a good idea, click on one and press "L" to relabel it

