# 문제
Daddy, teach me how to use random value in programming!

ssh random@pwnable.kr -p2222 (pw:guest)

---
# 개요
서버에 접속하여 ls 명령을 입력해보면 random, random.c, flag 3개의 파일이 존재하는 것을 알 수 있습니다. cat 명령어를 통해 random.c 파일을 확인합니다.  
```c
#include <stdio.h>

int main(){
	unsigned int random;
	random = rand();	// random value!

	unsigned int key=0;
	scanf("%d", &key);

	if( (key ^ random) == 0xdeadbeef ){
		printf("Good!\n");
		system("/bin/cat flag");
		return 0;
	}

	printf("Wrong, maybe you should try 2^32 cases.\n");
	return 0;
}
```
입력받은 key와 random 함수를 통해 생성된 값을 XOR 연산하여 그 값이 0xdeadbeef 이면 플래그를 출력합니다. rand 함수의 경우 seed 값이 없으면 매번 똑같은 값을 생성하게 됩니다. 이를 이용하여 문제 해결이 가능합니다. 분석을 통해 random 변수의 값을 찾고 XOR 연산을 통해 조건을 만족시킬 수 있습니다.  

# 분석
```
(gdb) disas main
Dump of assembler code for function main:
   0x00000000004005f4 <+0>:	push   rbp
   0x00000000004005f5 <+1>:	mov    rbp,rsp
   0x00000000004005f8 <+4>:	sub    rsp,0x10
   0x00000000004005fc <+8>:	mov    eax,0x0
   0x0000000000400601 <+13>:	call   0x400500 <rand@plt>
   0x0000000000400606 <+18>:	mov    DWORD PTR [rbp-0x4],eax
   0x0000000000400609 <+21>:	mov    DWORD PTR [rbp-0x8],0x0
   0x0000000000400610 <+28>:	mov    eax,0x400760
   0x0000000000400615 <+33>:	lea    rdx,[rbp-0x8]
   0x0000000000400619 <+37>:	mov    rsi,rdx
   0x000000000040061c <+40>:	mov    rdi,rax
   0x000000000040061f <+43>:	mov    eax,0x0
   0x0000000000400624 <+48>:	call   0x4004f0 <__isoc99_scanf@plt>
   0x0000000000400629 <+53>:	mov    eax,DWORD PTR [rbp-0x8]
   0x000000000040062c <+56>:	xor    eax,DWORD PTR [rbp-0x4]
   0x000000000040062f <+59>:	cmp    eax,0xdeadbeef
   0x0000000000400634 <+64>:	jne    0x400656 <main+98>
   0x0000000000400636 <+66>:	mov    edi,0x400763
   0x000000000040063b <+71>:	call   0x4004c0 <puts@plt>
   0x0000000000400640 <+76>:	mov    edi,0x400769
   0x0000000000400645 <+81>:	mov    eax,0x0
   0x000000000040064a <+86>:	call   0x4004d0 <system@plt>
   0x000000000040064f <+91>:	mov    eax,0x0
   0x0000000000400654 <+96>:	jmp    0x400665 <main+113>
   0x0000000000400656 <+98>:	mov    edi,0x400778
   0x000000000040065b <+103>:	call   0x4004c0 <puts@plt>
   0x0000000000400660 <+108>:	mov    eax,0x0
   0x0000000000400665 <+113>:	leave  
   0x0000000000400666 <+114>:	ret    
End of assembler dump.
(gdb)
```
random 변수의 위치를 찾기 위해 rand 함수가 실행된 다음인 main+18에 브레이크 포인트를 걸고 디버깅을 시작합니다.
```
(gdb) b *main+18
Breakpoint 1 at 0x400606
(gdb) r
Starting program: /home/random/random

Breakpoint 1, 0x0000000000400606 in main ()
(gdb) i r
rax            0x6b8b4567	1804289383
rbx            0x0	0
rcx            0x7f16c7ee10a4	139735820275876
rdx            0x7f16c7ee10a8	139735820275880
rsi            0x7ffcec99c17c	140724277985660
rdi            0x7f16c7ee1620	139735820277280
rbp            0x7ffcec99c1b0	0x7ffcec99c1b0
rsp            0x7ffcec99c1a0	0x7ffcec99c1a0
r8             0x7f16c7ee10a4	139735820275876
r9             0x7f16c7ee1120	139735820276000
r10            0x47f	1151
r11            0x7f16c7b57f60	139735816568672
r12            0x400510	4195600
r13            0x7ffcec99c290	140724277985936
r14            0x0	0
r15            0x0	0
rip            0x400606	0x400606 <main+18>
eflags         0x202	[ IF ]
cs             0x33	51
ss             0x2b	43
ds             0x0	0
es             0x0	0
fs             0x0	0
gs             0x0	0
(gdb)
```
rax 레지스터의 값이 rand 함수를 거친 random 변수의 값입니다.
0x6b8b4567과 0xdeadbeef를 XOR 하면 조건을 만족하는 값이 나올 것입니다.
```
random@ubuntu:~$ python
Python 2.7.12 (default, Jul  1 2016, 15:12:24)
[GCC 5.4.0 20160609] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> 0x6b8b4567^0xdeadbeef
3039230856
>>>
```
```
random@ubuntu:~$ ./random
3039230856
Good!
Mommy, I thought libc random is unpredictable...
random@ubuntu:~$ 
```
