# constructor
_Reverse_

Credit: Zerotistic

## Description

Heard of constructor?

## Download

[constructor.tar.gz](https://github.com/jasperjjhe/Capture-the-Flag/blob/main/idek-2025/constructor.tar.gz)

## Approach

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

We've probably hit a function, let's take a look at it.We've hit something. Let's disassemble and see what's at this address.

```console
(gdb) x/30i $rip
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
   0x4012e7:    mov    %rdx,(%rax)
   0x4012ea:    cmpq   $0x0,(%rdx)
   0x4012ee:    je     0x4014a8
   0x4012f4:    xor    %eax,%eax
   0x4012f6:    cs nopw 0x0(%rax,%rax,1)
   0x401300:    mov    %rax,%rcx
   0x401303:    add    $0x1,%rax
   0x401307:    cmpq   $0x0,(%rdx,%rax,8)
   0x40130c:    jne    0x401300
   0x40130e:    lea    0x10(,%rcx,8),%rax
```
The first thing this function does at `0x401290` is an unconditional `jmp` to `0x401210`; the real logic is actually elsewhere. This is a common pattern in C++ programs using constructor functions registered in `.init_array`: the entry stubs jump to the real implementation. The actual meat starts at `0x4012c0`, which allocates a `0x158`-byte stack frame and starts working.

Let's extend the disassembly further to understand what it's doing:

```console
(gdb) x/50i $rip
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
   0x4012e7:    mov    %rdx,(%rax)
   0x4012ea:    cmpq   $0x0,(%rdx)
   0x4012ee:    je     0x4014a8
   0x4012f4:    xor    %eax,%eax
   0x4012f6:    cs nopw 0x0(%rax,%rax,1)
   0x401300:    mov    %rax,%rcx
   0x401303:    add    $0x1,%rax
   0x401307:    cmpq   $0x0,(%rdx,%rax,8)
   0x40130c:    jne    0x401300
   0x40130e:    lea    0x10(,%rcx,8),%rax
   0x401316:    add    %rdx,%rax
   0x401319:    mov    %rax,0x3e88(%rip)        # 0x4051a8
   0x401320:    mov    (%rax),%rdx
   0x401323:    add    $0x8,%rax
   0x401327:    test   %rdx,%rdx
   0x40132a:    je     0x4014b8
   0x401330:    cmp    $0x25,%rdx
   0x401334:    ja     0x40133e
   0x401336:    mov    (%rax),%rcx
   0x401339:    mov    %rcx,0x20(%rsp,%rdx,8)
   0x40133e:    mov    0x8(%rax),%rdx
   0x401342:    add    $0x10,%rax
   0x401346:    test   %rdx,%rdx
   0x401349:    jne    0x401330
   0x40134b:    mov    0xa0(%rsp),%rcx
   0x401353:    mov    0x120(%rsp),%rax
   0x40135b:    mov    0x50(%rsp),%rdx
   0x401360:    mov    %rcx,0x3e29(%rip)        # 0x405190
   0x401367:    test   %rax,%rax
   0x40136a:    je     0x401373
```
This is parsing `envp` (the environment pointer passed as `rdi`). It's iterating over the environment variable entries, checking that values are non-null, and storing them into a local array on the stack indexed by key. The `cmp $0x25,%rdx` (comparing to 37 decimal) is a bounds check, so it's only storing values for indices 0–37.

There are also two jump targets we should pay attention to: `0x4014a8` and `0x4014b8`, which are the "empty `envp`" and "done parsing" exit paths respectively. Let's set breakpoints on those too and let the constructor run:
```console
(gdb) break *0x4014a8
Breakpoint 4 at 0x4014a8
(gdb) break *0x4014b8
Breakpoint 5 at 0x4014b8
(gdb) continue
Continuing.
```
Instead of hitting breakpoints 4 or 5, we hit Breakpoint 3 at `0x401050`. This means the first constructor at `0x401290` finished cleanly without going through either of the "error" paths and we're now in the second constructor function.
```console
Breakpoint 3, 0x0000000000401050 in ?? ()
(gdb) x/30i $rip
=> 0x401050:    endbr64
   0x401054:    push   %r13
   0x401056:    push   %r12
   0x401058:    push   %rbp
   0x401059:    push   %rbx
   0x40105a:    sub    $0x1000,%rsp
   0x401061:    orq    $0x0,(%rsp)
   0x401066:    sub    $0x18,%rsp
   0x40106a:    xor    %ecx,%ecx
   0x40106c:    lea    0x40cd(%rip),%rbx        # 0x405140
   0x401073:    lea    0x1fc6(%rip),%rdi        # 0x403040
   0x40107a:    mov    %fs:0x28,%rax
   0x401083:    mov    %rax,0x1008(%rsp)
   0x40108b:    xor    %eax,%eax
   0x40108d:    nopl   (%rax)
   0x401090:    movzbl (%rdi,%rax,1),%edx
   0x401094:    mov    %rax,%rsi
   0x401097:    shr    %rsi
   0x40109a:    xor    %ecx,%edx
   0x40109c:    add    $0x1f,%ecx
   0x40109f:    xor    %esi,%edx
   0x4010a1:    xor    $0x5a,%edx
   0x4010a4:    mov    %dl,(%rbx,%rax,1)
   0x4010a7:    add    $0x1,%rax
   0x4010ab:    cmp    $0x2a,%rax
   0x4010af:    jne    0x401090
   0x4010b1:    xor    %esi,%esi
   0x4010b3:    lea    0x1f46(%rip),%rdi        # 0x403000
   0x4010ba:    xor    %eax,%eax
   0x4010bc:    movb   $0x0,0x40a7(%rip)        # 0x40516a
```
This is the interesting one. The function is:
1. Loading a byte from `0x403040` (likely an obfuscated/encrypted blob of data)
2. XOR-ing it with a counter-based value (`ecx`, which increments by `0x1f` each iteration)
3. XOR-ing again with `rsi` (the loop index right-shifted by 1)
4. XOR-ing again with the constant `0x5a`
5. Writing the result to `0x405140` (the destination buffer, stored in `rbx`)

It runs for `0x2a` (42) iterations, decoding 42 bytes in total. This is a classic XOR decode loop; the flag is being decrypted in the `.init_array` constructor before `main()` even runs! The challenge name "constructor" is a very direct hint at exactly this.

Let's set a breakpoint right after the decode loop finishes, at `0x4010b1`:
```console
(gdb) nexti
0x0000000000401054 in ?? ()
(gdb) break *0x4010b1
Breakpoint 6 at 0x4010b1
(gdb) continue
Continuing.

Breakpoint 6, 0x00000000004010b1 in ?? ()
```
We're now past the decode loop. Let's inspect the destination buffer at `0x405140`:
```console
(gdb) x/s 0x405140
0x405140:       "idek{he4rd_0f_constructors?_now_you_d1d!!}"
```
🚩

## Summary
The challenge was a stripped ELF binary that printed only an emoji when run normally, making it look unsolvable at first glance. The flag was never in plaintext in the binary; it was stored obfuscated at `0x403040` and decoded at runtime by a C++ constructor function registered in `.init_array`, which runs before `main()`.

The decode routine used a rolling XOR with three components:

- A counter value (`ecx`) incrementing by `0x1f` per byte
- The loop index right-shifted by 1 (`rsi >> 1`)
- A static XOR key of `0x5a`

By using `readelf` to find both the entry point and `.init_array`, then setting a GDB breakpoint after the decode loop completed, we were able to read the flag directly from memory before the program continued executing. No manual XOR decoding required since the binary did all the work for us.
So, when reversing stripped binaries, always check `.init_array` and `.fini_array` for constructor/destructor functions. Interesting logic (like flag decryption) is often deliberately hidden there precisely because many reverse engineers go straight to `main()`.
