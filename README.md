# CTF-challenge-hack-the-flag
This project was an attempt to modify an ELF binary to bypass a function `check_flag` by patching ARM thumb assembly instructions.

# Instructions for Getting the Debugger Flag

This tutorial will show you how to use the GDB debugger to walk through and
interact with a MAX78000fhtr board.

## 1. Launch in Debug Mode with Docker and OpenOCD
Using a nix-shell start an openocd connection to get the debugger running:
```bash
nix-shell
 sudo openocd -s scripts/ -f interface/cmsis-dap.cfg -f target/max78000.cfg -c "bindto 0.0.0.0; init"
```
Now use docker to start the debugger in the decoder/ directory of the project and connect to the openocd server:
```bash
sudo docker run --rm -it -p 3333:3333/tcp -v $(pwd)/./build_out:/out --workdir=/root --entrypoint /bin/bash decoder -c " cp -r /out/* /root/ && gdb-multiarch gdb_challenge_25.elf "
```

## 2. Getting Your Bearings
When the deployment finishes spinning up, you should see output along the lines of:
```
GNU gdb (Ubuntu 15.0.50.20240403-0ubuntu1) 15.0.50.20240403-git
Copyright (C) 2024 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from gdb_challenge_25.elf...
(gdb) target remote host.docker.internal:3333
Remote debugging using host.docker.internal:3333
0x1000ee6e in MXC_Delay (us=<optimized out>)
    at /root/msdk-2024_02/Libraries/CMSIS/../PeriphDrivers/Source/SYS/mxc_delay.c:233
233         while (SysTick->VAL > endtick) {}
```

You are now using GDB.

Let's view the current register values with `info`:
```
(gdb) info registers
r0             0x363c7f            3554431
r1             0xe000e000          -536813568
r2             0x363c7f            3554431
r3             0x7b0d32            8064306
r4             0x20000004          536870916
r5             0x0                 0
r6             0x0                 0
r7             0x2001fff8          537001976
r8             0x0                 0
r9             0x0                 0
r10            0x0                 0
r11            0x0                 0
r12            0xf4240000          -198967296
sp             0x2001ffe8          0x2001ffe8
lr             0x1000ed1d          268496157
pc             0x1000ee6e          0x1000ee6e <MXC_Delay+46>
xPSR           0x21000000          553648128
fpscr          0x0                 0
msp            0x2001ffe8          0x2001ffe8
psp            0x0                 0x0
primask        0x0                 0
basepri        0x0                 0
faultmask      0x0                 0
control        0x0                 0
```

We can also view values in memory with `x` specified by address or symbol:
```
(gdb) x gdb_challenge
0x1000e5d0 <gdb_challenge>:     0xb084b580
(gdb)  x to_hex
0x1000e738 <to_hex>:    0x4c09b410

```
Now because the addresses are read-only there will be a signifcant problem
later when modifying registers, the designers did not want the program to be
modified. Instead, we will be modifying the memory at the address of the
instruction to change the program's behavior.

**Write down the raw value of the instruction in memory at to_hex for later
as value1**


Let's try that with `do_some_math`:
```
(gdb) b do_some_math
Breakpoint 2 at 0x1000e290: file src/debugger_challenge.c, line 39.
(gdb) info registers sp
sp             0x2001fff8          0x2001fff8
(gdb) c
Continuing.

Thread 2 "max32xxx.cpu" hit Breakpoint 2, do_some_math (a=0, b=0, c=268496157, d=537001936)
    at src/debugger_challenge.c:39
39      in src/debugger_challenge.c
(gdb) info registers sp
sp             0x2001ffe0          0x2001ffe0
(gdb)
```

**Inspect the registers and write down the value in `sp` as value2**


## 3. Stepping Through Code
Let's skip ahead to a function called `do_some_math` (without skipping the
function prologue):
```
(gdb) (gdb) p do_some_math
$1 = {int (int, int, int, int)} 0x1000e290 <do_some_math> 
(gdb) b *$1
Note: breakpoint 2 also set at pc 0x1000e290.
Breakpoint 3 at 0x1000e290: file src/debugger_challenge.c, line 39.
(gdb) c
Continuing.

Thread 2 "max32xxx.cpu" hit Breakpoint 1, gdb_challenge () at src/debugger_challenge.c:102
102     in src/debugger_challenge.c
```

