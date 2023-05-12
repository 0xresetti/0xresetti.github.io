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

- ```seach-pattern``` in **gef** and ```search``` in **pwndbg** are very useful, use them.

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/c7a3b729-0594-45a6-bdba-bbeb12446965)

- See above, the function ```gets``` has executed and has our input of ```15935728``` (random number inputted). So that means that the ```rip``` (return address, because the final gets function is finished so now we are ```ret```'ing) register is at ```0x7fffffffdac8```, and the start of our  input is ```0x7fffffffdaa0```. I ran ```search-pattern 15935728``` in order to find out where my input started on the stack.
- Another note, in x64 the saved base pointer is stored at ```rbp+0x0``` and the saved instruction pointer is stored at ```rbp+0x8```
- ```0x7fffffffdac8 - 0x7fffffffdaa0 = 0x28``` byte offset (```0x28 = 40 in decimal```), WE HAVE TO WRITE 40 BYTES WORTH OF INPUT AND WE CAN OVERWRITE THE RETURN ADDRESS!!! BASICALLY REDIRECTING THE CODE FLOW TO ANOTHER FUNCTION IF WE WANT!!!
- ***^^^^^ REMEMBER!!! ^^^^^ Sometimes, when calculating how many bytes of data are needed to fill up a buffer (or for padding), decimal values may need to be used***

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/01dd8a62-e9b1-4978-b8d8-d7679ac14857)

- Sometimes, exploits will crash on execution due to ***"The Movaps Problem.*** This is where a general protection fault is triggered when the ```movaps``` instruction is operating on unaligned data, aka any time ```movaps``` isn't aligned prior to a call. It is a ```libc``` problem that means we need to find a way to align bytes properly.
- I solved this by changing the exploit code to use ```0x04005ba```, which is 4 bytes away from ```*give_shell+0```.

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/c66b5cd4-980c-4762-9455-91bc43f80d4e)

- I believe char arrays move up in the stack, therefore calculating offsets for the below stack frame would be as follows: ```0x31 - 0x1d = 0x14 bytes``` and ```0x1d - 0x9 = 0x14.``` Both offets are ```0x14```

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/54636ee7-0579-46e0-a753-0519d4f0c7fc)

- Also, keep an eye out for **char sequences** by hovering over ```0x``` addresses. For example, hovering over ```0x73303325``` shows a ```char[]``` value of ```s03%```, and because this is a 32-bit executable the endinness is little, meaning we need to reverse this to ```%30s```, which ends up giving us ```dword ptr [EBP + fmt],"%30s"``` instead of ```dword ptr [EBP + fmt],0x73303325```

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/4364542f-f061-4550-af6d-63b7ecf1ca15)

