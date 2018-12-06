# 문제
0부터 5,555,555 ( 555만 5555 ) 를 포함하는 모든 수 들을 살펴보면
숫자'5'는 몇번나타날까요?

---
# 개요
파이썬 string.count를 이용하여 쉽게 구할 수 있습니다.

```python
counter = 0

for i in range(0,5555556):
	string = str(i)
	count = string.count('5')
	counter += count

print counter
```
