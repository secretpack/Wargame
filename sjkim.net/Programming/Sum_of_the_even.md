# 문제

매우 간단한 문제 입니다. 1 부터 5000을 포함하는 사이 수 중
짝수의 총 합계를 구하여 플래그로 등록하세요.

---
# 개요

```c
#include <stdio.h>

int main()
{
  int i, sum=0;

  for(i = 1; i<=5000; i++){
    if(i % 2 == 0)
      sum += i;
  }
  printf("%d\n",sum);

  return 0;
}
```
