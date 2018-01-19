# Security名词

## 工具箱

* beEF
  * JS代码利用

* nmap

  * 主机发现

    >-sn：跳过端口扫描阶段
    >
    >TCP检测：
    >
    >-PS[port]：TCP SYN扫描。和指定端口（默认80）尝试建立TCP连接，以此来判断主机是否存活
    >
    >-PA：TCP ACK扫描。
    >
    >UDP检测：
    >
    >-PU: 发UDP到默认40125端口，看返回情况
    >
    >ICMP检测：
    >
    >-PE
    >
    >-PP
    >
    >-PM
    >
    >SCTP检测：
    >
    >-PY:
    >
    >IP协议检测：
    >
    >-PO
    >
    >会用IP协议发各种不同的包，以此检测主机
    >
    >ARP检测：
    >
    >-PR
    >
    >--send-ip：（在局域网内）强制不使用ARP
    >
    >​
    >
    >​
    >
    >​

    ​

  * 端口扫描

  * 服务发现

  * 漏洞发现

  * 漏洞利用

  * 暴力破解

  * 保护自己

    >隐藏踪迹
    >
    >1. MAC地址伪造
    >
    >--spoof-mac <max_address>
    >
    >2. 使用代理
    >
    >   对TCP使用代理，不要使用ping，会泄露原始IP
    >
    >3. IP地址伪造
    >
    >4. ​

  * 脚本

    * 广播

      >主机发现：
      >
      >nmap --script broadcast-ping: 发送ICMP echo报文至本地广播地址（255.255.255.255）
      >
      >网络服务发现：
      >
      >nmap --script broadcast -e <interface>
      >
      >会使用broadcast目录下所有脚本。。。。。
      >
      >​

    ​

* burpsuite

* metasploit



## 账号

[我的通行你的证](http://static.hx99.net/static/drops/web-12695.html)

### 密码

#### 主流盗号的八十一种姿势

- **密码类漏洞**
  ——密码泄露、暴力破解、撞库、密码找回漏洞、社工库、钓鱼…
- **认证cookie被盗**
  ——xss攻击、网络泄露、中间人攻击
- **其他漏洞**
  ——二维码登录、单点登录、第三方登录、客户端web自动登录、绑定其他账号登录、oauth登陆漏洞…





## cookie

#### cookie安全注意点

Httponly：防止cookie被xss偷

https：防止cookie在网络中被偷

Secure：阻止cookie在非https下传输，很多全站https时会漏掉

Path :区分cookie的标识，安全上作用不大，和浏览器同源冲突



## XSS

* 反射型
* 存储型
* DOM型
* 通用型





## 社会工程学

* 使用CURL克隆网站