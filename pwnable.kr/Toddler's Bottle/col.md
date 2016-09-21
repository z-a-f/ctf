## Question

> Daddy told me about cool MD5 hash collision today.
> I wanna do something like that too!
> 
> ssh col@pwnable.kr -p2222 (pw:guest)

<button class="section" target="solution" show="Show solution" hide="Hide solution"></button>

<!--sec data-title="Solution" data-id="solution" data-show=false ces-->

### Step 1: SSH to the remote, and look around

```sh
$> ssh col@pwnable.kr -p2222
col@pwnable.kr's password:
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
No mail.

col@ubuntu:~$ ls
col  col.c  flag

col@ubuntu:~$ cat flag
cat: flag: Permission denied
```

The flag is again not readable (duh-duh!) - Maybe the other file has something :)


### Step 2: See what is inside other files

```bash
col@ubuntu:~$ cat col.c
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
        int* ip = (int*)p;
        int i;
        int res=0;
        for(i=0; i<5; i++){
            res += ip[i];
        }
        return res;
}

int main(int argc, char* argv[]){
        if(argc<2){
            printf("usage : %s [passcode]\n", argv[0]);
            return 0;
        }
        if(strlen(argv[1]) != 20){
            printf("passcode length should be 20 bytes\n");
            return 0;
        }

        if(hashcode == check_password( argv[1] )){
            system("/bin/cat flag");
            return 0;
        }
        else
            printf("wrong passcode.\n");
        return 0;
}
col@ubuntu:~$
```

### Step 2.5: What the hell is going on?

This is a weird hashing function. It takes 20 character, and casts them as `int`.
Because the size of `int` is the `4 x sizeof(char)`, there are 5 `int`s. 
After that the hashing code (aka `check_password`), iterates through all of them and adds them up.

### Step 3: Grab the flag

To grab the flag, we need to make sure the `check_password(_our_input_)` returns `hashcode =  568134124` (`0x21DD09EC` in decimal).
Because `check_password` splits the input into 5 parts, and adds them up, we can get the flag by providing it with some 5 numbers that add up to 568134124: 

```python
$> python
>>> num1 = 0x21DD09EC / 5
>>> num2 = 0x21DD09EC - num1 * 4
>>> print hex(num1), hex(num2)
0x6c5cec8 0x6c5cecc
```

Let's try passing the numbers above - but don't forget that it is __little-endian__!

```bash
$> ssh col@pwnable.kr -p2222
col@pwnable.kr\'s password:
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

col@ubuntu:~$ ls
col  col.c  flag

col@ubuntu:~$ ./col $(python -c "print '\xc8\xce\xc5\x06'*4 + '\xcc\xce\xc5\x06'")

```




So the flag is `mommy! I think I know what a file descriptor is!!`
<!--endsec-->


