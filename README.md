# Buffer Overflow Vulnerability

This is a short tutorial on running a simple buffer overflow on a virtual machine
running Ubuntu. It shows how one can use a buffer overflow to obtain a root 
shell. The end of the tutorial also demonstrates how two defenses in the Ubuntu
OS prevent the simple buffer overflow attack implemented here.

This tutorial leverages the paper written by alephOne on buffer overflows:
<http://cecs.wright.edu/~tkprasad/courses/cs781/alephOne.html>

## Initial Setup of the Virtual Machine

Let us first go through the initial setup of the virtual machine used to
experiment with buffer overflow. The virtual machine used runs Ubuntu OS `12.04
LTS`. To make our attack easier, we first need to disable address space
randomization, a defense against buffer overflows making guessing addrsses in
the heap and stack more difficult.  To do so, we simply need to run the
following command under root privileges:

```
su root 
sysctl -w kernel.randomize_va_space=0
```

A confirmation of the variable's value is printed `kernel.randomize_va_space = 0`
by the terminal. 

## Example of a Shellcode

The file `call_shellcode.c` containes an example shellcode, which allows one to 
store a `char` in a buffer and then call the shell by a buffer overflow. To run
the shell, we can compile `call_shellcode.c` using the executable stack option
in `gcc`. Running the program `./call_shellcode` from the terminal starts a
shell, which can for instance be used to run programs (e.g., `vim`).

This shellcode was however not injected so this example does not really
correspond to a realistic threat model. Instead, we now show how one can inject
the buffer using a file loaded by a vulnerable program. 

## Vulnerable Program

The vulnerable program is provided in the `stack.c` file. It needs to be made
a set-root-uid in order for the adversary exploiting the buffer overflow to be
able to gain access to a root shell. For that purpose, we compile the file using
root privileges. Furthermore, if `GCC>4.3.3` is used, since the Stack Guard
option is enabled by default, one needs to disable it at compile time (cf. 
below). Note that we also use the executable stack option (to be able to run 
our shellcode from the buffer). Finally, to make the file executable, we `chmod`
the permissions to `4755` on the compiled program `stack`.  

```
su root 
gcc -o stack -z execstack -fno-stack-protector stack.c
chmod 4755 stack 
```

## Exploiting the Vulnerability: Demonstration of the Buffer Overflow Attack

We now need to craft the `badfile` file that will be read by this vulnerable
program 'stack' and stored in the buffer, which will be overflowed. The file
`exploit.c` contains code that dumps the buffer that will be read by the
vulnerable program. The code is well commented and should be fairly
understandable and leverages sample code provided in the paper by alephOne. 

To demonstrate the buffer flow attack, we run the following commands:

```
gcc -o exploit exploit.c 
./exploit
./stack
```

This simply compiles and runs the exploit file. The exploit file evaluates the 
stack pointer and crafts a buffer (with the stack pointer and the shellcode) 
and saves it to `badfile`. The vulnerable program `stack` is then executed, it 
reads the file `badfile` and loads the buffer, which triggers the buffer overflow
and executes the shellcode, thus giving us a root shell (designated by `#`). 

Note that the root shell we have obtained is still using our user ID, as proved
by running the `id` command. To solve this and have both the real and effective 
user ids set to root, one can run the included `set_uid_root.c` file.

## Address Randomization: a first defense

One can set Ubuntu's address randomization back on using:

```
$ su root
# /sbin/sysctl -w kernel.randomize_va_space=2
```

Running the attack described in the previous section gives a 
`segmentation fault (core dumped)` error because the address is randomized each
time the program is executed. Therefore, the stack pointer is different and the
`exploit.c` program will not set the address properly anymore for the buffer
flow to run the shellcode. 

## Stack Guard: a second defense

To analyze one defense at a time, it is best to first turn off again address
randomization, as performed in the initial setup. One can then repeat the
buffer overflow attack but this time compiling the vulnerable program `stack`
with the Stack Guard protection mechanism (i.e. removing the flag previously
used: `-fno-stack-protector`). 

```
su root 
gcc -o stack -z execstack stack.c
```

This time, the Stack Guard option in `gcc` was able to allow us to detect the
smashing attemp. This effectively terminates the program and prevents the 
attack. Here is a screen dump:

```
*** stack smashing detected ***: ./stack terminated
======= Backtrace: =========
/lib/i386-linux-gnu/libc.so.6(__fortify_fail+0x45)[0xb7f240e5]
/lib/i386-linux-gnu/libc.so.6(+0x10409a)[0xb7f2409a]
./stack[0x8048513]
[0xbffff33c]
[0x2f6850c0]
======= Memory map: ========
08048000-08049000 r-xp 00000000 08:01 1582540    /home/***/Documents/stack
08049000-0804a000 r-xp 00000000 08:01 1582540    /home/***/Documents/stack
0804a000-0804b000 rwxp 00001000 08:01 1582540    /home/***/Documents/stack
0804b000-0806c000 rwxp 00000000 00:00 0          [heap]
b7def000-b7e0b000 r-xp 00000000 08:01 2360149    /lib/i386-linux-gnu/libgcc_s.so.1
b7e0b000-b7e0c000 r-xp 0001b000 08:01 2360149    /lib/i386-linux-gnu/libgcc_s.so.1
b7e0c000-b7e0d000 rwxp 0001c000 08:01 2360149    /lib/i386-linux-gnu/libgcc_s.so.1
b7e1f000-b7e20000 rwxp 00000000 00:00 0 
b7e20000-b7fc3000 r-xp 00000000 08:01 2360304    /lib/i386-linux-gnu/libc-2.15.so
b7fc3000-b7fc5000 r-xp 001a3000 08:01 2360304    /lib/i386-linux-gnu/libc-2.15.so
b7fc5000-b7fc6000 rwxp 001a5000 08:01 2360304    /lib/i386-linux-gnu/libc-2.15.so
b7fc6000-b7fc9000 rwxp 00000000 00:00 0 
b7fd9000-b7fdd000 rwxp 00000000 00:00 0 
b7fdd000-b7fde000 r-xp 00000000 00:00 0          [vdso]
b7fde000-b7ffe000 r-xp 00000000 08:01 2364405    /lib/i386-linux-gnu/ld-2.15.so
b7ffe000-b7fff000 r-xp 0001f000 08:01 2364405    /lib/i386-linux-gnu/ld-2.15.so
b7fff000-b8000000 rwxp 00020000 08:01 2364405    /lib/i386-linux-gnu/ld-2.15.so
bffdf000-c0000000 rwxp 00000000 00:00 0          [stack]
Aborted (core dumped)
```

## How to contact me

Ways to get in touch with me:
* Twitter: <https://www.twitter.com/nicolaspapernot>
* Webpage: <https://www.papernot.fr> 

## License

This was adapted from a SEED lab.

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.






