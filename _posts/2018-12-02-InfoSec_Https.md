---
layout: post
title: InfoSec:Https
category: InfoSec_LAB
description: InfoSec:Https
published: true
---

## 实验介绍
本次实验是关于Https连接
要求使用OpenSSL建立个人CA
使用Apache搭建web服务器
从而为自己的网站建立一个https连接
在开始实验之前，我们首先需要搞清楚本次实验所涉及的概念

## HTTPS
说HTTPS之前需要知道什么是HTTP
HTTP:HyperText Transfer Protocol 超文本传输协议
顾名思义，是用来传输超文本，而超文本即我们所浏览的网页
所以HTTP诞生之初就是用来传输网页内容
后来随着浏览器功能愈发强大，网页可以做到的功能越来越多，HTTP也可以用来传输更多东西
例如向服务器提交请求，传输图片等
而HTTP协议的传输过程都是明文传输，当我们需要传输的信息涉及隐私时，这种传输方式就非常不安全，因此诞生了HTTPS
全称Hyper Text Transfer Protocol over Secure Socket Layer
即在HTTP之下增加了一层SSL，得以保证需要传输的数据是以加密的形式传出去的

## SSL
SSL保证服务器与客户端通信安全的机制是：
<ul>
<li>利用对称加密算法对传输的数据进行加密</li>
<li>利用数字签名证书的方法验证通信双方的身份</li>
</ul>

使用数字签名验证身份时。须要确保被验证者的公钥是真实的，否则。非法用户可能会冒充被验证者与验证者通信。如图1所看到的。Cindy冒充Bob，将自己的公钥发给Alice，并利用自己的私钥计算出签名发送给Alice，Alice利用“Bob”的公钥（实际上为Cindy的公钥）成功验证该签名，则Alice觉得Bob的身份验证成功，而实际上与Alice通信的是冒充Bob的Cindy。

![image.png-13.5kB][1]

验证身份时使用的签名证书一般由受信任的第三方机构Certification Authority（CA）颁发给服务提供商
具体流程如下图

![image.png-153.1kB][2]

<ol>
<li>CA将自己的根证书颁发给浏览器厂商，里面保存了自己的公钥等验证信息</li>
<li>服务商付费在CA处使用CA的私钥对自己的信息进行数字签名，其中包括服务商的URL、公钥等用于验证身份的信息</li>
<li>当服务器与用户进行通信之前，在SSL握手过程中，服务商会把自己的证书发给用户，用户使用根证书解密之后，与自己访问的网站信息进行对比，确认访问的网站的身份</li>
</ol>

知道了上面这些，对本次实验要干的事情也有了一个完整的了解
即在自己本地的环境中，构建一个完整的如上图所描述的HTTPS流程

## 实验平台
Ubuntu 16.04 虚拟机
FireFox 浏览器

