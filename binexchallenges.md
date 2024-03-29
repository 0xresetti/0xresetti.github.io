# Binary Exploitation Challenge Writeups

### Here are my writeups from any Binary Exploitation Challenges I have done. Mainly from [CTF Workshop](https://github.com/kablaa/CTF-Workshop) which you can download yourself and follow along if you like.

Recently, I've been learning Binary Exploitation and Exploit Development, I have been putting my notes into my [Binary Exploitation Notes](./binex.html) page, and after a few weeks of learning, I'm starting to get the hang of some Buffer Overflow concepts, so I thought I would do some writeups on some challenges I have been doing.

### Buffer Overflows.

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

In the ```main``` function, we can see that we have a ```local_14``` variable which calls the ```lose``` function, then we have our input scanned in, which again has a 64 byte buffer, then finally, the ```local_14``` variable is called which in turn calls the ```lose``` function.

In the list of functions, we can see the ```lose``` function, and another hidden function which isnt called anywhere else named ```win```

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/e558a2ae-8da7-4e98-a565-8aee94e2e755)

The ```lose``` function just prints "you suck!" however the ```win``` function prints "you win!" and gives us a shell:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/1dfa2cf8-e65b-4b82-aa0e-f34bf1cc4d3d)

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/d78187c1-0a9c-44d3-a95c-373f72f98e7d)

So, how can we call the ```win``` function before the ```lose``` function is called?

Simple, by overwriting the **return address** with the memory address of the ```win``` function...

In this case, the memory address of the ```win``` function is ```0x080484ab``` which you can see highlighted in green below:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/704d1642-2cdd-4966-946f-eb18747f38da)

What I mean by **overwriting the return address** is, when a function is called and it completes (for example, the ```lose``` function), the ```return()``` function is called at the end, which contains a memory address which either calls back to the ```main``` function memory address, another function memory address (if we were in a loop for example) or exits entirely.

***REMEMBER: The ```eip``` register (```rip``` on 64-bit binaries) is the Instruction Pointer, if the next process in the code flow is to return back to another function for example, the ```eip``` register would have the memory address of the function to return to inside it***

Lets take a look at the way this would be exploited in GEF debugger before writing anything. (Feel free to use your favorite debugger, aka plain old GDB, pwndbg, r2 or PEDA):

I'm going to debug it in GEF:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/7eb5b68e-d449-423d-a6ec-0ec378572fea)

As you can see, there is a line at ```main+36``` which reads ```0x08048518 <+36>:	call   0x80483a0 <__isoc99_scanf@plt>```, this is where our input is scanned into the binary, I'm going to set a breakpoint at the instruction just after this one at ```main+41``` or ```0x0804851d``` using ```b *0x0804851d```

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/b3c8d8b8-09ec-4e94-bb86-4aae4d272b74)

Once this breakpoint is set, I can run the program, and it will ask me for my input, since we know the buffer of the ```input``` variable is 64 from Ghidra and I am doing a demonstration, I'm going to use [this buffer overflow pattern generator](https://wiremask.eu/tools/buffer-overflow-pattern-generator/?) to generate a 64 character long pattern:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/ff01ec56-8ef2-45f3-bd5a-1efb5b6e2057)

Once I run the program and paste this 64 character long pattern, anything AFTER those 64 characters should go into the ```eip``` register, to prove this, I'm going to add 4 B's to the end of this 64 character pattern, making it a total of 68 characters, overflowing the buffer by 4 characters.

If everything goes correctly, the ```eip``` register will be filled with ```BBBB```

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/6dd84aab-0d27-42a4-950f-890dc9d126b1)

As you can see, we hit our breakpoint, but the ```eip``` register has not been filled up yet, that is because we need to continue through some more instructions before until we see the **return address** being called... You can use the ```ni``` command in most GDB-based debuggers to do this:

**After one ```ni```, you can see the stack being filled up, but the ```eip``` register still hasnt changed yet:**

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/ec88dca6-87fc-41b6-b91d-f378ff53db51)

**After two ```ni```, we can see the ```eax``` register has been filled with "BBBB", this is a good sign:**

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/8b9cceba-813d-4198-b5e4-c47cfa17591b)

**And after a third ```ni```, boom! We can see the program crashed and is no longer running, and the eip register has been filled with "BBBB":** 

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/51407f3a-5714-49ae-b679-3466d658af51)

Perfect! So, now all we have to do to exploit the program and call the ```win``` function is to replace the "BBBB"'s we filled the ```eip``` register up with, with the memory address of the ```win``` function, lets write an exploit for this with pwntools!

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/f0cceb57-4f0a-471a-9581-4fbf5ee0a3bf)

Here is the exploit for you to copy:

```
from pwn import *

target = process("./bof3")

payload = b""
payload += b"A"*64
payload += p32(0x80484ab)

f = open("payload", "wb") # make "payload" file
f.write(payload) # Write payload to "payload" file
f.close() # finito


target.sendline(payload)

target.interactive()
```

And when we run the exploit:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/c3e4e45c-3b65-4ae4-93b3-e013aede6f30)

We successfully overwrote the ```eip``` register with the ```win``` function memory address, called the ```win``` function which printed "you win!", and popped a shell.

Now, you're probably wondering what the whole ```f = open``` and ```f.write(payload)``` business is about. That is just the script outputting the payload to a file called "payload", you can use these files with GDB-like debuggers to input the payload straight into the execution of the binary, like below, you can see inside the "payload" file is all 64 bytes of A's to fill up the buffer until we get to the return address, then the final ```��^D^H```, is actually the ```win``` functions memory address in 32-bit endianness.

