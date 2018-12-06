# 문제
지뢰찾기 잘 하시나요?
제가 약간 손봐둔 지뢰찾기는 조금 어려울겁니다.

성공하기만 한다면 플래그를 출력할테니 다운받아
게임을 성공해보세요.

---
# 개요

문제를 해결하기 위한 여러가지 접근 방법이 있습니다.

필자가 직접 삽질하며 풀어본 결과 입문자에게 추천할만한 난이도는 아닌것 같습니다.

하지만 리버싱을 어느정도 접한 사람들에게는 좋은 문제인 것 같습니다.

해당 문제를 해결하기 위해 필자는 다음과 같이 생각했습니다.

#### 1. 플래그를 출력하는 이벤트 호출함수를 찾아 그 주소로 JMP

IDA 를 열어 확인한 결과 flag와 관련된 직접적인 string이 없어 찾기가 힘듭니다.

#### 2. 지뢰는 10 * 10 총 100개이며 지뢰를 피할 수 있는 칸은 오직 2개 뿐이다.

그렇다면 그 두 곳에 대한 좌표가 있을 것입니다.

그 좌표를 찾아 누르고 플래그를 획득해봅시다.

---

# 해결

좌표를 구하기 위해 게임 시작 루틴을 찾아야 합니다.

게임 시작과 동시에 폭탄이 세팅될 것이기 때문입니다.

```c
int __stdcall StartGame()
{
  signed int v0; // ebx@5
  int v1; // exi@6
  int v2; // eax@6
  signed int v4; // [sp-4h] [bp-10h] @3
  fTimer = 0;
  if( dword_10056AC != xBoxMac || uValue !- yBoxMac )
    v4 = 6;
  else
    v4 =- 4
  v0 = v4;
  xBoxMac = dword_10056AC;
  yBoxMac = uValue;
  ClearField();
  iButtonCur = 0;
  cBomStart = dword_10056A4;
  do
  {
    do
    {
      v1 = Rnd(xBoxMac) + 1;
      v2 = Rnd(yBoxMac) + 1;
    }
    while ( *(&rgBlk[32 * v2] + v1) & 0x80 );
    *(&rgBlk[32 * v2] + v1) |= 0x80u;
    --cBomStart;
  }
  while( cBomStart );
  cSec = 0;
  cBomStart = dword_10056A4;
  cBomLeft = dword_10056A4;
  cBoxVisit = 0;
  cBoxVisitMac = xBoxMac * yBoxMac - dword_10056A4;
  fStatus = 1;
  UpdateBombCount(0);
  return AdjustWindow(v0);
```
루틴을 찾기 위해서는 String, 함수명, 메모리 엑세스 로그 등의 정보가 필요합니다.

하지만 분석 당시 심볼이 살아 있어 비교적 쉽게 찾을 수 있었습니다.

여기서 주목해야 할 코드는 아래와 같습니다.

```c
do
{
  do
  {
    v1 = Rnd(xBoxMac) + 1;
    v2 = Rnd(yBoxMac) + 1;
  }
  while ( *(&rgBlk[32 * v2] + v1) & 0x80 );
  *(&rgBlk[32 * v2] + v1) |= 0x80u;
  --cBomStart;
}
while( cBomStart );
```

* cBomStart가 무한루프 합니다.

그리고 Rnd 함수에 들어 있는 xBoxMac, yBoxMac이 보입니다.

```c
int __stdcall Rnd( signed int a1 )
{
  return _rand() % a1;
}
```
Rnd 함수는 rand() 함수를 이용하여 난수를 생성하는 함수이고

즉 v1 에는 랜덤한 X 좌표가, v2에는 랜덤한 y좌표가 저장됩니다.

--cBomStart 되면서 특정 메모리 주소에 값을 넣는 것도 보입니다.

cBomStart가 지뢰의 갯수라면 초기 값은 두 개를 제외한 98이 되야 합니다.

해당 사실을 확인 하기 위해 Cheat Engine을 사용합니다.

![mine1](D:\git\Wargame\sjkim.net\Reversing\image\mine1.png)

예상대로 초기값이 98로 설정되어 있습니다.

이제 저 루틴에 대해 신뢰성이 생겼으니 X와 Y좌표를 구해야 합니다.

X좌표와 Y좌표를 구할 수 있는 방법은 다음과 같습니다.

* seed 값을 알아낸 뒤 rand 함수를 직접 코딩한다.
* Code Injection을 통해 좌표 값을 알아낸다.
* 값이 들어간 테이블을 분석하여 좌표값을 알아낸다.

필자는 코드인젝션을 통해 X좌표와 Y좌표를 가져오는데 성공 했습니다.

Code Injection을 수행하기 위해 먼저 인젝션 포인트를 잡습니다.

![mine2](D:\git\Wargame\sjkim.net\Reversing\image\mine2.png)

Rnd(x)의 결과 값이 eax에 들어가므로 밑의 인스트럭션 아무거나 잡아서 출력하면 됩니다.

필자의 경우 call rnd 바로 밑의 인스트럭션 주소를 잡았습니다.

### x : 0x10036DE

Rnd(y) 또한 마찬가지로 결과 값이 eax에 저장되므로 밑의 인스트럭션을 잡습니다.

### y : 0x10036E0

두 주소 모두 eax 레지스터에 결과를 저장한다는 것을 파악 했습니다.

이제 Cheat Engine을 이용하여 코드 인젝션을 해봅시다.

![mine3](D:\git\Wargame\sjkim.net\Reversing\image\mine3.png)

Lua Engine을 이용하여 아래와 같이 코드를 작성합니다.

```lua
x = 0x10036D2
y = 0x10036E0

x_val = 0
y_val = 0
function debugger_onBreakpoint()
  if x == EIP then
    x_val = EAX
  end

  if y == EIP then
    y_val = EAX
    print(x_val..","..y_val)
    end
    return 1
  end

debug_setBreakpoint(x)
debug_setBreakpoint(y)
```

코드를 작성하고 execute 버튼을 누르면 코드가 적용됩니다.

그리고 스마일 버튼을 눌러 테이블을 초기화 합니다.

![mine4](D:\git\Wargame\sjkim.net\Reversing\image\mine4.png)

초기화와 동시에 Output 란에 좌표 값을 출력하는 것을 볼 수 있습니다.

여기서 주의해야 할 점은 앞에서 +1 값을 좌표에 넣어 줬으므로 좌표에도 +1씩 더해줘야 합니다.

좌표를 정리하고 올바른 좌표를 클릭하면 플래그가 출력됩니다.

##### Thx to tonix
