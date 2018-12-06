# 문제  
첨부된 실행파일을 내려받아 실행하면
1분마다 한번씩 반복하여 임의의 로또 번호를 추첨합니다.

프로그램이 발생할 난수를 예측하여 1등에 당첨되면 플래그를
얻을 수 있습니다.

---
# 개요

원래 rand 함수의 특성을 이용하여 문제를 풀이하려 했으나 화가나서 리버싱을 했습니다.

해당 파일을 IDA 로 열어봅시다.

```c
if ( v2 == 6 )
{
  CWnd::MessageBoxA((CWnd *)v0, "6개 맞았습니다. 1등입니다.", "축하합니다.", 0);
  CWnd::SetWindowTextA((CWnd *)(v0 + 444), "1등입니다.");
  v3 = &v13;
  v4 = 0;
  do
    v5 = *v3++;
  while ( v5 );
  if ( v3 != v14 )
  {
    do
    {
      *(&v13 + v4) ^= 2u;
      ++v4;
    }
    while ( v4 < strlen(&v13) );
  }
  sprintf(&Text, "f|a9 : %s", &v13);
  CWnd::MessageBoxA((CWnd *)v0, &Text, 0, 0);
  Sleep(1000u);
}
```
문자열 검색을 통해 수상한 문자열을 발견하였고 XREF 검색 결과 다음의 코드를 발견할 수 있습니다.

sprintf(&Text, "f|a9 : %s", &v13); 문구로 보아 v13 변수에 플래그가 저장되어 있는 것 같습니다.

```c
if ( v3 != v14 )
{
  do
  {
    *(&v13 + v4) ^= 2u;
    ++v4;
  }
  while ( v4 < strlen(&v13) );
}
```
해당 코드를 보면 XOR 2 연산을 수행하는것을 알 수 있습니다.

그렇다면 플래그는 연산된 암호문 형태로 존재할 것입니다.

```c
qmemcpy(&v13, "a`60;f`7aadf444d6:f`13g0:17;0;g6", 33u);
```
암호문은 a`60;f`7aadf444d6:f`13g0:17;0;g6 이며 이를 다시 역연산 합니다.

```python
string = "a`60;f`7aadf444d6:f`13g0:17;0;g6"

length = len(string)
i = 0

for i in range(0,length):
  print chr(ord(string[i])^2)
```
