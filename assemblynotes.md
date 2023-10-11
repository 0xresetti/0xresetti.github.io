# Architecture 1001: x86-64 Assembly Notes

**This is just a page of notes and important things to me to remember while learning Assembly**

# Constants, Registers, Memory

"12" means decimal 12; "```0xF0```" is hex.  "some_function" is the address of the first instruction of the function.  Memory access (use register as pointer): "```[rax]```".  Same as C "```*rax```".
Memory access with offset (use register + offset as pointer): "```[rax+4]```".  Same as C "```*(rax+4)```".
Memory access with scaled index (register + another register * scale): "```[rax+rbx*4]```".  Same as C "```*(rax+rbx*4)```".

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

**Registers are also called general purpose registers and their capacity is 32 bits: 4 bytes (4 sets of 8 bits).**

**The Program Status and Control Register is EFLAGS, which is a collection of 1-bit flags.**

**The Flags aint important, apart from the Trap Flag which basically allows debuggers to single-step through instructions.**

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

### PUSH
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

![Screenshot_17](https://user-images.githubusercontent.com/114181159/234862219-51cfe178-aca9-495b-b295-37e8d074373b.png)

**A scenario: ```push RAX```**

![Screenshot_15](https://user-images.githubusercontent.com/114181159/234850844-3a2b162c-d691-46cc-8b2f-c9b0e10d2921.png)

**What happened?**

1. ```RSP``` value decremented by 8 because the stack pointer ```(RSP)``` is automatically decremented by 8 after being pushed onto the stack
2. ```RSP``` ***WAS*** at memory address ```0x014FE08```. ***AFTERWARDS*** it is at ```0x014FE00```. Again because the stack pointer is automatically decremented by 8 after being pushed.
3. Also, the value of memory address ```0x014FE00``` is now ```3```, because the ```RAX``` value was ```3```. Previously, it was ```undefined```
4. In the end, ```0x014FE00``` is the new stack pointer, and its value is still ```3```. (it was always going to be 3 anyways, because we are pushing a value onto the stack from the predefined register ```RAX```)

### POP
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

### CALL
- Transfer control to a different function
- First it _pushes_ the address of the next instruction onto the stack (for use by **return address** for when the procedure is done)
- Then changes ```RIP``` to the address given in the instruction
- Destination address for the target function can be specified in multiple ways
- - Absolute address
- - Relative address

### RET - Return from Procedure
- ***Two forms:***
- Pop the top of the stack into ```RIP``` (remember that _pop_ implicitly increments the stack pointer, ```RSP```)
- - ^^^ In this form, the instruction is just written as ```ret```
- Pop the top of the stack into ```RIP``` and also add a constant number of bytes to ```RSP```
- - ^^^ In this form, the instruction is written as ```ret 0x8```, or ```ret 0x20```, etc etc.

### How to read two-operand instructions: Intel vs AT&T Syntax

![Screenshot_18](https://user-images.githubusercontent.com/114181159/234865033-f2cce2ba-01d9-436f-a0ce-afa0f8cbc5d6.png)

### MOV (aka Move)
Can move:
- Register to Register
- Memory to register, register to memory
- Immediate to register, immidate to memory
- NEVER memory to memory!
- Memory addresses are given in ```r/mX``` form

**See examples below:**

![Screenshot_19](https://user-images.githubusercontent.com/114181159/234865564-41a19f9b-5149-49f0-a3d5-3f6d7c1b3686.png)

### ADD and SUB
- Adds or Subtracts, just as expected
- Destination operant can be ```r/mX``` or register
- Source operand can be ```r/mX``` or register or immediate
- No source ***and*** desination as ```r/mX```, because that could allow for memory to memory transfer which isn't allowed on x86

**Examples:**

```add rsp, 8``` -> (rsp = rsp + 8)

```sub rax, [rbx*2]``` -> (rax = rax - memorypointedtoby(rbx * 2))

### Writing in Visual Studio
Writing some simple subroutine call code:

```
int func() {
    return 0xbeef;
}
int main() {
    func();
    return 0xf00d;
}
```

Debugging the code results in the ```RSP``` register being changed after hitting the ```main()``` breakpoint and stepping into the function:

**Before**

![Screenshot_20](https://user-images.githubusercontent.com/114181159/234883443-3c41278b-97d0-41e6-8dcf-ed86731abe02.png)

**After**

![Screenshot_21](https://user-images.githubusercontent.com/114181159/234883461-32d3c245-d45a-4ede-a53f-1cc0cce44085.png)

