# Reverse Engineering Notes

**This is just a page of notes and important things to me to remember while learning Reverse Engineering**

### ...

- **Each thread executing inside a process has its own stack. // 5 Threads = 5 Stacks**

- **When a function is called, it places its parameters on the stack, PUSH places data on top of the stack and POP removes that value from the top of the stack. LIFO (Last In - First Out).**

- **When an item is pushed onto the stack, the ESP register, which always points to the Top of the Stack, is decremented in order to point to the new item placed on the Top. Works the same the other way around, POPing from the stack will increment in order to point to the item below. EBP just points to the bottom becauSE IT IS A BASE POINTER.**

- **CALL is obviously responsible for executing functions, it loads the entry point of the function into the EIP register in order to run it. RET for exiting functions**

- **The program in execution needs to "remember" the return address each time the execution flow enters a function and that function needs to have its own memory area where it can store return addresses, local variables, etc. This is done by splitting the memory assigned for the stack into stack frames, each frame holds the information just mentioned for each function.**

