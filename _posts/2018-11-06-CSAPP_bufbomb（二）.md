---
layout: post
title: CSAPP:bufbomb（二）
category: CSAPP_LAB
description: CSAPP:bufbomb
published: true
---

## Level 1:Sparkler
```c
void fizz(int val)
{
  entry_check(1); /* Make sure entered this function properly */
  if (val == cookie) {
    printf("Fizz!: You called fizz(0x%x)\n", val);
    validate(1);
  } else
    printf("Misfire: You called fizz(0x%x)\n", val);
    exit(0);
}
```

阅读函数，发现这次不但需要我们调用`fizz()`函数，而且函数中还多了一个参数`fizz(int val)`我们需要传入参数并且需要与cookie相等，此处的cookie即[上文](https://www.zybuluo.com/windmelon/note/1332160)提到的经过哈希函数转换后的八个十六进制数

先看看`<fizz>`函数的汇编代码

![image.png-174.3kB][1]

发现比较是发生在第九行

```asm
8048dd6:	3b 1d cc a1 04 08    	cmp    0x804a1cc,%ebx
```
而%ebx中的值来源于%ebp+0x8的地址指向的四个字节的值
```asm
8048dc7:	8b 5d 08             	mov    0x8(%ebp),%ebx
```

那么我们的目标就是想办法把%ebp+0x8的值给改成cookie的值，在这里是
**26 f2 53 e8**

在[Level 0](https://www.zybuluo.com/windmelon/note/1332160)中我们通过改写%ebp+0x4的值，改写了程序跳转地址，那么我们在此次是否把跳转地址改成`<fizz>`然后在后面接上四个和cookie值一样的字节就行了呢
**xx xx xx xx xx xx xx xx xx xx xx xx xx xx xx xx c0 8d 04 08 e8 53 f2 26**

非也，因为我们是直接跳转到`<fizz>`函数的第一条指令，而不是用`call`指令调用，所以我们没有往栈中push`<fizz>`返回后执行的下一条指令，所以进入到`<fizz>`函数中%ebp和`<getbuf>`中的%ebp会有四个字节的差距（一个指向指令的地址的长度）。

当然，不知道这个点我们也能做出来，也就是使用gdb工具分别观察两个函数中%ebp中的值的差别

在`<getbuf>`中

![image.png-141.9kB][2]

在`<fizz>`中

![image.png-141.8kB][3]

可以看到`<fizz>`中%ebp的值代表的地址比`<getbuf>`中%ebp的值代表的地址高出四个字节，那么我们需要覆写的`<fizz>`中%ebp+0x8的值在`<getbuf>`中自然是%ebp+0xc

类似
**00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 c0 8d 04 08 00 00 00 00 e8 53 f2 26**

验证一下

![image.png-70.9kB][4]

成功调用`fizz()`，并且传入值正确


  [1]: http://static.zybuluo.com/windmelon/43xr7augfv62wm6b575anbzw/image.png
  [2]: http://static.zybuluo.com/windmelon/6fgtjmjfgkmg3u2fgzodyp9b/image.png
  [3]: http://static.zybuluo.com/windmelon/nljkw5vpal8p5f9etjppfhgt/image.png
  [4]: http://static.zybuluo.com/windmelon/2j0ywmun0miiu80qc0pzsrv1/image.png