Now, let's inspect the disassembly of this function with `disass`:
```
(gdb) disass do_some_math
Dump of assembler code for function do_some_math:
=> 0x1000e290 <+0>:     push    {r7}
   0x1000e292 <+2>:     sub     sp, #20
   0x1000e294 <+4>:     add     r7, sp, #0
   0x1000e296 <+6>:     str     r0, [r7, #12]
   0x1000e298 <+8>:     str     r1, [r7, #8]
   0x1000e29a <+10>:    str     r2, [r7, #4]
   0x1000e29c <+12>:    str     r3, [r7, #0]
   0x1000e29e <+14>:    ldr     r2, [r7, #12]
   0x1000e2a0 <+16>:    ldr     r3, [r7, #8]
   0x1000e2a2 <+18>:    add     r3, r2
   0x1000e2a4 <+20>:    ldr     r1, [r7, #8]
   0x1000e2a6 <+22>:    ldr     r2, [r7, #4]
   0x1000e2a8 <+24>:    sdiv    r2, r1, r2
   0x1000e2ac <+28>:    mul.w   r2, r3, r2
   0x1000e2b0 <+32>:    ldr     r3, [r7, #0]
   0x1000e2b2 <+34>:    ldr     r1, [r7, #12]
   0x1000e2b4 <+36>:    sdiv    r1, r3, r1
   0x1000e2b8 <+40>:    ldr     r0, [r7, #12]
   0x1000e2ba <+42>:    mul.w   r1, r0, r1
   0x1000e2be <+46>:    subs    r3, r3, r1
   0x1000e2c0 <+48>:    mul.w   r3, r2, r3
   0x1000e2c4 <+52>:    ldr     r1, [r7, #12]
   0x1000e2c6 <+54>:    ldr     r2, [r7, #8]
   0x1000e2c8 <+56>:    eors    r1, r2
```

Instead of using breakpoints, we can instead step through this function
instruction by instruction using `si` for step instruction:
```
(gdb) si
0x1000e292      39      in src/debugger_challenge.c
(gdb) si
0x1000e294      39      in src/debugger_challenge.c
(gdb) si
0x1000e296      39      in src/debugger_challenge.c
```

You can see that we are stepping through the instructions of the function.
If you run `disass` again, you will see our position has changed:
```
(gdb) disass
Dump of assembler code for function do_some_math:
   0x1000e290 <+0>:     push    {r7}
   0x1000e292 <+2>:     sub     sp, #20
   0x1000e294 <+4>:     add     r7, sp, #0
=> 0x1000e296 <+6>:     str     r0, [r7, #12]
   0x1000e298 <+8>:     str     r1, [r7, #8]
   0x1000e29a <+10>:    str     r2, [r7, #4]
   0x1000e29c <+12>:    str     r3, [r7, #0]
   0x1000e29e <+14>:    ldr     r2, [r7, #12]
   0x1000e2a0 <+16>:    ldr     r3, [r7, #8]
   0x1000e2a2 <+18>:    add     r3, r2
   0x1000e2a4 <+20>:    ldr     r1, [r7, #8]
   0x1000e2a6 <+22>:    ldr     r2, [r7, #4]
   0x1000e2a8 <+24>:    sdiv    r2, r1, r2
   0x1000e2ac <+28>:    mul.w   r2, r3, r2
   0x1000e2b0 <+32>:    ldr     r3, [r7, #0]
   0x1000e2b2 <+34>:    ldr     r1, [r7, #12]
   0x1000e2b4 <+36>:    sdiv    r1, r3, r1
   0x1000e2b8 <+40>:    ldr     r0, [r7, #12]
   0x1000e2ba <+42>:    mul.w   r1, r0, r1
   0x1000e2be <+46>:    subs    r3, r3, r1
   0x1000e2c0 <+48>:    mul.w   r3, r2, r3
   0x1000e2c4 <+52>:    ldr     r1, [r7, #12]
   0x1000e2c6 <+54>:    ldr     r2, [r7, #8]
   0x1000e2c8 <+56>:    eors    r1, r2
   0x1000e2ca <+58>:    ldr     r2, [r7, #0]
   0x1000e2cc <+60>:    add     r2, r1
   0x1000e2ce <+62>:    sdiv    r1, r3, r2
   0x1000e2d2 <+66>:    mul.w   r2, r1, r2
   0x1000e2d6 <+70>:    subs    r3, r3, r2
   0x1000e2d8 <+72>:    mov     r0, r3
   0x1000e2da <+74>:    adds    r7, #20
   0x1000e2dc <+76>:    mov     sp, r7
   0x1000e2de <+78>:    pop     {r7}
   0x1000e2e0 <+80>:    bx      lr
```
## 4. Set watchpoints
We can set watchpoints to stop the program when a certain memory location is
modified. Let's set a watchpoint at the address of the instruction at $r2:
```
(gdb) watch $r2
(gdb) c
Continuing.

Thread 2 "max32xxx.cpu" hit Watchpoint 4: $r2

Old value = -1057017071
New value = -515934414
do_some_math (a=-559038737, b=-17958194, c=-889271554, d=-1057017071) at src/debugger_challenge.c:41
41      in src/debugger_challenge.c
(gdb) info registers r2
r2             0xe13f7732          -515934414
```

**Step through the instructions until the value of r2 starts with 0xe1 and
record that value as value3**

