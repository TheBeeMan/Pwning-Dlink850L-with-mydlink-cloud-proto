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