This payload can be ran within GEF and other debuggers like this:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/6ecf7988-1043-4e9a-ba23-cb428bc742b5)

As you can see above, I used ```r < payload``` to run the program with the "payload" string as the input, and we got the "you win!" message.

### ROP

### Challenge 1: ROP1

The first ROP challenge fried my head a bit, but I managed to figure it out. The file is a 32-bit executable, with no RELRO, no canary, no PIE/ASLR, but it does have the NX bit enabled:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/37c66346-8885-4860-93cb-d33b07e33dfa)

Opening the file in Ghidra, the ```main``` function just calls the ```func``` function.

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/2a5ced7f-92b4-4f75-a664-8bde6adcbac6)

Inside the ```func``` function, the porgram scans in our input into a 64 byte buffer and we have a check that is similar to the previous BOF2 challenge, however, no ```win``` function is called after the check is complete, the program just exits:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/95cef4c4-fa8b-4f64-bc02-4b8144b90efa)

On the other hand, we do have a ```win``` function, at memory address ```0x80484ab```

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/b5250d4b-e4d0-4c7f-a5cf-cc303188af1a)

So, what do we need to do in order to exploit this?

- We need to overflow the buffer with 64 bytes
- We need to complete the ```deadbeef``` check in the ```func``` function
- We need to fill up the rest of the buffer with padding that will reach the eip (return address) register
- We need to overwrite the eip (return address) register with the memory address of the ```win``` function

Since we have done the first ```deadbeef``` check before in one of the previous challenges, we know how that is going to work in the exploit:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/d78aca1c-ccc9-48f3-aff8-c999acab9ae8)

But what about overwriting the return address? 

Well, first off we need to find the offset between our scanned in input, and the ```eip``` register (return address), which can be found in Ghidra or GEF

***In Ghidra:***

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/e340ffd7-cfa5-40e7-8569-0aa04fe7666e)

The ```[-0x50]``` is the offset between the ```input``` (start of our scanned in input) and the ```eip``` register (return address)

***In GEF:***

This is a bit more complex, but will give 100% accurate results, as sometimes, Ghidra doesnt figure out the offsets, or something, idk i heard it somewhere. Either way its good to know both methods

First off, set a breakpoint for just after the input is entered:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/3692ecfc-05cb-483a-b7b7-1f61810808dc)

Second, run the binary and input something, I inputted ```33012002```, and use the ```search-pattern``` and ```i f (shortened version of "info frame")``` commands to get the addresses of the start of our input, and the ```eip``` register

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/a71c45e5-2e6c-48c6-afe5-bdd7182529f7)

As seen above, the start of my input is at ```0xffffd0dc```, and as you can see near the bottom, the ```eip``` at ```0xffffd12c```

If you go into any hex calculator or just use Python, you can figure out the offset by subtracting the value of the start of the input with the ```eip``` register, like this:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/8a854395-1b43-4d63-b6f2-97f2b09248ec)

As you can see, we get the same offset from Ghidra, which is ```0x50```, or ```80``` in decimal (this will become important soon)

So, we have the first check done by filling up the input buffer with 64 bytes and adding ```0xdeadbeef``` to the end of it, but if we just run that, we dont actually win, we just don't get the "you suck!" message

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/1546c558-1157-4fc8-91a5-47473178c841)

We still need to overwrite the return address with the address of the ```win``` function. Now, we know that the offset is ```0x50``` which is ```80``` bytes (0x50 in decimal is 80), and we have already filled up 68 bytes of the stack. **The reason i say 68 bytes instead of 64 is because we filled up the initial buffer with 64 A's, then added the final ```0xdeadbeef``` onto the end which allowed us to complete the inital check, and since ```deadbeef``` is 4 bytes (```de```, ```ad```, ```be```, ```ef```) (each 2 letters is 1 byte), we get a total of 68 bytes filled up because ```64 + 4 = 68```**.

So, we have already filled up 68 bytes of the stack, and we need to pad the rest in order to get to the ```eip``` register so we can overwrite it, so lets do ```80 - 68 = 12```, so we need to pad ```12``` more bytes in order to reach the ```eip``` register (return address)

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/7a6571f7-f462-4158-a26c-92a338a8584a)

(For less confusion, the initial A's are for the first 64 byte overflow, and the B's are for the padding to the ```eip`` register)

After we have done that, we are now at the ```eip``` register (return address) and we should now be able to add the address of the ```win``` function to the end of our payload, and that should overwrite the return address with the address of the ````win``` function

The address of the ```win``` function is ```0x080484ab``` and can be found in Ghidra, seen below highlighted in green:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/fb414dbb-ad0a-40ae-93cb-ac88ae5944fa)

Below is the complete exploit, which you can copy if you want:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/65a76f57-333b-448b-b049-021ed75bd045)

```
from pwn import *

target = process("./rop1")

payload = b""
payload += b"A"*64
payload += p32(0xdeadbeef)
payload += b"B"*12
payload += p32(0x080484ab)

f = open("payload", "wb") # make "payload" file
f.write(payload) # Write payload to "payload" file
f.close() # finito

target.send(payload)
target.interactive()
```

And as you can see, the exploit worked flawlessly:

![image](https://github.com/0xwyvn/0xwyvn.github.io/assets/114181159/a5756c60-88b2-4dde-af9e-3c6a0d34b687)

***Count ya bytes, people!***
