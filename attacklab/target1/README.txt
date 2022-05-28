level 1:
    sub rsp, 0x28 --> BUFFER_SIZE = 40 bytes
    so we need to overwrite the original return address with touch1's address

level 2:
    similar to level 1, but this time, we need to make the return address pointing to the beginning of stack
    base(the one after sub rsp, 0x28), where we will store the byte representation of our exploit code;
    to do so, we set up a break point at 0x4017ac, and use 'info reg' to determine the stack base address
    
level 3:
    similar to level 2, where we make the return address pointing to the beginning of stack base to execute
    our exploit code. However, we also need to find a place to put the cookie; Now because hexmatch and strncmp 
    together overwrite getbuf's stack frame, thus we cannot put the cookie within getbuf's stack frame. Instead, 
    we will put it somewhere in the test's stack frame. Let's put it at the end. Notice that this address is
    equivalent to rsp after ret executes. [two things to be considered 1). need to fill the return address with
    0s to make it exactly 8 bytes since we are using rsp 2). need to add 00 after the cookie]
    
level 4:
    the idea of the return-oriented attack is that, we can use some of the existing assembly instruction provided
    to reach the target. In this case, we want to seek instructions from farm.c that can 1) popq %rdi (retrieve 
    cookie from stack) 2) ret (jump to touch2). The instruction for 1) is 5a, nevertheless, it cannot be found in the 
    farm.c, thus, we need to separate 1) into 2 instructions, popq %rax, movq %rax, %rdx, which happens to be located
    at 0x4019ab and 0x4019a2 respectively. 
    flow: overwrite return address with 0x4019ab, which will execute 58 90(NOP) c3 and jumps to 0x4019a2, which will
    then execute 48 89 c7 c3 that will jump to touch2

level 5:
    because the stack needs to be constantly popped, so we cannot directly move cookie's address to a register. 
    However, notice that there is a code lea (%rdi,%rsi,1), %rax that let us calculate the address relative to stack 
    top. So, we can put the offset in rsi and stack top address(when stack doesn't need to be popped anymore) in rdi. 
    Analysing the code, there is an instruction movl %ecx,%esi that fulfills our goal. Going backwards, 
    movl %edx, %ecx -> movl %eax,%edx. Thus, we know that we can store the offset in rax at the start. This offset can 
    be determined easily by observing that we are basically left with 4 additional instructions(movq %rsp, %rax -> 
    movq %rax, %rdi -> lea (%rdi,%rsi,1), %rax -> movq %rax, %rdi), which amounts to 32 bytes(0x20). 
