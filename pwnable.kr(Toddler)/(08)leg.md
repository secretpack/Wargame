# 문제
Daddy told me I should study arm.
But I prefer to study my leg!

Download : http://pwnable.kr/bin/leg.c
Download : http://pwnable.kr/bin/leg.asm

ssh leg@pwnable.kr -p2222 (pw:guest)

---
# 개요
위의 문제를 해결하기 위해 파이프라인과 ARM architecture 에 대한 이해가 필요합니다.  
##### 파이프라인
CPU는 CLOCK 신호에 따라 명령을 처리하도록 설계되어 있습니다. 하지만 파이프라인 매커니즘을 적용하면 주어진 CLOCK 속도보다 더욱 빠르게 처리할 수 있게 됩니다. 기본 명령어는 순차적으로 명령을 처리하기 전에 일련의 종속단계로 나뉘어져 서로 다른 단계가 병렬로 실행될 수 있습니다. 보통 파이프라인에 대한 설명을 할 때 명령 처리 단계를 4단계로 나누어 설명합니다.
* Instruction Fetch : 다음에 실행할 명령어를 읽음
* Instruction Decode : 명령어 해석
* Execute : 명령어 수행
* Write-Back : 처리된 결과를 저장

파이프라인은 하나의 명령어를 처리할 수 있는 여러 단계로 나누어 종속된 일련의 명령어들이 처리되는 것에 있어 서로에게 영향을 주지 않는 범위에서 동시에 다른 명령어를 처리할 수 있게 됩니다. 따라서 PC(Program Counter)는 단순히 다음 명령어를 나타내기 보다는 Fetch를 진행해야 하는 위치를 나타낸다고 하는것이 옳은 표현입니다.

##### ARM architecture
문제 풀이에 있어 다음을 숙지하면 좋습니다.
* ARM : 32bit RISC Machine
* Thum b : 16bit RISC Machine

#분석
먼저 C코드를 분석해봅시다.
```c
#include <stdio.h>
#include <fcntl.h>
int key1(){
	asm("mov r3, pc\n");
}
int key2(){
	asm(
	"push	{r6}\n"
	"add	r6, pc, $1\n"
	"bx	r6\n"
	".code   16\n"
	"mov	r3, pc\n"
	"add	r3, $0x4\n"
	"push	{r3}\n"
	"pop	{pc}\n"
	".code	32\n"
	"pop	{r6}\n"
	);
}
int key3(){
	asm("mov r3, lr\n");
}
int main(){
	int key=0;
	printf("Daddy has very strong arm! : ");
	scanf("%d", &key);
	if( (key1()+key2()+key3()) == key ){
		printf("Congratz!\n");
		int fd = open("flag", O_RDONLY);
		char buf[100];
		int r = read(fd, buf, 100);
		write(0, buf, r);
	}
	else{
		printf("I have strong leg :P\n");
	}
	return 0;
}
```
key1+key2+key3의 값이 내가 입력한 key와 같으면 플래그를 출력합니다.  
##### key1
```c
int key1(){
	asm("mov r3, pc\n");
}
```

```
(gdb) disass key1
Dump of assembler code for function key1:
   0x00008cd4 <+0>:	push	{r11}		; (str r11, [sp, #-4]!)
   0x00008cd8 <+4>:	add	r11, sp, #0
   0x00008cdc <+8>:	mov	r3, pc
   0x00008ce0 <+12>:	mov	r0, r3
   0x00008ce4 <+16>:	sub	sp, r11, #0
   0x00008ce8 <+20>:	pop	{r11}		; (ldr r11, [sp], #4)
   0x00008cec <+24>:	bx	lr
```
pc 레지스터에 있는 값을 r3으로 옮깁니다.
파이프라인 매커니즘을 이해했다면 pc는 fetch할 주소를 담고 있고 현재 명령어가 execute 단계라면 다음 명령어는 decode 단계, 그 다음은 fetch 단계가 될 것입니다.
따라서 key1에서의 pc의 값은 0x00008ce4가 됩니다.

##### key2
```c
int key2(){
	asm(
	"push	{r6}\n"
	"add	r6, pc, $1\n"
	"bx	r6\n"
	".code   16\n"
	"mov	r3, pc\n"
	"add	r3, $0x4\n"
	"push	{r3}\n"
	"pop	{pc}\n"
	".code	32\n"
	"pop	{r6}\n"
	);
}
```
```
(gdb) disass key2
Dump of assembler code for function key2:
   0x00008cf0 <+0>:	push	{r11}		; (str r11, [sp, #-4]!)
   0x00008cf4 <+4>:	add	r11, sp, #0
   0x00008cf8 <+8>:	push	{r6}		; (str r6, [sp, #-4]!)
   0x00008cfc <+12>:	add	r6, pc, #1
   0x00008d00 <+16>:	bx	r6
   0x00008d04 <+20>:	mov	r3, pc
   0x00008d06 <+22>:	adds	r3, #4
   0x00008d08 <+24>:	push	{r3}
   0x00008d0a <+26>:	pop	{pc}
   0x00008d0c <+28>:	pop	{r6}		; (ldr r6, [sp], #4)
   0x00008d10 <+32>:	mov	r0, r3
   0x00008d14 <+36>:	sub	sp, r11, #0
   0x00008d18 <+40>:	pop	{r11}		; (ldr r11, [sp], #4)
   0x00008d1c <+44>:	bx	lr
End of assembler dump.
```
r3에 pc(0x00008d08)를 대입하고 4를 더해서 r0에 저장합니다.

##### key3
```c
int key3(){
	asm("mov r3, lr\n");
}
```
```
(gdb) disass key3
Dump of assembler code for function key3:
   0x00008d20 <+0>:	push	{r11}		; (str r11, [sp, #-4]!)
   0x00008d24 <+4>:	add	r11, sp, #0
   0x00008d28 <+8>:	mov	r3, lr
   0x00008d2c <+12>:	mov	r0, r3
   0x00008d30 <+16>:	sub	sp, r11, #0
   0x00008d34 <+20>:	pop	{r11}		; (ldr r11, [sp], #4)
   0x00008d38 <+24>:	bx	lr
End of assembler dump.
```
lr은 함수 호출 전 다시 되돌아 갈 실행할 주소가 저장되어 있습니다.
main함수를 살펴보면 다음과 같이 lr을 구할 수 있습니다.
```
0x00008d7c <+64>:	bl	0x8d20 <key3>
0x00008d80 <+68>:	mov	r3, r0
0x00008d84 <+72>:	add	r2, r4, r3
```
lr의 값은 0x00008d80이 됩니다.

구해진 위 3개의 값을 더한 후 정수로 변환하여 입력하면 플래그가 출력됩니다.
```
/ $ ./leg
Daddy has very strong arm! : 108400
Congratz!
My daddy has a lot of ARMv5te muscle!
/ $
```
