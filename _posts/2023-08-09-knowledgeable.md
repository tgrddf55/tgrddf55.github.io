# 前言

本文记录一下打国外比赛或者自己做题时遇到的一些有意思的考点知识。

> https://github.com/tgrddf55/any/tree/main/challenges

# 2023-08-07 LITCTF

1. obf.py

```python
eval(compile(b64decode(eval('\x74\x72\x75\x73\x74')),'<string>','exec'))
```

python中 compile的用法   字符串混淆方法    import  as修改名称





2. budget-mc

   题目是在远端服务器上运行的，因此这里的flag.txt在本地是无法看到的

   ![image-20230809170416414](../AppData/Roaming/Typora/typora-user-images/image-20230809170416414.png)

题目本身是一个带有重力的二维游戏

![image-20230809170559194](https://raw.githubusercontent.com/tgrddf55/tgrddf55.github.io/main/imgs/202308091706378.png)

程序中开了一个数组，用一维数组来表示二维数组，向上走只能通过改变当前元素值为1。程序中唯一可以显示元素值的是

![image-20230809171009319](https://raw.githubusercontent.com/tgrddf55/tgrddf55.github.io/main/imgs/202308091710891.png)

看起来程序没有地方直接输出flag，我们要想获得flag只能依据这个数组和flag数组的位置关系，通过类似于pwn的方式来越界访问数组，从而获取flag。