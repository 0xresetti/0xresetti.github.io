# Architecture 1001: x86-64 Assembly Notes

**This is just a page of notes and important things to me to remember while going through the "Architecture 1001: x86-64 Assembly" Course.**

# Endianness
Little Endian Example:
```0x12345678``` = ```0x78, 0x56, 0x34, 0x12```

Big Endian Example:
```0x12345678``` = ```0x12, 0x34, 0x56, 0x78```

![Screenshot_12](https://user-images.githubusercontent.com/114181159/234832477-8b33f872-7e4d-496f-8780-bfdd3378244a.png)

Network traffic is sent in Big Endian
- Endianness applies to _**memory, not registers!**_
- Endianness applies to _**bytes, not bits!**_

# Memory Hierarchy
Top of Hierarchy is small storange, quick access, volatile.
- Registers, Cache, RAM
- Small size
- Small capacity
- Fast/very fast to access

Lower part of Hierarchy is large, slow, non volatile storage.
- Flash/USB memory, Harddrives, Tape drives
- Slow/very slow to access
- Large/Very large capacity

# Architecture Registers
- Small memory storage areas built into the processor (still volatile)
- Intel has 16 "general purpose" registers + the instruction pointer
- On x86-32, registers are 32 bits wide
- on x86-64, registers are 64 bits wide

**Register Evolution:** ```AL, AH, AX, EAX, RAX.```

![Screenshot_11](https://user-images.githubusercontent.com/114181159/234832113-5ff7cd3d-8230-4d61-bc3a-130afee14c7e.png)

**Register Conventions**
- RAX - Stores function return values
- RBX - Base pointer to the data section
- RCX - **C**ounter for string and loop operations
- RDX - I/O pointer
- RSI - **S**ource **I**ndex pointer for string operations
- RDI - **D**estination **I**ndex pointer for string operations
- RSP - **S**tack (top) **P**ointer
- RBP - Stack frame **B**ase **P**ointer
- RIP - Pointer to next instruction to execute (**I**nstruction **P**ointer)

# My First Instruction: NOP
- No-Operation! No registers, no values.
- Just there to pad/align bytes, or to delay time.
- Attackers use it to make simple exploits more reliable.
- ```xchg eax, eax``` is an alias of NOP, because exchanging two of the same registers does nothing.

# The Stack
- Last In First Out (LIFO) data structure where data is "pushed" onto the top and "popped" off of the top
- Conceptual area of memory (RAM) which is designated by the OS when a program is started
- The Stack grows toward lower memory addresses
- Adding something to the stack means the top of the stack is now at a lower memory address
- Up is down.
- **RSP** points to the top of the stack - _the lowest address which is being used_

**What can be found on the stack?**
- Return addresses so a called function can return back to the function that called it
- Local variables
- Sometimes used to pass arguments between functions
- Save space for registers so functions can share registers without smashing the value for eachother
- Save space for registers when the compiler has to juggle too many in a function
- Dynamically allocated memory via ```alloca()```

![Screenshot_13](https://user-images.githubusercontent.com/114181159/234846964-cce0f7ee-5c58-42af-906c-b48865a1a18b.png)

# Push & Pop Instructions
- Why do the GCC/Clang compilers have balanced Push/Pop instructions but Visual Studio doesnt?

**PUSH**
- Places (pushes) an operand onto the top of stack
- Automatically decrements the stack pointer RSP by 8 (ESP by 4)

**r/mX**
- ```r/m8, r/m16, r/m32, r/m64```
- It is a way to specify a register or memory value.
- Square brackets meant to treat the value within as a memory address and to fetch the value at that address

**r/mX can take 4 forms:**

1. Register -> ```rbx```
2. Memory, base-only -> ```[rbx]```
3. Memory, base + index * scale -> ```[rbx+rcx*X]```
- For X = 1, 2, 4 or 8
4. Memory, base + index * scale + displacement -> ```[rbx+rcx*X+Y]```
- For Y of 1 byte (0-2^8) or 4 bytes (0-2^32)

_**Basically a complicated way of addressing memory locations**_

_**r/mX could be a single register like ```rbx``` or it could be a complicated memory address calculation like ```[rbx+rcx*X+Y]```**_

**r/mX Examples:**

|      **push register**      |<br>
|:---------------------------:|<br>
| ```push rbx```              |<br>
|       **push memory**       |<br>
| ```push [rbx]```            |<br>
| ```push [rbx+rcx*4]```      |<br>
| ```push [rbx+rcx*8+0x20]``` |<br>

**A scenario: ```push RAX```**

![Screenshot_15](https://user-images.githubusercontent.com/114181159/234850844-3a2b162c-d691-46cc-8b2f-c9b0e10d2921.png)

**What happened?**

1. ```RSP``` value decremented by 8 because the stack pointer ```(RSP)``` is automatically decremented by 8 after being pushed onto the stack
2. ```RSP``` ***WAS*** at memory address ```0x014FE08```. ***AFTERWARDS*** it is at ```0x014FE00```. Again because the stack pointer is automatically decremented by 8 after being pushed.
3. Also, the value of memory address ```0x014FE00``` is now ```3```, because the ```RAX``` value was ```3```. Previously, it was ```undefined```
4. In the end, ```0x014FE00``` is the new stack pointer, and its value is still ```3```. (it was always going to be 3 anyways, because we are pushing a value onto the stack from the predefined register ```RAX```)

**POP**
- Pop a value from the stack

**Opposite scenario: ```pop RAX```**

![Screenshot_16](https://user-images.githubusercontent.com/114181159/234853608-e4f149bf-aaa0-4b78-94bd-33198b64d66c.png)

**What happened?**

1. ```RAX``` had some random value before the POP, in this case it is ```5007```
2. ```RSP``` was pointing at our previous memory address: ```0x014FE00``` with the value still being ```3```
3. After execution, the ```3``` value from the stack is popped off of the stack and into the ```RAX``` register, overwriting the ```5007```
4. ```RSP``` register is incremented by 8 as a side effect automatically after being popped off of the stack
5. Memory address ```0x014FE00``` is now known as ```undefined```
6. ```RSP``` is now at memory address ```0x014FE08``` with a memory address value of ```2```
7. The value ```3``` is now stored in the ```RAX``` register
8. Data does still exist past the end of the stack.

**32-bit Information**
- Executing in 32-bit mode, push/pop will add/remove values 32 bits at a time rather than 64 bits, therefore they decrement/increment the ```RSP``` register by 4 instead of 8.
- Likewise with 16-bit mode, push/pop 16 bits at a time, and decrement/increment by 2.
