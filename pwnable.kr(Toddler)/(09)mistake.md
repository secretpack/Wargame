# 문제
We all make mistakes, let's move on.
(don't take this too seriously, no fancy hacking skill is required at all)

This task is based on real event
Thanks to dhmonkey

hint : operator priority

ssh mistake@pwnable.kr -p2222 (pw:guest)

---
# 개요
```
mistake@ubuntu:~$ ls -l
total 24
-r-------- 1 mistake_pwn root      51 Jul 29  2014 flag
-r-sr-x--- 1 mistake_pwn mistake 8934 Aug  1  2014 mistake
-rw-r--r-- 1 root        root     792 Aug  1  2014 mistake.c
-r-------- 1 mistake_pwn root      10 Jul 29  2014 password
mistake@ubuntu:~$
```
총 4개의 파일을 볼 수 있습니다. 먼저 mistake.c를 살펴봅시다.
```c
#include <stdio.h>
#include <fcntl.h>

#define PW_LEN 10
#define XORKEY 1

void xor(char* s, int len){
	int i;
	for(i=0; i<len; i++){
		s[i] ^= XORKEY;
	}
}

int main(int argc, char* argv[]){

	int fd;
	if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){
		printf("can't open password %d\n", fd);
		return 0;
	}

	printf("do not bruteforce...\n");
	sleep(time(0)%20);

	char pw_buf[PW_LEN+1];
	int len;
	if(!(len=read(fd,pw_buf,PW_LEN) > 0)){
		printf("read error\n");
		close(fd);
		return 0;		
	}

	char pw_buf2[PW_LEN+1];
	printf("input password : ");
	scanf("%10s", pw_buf2);

	// xor your input
	xor(pw_buf2, 10);

	if(!strncmp(pw_buf, pw_buf2, PW_LEN)){
		printf("Password OK\n");
		system("/bin/cat flag\n");
	}
	else{
		printf("Wrong Password\n");
	}

	close(fd);
	return 0;
}
```
---
# 분석  
우리가 주의 깊게 봐야할 부분은 다음과 같습니다.
```c
if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){
  printf("can't open password %d\n", fd);
  return 0;
}
```
연산자 우선순위에 의해 비교 연산자가 대입 연산자보다 우선순위가 높으므로 먼저 수행됩니다.
파일이 존재하므로 open 함수는 0이 아닌 양수를 반환하게 됩니다. 그리고 이를 0과 비교하는데 해당 if 문은 당연히 fale 라는 것을 알 수 있으며 fd는 0이 됩니다.

따라서 pw_buf, pw_buf2 모두 값을 입력할 수 있게 되었으므로 두 수를 XOR 연산 했을 때 자기 자신이 나오는 숫자를 구성하면 됩니다.
A xor B = A 를 만족하는 숫자의 예시는
* A = 1111111111
* B = 0000000000
* A xor B = 1111111111

입니다.
```
mistake@ubuntu:~$ ./mistake
do not bruteforce...
1111111111
0000000000input password :
Password OK
Mommy, the operator priority always confuses me :(
mistake@ubuntu:~$
```
