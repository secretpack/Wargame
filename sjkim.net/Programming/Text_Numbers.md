# 문제
첨부한 텍스트 파일 내용중 '숫자'만 추출하여 그 수들의
모든 합을 구하세요.

---
# 개요
정규표현식을 사용하여 만족하는 값을 구할 수 있습니다.

비교적 문법이 쉬운 python 을 사용하면 답을 구할 수 있습니다.

```python
import re

sum = 0
f = open('/home/secretpack/txt_sum.txt','r')

data = f.readline()
number = (re.sub('[^0-9]','',data))

for i in range(0,len(number)):
	sum += int(number[i])

print sum
```
