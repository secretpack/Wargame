# 문제
Hey! check out this C implementation of blackjack game!
I found it online
* http://cboard.cprogramming.com/c-programming/114023-simple-blackjack-program.html

I like to give my flags to millionares.
how much money you got?


Running at : nc pwnable.kr 9009

---
# 개요
문제에 제시된 링크에서 사용된 C소스코드를 볼 수 있습니다.
우리가 주목해야 할 코드는 다음과 같습니다.
```c
int betting() //Asks user amount to bet
{
 printf("\n\nEnter Bet: $");
 scanf("%d", &bet);

 if (bet > cash) //If player tries to bet more money than player has
 {
        printf("\nYou cannot bet more money than you have.");
        printf("\nEnter Bet: ");
        scanf("%d", &bet);
        return bet;
 }
 else return bet;
} // End Function
```
---
# 분석
if문을 통해 한번만 검증하기 때문에 돈을 크게 넣고 이기거나 음수 값을 넣고 게임을 지는 것도 하나의 방법이 될 것입니다.
```
YaY_I_AM_A_MILLIONARE_LOL

Cash: $1215742792
-------
|D     |
|  9   |
|     D|
-------

Your Total is 9

The Dealer Has a Total of 10

Enter Bet: $
```
