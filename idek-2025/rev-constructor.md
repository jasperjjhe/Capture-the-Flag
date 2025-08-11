# Challenge:

## rev/constructor

### Credit: Zerotistic

"Heard of constructor?"

### Downloads:

[constructor.tar.gz](https://github.com/jasperjjhe/Capture-the-Flag/blob/0d36de7207c600e87faa2dcba42eb955f6c3f629/idek-2025/constructor.tar.gz)

# Approach:

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

No debugging symbols means I'll have to find the insertion point manually.

```console
jasperjjhe@machine:~/constructor$ readelf -h ./chall | grep "Entry point"
  Entry point address:               0x40118b
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
(gdb)
```

Great! We are paused right at the beginning of the program.
