## Question

> Papa brought me a packed present! let's open it.
>
> Download : http://pwnable.kr/bin/flag
> 
> This is reversing task. all you need is binary

<button class="section" target="solution" show="Show solution" hide="Hide solution"></button>

<!--sec data-title="Solution" data-id="solution" data-show=false ces-->
I finally got a machine for debugging running!!! :))))


### Step 1: Examine the file

After downloading the file (`wget http://pwnable.kr/bin/flag`), I spent five minutes trying to run in in GDM/LLDB/R2 - no luck.
Furstrated, I catted it, and here what it says:

```bash
$> ./flag
I will malloc() and strcpy the flag there. take it.

$> cat flag | tail -n1
�)�T �+���5�Y�@d`���C)�0��Y0!�Upbrk��j6���z ηwĊ���is���#�makBN�H,���2*E�        � �&V"�I��I���!�1E^p!|�hX��de㵋zT�su`"]R����%��mI��A���$�UPX!UPX!
```

Note the last thing it says is `UPX!`.
Quick search has shown that it's an executable packer - so we probably need to unpack it.
I just got the UPX from their [website](https://upx.github.io/), and here is the result:

```bash
$> sudo apt install upx
$> man upx
   Decompress
       All UPX supported file formats can be unpacked using the -d switch, eg.
       upx -d yourfile.exe will uncompress the file you've just compressed.
$> upx -d flag
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2013
UPX 3.91        Markus Oberhumer, Laszlo Molnar & John Reiser   Sep 30th 2013

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
    887219 <-    335288   37.79%  linux/ElfAMD   flag

Unpacked 1 file.
$>
```

I guess it is unpacked now:

```bash
$> cat flag | tail -n1
[...]l_map_object_deps_nl_C_LC_IDENTIFICATION_dl_ns_nl_load_locale_from_archivewctransfopen64__cache_sysconf
```

### Step 2: GDB time

```
(gdb) disas main
Dump of assembler code for function main:
   0x0000000000401164 <+0>: push   %rbp
   0x0000000000401165 <+1>: mov    %rsp,%rbp
   0x0000000000401168 <+4>: sub    $0x10,%rsp
   0x000000000040116c <+8>: mov    $0x496658,%edi
   0x0000000000401171 <+13>:    callq  0x402080 <puts>
   0x0000000000401176 <+18>:    mov    $0x64,%edi
   0x000000000040117b <+23>:    callq  0x4099d0 <malloc>
   0x0000000000401180 <+28>:    mov    %rax,-0x8(%rbp)
   0x0000000000401184 <+32>:    mov    0x2c0ee5(%rip),%rdx        # 0x6c2070 <flag>
   0x000000000040118b <+39>:    mov    -0x8(%rbp),%rax
   0x000000000040118f <+43>:    mov    %rdx,%rsi
   0x0000000000401192 <+46>:    mov    %rax,%rdi
   0x0000000000401195 <+49>:    callq  0x400320
   0x000000000040119a <+54>:    mov    $0x0,%eax
   0x000000000040119f <+59>:    leaveq 
   0x00000000004011a0 <+60>:    retq   
End of assembler dump.
(gdb) break *0x0000000000401184
Breakpoint 1 at 0x401184
```

How cute - there is a even a comment with a pointer to the flag. 
I put a breakpoints right after the `<flag>`, and we will check what is stored in the `$rdx`.
Note that the assumption is that the flag is a well-formatted string, and `x/s` should be able to read it.

```
(gdb) br *0x000000000040118b
Breakpoint 2 at 0x40118b
(gdb) r
Starting program: [...]/flag 
I will malloc() and strcpy the flag there. take it.

Breakpoint 1, 0x0000000000401184 in main ()
(gdb) x/s $rdx
0x6c27c0 <main_arena>:  ""
(gdb) c
Continuing.

Breakpoint 2, 0x000000000040118b in main ()
(gdb) x/s $rdx
0x496628:   "UPX...? sounds like a delivery service :)"
(gdb)
```

### Step 000: Another way of doing it

Actually while running the whole thing, I accidentally dictionary-brute-forced the flag, and it worked :)

```bash
$> upx -d flag
[...]
$> strings flag | grep -i flag
I will malloc() and strcpy the flag there. take it.
s->_flags2 & 4
version == ((void *)0) || (flags & ~(DL_LOOKUP_ADD_DEPENDENCY | DL_LOOKUP_GSCOPE_LOCK)) == 0
imap->l_type == lt_loaded && (imap->l_flags_1 & 0x00000008) == 0
flag.c
flag
_dl_stack_flags
$> strings flag | grep -i papa
$> strings flag | grep -i present
$> strings flag | grep -i upx
UPX...? sounds like a delivery service :)
$> 

```

### Flag

`UPX...? sounds like a delivery service :)`
<!--endsec-->


