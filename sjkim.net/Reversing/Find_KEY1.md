# 문제
첨부한 실행파일을 내려받아 키를 찾아내고
플래그를 등록하세요

---
# 개요

IDA를 사용하여 해당 파일을 열어봅시다.

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v4; // [sp+0h] [bp-4h]@1

  v4 = 0;
  sub_401000();
  printf(" SJKIM BINARYGAME Reversing 5 point \n");
  printf(" input key : ");
  scanf("%d", &v4);
  if ( (v4 ^ 0x57) == 1320553664 )
    printf(" 정답입니다. 플래그를 등록하세요 . \n");
  else
    printf(" 다시 시도하세요. \n");
  system("pause");
  return 0;
}
```

v4 값과 0x57(84) 를 XOR 연산하여 그 값이 1320553664 일때 플래그를 출력합니다.

XOR 연산의 특성을 이용하여 13205533664 ^ 87 을 수행하여 플래그를 구할 수 있습니다.
