# 문제
Are you tired of hacking?, take some rest here.
Just help me out with my small experiment regarding memcpy performance.
after that, flag is yours.

http://pwnable.kr/bin/memcpy.c
ssh memcpy@pwnable.kr -p2222 (pw:guest)

---
# 개요
```c++
// compiled with : gcc -o memcpy memcpy.c -m32 -lm
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <sys/mman.h>
#include <math.h>

unsigned long long rdtsc(){
        asm("rdtsc");
}

char* slow_memcpy(char* dest, const char* src, size_t len){
	int i;
	for (i=0; i<len; i++) {
		dest[i] = src[i];
	}
	return dest;
}

char* fast_memcpy(char* dest, const char* src, size_t len){
	size_t i;
	// 64-byte block fast copy
	if(len >= 64){
		i = len / 64;
		len &= (64-1);
		while(i-- > 0){
			__asm__ __volatile__ (
			"movdqa (%0), %%xmm0\n"
			"movdqa 16(%0), %%xmm1\n"
			"movdqa 32(%0), %%xmm2\n"
			"movdqa 48(%0), %%xmm3\n"
			"movntps %%xmm0, (%1)\n"
			"movntps %%xmm1, 16(%1)\n"
			"movntps %%xmm2, 32(%1)\n"
			"movntps %%xmm3, 48(%1)\n"
			::"r"(src),"r"(dest):"memory");
			dest += 64;
			src += 64;
		}
	}

	// byte-to-byte slow copy
	if(len) slow_memcpy(dest, src, len);
	return dest;
}

int main(void){

	setvbuf(stdout, 0, _IONBF, 0);
	setvbuf(stdin, 0, _IOLBF, 0);

	printf("Hey, I have a boring assignment for CS class.. :(\n");
	printf("The assignment is simple.\n");

	printf("-----------------------------------------------------\n");
	printf("- What is the best implementation of memcpy?        -\n");
	printf("- 1. implement your own slow/fast version of memcpy -\n");
	printf("- 2. compare them with various size of data         -\n");
	printf("- 3. conclude your experiment and submit report     -\n");
	printf("-----------------------------------------------------\n");

	printf("This time, just help me out with my experiment and get flag\n");
	printf("No fancy hacking, I promise :D\n");

	unsigned long long t1, t2;
	int e;
	char* src;
	char* dest;
	unsigned int low, high;
	unsigned int size;
	// allocate memory
	char* cache1 = mmap(0, 0x4000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
	char* cache2 = mmap(0, 0x4000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
	src = mmap(0, 0x2000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);

	size_t sizes[10];
	int i=0;

	// setup experiment parameters
	for(e=4; e<14; e++){	// 2^13 = 8K
		low = pow(2,e-1);
		high = pow(2,e);
		printf("specify the memcpy amount between %d ~ %d : ", low, high);
		scanf("%d", &size);
		if( size < low || size > high ){
			printf("don't mess with the experiment.\n");
			exit(0);
		}
		sizes[i++] = size;
	}

	sleep(1);
	printf("ok, lets run the experiment with your configuration\n");
	sleep(1);

	// run experiment
	for(i=0; i<10; i++){
		size = sizes[i];
		printf("experiment %d : memcpy with buffer size %d\n", i+1, size);
		dest = malloc( size );

		memcpy(cache1, cache2, 0x4000);		// to eliminate cache effect
		t1 = rdtsc();
		slow_memcpy(dest, src, size);		// byte-to-byte memcpy
		t2 = rdtsc();
		printf("ellapsed CPU cycles for slow_memcpy : %llu\n", t2-t1);

		memcpy(cache1, cache2, 0x4000);		// to eliminate cache effect
		t1 = rdtsc();
		fast_memcpy(dest, src, size);		// block-to-block memcpy
		t2 = rdtsc();
		printf("ellapsed CPU cycles for fast_memcpy : %llu\n", t2-t1);
		printf("\n");
	}

	printf("thanks for helping my experiment!\n");
	printf("flag : ----- erased in this source code -----\n");
	return 0;
}
```
slow_memcpy와 fast_memcpy의 속도를 비교하는 프로그램입니다.

