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

- **A quick few notes on Mitigations**
- NX aka DEP - The No eXecute or the NX bit (also known as Data Execution Prevention or DEP) marks certain areas of the program as not executable, meaning that stored input or data cannot be executed as code. This is significant because it prevents attackers from being able to jump to custom shellcode that they've stored on the stack or in a global variable.
- ASLR - This is the randomization of the place in memory where the program, shared libraries, the stack, and the heap are. This makes can make it harder for an attacker to exploit a service, as knowledge about where the stack, heap, or libc can't be re-used between program launches. This is a partially effective way of preventing an attacker from jumping to, for example, libc without a leak. [Click here to learn more and how to bypass ASLR](https://github.com/hoppersroppers/nightmare/blob/master/modules/04-Overflows/5.1-mitigation_aslr_pie/readme.md)
- - PIE - Position Independent Executable (PIE) is another binary mitigation extremely similar to ASLR. It is basically ASLR but for the binary's code / memory regions
- Relocation Read-Only (RELRO) - Partial RELRO is the default setting in GCC, and nearly all binaries you will see have at least partial RELRO. From an attackers point-of-view, partial RELRO makes almost no difference, other than it forces the GOT to come before the BSS in memory, eliminating the risk of a buffer overflows on a global variable overwriting GOT entries. Full RELRO makes the entire GOT read-only which removes the ability to perform a "GOT overwrite" attack, where the GOT address of a function is overwritten with the location of another function or a ROP gadget an attacker wants to run. Full RELRO is not a default compiler setting as it can greatly increase program startup time since all symbols must be resolved before the program is started. In large programs with thousands of symbols that need to be linked, this could cause a noticable delay in startup time.
- Stack Canaries - Stack Canaries are a secret value placed on the stack which changes every time the program is started. The general idea is, a random value is placed at the bottom of the stack frame, which is below the stack variables where we actually have input. If had a buffer overflow to overwrite the saved return address, this value on the stack would be overwritten. Then before the return address is executed, it checks to see if that value is the same one it set. If it isn't then it knows that there is a memory corruption bug happening and terminates the program.  Stack Canaries seem like a clear cut way to mitigate any stack smashing as it is fairly impossible to just guess a random 64-bit value. However, leaking the address and bruteforcing the canary are two methods which would allow us to get through the canary check.
- ```<__stack_chk_fail@plt>``` is probably a Stack Canary
- **RE: Stack Canaries** - For x64 elfs, the pattern is an 0x8 byte qword, where the first seven bytes are random and the last byte is a null byte.
- **RE: Stack Canaries** - For x86 elfs, the pattern is a 0x4 byte dword, where the first three bytes are random and the last byte is a null byte.

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/c66e9190-ec78-4277-9f27-eb6dc281ad3c)


![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/95e7ac93-f1e8-431b-bc6e-f8c49e6e7697)

**^^^ Examples of Stack Canaries ^^^**

For more information on Stack Canary bruteforcing, [go here.](https://ctf101.org/binary-exploitation/stack-canaries/)

- libc is where standard functions like ```fgets``` and ```puts``` live.

- **While the addresses in a memory space will change, the offset between the addresses themselves will not change**

- Getting an infoleak for certain parts of the memory, like ```libc``` for example, means that that infoleak is only good for the ```libc``` region of memory, we cant use that infoleak for areas like ```stack``` or ```heap```

- [Shell Storm is a good place to get shellcode](http://shell-storm.org/shellcode/)

- Offset/Hex Calculation can be done with Python using ```hex(addr1 - addr2)```

- The ```info frame``` or ```i f``` command can show more information aboutthe ```ebx, ebp, and eip registers```

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/76d475c9-2783-4ac2-9bef-47ae040e2cd5)

- The ```search-pattern``` command is also very useful for finding input in the stack

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/a1b342dc-a953-49e3-ac87-e6ddb65401a2)

- A Format String attack is an alternate form of exploiting programming that doesn't necessarily require smashing the stack. Instead, it leverages the format characters in a format string to generate excessive data, read from arbitrary memory, or write to arbitrary memory

Example:
```user@si485H-base:demo$ ./format_error "Hello World"
Hello World
user@si485H-base:demo$ ./format_error "Go Navy"
Go Navy
user@si485H-base:demo$ ./format_error "%x"
b7fff000
```

- ```%x``` caused the program to output an address on the stack

```user@si485H-base:demo$ ./format_error "%s.%s.%s.%s.%s.%s.%s"
4.??u?.UW1?VS???????unull).(null).?$?U?
user@si485H-base:demo$ ./format_error "%s.%s.%s.%s.%s.%s.%s.%s"
Segmentation fault (core dumped)
```

- Using ```%s``` can also cause crashes as seen above

- List of formats:
- ```%d``` : signed number
- ```%u``` : unsigned number
- ```%x``` : hexadecimal number
- ```%f``` : floating point number
- ```%s``` : string conversion
- ```%n``` : printf has a ```%n``` flag. This will write an integer to memory equal to the amount of bytes printed

- Using ```%#x``` will output ```0xdeadbeef``` instead of ```deadbeef```

- Format String exploits are confusing asf, learn more about them.

- The GOT Table is a table of addresses in the binary that hold libc address functions

- Sometimes, random functions may not actually be random, they may be based off of a certain condition, for example. A random number is generated depending on the current TIME, or DATE, or current VARIABLE in memory... This is pretty rare to see, but still worth knowing.

- RAX - Stores function return values
- RBX - Base pointer to the data section
- RCX - Counter for string and loop operations
- RDX - I/O pointer
- RSI - Source Index pointer for string operations
- RDI - Destination Index pointer for string operations
- RSP - Stack (top) Pointer
- RBP - Stack frame Base Pointer
- RIP - Pointer to next instruction to execute (Instruction Pointer)

