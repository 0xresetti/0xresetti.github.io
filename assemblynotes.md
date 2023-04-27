# Architecture 1001: x86-64 Assembly Notes

**This is just a page of notes and important things to me to remember while going through the "Architecture 1001: x86-64 Assembly" Course.**

# Endianness
Little Endian Example:
```0x12345678``` = ```0x78, 0x56, 0x34, 0x12```

Big Endian Example:
```0x12345678``` = ```0x12, 0x34, 0x56, 0x78```

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

[CheatSheet_x86-64_Registers.pdf](https://github.com/0xwyvn/0xwyvn.github.io/files/11342258/CheatSheet_x86-64_Registers.pdf)
