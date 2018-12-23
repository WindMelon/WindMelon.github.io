---
layout: post
title: CSAPP:bufbomb（一）
category: CSAPP_LAB
description: CSAPP:bufbomb
published: true
---
# CSAPP:bufbomb（一）

标签（空格分隔）： csapp实验 逆向工程

---

##BufBomb简介
bufbomb是csapp课程的系列实验中的一个，要求掌握程序运行时栈帧的状态，通过往给定大小的数组中写入超过限定大小的字符达到覆写栈中的程序返回指针，使程序跳转到指定位置，或者执行操作者自定义的代码段

##程序主体
实验文件由三个可执行的ELF文件组成
makecookie:一个哈希函数，将一串字符串转换为四个字节长度的八个十六进制数字

![image.png-40.7kB][1]

这串数字后面有用，就把它叫做cookie

sendstring:把以空格分隔的ascii码转换为字符串，因为有的ascii码没有对应的可显示的字符串，但是在实验中我们需要输入这种字符串，需要使用这个程序进行转换

![image.png-41.5kB][2]

bufbomb:本实验的主程序，内含了makecookie的哈希函数，需要我们输入一串字符串(比如小组名)以生成一个独一无二的cookie，需要输入另一串字符串，作为输入流输入到buf[12]这个数组中

![image.png-54.2kB][3]

```c
void test()
{
  int val;
  volatile int local = 0xdeadbeef;
  entry_check(3); /* Make sure entered this function properly */
  val = getbuf();
  /* Check for corrupted stack */
  if (local != 0xdeadbeef) {
    printf("Sabotaged!: the stack has been corrupted\n");
  }
  else if (val == cookie) {
    printf("Boom!: getbuf returned 0x%x\n", val);
    validate(3);
  }
  else {
    printf("Dud: getbuf returned 0x%x\n", val);
  }
}
```

```c
int getbuf()
{
  char buf[12];
  Gets(buf);
  return 1;
}
```

程序开始后会调用`test()`，其中我们输入的第二串字符串是通过`getbuf()`输入到buf[12]中
上图中我们输入的字符串长度不足11，所以程序正常执行，返回1，结束

我们要做的就是利用buf[12]，输入一段字符串，溢出并覆写栈帧中的返回地址，返回到我们指定的位置

当调用`getbuf()`后，栈内的情况如下图所示

![image.png-105.4kB][4]

%ebp为该帧的起始位置，%ebp+0x4的位置为返回后，下一个指令的地址，即`test()`函数的第六行之后的操作，而%ebp-0xc的位置即buf[12]数组的首地址(注:在不同系统下，buf[12]的起始位置可能会有不同，但都会在%ebp和%esp的区间)。
%esp则为栈顶指针

##Level 0:Candle
Level 0很简单，题目要求我们通过向buf[12]写入数据，使得`getbuf()`在读入字符串之后，不执行`return 1`，而是跳转到`smoke()`函数，并且我们不需要考虑是否破坏了栈中的原来数据

```c
void smoke()
{
  entry_check(0); /* Make sure entered this function properly */
  printf("Smoke!: You called smoke()\n");
  validate(0);
  exit(0);
}
```

那么，我们首先对bufbomb进行反汇编，此处需要使用objdump，一般linux的发行版本都已经预装好了此工具，我们直接使用

![image.png-37.5kB][5]

使用notepad看一下bufbomb.asm

![image.png-107.2kB][6]

可以看到`<smoke>`函数的首地址为08048e20

再来看`<getbuf>`

![image.png-70.5kB][7]

可以看到`<getbuf>`中有一个`ret`指令，该指令经我们上面分析，是跳向%ebp+0x4指向的位置，那么我们就要想办法把%ebp+0x4里面的内容改写掉

再看一下栈帧的结构

![image.png-41.5kB][8]

可以看到在不断往栈中添加数据时地址是有高地址变到低地址，而数组中存储的数据是由低地址到高地址，那么我们就可以得到如下结论

%ebp-0xc到%ebp-0x1这个区间存储的是buf[12]中的内容，如果要覆盖%ebp+0x4到%ebp+0x7的内容就必须写入20个字节的内容，并且最后四个字节为`<smoke>`函数的首地址

所以我们需要往buf[12]中写入的内容要为

**xx xx xx xx xx xx xx xx xx xx xx xx xx xx xx xx 20 8e 04 08**

因为上面我们得出`<smoke>`的首地址为08048e20，那最后四个字节为什么不是
**08 04 8e 20**呢，因为我们输入的字符由左到右在内存中地址是由高到低，而在读取的时候是由低到高，所以顺序要返过来

那么我们怎么把这串数据传进去呢，我们不能直接写一串字符串类似
**00000000000000000000000000000000208e0408**

因为计算机会把我们输入的字符转为ascii码，所以我们输入的数据到内存中就会变为
**30 30 30 30 30 30 30 30 30 30 30 30…………32 30 38……34 30 38**

所以需要使用sendstring这个程序，把我们相要写到内存中的值写到一个文件中，每个字节用空格隔开，如下

![image.png-32.4kB][9]

然后使用sendstring进行转换

![image.png-34.1kB][10]

我们的结果就保存在`exploit-raw-string`中

最后运行bufbomb看看我们的结果是否正确

![image.png-45.6kB][11]

成功调用了`smoke()`，成功

  [1]: http://static.zybuluo.com/windmelon/1apt45eiuwn6zbhckqkuri08/image.png
  [2]: http://static.zybuluo.com/windmelon/szjbyu4zbtmrrppaaq0dzafl/image.png
  [3]: http://static.zybuluo.com/windmelon/f252r5j0n2zf141o8f1oithi/image.png
  [4]: http://static.zybuluo.com/windmelon/gdvjnkpb432l94su9pc7p3gx/image.png
  [5]: http://static.zybuluo.com/windmelon/jdzbnowezuxo1blo9ysdvmlh/image.png
  [6]: http://static.zybuluo.com/windmelon/1jiyeoi0pggpfq6fndsp18kp/image.png
  [7]: http://static.zybuluo.com/windmelon/bytx4yp8k5j6ys6q4uoxu051/image.png
  [8]: http://static.zybuluo.com/windmelon/nswxvbtq9vv3oz6cyue734fl/image.png
  [9]: http://static.zybuluo.com/windmelon/i1hey9djjooxq9i3ce0ixkd7/image.png
  [10]: http://static.zybuluo.com/windmelon/wanpcrsfk9l1ejofgubuo28r/image.png
  [11]: http://static.zybuluo.com/windmelon/k129n2r2hsksslsdgbvkkxmn/image.png