- LEAST SIGNIFICANT BYTE OVERWRITES EXIST, AND THEY WORK LIKE THIS:
- When we looked at the saved return address, we saw that it was equal to ```0x8048668```. The function we are trying to call (```printFlag```) is at ```0x8048672```. Since the only difference between the two addresses is the least significant byte, WHICH IS ```72``` AND ```68```, BECAUSE BOTH THE ADDRESSES HAVE THE SAME FIRST 5 VALUES: ```0x80486``` AND ```0x80486```, THE LEAST SIGNIFICANT BYTE FOR EACH ADDRESS IS ```72``` AND ```68```... And because we want to call the ```printFlag``` function which is at ```0x8048672```, we need to overwrite that with ```0x72``` bytes at the end to call the ```printFlag``` function

- ANOTHER EXAMPLE:
- The address that it is initialized to is ```0x565556ad```, and the address we want to set it to is ```0x565556d8``` (for ```print_flag```). The difference between these two is just the least significant byte. So we can just overwrite the least significant byte to be ```0xd8```, and that will call ```print_flag```.

- ROP (Return Oriented Programming) is a technique in exploitation to reuse existing code gadgets in a target binary as a method to bypass DEP
- A Gadget is a sequence of meaningful instructions typically followed by a return instruction
- Usually multiple Gadgets are chained together to complete malicious actions similar to what shellcode would do
- These are called ROP Chains

- Using ```shell``` in gdb will pop you back to a shell, and searching for the process with ```ps -aux | grep <processname>``` will get you the process ID which you can use with ```cat /proc/<PROCESSID>/maps``` to get linked libraries
- ```info proc``` in gdb or gef will also show the process ID for you

![Screenshot_20230602_113606](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/f8b7303e-dc20-496d-a182-a64ebb6ebc1b)

- You can use ```ropper``` to search for ROP gadgets 

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/85ba877d-b1d3-4258-a690-8bd6fc782afe)

- When ROPPing, the best thing to do is planning what you want to do and actualizing the plan in bullet points.

- Keep in mind that our ROP chain consists of addresses to instructions, and not the instructions themselves. So we will overwrite the return address with the first gadget of the ROP chain, and when it returns it will keep on going down the chain until we get our shell

- ROPgadget is also a good tool to use to look for ROP gadgets, probably better than ropper

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/2935489c-62b7-4665-8ee4-9a7649539ee1)

- Sometimes, searching in HEX values is required for ```search-pattern```

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/4f8a01b5-dbc5-4313-999a-84d483343a91)

- Using python to get PLT and GOT addresses is also useful:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/b8cabd95-5ae7-46a9-9628-e24a5d6fc877)

- Now we talking about the HEAP!
- Heap is a pool of memory used for dynamic allocations at runtime
- ```malloc()``` grabs memory on the heap
- ```free()``` releases memory on the heap

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/1449241f-e42e-4510-91cb-bc1a71dea64b)

- Heap is slower, and manual, whereas the Stack is faster, done by the compiler, 

- Bytes on the heap are fucking ***weird***

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/9869c424-8e25-48d4-8f7e-337a701e69fd)

- Heap grows DOWN to higher memory, Stack grows UP to lower memory

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/c6542cac-027b-499a-8c8b-a7da9715d31c)

- Heap overflows are basically the same as Stack overflows
- Heap Canaries/Cookies do not exist

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/761103e6-fa3c-47a5-951a-6ee0d63be1f9)

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/b65e6f40-5121-4f09-987a-582fd426000f)

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/fcf4372f-78ab-4e1e-b8cc-f3fb6ea259ba)

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/7c4193a4-b32d-4066-a2b2-1ea1407b28d5)

- Dangling pointers are left over pointers in code which reference to free'd data and is prone to being re-used.
- Since the memory it was pointing at was free'd there is no guarantees on what data is there now.  

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/cc1bf2eb-986f-4236-84b9-571a1c549c26)

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/15a069fa-83f6-4cd6-a66d-57261e0f57da)

- Memory corruption not required to exploit UAF, it is simply an implementation issue

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/1a8e2c38-4d1f-4a08-9e53-d3905893bf7b)

-  UAF only exists through certain states of execution

-  Usually only found through crashes

- Heap Spraying, a technique used to increase exploit reliability, by filling the heap with large chunks of data relevant to the exploit you are trying to do

- Heap Spraying can help bypass ASLR

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/7bd348aa-0028-40f8-baba-a1a78c51dc06)

- Heap Spraying on 64bit can't really be used to bypass ASLR

- Metadata Corruption exploits involve corrupting heap metadata in a way that allows you to use the allocator's internal functions to cause a controlled write of some sort

- This generally involes faking chunks and abusing different its different coalesent or unlinking processes

- Metadata exploits are hard to pull off due to heaps being fairly hardened on modern OS's

- Quick note, see below, the registers area in gdb can be used to check if inputs or values in registers are within the Heap or not (see the ```$rsi``` address highlighted in green, it is ```0x00005555556036b0```

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/82bd2127-cf8c-4253-94b7-53aa5b150df1)

- Then below, after running ```vmmap```, we can see that the Heap starts at ```0x0000555555603000``` and ends at ```0x0000555555624000```

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/ee182c46-a189-4fc3-8b05-daced47ae997)

- Because ```0x00005555556036b0``` is between the start and end addresses of the heap, we know that the value in the ```$rsi``` register is within the heap

- Malloc will reuse previously freed chunks if they are the right size for performance reasons