## 实验步骤
[参考文档:OpenSSL](https://help.ubuntu.com/community/OpenSSL)

### 安装所需要的工具
OpenSSL
检查是否已安装
>~$openssl version

如果没有安装则
>~$sudo apt-get install openssl

Apache2
检查是否已安装
>~$apache2 -v

如果没有安装则
>~$sudo apt install apache2

###工作目录准备
>~$cd && mkdir -p myCA/signedcerts && mkdir myCA/private && cd myCA

创建myCA文件夹，并在此文件夹下创建两个子文件夹signedcerts，private
<ol>
<li>~/myCA：包含CA证书，证书数据库，生成的证书，密钥和请求</li>
<li>~/myCA/signedcerts:保存签名证书的copy</li>
<li>~/myCA/private: 包含私钥</li>
</ol>

### 在myCA中配置参数文件
创建初始数据库
>~$echo '01'>serial && touch index.txt

创建初始caconfig.cnf
>~$sudo nano ~/myCA/caconfig.cnf

将以下内容写入文件
```
# My sample caconfig.cnf file.
#
# Default configuration to use when one is not provided on the command line.
#
[ ca ]
default_ca      = local_ca
#
#
# Default location of directories and files needed to generate certificates.
#
[ local_ca ]
dir             = /home/<username>/myCA
certificate     = $dir/cacert.pem
database        = $dir/index.txt
new_certs_dir   = $dir/signedcerts
private_key     = $dir/private/cakey.pem
serial          = $dir/serial
#       
#
# Default expiration and encryption policies for certificates.
#
default_crl_days        = 365
default_days            = 1825
default_md              = sha256
#       
policy          = local_ca_policy
x509_extensions = local_ca_extensions
#
#
# Copy extensions specified in the certificate request
#
copy_extensions = copy
#       
#
# Default policy to use when generating server certificates.  The following
# fields must be defined in the server certificate.
#
[ local_ca_policy ]
commonName              = supplied
stateOrProvinceName     = supplied
countryName             = supplied
emailAddress            = supplied
organizationName        = supplied
organizationalUnitName  = supplied
#       
#
# x509 extensions to use when generating server certificates.
#
[ local_ca_extensions ]
subjectAltName = DNS:localhost
basicConstraints = CA:false
nsCertType = server
#       
#
# The default root certificate generation policy.
#
[ req ]
default_bits    = 2048
default_keyfile = /home/<username>/myCA/private/cakey.pem
default_md      = sha256
#       
prompt                  = no
distinguished_name      = root_ca_distinguished_name
x509_extensions         = root_ca_extensions
#
#
# Root Certificate Authority distinguished name.  Change these fields to match
# your local environment!
#
[ root_ca_distinguished_name ]
commonName              = MyOwn Root Certificate Authority
stateOrProvinceName     = NC
countryName             = US
emailAddress            = root@tradeshowhell.com
organizationName        = Trade Show Hell
organizationalUnitName  = IT Department
#       
[ root_ca_extensions ]
basicConstraints        = CA:true
```

注意，要将 `/home/<username>/myCA`改成自己的路径

### 生成CA 根证书和密钥
创建环境变量
>~$export OPENSSL_CONF=~/myCA/caconfig.cnf

生成CA证书和密钥
>~$openssl req -x509 -newkey rsa:2048 -out cacert.pem -outform PEM -days 1825

在这一步可能遇到如下问题
>unable to write 'random state'

使用如下命令即可解决
>sudo rm ~/.rnd

这个`.rnd`文件是生成密钥所需要的随机文件，当它被root用户所占有时就会出现上述错误，此处把它删除就没问题了，因为需要时它又会生成

返回如下提示，设置密码即生成成功
>Generating a 2048 bit RSA private key
.................................+++
.................................................................................................+++
writing new private key to '/home/bshumate/myCA/private/cakey.pem'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:

生成两个文件
<ol>
<li>~/myCA/cacert.pem : CA 公共证书</li>
<li>~/myCA/private/cakey.pem : CA 私钥</li>
</ol>

### 生成CA签名的服务器证书
创建服务器配置文件
>~$sudo nano ~/myCA/exampleserver.cnf

将以下内容写入文件
```
#
# exampleserver.cnf
#

[ req ]
prompt = no
distinguished_name = server_distinguished_name

[ server_distinguished_name ]
commonName              = tradeshowhell.com
stateOrProvinceName     = NC
countryName             = US
emailAddress            = root@tradeshowhell.com
organizationName        = My Organization Name
organizationalUnitName  = Subunit of My Large Organization
```

更改ssl配置文件环境变量
>~$export OPENSSL_CONF=~/myCA/exampleserver.cnf

生成服务器证书和密钥
>~$openssl req -newkey rsa:1024 -keyout tempkey.pem -keyform PEM -out tempreq.pem -outform PEM

生成未加密的私钥
>~$openssl rsa < tempkey.pem > server_key.pem

当然，需要输入上面的passphrase
>Enter pass phrase:
writing RSA key

还原环境变量
>~$export OPENSSL_CONF=~/myCA/caconfig.cnf

使用CA私钥对服务器证书签名
>~$openssl ca -in tempreq.pem -out server_crt.pem

输入生成CA私钥时设置的加密密码，生成如下两个文件
<ol>
<li>server_crt.pem : 服务器证书文件</li>
<li>server_key.pem : 服务器密钥文件</li>
</ol>

接下来，就是服务器的部分

### Apache2的配置
启用SSL MOD
>~$sudo a2enmod ssl

重启apache2
>~$service apache2 restart

进入apache2配置目录
>~$cd /etc/apache2

配置默认的SSL页面
>~$a2ensite default-ssl.conf

此时进入sites-enabled文件夹修改配置文件
>~$cd /etc/apache2/sites-enabled

将“	SSLCertificateFile ”一项改为”sever_crt.pem” 的绝对路径（从根目录开始）
将”SSLCertificateKeyFile ”一项改为“server_key.pem”的绝对路径。
将DocumentRoot指向存放index.html的目录
>~$sudo vim default-ssl.conf

### 证书导入
此时打开firefox浏览器，输入https://localhost，浏览器会自动拦截，因为此时我们还没有将CA根证书导入浏览器，所以浏览器认为这不是一个安全的链接

![image.png-143.2kB][3]

点击浏览器右上角
Preferences->Privacy&Security->View Certificates->Import
导入~/myCA目录下的cacert.pem

![image.png-120kB][4]

###将HTTPS绑定到8080端口
想要使HTTPS绑定到8080端口，只需修改
/etc/apache2/ports.conf
```
# If you just change the port or add more ports here, you will likely also
# have to change the VirtualHost statement in
# /etc/apache2/sites-enabled/000-default.conf

Listen 80

<IfModule ssl_module>
        Listen 8080
        #原来为443
</IfModule>

<IfModule mod_gnutls.c>
        Listen 8080
        #原来为443
</IfModule>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```

和/etc/apache2/sites-enabled/default-ssl.conf
```
<VirtualHost _default_:8080>#原来为443
```

![image.png-120.9kB][5]


  [1]: http://static.zybuluo.com/windmelon/lg1fm5l7waqcb59znei8ay08/image.png
  [2]: http://static.zybuluo.com/windmelon/1t7fdhj2cktyeb0cgmcc8ndd/image.png
  [3]: http://static.zybuluo.com/windmelon/0zihul1ogchc14lbfs17e9q0/image.png
  [4]: http://static.zybuluo.com/windmelon/utjkhtjokd2n6zelsmu2cj24/image.png
  [5]: http://static.zybuluo.com/windmelon/blzrpraks55bq80klg5fzypa/image.png