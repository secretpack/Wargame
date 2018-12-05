# 문제  
Daddy told me about cool MD5 hash collision today.
I wanna do something like that too!

ssh col@pwnable.kr -p2222 (pw:guest)  

---
# 개요  
서버에 접속하여 ls 명령을 입력해보면 col, col.c, flag 3개의 파일이 존재하는 것을 알 수 있습니다.  
cat 명령어를 통해 fd.c 파일을 확인합니다.  
```c
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
```
hashcode와 check_password함수의 리턴값이 같으면 system("/bin/cat flag") 명령이 실행되어 flag를 출력하게 됩니다. 여기서 주목해야 할 코드는 다음과 같습니다.
```c
unsigned long check_password(const char* p)
{

	int* ip = (int*)p;
	int i;
	int res=0;
	for(i=0; i<5; i++){
		res += ip[i];
	}
	return res;
}
```
check_password 함수의 인자로 argv[1] 을 받지만 함수 내에서 char 형 포인터를 int형 포인터로 바꿉니다.그리고 반복문을 통해 사용자가 입력한 20byte를 4byte 단위로 다섯번씩 읽어들여 res 변수에 저장합니다. 즉 res 변수에 저장된 값이 hashcode인 0x21DD09EC와 같게 하면 조건이 참이됩니다.  

##### 풀이과정
* 0x21DD09EC / 5 = 0x6C5CEC8
* 검산 0x6C5CEC8 * 5 = 0x21DD09E8 (4만큼 작다)
* 0x21DD09EC = 0x6C5CEC8 * 4 + (6C5CECC + 4)

```
col@ubuntu:~$ ./col `python -c 'print "\xc8\xce\xc5\x06"*4+"\xcc\xce\xc5\x06"'`
daddy! I just managed to create a hash collision :)
col@ubuntu:~$ 
```
