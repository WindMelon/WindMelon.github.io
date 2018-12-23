---
layout: post
title: CSAPP:bufbomb（三）
category: CSAPP_LAB
description: CSAPP:bufbomb
published: true
---

## Level 2:Firecracker
```c
int global_value = 0;
void bang(int val)
{
  entry_check(2); /* Make sure entered this function properly */
  if (global_value == cookie) {
    printf("Bang!: You set global_value to 0x%x\n", global_value);
    validate(2);
  } else
    printf("Misfire: global_value = 0x%x\n", global_value);
  exit(0);
}
```
Level 2在[Level 1](https://www.zybuluo.com/windmelon/note/1333271)的基础上增加了一个全局变量`int global_value`，阅读代码我们可以发现这次我们不需要管参数`int val`的值，而是需要把全局变量`global_value`设置为与`cookie`相同的值。

我们知道，在程序中，局部变量、函数参数是保存在栈中，因此我们可以通过缓冲区溢出的方式覆写栈中内容达到修改一些局部变量或者函数参数的目的，而全局变量是保存在内存的其他区域，因此，要修改全局变量，必须要知道变量的地址，并且要用指令对其进行修改，那我们先来看一下`<bang>`的汇编代码

![image.png-174.6kB][1]

可以看到七八行的代码有两个静态的地址0x804a1dc和0x804a1cc
```asm
8048d72:	a1 dc a1 04 08       	mov    0x804a1dc,%eax
8048d77:	3b 05 cc a1 04 08    	cmp    0x804a1cc,%eax
```

猜测应该就是`global_value`和`cookie`的地址，使用gdb验证一下

![image.png-138kB][2]

值分别为0和0x26f253e8即`global_value`和`cookie`的值

接下来就要想办法把0x804a1dc里面的值修改为0x26f253e8

有了前面的经验，我们知道buf+16到buf+19存的是跳转地址，那么我们就可以把指令写到buf+0到buf+15中，然后在把跳转地址写成buf就可以了。需要注意的是指令不能超过16个字节，不然不够空间储存跳转地址

首先，我们先明确我们需要进行的操作
```asm
movl $0x26f253e8,%eax
movl %eax,0x804a1dc
push  $0x08048d60
ret
```

首先将0x26f253e8赋值给%eax寄存器，再将%eax寄存器中的值存到0x804a1dc这个地址中，完成对`global_value`的赋值，然后再将`<bang>`的首地址push入栈，然后调用ret指令，即取栈顶值作为跳转目的地

指令有了，我们如何知道它的机器码呢，我们可以先用gcc -c对这些指令进行汇编，然后再用objdump进行反汇编，就可以得到需要的机器码了

```asm
0000000000000000 <.text>:
   0:	b8 e8 53 f2 26       	mov    $0x26f253e8,%eax
   5:	a3 dc a1 04 08 	        mov    %eax,0x804a1dc
   a:	68 60 8d 04 08       	pushq  $0x8048d60
  f:	c3                   	retq   
```

正好十六个字节，接下来需要知道buf[12]数组的首地址，用gdb观察

![image.png-134.9kB][3]

buf[12]首地址为0xffffb8fc，那么我们需要输入的字节就为
**b8 e8 53 f2 26 a3 dc a1 04 08 68 60 8d 04 08 c3 fc b8 ff ff**

但是如果此时直接运行程序会出现段错误，因为现在的计算机都对栈进行了保护，不可以在栈中直接执行指令，我们需要使用execstack工具解除栈执行限制

![image.png-49.8kB][4]

execstack -s解除栈执行限制，使用execstack -q查看，如果为X则可以在栈中执行指令

测试

![image.png-157.4kB][5]

不知道为什么，只能在gdb中运行成功，如果直接运行程序则会出现段错误

![image.png-64.3kB][6]

经查阅资料，是因为Linux系统中存在随机地址空间分配机制（ASLR），而在使用gdb的时候，gdb会关闭ASLR，而在实际运行的时候，计算机会对地址进行随机分配，在GDB中找到的地址不是程序实际运行时的地址，要找到实际运行时的地址还得费一番功夫，这里就不展开了

## Level 3:Dynamite

Level 3比较简单，不需要我们调用额外的函数，只需要我们执行完`getbuf()`后，返回的值是`cookie`而不是1，并且不破坏栈的结构。

即，我们要保证我们的返回要像正常的步骤，即在`test()`函数中调用`getbuf()`前后，%ebp的值保持不变，那么我们就需要知道正常返回的%ebp值是什么，然后使相应位置的内容保持不变，就可以维持栈结构了

![image.png-138.1kB][7]

![20181106171142.png-65.4kB][8]

将断点设置在`<getbuf>`首地址，此时查看%ebp的值，即为返回时应保存在%ebp中的值，可以看到为0xffffb928

接下来我们需要把返回值设为`cookie`，观察`<getbuf>`的汇编码可以发现返回值是保存在%eax中，所以我们构建指令
```asm
0000000000000000 <.text>:
   0:	b8 e8 53 f2 26       	mov    $0x26f253e8,%eax
   5:	68 1e 90 04 08       	pushq  $0x804901e
   a:	c3                   	retq   
   b:	90                   	nop
```

最后push入栈的是`<test>`中执行完`call <getbuf>`后下一条指令的地址，所以我们需要输入的字节为
**b8 e8 53 f2 26 68 1e 90 04 08 c3 90 28 b9 ff ff fc b8 ff ff**

在gdb中测试

![image.png-156.5kB][9]

成功


  [1]: http://static.zybuluo.com/windmelon/uloeady8xnufj1nphgr3q97q/image.png
  [2]: http://static.zybuluo.com/windmelon/vecuwihi3971x0xz1o1wu50o/image.png
  [3]: http://static.zybuluo.com/windmelon/n81hvv6ja16dn8amlk4gwfnv/image.png
  [4]: http://static.zybuluo.com/windmelon/pnj3sfj1zu320b7696w1hmft/image.png
  [5]: http://static.zybuluo.com/windmelon/w2qzlwu61rxc0f0jc6cnbfn9/image.png
  [6]: http://static.zybuluo.com/windmelon/btm5tuyv5gnqp98wcakts27z/image.png
  [7]: http://static.zybuluo.com/windmelon/847k2gsuozunif0zsugw5htl/image.png
  [8]: http://static.zybuluo.com/windmelon/kz86ph70roalx8u6vbon8ijx/20181106171142.png
  [9]: http://static.zybuluo.com/windmelon/wpvg9pauymucw96j3r2mhdx5/image.png