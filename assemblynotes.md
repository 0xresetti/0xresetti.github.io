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


