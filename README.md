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

## 0x01 漏洞分析

:zero: **Firmware**

:one: **xss**

:two: **Retrieving admin password**

***获取web管理员密码漏洞***本质缺陷在于Mydlink云协议在路由器端没有对发送请求的用户身份进行鉴权，导致任意用户可以请求路由器将其注册到远程的Mydlink云端，然后用户通过注册时提供的账号和密码，登陆到Mydlink云端的web管理界面，虽然协议采用https，但对通信双方而言数据是透明的，只是对中间人是加密的数据。这部分https数据中就包含明文的路由器的web管理界面密码。简言之，非管理员的局域网用户通过Mydlink云协议能够获取管理员的账号密码。

#### Mydlink UI注册：

#### Mydlink 命令行注册：

#### 如何获取web管理员密码：

#### 总结与思考

:question:

:question:

:question:

:three: **Weak Cloud protocol**

:four: **Backdoor access**

:five: **Stunnel private keys**

:six: **Nonce bruteforcing for DNS configuration**

:seven: **Weak files permission and credentials stored in cleartext**

:eight: **Pre-Auth RCEs as root (L2)**

:nine: **DoS against some daemons**
