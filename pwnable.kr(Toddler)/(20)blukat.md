# 문제
Sometimes, pwnable is strange...
hint: if this challenge is hard, you are a skilled player.

ssh blukat@pwnable.kr -p2222 (pw: guest)

---  
# 개요  
```
blukat@ubuntu:~$ ls
blukat	blukat.c  password
blukat@ubuntu:~$ cat password
cat: password: Permission denied
blukat@ubuntu:~$
```
cat: password: Permission denied.....? 뭔가 이상합니다. 그대로 입력해볼까요?
```
blukat@ubuntu:~$ ./blukat
guess the password!
cat: password: Permission denied
congrats! here is your flag: Pl3as_DonT_Miss_youR_GrouP_Perm!!
blukat@ubuntu:~$

```
#???
어쨋든 해결했습니다...
