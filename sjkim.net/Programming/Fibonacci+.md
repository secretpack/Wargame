# 문제
아래의 수열은 처음 두 항을 1과 1로 한 후 ,
그 다음 항 부터는 바로 앞의 두개의 항을 더해 만드는 피보나치 수열 입니다.
이 수열에 속하는 수를 피보나치 수 라고 이야기 하며,
아래와 같이
1번째 피보나치 수 : 1
2번째 피보나치 수 : 1
3번째 피보나치 수 : 2
4번째 피보나치 수 : 3
5번째 피보나치 수 : 5 라고 할때 ,
피보나치 수 1 부터 75 까지(75포함)의 수 중 3의 배수이거나 5의 배수인 수를 골라
그중 "짝수" 의 합만 구한 값은 ?

---
# 개요

피보나치 수열만 구현할 줄 안다면 별로 어려운 문제는 아닙니다.
```python
def fibonacci(n):
	b = True
	f1 = 1
	f2 = 1

	while n > 2:
		if b :
			f1 = f1 + f2

		else :
			f2 = f1 + f2

		b = not b
		n -= 1

	if b:
		return f2

	else :
		return f1

if __name__ == '__main__':
	hab = 0

	for i in range(1,76):
		if fibonacci(i)%3 == 0 or fibonacci(i)%5 == 0:
			if fibonacci(i)%2 == 0:
				hab += fibonacci(i)

	print hab
```
