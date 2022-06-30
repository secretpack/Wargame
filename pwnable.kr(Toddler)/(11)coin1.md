# 문제
Mommy, I wanna play a game!
(if your network response time is too slow, try nc 0 9007 inside pwnable.kr server)

Running at : nc pwnable.kr 9007

---
# 개요
```
        ---------------------------------------------------
        -              Shall we play a game?              -
        ---------------------------------------------------

        You have given some gold coins in your hand
        however, there is one counterfeit coin among them
        counterfeit coin looks exactly same as real coin
        however, its weight is different from real one
        real coin weighs 10, counterfeit coin weighes 9
        help me to find the counterfeit coin with a scale
        if you find 100 counterfeit coins, you will get reward :)
        FYI, you have 60 seconds.

        - How to play -
        1. you get a number of coins (N) and number of chances (C)
        2. then you specify a set of index numbers of coins to be weighed
        3. you get the weight information
        4. 2~3 repeats C time, then you give the answer

        - Example -
        [Server] N=4 C=2        # find counterfeit among 4 coins with 2 trial
        [Client] 0 1            # weigh first and second coin
        [Server] 20                     # scale result : 20
        [Client] 3                      # weigh fourth coin
        [Server] 10                     # scale result : 10
        [Client] 2                      # counterfeit coin is third!
        [Server] Correct!

        - Ready? starting in 3 sec... -
```
* N = 동전의 총 갯수
* C = 시도 가능 횟수
* 진짜 동전의 무게 = 10
* 가짜 동전의 무게 = 9
* 정수 N을 보내면 N+1번째 동전의 무게를 알려준다
* 정수 A B C D E 와 같이 보낼경우 A B C D E번째 동전 무게의 총합을 알려준다.
---
# 분석 
특정 배열의 총합을 알아재서 무개가 10의 배수가 아니면 그 배열에 가짜 동전이 있다는 뜻입니다. 전체 동전을 반으로 나누어서 무게를 구하고 다시 10의 배수가 아닌 배열을 반으로 나누어서 무게를 구합니다. 즉 이진탐색 알고리즘을 사용하여 10의 배수가 아닌 배열의 범위를 좁혀 나갑니다. 30초 내에 100개의 가짜 동전을 찾아야 하므로 코드를 짜서 해결합니다.
```python
#thx to crater0516

from pwn import *
import re

def check (start, end, weight) :
	num = (end - start)
	print ("[!] check : num is %d" % num)
	if (10 * num == weight) :
		return 1
	else :
		return 0

def main () :
	HOST = "pwnable.kr"
	PORT = 9007
	r = remote(HOST, PORT)
	data = r.recvuntil("sec... -\n")
	print data
	sleep(3)
	r.recv(1024)
	# start game!
	for i in range (0, 100) :
		print ("[+] recving data..."),
		sleep(0.1)
		#	r.recv(1024)
		data = r.recv(1024)
		print (": Done, data is %s" % data),
		print ("[*] data parsing..."),
		arr = re.findall("\d+", data)
		N = int(arr[0])
		C = int(arr[1])
		print (": Done, N is %d, C is %d" % (N, C))
		# data has been parsed

		start = 0
		end = N
		while (start <= end) :
			msg = ""
			mid = (start + end) / 2
			print ("[+] sending msg start...")
			for j in range (start, mid + 1) :
				msg += str(j) + " "
			msg += '\n'
			r.send(msg)
			print ("[*] msg : %s" % msg),
			dt = r.recv(100)
			print ("[*] dt : %s" % dt),
			if (dt.find("Correct") != -1) :
				break
			#	sleep(3)
			weight = int(dt)
			print ("[+] weight : %d" % weight)
			#	sleep(3)
			ck = check(start, mid+1, weight)
			if (ck == 1) :
				print ("[*] counterfeit coin not found")
				start = mid + 1
			elif (ck == 0) :
				print ("[*] counterfeit coin found")
				end = mid
		print ("[+] Done, counterfeit coin has been found")
	while True :
	   	data = r.recvline ()
		print data
		if (data.find("bye!") != -1) :
			break

if __name__ == "__main__" :
	main ()
```
