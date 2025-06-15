## 前言
有些哥们发布的特定服务可能需要占用大量CPU、内存或者硬盘空间，买高配的云服务器又不划算，所以部署在本地通过内网穿透发布出去，于是就有了这篇文章。

本文使用Zerotier，这个软件官方也提供免费通道，但是很不稳定，所以本文是自建Planet实现稳定的内网穿透。

力求小白看了也能直接配好，每行要执行的代码前面会标注执行原因。但是你如果连怎么连接云服务器都不清楚的话，你需要先搜一下怎么SSH连接云服务器。

不知道选择什么样的云服务器合适？可以参阅我的另一篇文章：[VPS/云服务器的选择策略](https://www.cnblogs.com/xiaotie666/p/18926245/vps-tuijian "VPS/云服务器的选择策略")。

感谢zerotier官方以及[docker-zerotier-planet](https://github.com/xubiaolin/docker-zerotier-planet "docker-zerotier-planet")的作者[xubiaolin](https://github.com/xubiaolin "xubiaolin")。

**配置前提：**
- 一台拥有公网IP的云服务器，最好是干净的，刚装完系统的，可以是NAT型的，不影响。
- 如果有云服务器提供商开了默认的防火墙或者安全组，请把所有端口全部开放。
- 本地客户端能正常上网。
- 确保客户端的网络运营商没有对UPD进行限制（因为Zerotier使用UDP协议），同时确保网络运营商没有限制上传速率。
- 要求云服务器能够ping通回环地址（127.0.0.1），否则**zerotier-one**无法正常启动。

**云服务器的推荐配置：**
- 系统：Ubuntu 22.0 版本以上/Debian 12.0 版本以上
- CPU：1核
- 内存：1G

------------

# 目录
- [1：ZeroTier 介绍](#1zerotier-介绍)
- [2：服务器端配置](#2服务器端配置)
  - [2.1：准备条件](#21准备条件)
    - [2.1.1：切换国内软件源](#211切换国内软件源)
    - [2.1.2：安装curl](#212安装curl)
    - [2.1.3：安装1Panel和Docker](#213安装1panel和docker)
    - [2.1.4：安装git](#214安装git)
    - [2.1.5：安装ufw防火墙（用于配置端口转发）](#215安装ufw防火墙用于配置端口转发)
  - [2.2：下载项目源码](#22下载项目源码)
  - [2.3：执行安装脚本](#23执行安装脚本)
  - [2.4：配置网络](#24配置网络)
    - [2.4.1：新建网络](#241新建网络)
    - [2.4.2：分配网络IP](#242分配网络ip)
  - [2.5：配置ufw防火墙](#25配置ufw防火墙)
    - [2.5.1：通过命令开放ssh以及1Panel端口](#251通过命令开放ssh以及1panel端口)
    - [2.5.2：通过1Panel开放所有端口（可仅开放Zerotier需要的端口）](#252通过1panel开放所有端口可仅开放zerotier需要的端口)
  - [2.6：配置Zerotier](#26配置zerotier)
    - [2.6.1：安装Zerotier](#261安装zerotier)
    - [2.6.2：替换planet文件](#262替换planet文件)
    - [2.6.3：加入网络](#263加入网络)
    - [2.6.4：授权并分配服务器IP](#264授权并分配服务器ip)
- [3：客户端配置](#3客户端配置)
  - [3.1：Windows 配置](#31windows-配置)
    - [3.1.1：安装zerotier-one](#311安装zerotier-one)
    - [3.1.2：替换planet文件](#312替换planet文件)
    - [3.1.3：使新的planet生效](#313使新的planet生效)
    - [3.1.4：加入网络](#314加入网络)
    - [3.1.5：授权并分配客户端IP](#315授权并分配客户端ip)
    - [3.1.6：验证网络连通性](#316验证网络连通性)
  - [3.2：Linux 配置](#32linux-配置)
    - [3.2.1：安装zerotier-one](#321安装zerotier-one)
    - [3.2.2：替换planet文件](#322替换planet文件)
    - [3.2.3：使新的planet生效](#323使新的planet生效)
    - [3.2.4：加入网络](#324加入网络)
    - [3.2.5：授权并分配客户端IP](#325授权并分配客户端ip)
    - [3.2.6：验证网络连通性](#326验证网络连通性)
  - [3.3：安卓 配置](#33安卓-配置)
  - [3.4：MacOS 配置](#34macos-配置)
    - [3.4.1：安装zerotier-one](#341安装zerotier-one)
    - [3.4.2：替换planet文件](#342替换planet文件)
    - [3.4.3：使新的planet生效](#343使新的planet生效)
    - [3.4.4：加入网络](#344加入网络)
    - [3.4.5：授权并分配客户端IP](#345授权并分配客户端ip)
    - [3.4.6：验证网络连通性](#346验证网络连通性)
  - [3.5：OpenWRT 配置](#35openwrt-配置)
    - [3.5.1：安装zerotier-one](#351安装zerotier-one)
    - [3.5.2：替换planet文件](#352替换planet文件)
    - [3.5.3：使新的planet生效](#353使新的planet生效)
    - [3.5.4：加入网络](#354加入网络)
    - [3.5.5：授权并分配客户端IP](#355授权并分配客户端ip)
    - [3.5.6：验证网络连通性](#356验证网络连通性)
  - [3.6：iOS 配置](#36ios-配置)
- [4：端口转发配置](#4端口转发配置)

------------

# 1：ZeroTier 介绍

`ZeroTier` 是一款强大的 内网穿透 工具，它能让你在互联网上搭建属于自己的虚拟局域网。通过它，你可以轻松实现远程访问家中设备的需求 - 比如在公司用手机直接访问家里的 NAS。最重要的是，设备之间是点对点直连的，无需经过中转服务器，既保证了速度，又提升了安全性。

它的工作原理是这样的：通过 `ZeroTier One` 客户端，在不同设备（如笔记本、手机、服务器等）之间建立 内网 连接，即使这些设备都在 NAT 后面也没问题。它使用了 STUN 等技术，可以穿透大多数类型的 NAT，实现设备间的直接通信。如果实在无法直连，才会通过中转服务器进行通信。

简单来说，`ZeroTier` 就像是一个跨越互联网的"虚拟交换机"，让分布在世界各地的设备，都能像在同一个局域网内一样方便地相互访问。

![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615140728908-621905746.png)


**ZeroTier 网络中的关键概念**

`PLANET`（行星服务器）：ZeroTier 网络的核心根服务器，负责网络发现和初始连接。相当于整个网络的"中枢"。

`MOON`（卫星服务器）：用户可以自建的私有根服务器。它可以作为区域性的代理节点,帮助就近的设备更快地建立连接,提升网络性能。

`LEAF`（叶子节点）：所有接入 ZeroTier 网络的终端设备,如电脑、手机、服务器等。这些设备通过 PLANET 和 MOON 的协调来相互发现和通信。

本教程将指导您搭建一个私有的 PLANET 服务器,让您完全掌控自己的 ZeroTier 网络。

------------

# 2：服务器端配置
### 2.1：准备条件
##### 2.1.1：切换国内软件源
首先SSH连接到云服务器，记住要使用root用户。连上去以后不要切换目录。

Ubantu/Debian的默认软件源很多是国外的，下载可能会中断或者速度很慢，所以切换软件源为国内清华源（如果你的VPS是海外线路，那么这块不执行也可以）：
```
cat > /etc/apt/sources.list << EOF
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free non-free-firmware
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-updates main contrib non-free non-free-firmware
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-backports main contrib non-free non-free-firmware
deb https://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
EOF
```

修改软件源配置文件以后，我们更新一下软件包索引，否则白改了：
```
apt update
```

------------

##### 2.1.2：安装curl
安装curl工具，用来下载特定软件。
```
apt install curl -y
```

------------

##### 2.1.3：安装1Panel和Docker
安装Docker和1Panel面板，用这个就图个配置方便，用命令也是能同样实现的。
这是docker的安装命令，默认选第一个一直回车就行。
```
curl -sSL https://linuxmirrors.cn/docker.sh -o docker.sh && bash docker.sh
```

这是1Panel面板的安装命令，用这个就图个配置方便，用命令也是能同样实现的。可以全部采用默认配置，也可以自己修改一下，安装完成后会输出访问地址和用户名密码，记得保存下来。
```
curl -sSL https://resource.fit2cloud.com/1panel/package/quick_start.sh -o quick_start.sh && bash quick_start.sh
```

配置docker加速镜像，更修改软件源一样，用来加速下载docker镜像。
```
tee /etc/docker/daemon.json <<EOF
{
    "registry-mirrors": [
        "https://docker.mirrors.aster.edu.pl",
        "https://docker.mirrors.imoyuapp.win"
    ]
}
EOF
```

应用docker配置。
```
systemctl daemon-reload
```

重启docker，使配置生效。
```
systemctl restart docker
```

------------

##### 2.1.4：安装git
安装git，后面下载docker-zerotier-planet存储库要用到。
```
apt install git -y
```

------------

##### 2.1.5：安装ufw防火墙（用于配置端口转发）
执行命令安装ufw防火墙，用来端口转发，下面的命令一行一行执行，井号后面的可别执行嗷，那是注释！
```
apt install ufw -y
```

------------

### 2.2：下载项目源码
下载docker-zerotier-planet存储库，该项目用于自建Zerotier Planet。
```
git clone https://ghproxy.imoyuapp.win/https://github.com/xubiaolin/docker-zerotier-planet.git
```

------------

### 2.3：执行安装脚本
执行docker-zerotier-planet的安装脚本，按照默认指示就行，安装完成后会打印访问信息，记录下来。
```
./docker-zerotier-planet/deploy.sh
```

------------

### 2.4：配置网络
浏览器访问<云服务器ip>:3443，输入默认的用户名和密码（admin/password），进去后先改一下密码，默认的不安全。

##### 2.4.1：新建网络
进入以后，先点击**Add network**按钮，然后输入**Network name**，这个随便输，最后点击**Create Network**按钮。
![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615002825227-13619324.png)

------------

##### 2.4.2：分配网络IP
点击**Easy setup**按钮。
![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615002955485-1136058412.png)

**Network address in CIDR notation** 输入`11.11.11.0/24`，下面两个输入框的内容应该会自动生成，然后点击Submit按钮，看到xxx Successful的字样，点击上方**Networks**按钮，这个页面先放着，不要关掉。
![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615003119037-728477076.png)

------------

### 2.5：配置ufw防火墙
##### 2.5.1：通过命令开放ssh以及1Panel端口
```
# 开放ssh端口，如果你用NAT型云服务器，那你得把22改成自己连接ssh的端口号
ufw allow 22/tcp

# 开放1Panel访问端口，这个每个人不一样，如果你忘了那么你得去看一下刚刚安装完1Panel以后打印的内容。把下main命令里的尖括号内容改成1Panel访问端口。
ufw allow <1Panel端口>/tcp

# 启用ufw防火墙
ufw --force enable
```

------------

##### 2.5.2：通过1Panel开放所有端口（可仅开放Zerotier需要的端口）
打开1Panel页面，开放防火墙所有端口，也可以只开发特定的，按自己的需求来，我这个只是演示用。
![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615010359890-1912545788.png)
![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615010546874-1121313631.png)

------------

### 2.6：配置Zerotier
##### 2.6.1：安装Zerotier
在服务器中安装zerotier，用来与客户端取得联系，以便后续把请求转发到客户端。
```
curl -s https://install.zerotier.com | bash
```

------------

##### 2.6.2：替换planet文件
转移自建的planet文件至zerotier-one目录。
```
cp /root/docker-zerotier-planet/data/zerotier/dist/planet /var/lib/zerotier-one/
```

重启zerotier-one服务，使配置生效。
```
service zerotier-one restart
```

------------

##### 2.6.3：加入网络
Zerotier加入服务器的**Network ID**，这个**Network ID**可以在刚刚保留的页面获取到，看下图红框的位置，每个人的**Network ID**不一样，别把我图里的给放上去了。
![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615005759656-122404014.png)

执行命令，把`<Network ID>`替换成自己的。
```
zerotier-cli join <Network ID>
```

看到**200 join OK**字样，我们再次返回之前保留的zerotier页面，等个十几秒刷新一下。

------------

##### 2.6.4：授权并分配服务器IP
看到列表中多一行，Member name输入**本机**，然后点击**IP assignment**。
![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615011051052-520505065.png)

输入`11.11.11.99`，点左边的➕，然后点右上角的**Back**按钮。
![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615011145203-1461867257.png)

勾选**Authorized**和**Active bridge**两个复选框，然后点击左下角**Refresh**按钮。
![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615011423897-876857326.png)

------------

至此，服务器端配置完成。

# 3：客户端配置
### 3.1：Windows 配置
##### 3.1.1：安装zerotier-one
下载zerotier-one并安装，下载地址：https://download.zerotier.com/dist/ZeroTier%20One.msi 。

------------

##### 3.1.2：替换planet文件
将 **planet** 文件覆盖粘贴到`C:\ProgramData\ZeroTier\One`中。
- 这个目录是个隐藏目录，需要设置允许查看隐藏目录，不会的话百度下。
- planet文件在云服务器的`/root/docker-zerotier-planet/data/zerotier/dist/planet`

------------

##### 3.1.3：使新的planet生效
按Win+S键，输入service，打开服务。
![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615012202671-473647605.png)

找到ZeroTier One，并且重启服务。
![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615012215686-1805402147.png)

------------

##### 3.1.4：加入网络
使用管理员身份打开PowerShell。
执行如下命令，看到join OK字样就成功了。
```
PS C:\Windows\system32> zerotier-cli.bat join <Network ID>(刚服务器端那边用的Network ID)
200 join OK
PS C:\Windows\system32
```

------------

##### 3.1.5：授权并分配客户端IP
去zerotier管理端网页做刚刚服务器端一样的设置，只不过IP换成`11.11.11.11`，记得勾选**Authorized**和**Active bridge**两个复选框并刷新页面。
![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615012617462-1109459281.png)

------------

##### 3.1.6：验证网络连通性
等个十几秒之后，试着在客户端的cmd里ping一下`11.11.11.99`，能ping通就可以了。

------------

### 3.2：Linux 配置
##### 3.2.1：安装zerotier-one
下载zerotier-one并安装
```
curl -s https://install.zerotier.com | sudo bash
```

------------

##### 3.2.2：替换planet文件
将 **planet** 文件上传到linux客户端。

在 **planet** 文件所在目录执行以下命令，覆盖客户端安装的zerotier-one自带的planet。
```
cp planet /var/lib/zerotier-one/
```

------------

##### 3.2.3：使新的planet生效
重启zerotier-one服务。

```
service zerotier-one restart
```

------------

##### 3.2.4：加入网络
执行如下命令，看到`200 join OK`字样就成功了。
```
zerotier-cli join join <Network ID>(刚服务器端那边用的Network ID)
```

------------

##### 3.2.5：授权并分配客户端IP
去zerotier管理端网页做刚刚服务器端一样的设置，只不过IP换成`11.11.11.11`，记得勾选**Authorized**和**Active bridge**两个复选框并刷新页面。
![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615012617462-1109459281.png)

------------

##### 3.2.6：验证网络连通性
等个十几秒之后，试着在客户端ping一下`11.11.11.99`，能ping通就可以了。

------------

### 3.3：安卓 配置
[Zerotier 非官方安卓客户端](https://github.com/kaaass/ZerotierFix "Zerotier 非官方安卓客户端")

------------

### 3.4：MacOS 配置
##### 3.4.1：安装zerotier-one
下载地址：https://download.zerotier.com/dist/ZeroTier%20One.pkg

------------

##### 3.4.2：替换planet文件
进入 `/Library/Application\ Support/ZeroTier/One/` 目录，并替换目录下的 **planet** 文件

------------

##### 3.4.3：使新的planet生效
重启 ZeroTier-One
```
cat /Library/Application\ Support/ZeroTier/One/zerotier-one.pid | sudo xargs kill
```

------------

##### 3.4.4：加入网络
```
zerotier-cli join 网络 id
```

------------

##### 3.4.5：授权并分配客户端IP
去zerotier管理端网页做刚刚服务器端一样的设置，只不过IP换成`11.11.11.11`，记得勾选**Authorized**和**Active bridge**两个复选框并刷新页面。
![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615012617462-1109459281.png)

------------

##### 3.4.6：验证网络连通性
等个十几秒之后，试着在客户端的cmd里ping一下`11.11.11.99`，能ping通就可以了。

------------

### 3.5：OpenWRT 配置
##### 3.5.1：安装zerotier-one
这个在openwrt商店里可以下载到。

------------

##### 3.5.2：替换planet文件
进入 `/etc/config/zero/planet` 目录，并替换目录下的 **planet** 文件。

------------

##### 3.5.3：使新的planet生效
在openwrt网页后台先关闭zerotier服务，再开启zerotier服务。

------------

##### 3.5.4：加入网络
在openwrt网页后台输入服务器端生成的网络ID，然后加入。

------------

##### 3.5.5：授权并分配客户端IP
去zerotier管理端网页做刚刚服务器端一样的设置，只不过IP换成`11.11.11.11`，记得勾选**Authorized**和**Active bridge**两个复选框并刷新页面。
![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615012617462-1109459281.png)

------------

##### 3.5.6：验证网络连通性
等个十几秒之后，试着在客户端的cmd里ping一下`11.11.11.99`，能ping通就可以了。

------------

### 3.6：iOS 配置
方案一： 越狱后安装ZeroTie，然后替换planet文件

方案二： 使用Wireguard接入到ZeroTier网络

------------

# 4：端口转发配置

打开服务器端1Panel，切换到系统——防火墙——端口转发页面。
![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615013025492-1214540694.png)

协议选`tcp/udp`，源端口就是服务器这边对应的端口，目标IP填zerotier为客户端分配的IP，目标端口填写客户端的被转发的端口，最后点击确认。
![image](https://img2024.cnblogs.com/blog/1422798/202506/1422798-20250615013157828-286115694.png)

最后用云服务器IP:源端口访问一下，此时应该可以访问了。

------------

**最后的最后，如果按照我的教程配置成功了，请帮我点个右上角的星星吧，后续也会更新其他内网穿透工具的教程，谢谢哥们。**