## 5. Writing to Registers and Memory
Set a breakpoint at 0x1000e2e0 (the end of `do_some_math` and continue there:

```
(gdb) b *0x1000e2e0
Breakpoint 3 at 0x1000e2e0: file src/debugger_challenge.c, line 42.
(gdb) c
Continuing.

Breakpoint 3, 0x1000e2e0 in do_some_math (a=-559038737, b=-17958194, c=-889271554, d=-1057017071)
    at src/debugger_challenge.c:42
42      in src/debugger_challenge.c
(gdb) 
```

With the `set` command we can now modify registers (make sure to reset them):
```
(gdb) info registers r0
r0             0x0                 0
(gdb) set $r0=111
(gdb) info registers r0
r0             0x6f                111
(gdb) set $r0=0
```

And memory:
```
(gdb) x 0x20000000
0x20000000 <pulStack>:	0x00000000
(gdb) set *0x20000000=0x111
(gdb) x 0x20000000
0x20000000 <pulStack>:	0x00000111
(gdb) set *0x20000000=0
```

## 6. Capturing the flag
With what you've learned, set a breakpoint at the first instruction of the
`check_flag` function and continue up to there.

`check_flag` has five arguments; let's check them out. The ARM calling convention
is to place the first four arguments in registers and further arguments are
pushed to the stack.

Print the registers and then the top value on the stack to view the arguments:
```
(gdb) info registers
r0             0x11111111          286331153
r1             0x22222222          572662306
r2             0x33333333          858993459
r3             0x44444444          1145324612
r4             0x0                 0
r5             0x0                 0
r6             0x0                 0
r7             0x2001ffe8          537001960
r8             0x0                 0
r9             0x0                 0
r10            0x0                 0
r11            0x0                 0
r12            0xf4240000          -198967296
sp             0x2001ffe0          0x2001ffe0
lr             0x1000e60b          268494347
pc             0x1000e4b0          0x1000e4b0 <check_flag>
xPSR           0x61000000          1627389952
fpscr          0x0                 0
msp            0x2001ffe0          0x2001ffe0
psp            0x0                 0x0
primask        0x0                 0
basepri        0x0                 0
faultmask      0x0                 0
control        0x0                 0
(gdb) 
```

We can see that arguments 1-4 (0x11111111, 0x22222222, 0x33333333, and
0x44444444) are in registers r0 through r3, and the top value of the stack
hold the fifth argument (0x55555555).
Now, using what you have learned, change the values of the function so that the
first argument is set to `value1`, the third argument is set to `value2`, and
the fifth argument is set to `value3`.

```
(gdb) set $r0 = 0x1000e738
(gdb) set $r2 = 0x2001ffe0
(gdb) set *(int *)($sp) = 0xe13f7732
(gdb) info registers
r0             0x1000e738          268494648
r1             0x22222222          572662306
r2             0x2001ffe0          537001952
r3             0x44444444          1145324612
r4             0x0                 0
r5             0x0                 0
r6             0x0                 0
r7             0x2001ffe8          537001960
r8             0x0                 0
r9             0x0                 0
r10            0x0                 0
r11            0x0                 0
r12            0xf4240000          -198967296
sp             0x2001ffe0          0x2001ffe0
lr             0x1000e60b          268494347
pc             0x1000e4b0          0x1000e4b0 <check_flag>
xPSR           0x61000000          1627389952
fpscr          0x0                 0
msp            0x2001ffe0          0x2001ffe0
psp            0x0                 0x0
primask        0x0                 0
basepri        0x0                 0
faultmask      0x0                 0
control        0x0                 0
(gdb) x *0x2001ffe0
0xe13f7732:     Cannot access memory at address 0xe13f7732
```
We have successfully set the arguments for the function. Now, continue the
program to see the flag:
```
(gdb) c
```

Now we know that the registers cannot be modified, but we can modify the 
binary ELF file to change the instructions at the address of the function.

## 7. Patching the Binary with Hopper Disassembler
Open the ELF file in Hopper Disassembler and navigate to the `check_flag`
function. You will see the instructions that are being executed
```
1000e4be ldr r3, =Ectfdebuggerni ; 0x1000e5b0, "ectf{debugger_nicetrybutnotyet}\\n"
```
This is a flag meant to trick you. The real flag is in the `check_flag` function.

At the address 1000e578
```assembly
movs r2, #0x21
ldr r1, =aTheFirstArgument
movs r0, #0x47
```
This is the address of the instruction that loads the first argument into
memory. There is no change to this because we are not changing the first
argument.

At the .text CODE_XREF=check_flag+198 you will see the following address:
1000e582
This is the address of the instruction that loads the flag into memory.
```assembly
ldr r3, [r7, #0x4]
ldr r2, =0x2001ffe0
cmp r3, r2
beq loc_1000e594
```
The instruction at 0x1000e582 loads the flag into memory. We can change the
value of the flag by changing the instruction at 0x1000e582.

At the .text CODE_XREF=check_flag+216 you will see the following address:
1000e594
This is the address of the instruction that loads the flag into memory.
```assembly
ldr r3, [r7, #0x58]
ldr r2, =0xe13f7732
cmp r3, r2
beq loc_1000e5a6
```
The instruction at 0x1000e594 loads the flag into memory. We can change the
value of the flag by changing the instruction at 0x1000e594.

To always accept the flag i attempted to patch the cmp and beq instructions to
nop instructions. This will always accept the flag.



