# 문제

아래는 첨부파일에 있는 실행파일을 작성한 소스코드 입니다.
소스코드를 잘 읽어보고 , 어떻게 하면 Flag 를 찾을 수 있을지 찾아 등록하세요.
strcmp 에서 비교하는 문자열은 아래 문자열을 복사하여 사용하면됩니다.

문자열 : Fosv~$qa}lfx,`mw3f}s7yhmo:hmhpE
```c
char* encrypt(char* str)
{
	int j = strlen(str);
	int i = 0 ;

	for( i = 0 ; i < j ; i ++)
	{
	   str[i] += 1; str[i] ^= i;
	}

	return str;
}

int main( int argc, char* argv[] )
{
	char buffer[50] = { 0 } ;
	if ( argc != 2 )
	{
		printf(" NoNo.. Try again.. ");
		system("pause");
		return ;
	}
	else
	strncpy(buffer,argv[1], sizeof(buffer)-1);

	if( !strcmp("Fosv~$qa}lfx,`mw3f}s7yhmo:hmhpE",
	encrypt(buffer))
	printf("Flag : &s ", argv[1]);
	else
	printf("NoNo.. try again.. ");

	return 0;
}
```
---
# 개요

암호화를 수행하는 부분은 다음과 같습니다.
```c
for( i = 0 ; i < j ; i ++)
{
  str[i] += 1; str[i] ^= i;
}
```
str[i] 배열 내의 값에 +1을 하고 그 때의 반복변수 i와 XOR 연산을 수행합니다.

역연산 코드를 작성하여 플래그를 구할 수 있습니다.

```c
#include <stdio.h>

char *decrypt(char *str)
{
  int j = strlen(str), i;

  for( i = 0 ; i < j ; i ++)
  {
     str[i] ^= i;
     str[i] -= 1;
  }

  return str;
}

int main()
{
  char str[50] = "Fosv~$qa}lfx,`mw3f}s7yhmo:hmhpE";

  printf("FLAG : %s\n",decrypt(str));

  return 0;
}
```
