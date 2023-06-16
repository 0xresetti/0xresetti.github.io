# Binary Exploitation Challenge Writeups

### Here are my writeups from any Binary Exploitation Challenges I have done. Mainly from [CTF Workshop](https://github.com/kablaa/CTF-Workshop) which you can download yourself and follow along if you like.

Recently, I've been learning Binary Exploitation and Exploit Development, I have been putting my notes into my [Binary Exploitation Notes](./binex.html) page, and after a few weeks of learning, I'm starting to get the hang of some Buffer Overflow concepts, so I thought I would do some writeups on some challenges I have been doing.

### Challenge 1: BOF1

The first challenge we have is BOF1, which is a 32-bit executable, with no RELRO, no canary, no PIE/ASLR, however NX is enabled, which means we cannot jump to custom shellcode for example.

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/09e715ca-2a6e-4da4-9c93-abad584fab02)

Running the file with some input doesn't do anything, so lets have a look at the file in Ghidra

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/5e58e5e2-d4ba-4a5d-9341-3e58bcde43e7)

Ghidra shows us the ```main``` function, which asks the user to scan in input, the ```input``` variable has a 64 byte buffer, after that, the function checks to see if the ```local_14``` variable is "not-equal-to" (```!=```) 0, if it isn't, then it will print "you win!" and call ```system("/bin/sh")``` which will pop a shell. 

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/83ba27df-8096-48ef-96bd-6bb3c93dcc45)

To exploit this, we just need to scan in more than 64 bytes into the input! Which should be easy, just spam ```A```'s!

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/67cee650-166b-44d2-8b3f-8958be2e7bac)

Now that was easy, but kinda boring, lets move onto the next challenge.

### Challenge 2: BOF2

The second challenge has the same protections and compilation as the BOF1 binary, 32-bit executable, no RELRO, no canary, no PIE/ASLR, but the NX bit is enabled

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/2d865daf-ad34-442b-8c71-dafa08535d4e)

Running the file and giving it some random input, we can see the binary thinks we suck because we didn't exploit it!

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/81f6b059-46b8-42be-a102-7064ce9bd7c7)

Lets take a look at the binary in Ghidra and see what it's doing...

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/7764f442-5edf-4476-a4c7-e32cbdc378bf)

Now I renamed some of the variables here to make it a bit easier to understand (you can do this by clicking on a variable and pressing "L" on your keyboard to change the name).

The binary has a few variables, the ones that matter are ```input``` and ```isDEADBEEF```. ```input``` has a buffer size of 64 bytes, and ```isDEADBEEF``` is at first set to 0. Then, the user input is scanned in, after that, a check is made to see if the ```isDEADBEEF``` variable is set to ```0x21524111```. If we hover over this value, we can see that in hex it translates to ```DEADBEEF```

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/80749551-41e3-4d30-9137-a4d19c8773af)

We can also see this more clearly in the disassembly without having to hover over the ```0x21524111``` value:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/91312177-1597-401c-b61b-c8d6f3811be9)

As you can see the ```!=``` operator is in the if statement again, remember that means "not-equal-to", in this case, if the ```isDEADBEEF``` variable is NOT equal to ```0xdeadbeef```, then the binary prints out "you suck!" and we lose.

In order to get the binary to print out "you win!" and pop a shell, we need to write ```0xdeadbeef``` to the end of our input, lets write an exploit using ```pwntools``` to do this.

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/45247296-ae2c-44c0-9a6e-288a641e1bcf)

You can copy the above exploit below:

```
from pwn import *

target = process("./bof2") # set the process we are targeting

payload = b"" 
payload += b"A"*64 # fill up the `input` variable with 64 bytes, since it has a 64 byte buffer
payload += p32(0xdeadbeef) # write 0xdeadbeef with 32-bit endianness, since the binary is 32-bit


target.send(payload) # fire the payload

target.interactive() # go into interactive mode
```

This is what the exploit is doing:

- importing the ```pwntools``` library
- setting the target process to ```bof2```
- building a payload which sends 64 A's into the input, and because the input buffer is 64 bytes this will fill it up completely
- it then writes ```0xdeadbeef``` in 32-bit endianness to the end of the payload, which should check out as good when the ```if``` statement checks to see if it matches
- it then sends the payload and starts interactive mode

Lets see if it worked:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/271dd0ce-d76a-4ed3-b63b-7feb7cdb809c)

As you can see, it worked perfectly and we popped a shell!

### Challenge 3: BOF3

The next binary is the exact same as the last two binaries, we load it up, input something, and it tells us we suck. It also has the same protections

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/0a5969f8-709d-4ebb-b2f5-27de67d7ab5a)

Lets dive into Ghidra to see whats going on in the background:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/815b9ceb-468c-415e-85a2-d46e2135aa58)

In the ```main``` function, we can see that we have a ```local_14``` variable which calls the ```lose`` function, then we have our input scanned in, which again has a 64 byte buffer, then finally, the ```local_14``` variable is called which in turn calls the ```lose``` function.

In the list of functions, we can see the ```lose``` function, and another hidden function which isnt called anywhere else named ```win```

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/e558a2ae-8da7-4e98-a565-8aee94e2e755)

The ```lose``` function just prints "you suck!" however the ```win``` function prints "you win!" and gives us a shell:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/1dfa2cf8-e65b-4b82-aa0e-f34bf1cc4d3d)

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/d78187c1-0a9c-44d3-a95c-373f72f98e7d)

So, how can we call the ```win``` function before the ```lose``` function is called?

Simple, by overwriting the **return address** with the memory address of the ```win``` function...

In this case, the memory address of the ```win``` function is ```0x080484ab``` which you can see highlighted in green below:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/704d1642-2cdd-4966-946f-eb18747f38da)

What I mean by **overwriting the return address** is, when a function is called and it completes (for example, the ```lose``` function), the ```return()``` function is called at the end, which contains a memory address which either calls back to the ```main```function memory address, another function memory address (if we were in a loop for example) or exits entirely.

***REMEMBER: The ```eip``` register (```rip``` on 64-bit binaries) is the Instruction Pointer, if the next process in the code flow is to return back to another function for example, the ```eip``` register would have the memory address of the function to return to inside it***

Lets take a look at the way this would be exploited in GEF debugger before writing anything. (Feel free to use your favorite debugger, aka plain old GDB, pwndbg, r2 or PEDA):

I'm going to debug it in GEF:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/7eb5b68e-d449-423d-a6ec-0ec378572fea)

As you can see, there is a line at ```main+36``` which reads ```0x08048518 <+36>:	call   0x80483a0 <__isoc99_scanf@plt>```, this is where our input is scanned into the binary, I'm going to set a breakpoint at the instruction just after this one (```main+41``` or ```0x0804851d```) using ```b *0x0804851d```

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/b3c8d8b8-09ec-4e94-bb86-4aae4d272b74)

Once this breakpoint is set, I can run the program, and it will ask me for my input, since we know the buffer of the ```input``` variable is 64 from Ghidra and I am doing a demonstration, I'm going to use [this buffer overflow pattern generator](https://wiremask.eu/tools/buffer-overflow-pattern-generator/?) to generate a 64 character long pattern:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/ff01ec56-8ef2-45f3-bd5a-1efb5b6e2057)

Once I run the program and paste this 64 character long pattern, anything AFTER those 64 characters should go into the ```eip``` register, to prove this, I'm going to do whats called ***The "B" Test***
