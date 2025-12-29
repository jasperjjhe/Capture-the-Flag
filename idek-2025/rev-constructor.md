# constructor (rev)

Credit: Zerotistic

## Description

Heard of constructor?

## Downloads

[constructor.tar.gz](https://github.com/jasperjjhe/Capture-the-Flag/blob/0d36de7207c600e87faa2dcba42eb955f6c3f629/idek-2025/constructor.tar.gz)

## Approach:

Just to cover my bases, I started by just opening the file in Notepad. As suspected, this yielded nothing but gibberish, so I turned to PowerShell and booted up Windows Subsystem for Linux (WSL).

Let's see what happens if I just:

```console
jasperjjhe@machine:~/constructor$ ./chall
👀
```

Okay, wasn't expecting it to be that easy anyway.

```console
jasperjjhe@machine:~/constructor$ gdb ./chall
GNU gdb (Ubuntu 12.1-0ubuntu1~22.04.2) 12.1
Copyright (C) 2022 Free Software Foundation, Inc.
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
Reading symbols from ./chall...
(No debugging symbols found in ./chall)
(gdb)
```

No debugging symbols means I'll have to find the insertion point manually. While I'm at it, it's a good idea to look for `.init_array`, too.

```console
jasperjjhe@machine:~/constructor$ readelf -h ./chall | grep "Entry point"
  Entry point address:               0x40118b
jasperjjhe@machine:~/constructor$ readelf -S ./chall | grep init_array
  [ 6] .init_array INIT_ARRAY 0000000000404fb0 00003fb0
```

So now I know where our entry point is in this program, which means I can set a breakpoint!

```console
jasperjjhe@machine:~/constructor$ gdb ./chall
GNU gdb (Ubuntu 12.1-0ubuntu1~22.04.2) 12.1
Copyright (C) 2022 Free Software Foundation, Inc.
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
Reading symbols from ./chall...
(No debugging symbols found in ./chall)
```

Even though the symbols have been intentionally removed, I've found the actual address of the program's entry point, so I can set that as a breakpoint:

```console
(gdb) break *0x40118b
Breakpoint 1 at 0x40118b
(gdb) run
Starting program: ~/constructor/chall

Breakpoint 1, 0x000000000040118b in ?? ()
```

Great! We are paused right at the beginning of the program. Let's examine four 8-byte values in `.init_array` which we know the address of from before.

```console
(gdb) x/4gx 0x404fb0
0x404fb0:       0x0000000000401290      0x0000000000401030
0x404fc0:       0x0000000000401050      0x0000000000401250
```

These new addresses probably point to functions; we'll set more breakpoints for them.

```console
(gdb) break *0x401290
Breakpoint 2 at 0x401290
(gdb) break *0x401050
Breakpoint 3 at 0x401050
(gdb) continue
Continuing.

Breakpoint 2, 0x0000000000401290 in ?? ()
```

We've probably hit a function, let's take a look at it.

```console
(gdb) x/20i $rip
=> 0x401290:    endbr64
   0x401294:    jmp    0x401210
   0x401299:    nopl   0x0(%rax)
   0x4012a0:    endbr64
   0x4012a4:    ret
   0x4012a5:    cs nopw 0x0(%rax,%rax,1)
   0x4012af:    nop
   0x4012b0:    endbr64
   0x4012b4:    ret
   0x4012b5:    cs nopw 0x0(%rax,%rax,1)
   0x4012bf:    nop
   0x4012c0:    endbr64
   0x4012c4:    sub    $0x158,%rsp
   0x4012cb:    mov    %rdi,%rdx
   0x4012ce:    xor    %eax,%eax
   0x4012d0:    mov    $0x26,%ecx
   0x4012d5:    lea    0x20(%rsp),%r8
   0x4012da:    mov    %r8,%rdi
   0x4012dd:    rep stos %rax,%es:(%rdi)
   0x4012e0:    mov    $0x405628,%rax
```