```
memcpy@ubuntu:~$ ls
memcpy.c  readme
memcpy@ubuntu:~$ cat readme
the compiled binary of "memcpy.c" source code (with real flag) will be executed under memcpy_pwn privilege if you connect to port 9022.
execute the binary by connecting to daemon(nc 0 9022).

memcpy@ubuntu:~$

```
플래그는 nc에 접속하여 문제를 해결해야지만 주는 것 같습니다.
분석을 위해 해당 소스코드를 가져와 컴파일 합니다. 주석에 가져온 옵션 그대로 컴파일 해야 하기 때문에 다음과 같이 두 개의 패키지를 설치해줘야 합니다.
```
secretpack@ubuntu:~$ sudo apt-get install gcc-multilib libc6-dev-i386
```
---
# 분석
이제 nc에 접속해서 문제를 보도록 합시다.
```
Hey, I have a boring assignment for CS class.. :(
The assignment is simple.
-----------------------------------------------------
- What is the best implementation of memcpy?        -
- 1. implement your own slow/fast version of memcpy -
- 2. compare them with various size of data         -
- 3. conclude your experiment and submit report     -
-----------------------------------------------------
This time, just help me out with my experiment and get flag
No fancy hacking, I promise :D
specify the memcpy amount between 8 ~ 16 : 8
specify the memcpy amount between 16 ~ 32 : 16
specify the memcpy amount between 32 ~ 64 : 32
specify the memcpy amount between 64 ~ 128 : 64
specify the memcpy amount between 128 ~ 256 : 128
specify the memcpy amount between 256 ~ 512 : 256
specify the memcpy amount between 512 ~ 1024 : 512
specify the memcpy amount between 1024 ~ 2048 : 1024
specify the memcpy amount between 2048 ~ 4096 : 2048
specify the memcpy amount between 4096 ~ 8192 : 4096
ok, lets run the experiment with your configuration
experiment 1 : memcpy with buffer size 8
ellapsed CPU cycles for slow_memcpy : 1458
ellapsed CPU cycles for fast_memcpy : 594

experiment 2 : memcpy with buffer size 16
ellapsed CPU cycles for slow_memcpy : 369
ellapsed CPU cycles for fast_memcpy : 639

experiment 3 : memcpy with buffer size 32
ellapsed CPU cycles for slow_memcpy : 564
ellapsed CPU cycles for fast_memcpy : 855

experiment 4 : memcpy with buffer size 64
ellapsed CPU cycles for slow_memcpy : 1002
ellapsed CPU cycles for fast_memcpy : 147

experiment 5 : memcpy with buffer size 128
ellapsed CPU cycles for slow_memcpy : 1839
```
최소를 입력했을때는 5번 최대를 입력했을 때는 4번에서 프로그램이 종료됩니다.
소스코드를 살펴봅시다.
```c
char* src;
char* dest;
unsigned int low, high;
unsigned int size;
// allocate memory
char* cache1 = mmap(0, 0x4000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
char* cache2 = mmap(0, 0x4000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
src = mmap(0, 0x2000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
```
src에 0x2000만큼의 메모리를 할당합니다.
```c
for(i=0; i<10; i++){
  size = sizes[i];
  printf("experiment %d : memcpy with buffer size %d\n", i+1, size);
  dest = malloc( size );

  memcpy(cache1, cache2, 0x4000);		// to eliminate cache effect
  t1 = rdtsc();
  slow_memcpy(dest, src, size);		// byte-to-byte memcpy
  t2 = rdtsc();
  printf("ellapsed CPU cycles for slow_memcpy : %llu\n", t2-t1);

  memcpy(cache1, cache2, 0x4000);		// to eliminate cache effect
  t1 = rdtsc();
  fast_memcpy(dest, src, size);		// block-to-block memcpy
  t2 = rdtsc();
  printf("ellapsed CPU cycles for fast_memcpy : %llu\n", t2-t1);
  printf("\n");
}

printf("thanks for helping my experiment!\n");
printf("flag : ----- erased in this source code -----\n");
return 0;
```
사용자가 입력한 만큼 동적할당해서 dest에 주소를 넣어줍니다.
그리고 slow_memcpy와 fast_memcpy를 호출합니다.
```c
char* slow_memcpy(char* dest, const char* src, size_t len){
	int i;
	for (i=0; i<len; i++) {
		dest[i] = src[i];
	}
	return dest;
}

char* fast_memcpy(char* dest, const char* src, size_t len){
	size_t i;
	// 64-byte block fast copy
	if(len >= 64){
		i = len / 64;
		len &= (64-1);
		while(i-- > 0){
			__asm__ __volatile__ (
			"movdqa (%0), %%xmm0\n"
			"movdqa 16(%0), %%xmm1\n"
			"movdqa 32(%0), %%xmm2\n"
			"movdqa 48(%0), %%xmm3\n"
			"movntps %%xmm0, (%1)\n"
			"movntps %%xmm1, 16(%1)\n"
			"movntps %%xmm2, 32(%1)\n"
			"movntps %%xmm3, 48(%1)\n"
			::"r"(src),"r"(dest):"memory");
			dest += 64;
			src += 64;
		}
	}
```
movdqa와 movntps가 사용된 것을 알 수 있습니다.
* movdqa (![링크잠조](http://www.jaist.ac.jp/iscenter-new/mpc/altix/altixdata/opt/intel/vtune/doc/users_guide/mergedProjects/analyzer_ec/mergedProjects/reference_olh/mergedProjects/instructions/instruct32_hh/vc183.htm))
  피 연산자(두 번째 피연산자)의 이중 쿼드 워드를 대상 피연산자(첫 번째 피연산자)로 이동합니다. 이 명령여는 더블 쿼드 워드를 XMM 레지스터와 128bit 메모리 위치간에 또는 두 개의 XMM 레지스터간에 이동하는데 사용할 수 있습니다. 소스 또는 대상 피연산자가 메모리 피연산자인 경우 피연산자는 16바이트 경계에 정렬되어야 하며 그렇지 않으면 일반 보호예외(#GP)가 생성됩니다.

피연산자인 dest의 주소를 16byte로 맞춰준다면 해결할 수 있을것 같습니다.
먼저 소스코드에 dest의 주소를 출력할 수 있도록 다음을 추가했습니다.
```c
printf("Dest : %p\n",dest);
```
```
secretpack@ubuntu:~/Desktop$ gdb -q memcpy
Reading symbols from memcpy...(no debugging symbols found)...done.
(gdb) r
Starting program: /home/secretpack/Desktop/memcpy
Hey, I have a boring assignment for CS class.. :(
The assignment is simple.
-----------------------------------------------------
- What is the best implementation of memcpy?        -
- 1. implement your own slow/fast version of memcpy -
- 2. compare them with various size of data         -
- 3. conclude your experiment and submit report     -
-----------------------------------------------------
This time, just help me out with my experiment and get flag
No fancy hacking, I promise :D
specify the memcpy amount between 8 ~ 16 : 8
specify the memcpy amount between 16 ~ 32 : 16
specify the memcpy amount between 32 ~ 64 : 32
specify the memcpy amount between 64 ~ 128 : 64
specify the memcpy amount between 128 ~ 256 : 128
specify the memcpy amount between 256 ~ 512 : 256
specify the memcpy amount between 512 ~ 1024 : 512
specify the memcpy amount between 1024 ~ 2048 : 1024
specify the memcpy amount between 2048 ~ 4096 : 2048
specify the memcpy amount between 4096 ~ 8192 : 4096
ok, lets run the experiment with your configuration
experiment 1 : memcpy with buffer size 8
ellapsed CPU cycles for slow_memcpy : 8874
Dest : 0x804c410
ellapsed CPU cycles for fast_memcpy : 1118

experiment 2 : memcpy with buffer size 16
ellapsed CPU cycles for slow_memcpy : 746
Dest : 0x804c420
ellapsed CPU cycles for fast_memcpy : 640

experiment 3 : memcpy with buffer size 32
ellapsed CPU cycles for slow_memcpy : 1074
Dest : 0x804c438
ellapsed CPU cycles for fast_memcpy : 798

experiment 4 : memcpy with buffer size 64
ellapsed CPU cycles for slow_memcpy : 1620
Dest : 0x804c460
ellapsed CPU cycles for fast_memcpy : 320

experiment 5 : memcpy with buffer size 128
ellapsed CPU cycles for slow_memcpy : 1906
Dest : 0x804c4a8

Program received signal SIGSEGV, Segmentation fault.
0x080487cc in fast_memcpy ()
```
128에서 dest가 8이므로 주소를 16배수로 맞춰주기 위해 이 자리를 0으로 만들 필요가 있습니다.
따라서 64 + 8 = 72를 대신 넣어준다면 8byte가 더 들어가므로
128에서의 dest 주소에 8byte를 채워줄 수 있습니다.
```
(gdb) r
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /home/secretpack/Desktop/memcpy
Hey, I have a boring assignment for CS class.. :(
The assignment is simple.
-----------------------------------------------------
- What is the best implementation of memcpy?        -
- 1. implement your own slow/fast version of memcpy -
- 2. compare them with various size of data         -
- 3. conclude your experiment and submit report     -
-----------------------------------------------------
This time, just help me out with my experiment and get flag
No fancy hacking, I promise :D
specify the memcpy amount between 8 ~ 16 : 8
specify the memcpy amount between 16 ~ 32 : 16
specify the memcpy amount between 32 ~ 64 : 32
specify the memcpy amount between 64 ~ 128 : 72
specify the memcpy amount between 128 ~ 256 : 136
specify the memcpy amount between 256 ~ 512 : 264
specify the memcpy amount between 512 ~ 1024 : 520
specify the memcpy amount between 1024 ~ 2048 : 1032
specify the memcpy amount between 2048 ~ 4096 : 2056
specify the memcpy amount between 4096 ~ 8192 : 4104
ok, lets run the experiment with your configuration
experiment 1 : memcpy with buffer size 8
ellapsed CPU cycles for slow_memcpy : 5546
Dest : 0x804c410
ellapsed CPU cycles for fast_memcpy : 754

experiment 2 : memcpy with buffer size 16
ellapsed CPU cycles for slow_memcpy : 772
Dest : 0x804c420
ellapsed CPU cycles for fast_memcpy : 554

experiment 3 : memcpy with buffer size 32
ellapsed CPU cycles for slow_memcpy : 660
Dest : 0x804c438
ellapsed CPU cycles for fast_memcpy : 694

experiment 4 : memcpy with buffer size 72
ellapsed CPU cycles for slow_memcpy : 1334
Dest : 0x804c460
ellapsed CPU cycles for fast_memcpy : 434

experiment 5 : memcpy with buffer size 136
ellapsed CPU cycles for slow_memcpy : 2452
Dest : 0x804c4b0
ellapsed CPU cycles for fast_memcpy : 814

experiment 6 : memcpy with buffer size 264
ellapsed CPU cycles for slow_memcpy : 4820
Dest : 0x804c540
ellapsed CPU cycles for fast_memcpy : 424

experiment 7 : memcpy with buffer size 520
ellapsed CPU cycles for slow_memcpy : 9222
Dest : 0x804c650
ellapsed CPU cycles for fast_memcpy : 546

experiment 8 : memcpy with buffer size 1032
ellapsed CPU cycles for slow_memcpy : 17982
Dest : 0x804c860
ellapsed CPU cycles for fast_memcpy : 798

experiment 9 : memcpy with buffer size 2056
ellapsed CPU cycles for slow_memcpy : 36358
Dest : 0x804cc70
ellapsed CPU cycles for fast_memcpy : 1812

experiment 10 : memcpy with buffer size 4104
ellapsed CPU cycles for slow_memcpy : 63388
Dest : 0x804d480
ellapsed CPU cycles for fast_memcpy : 2886

thanks for helping my experiment!
flag : ----- erased in this source code -----
[Inferior 1 (process 118584) exited normally]
(gdb)
```
저 값을 그대로 본 서버에 대입하면 플래그가 출력됩니다.  
