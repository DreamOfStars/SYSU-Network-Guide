# SYSU-Network-Guide
中山大学锐捷认证上网路由器配置指南 - R2S 例子

此文撰写于2022年7月26日，在此时间段一切运行正常，不保证以后此教程的可行性。

由于种种不可述说的原因，无线的 SYSU-SECURE 和宿舍的有线网往往不能满足我们的需求。（rnm，退钱！）自己动手丰衣足食，搭建一个软路由 AP 可以满足我们的一些奇怪需求（比如宿舍开黑、局域网传文件、到世界的另一端看看等）。下文将以 NanoPi R2S + 斐讯 K2P（任意 AP 均可）为例，介绍一下我在珠海校区的组网方案。

## 确定网络拓扑图

一开始只购入了斐讯 K2P A1，曾经的性价比之王。它本身可以刷入一些固件，但是它的性能还是不太能满足我的需求，内存跑个 V2Ray 网络很不稳定，所以后来我就放弃了折腾它的想法，让它老老实实当一个 AP 算了。

购买前需要自行准备好 SD 卡读卡器（用于系统烧录）和 网线两根 （一根从校园网拉出来的网线连 R2S WAN，一根从R2S LAN 连 K2P WAN），以及一个 USB 网线接口（现在大部分笔记本都不支持直接插网线，在前期网络配置时需要用到）。

<img src="https://user-images.githubusercontent.com/25294404/180906909-0d03f89f-e2a6-4d77-8113-aa70e205bdd8.png" width="60%">

## NanoPi R2S OpenWrt 安装及配置

安装过程以及固件推荐可以参考网站 [R2S第三方固件 - ARM NanoPi (gitbook.io)](https://bigdongdong.gitbook.io/nanopi-r2s/r2srotherfirmware)

尝试了几个固件之后我选择了[kiddin9/OpenWrt_x86-r2s-r4s-r5s-N1)](https://github.com/kiddin9/OpenWrt_x86-r2s-r4s-r5s-N1)，根据自己喜好选择就好。如果有更多自定义需求可以尝试自己编译 OpenWrt 系统，不过编译过程稍复杂，耗时较长，容易翻车，所以不太推荐（可以利用 GitHub Actions 实现云端编译），本文将不会涉及这些内容。

然后将写好系统的内存卡插入 R2S 的屁屁里面，插腚揩机！不同的固件指示灯会不一样（可以进去 OpenWrt 后台 LED 配置更改），现在要关注的有两点：1、默认管理地址 2、是否互换 LAN 和 WAN。

需要将网线插入 R2S 的 LAN 口和电脑连接，之后在浏览器输入管理地址（例 192.168.1.1）如无意外可以顺利进入后台管理。设置完管理账号密码后，我们需要修改 LAN 的地址，把它和 AP(K2P) 放在同一个网段下，这样才能实现正常上网。（你也可以修改 AP 的地址）在本例中，K2P(Padavan 固件) 的后台管理地址为 192.168.123.1，我把 LAN 口的 IP 地址 (原 192.168.1.1) 修改为 192.168.123.123。（同一个网段需要通过子网掩码来判断，不过为了便于理解和操作，只要前几个数字相同就可以保证正常上网）。

> 注意：修改 LAN 地址后管理地址也会随之更改，此时你不能再通过原 192.168.1.1 来访问 OpenWrt，而应该访问 192.168.123.123。

### 修改 LAN 口地址/后台管理地址

有些固件都可以直接进入系统->设置向导中修改 IPv4 地址（即后台访问地址），或者在网络->接口中修改LAN口IPv4地址。

或者使用远程连接的方法。使用 SSH 连接 R2S，确保路由器的 SSH 功能打开（默认是打开的）。

以 Windows 为例，按 Win+R 键输入 cmd ，或者开始菜单右键找到命令提示符 or Windows Terminal，打开。输入 `ssh root@192.168.1.1`（root 为后台用户名，192.168.1.1 替换成你选择的固件的默认管理地址），然后输入密码成功连接。

输入以下代码，完成修改

````bash
uci set network.lan.ipaddr='192.168.123.123'
uci commit network
reboot
````

 等待重启之后，访问修改后的后台管理地址，一切正常。

###  编译 Minieap

使用了一年多的 `mentohust` ，由于其历史悠久，实际使用的时候会出现频繁掉线，或者延迟不稳定等情况，因此不再使用。我们将使用 KumaTea 提供的 SYSU 修改版 minieap [KumaTea/minieap: 可扩展的 802.1x 客户端，带有锐捷 v3 (v4) 算法插件支持 (github.com)](https://github.com/KumaTea/minieap)

注意，我们需要搭建一个Linux环境用于软件包的编译。如果有别人已编译好的版本最好，但是在我写出这篇文章之前似乎是没有找到R2S对应版本的软件包，不得已自己手动操作了。其实也很简单，懒得编译的可以直接下载本仓库的ipk文件。

首先到OpenWrt SDK找到自己版本的源码下载https://downloads.openwrt.org/。

```bash
cd /path/to/your/sdk
git clone https://github.com/KumaTea/openwrt-minieap.git package/minieap
make menuconfig # choose `minieap` in section `Network`
make package/minieap/compile V=s
```

然后就能够在`bin\package\aarch64_generic\base`目录下找到我们生成的`minieap_0.93-1_aarch64_generic.ipk` 软件包了。

### 安装 Minieap 并实现锐捷认证

打开OpenWrt后台，进入系统->软件包->上传软件包，将minieap的软件包上传并安装。或者打开SCP手动将软件上传然后使用`opkg install`进行安装。

`ssh`到OpenWrt命令行，输入命令

`minieap -u 你的NetID -p 你的NetID密码 -n eth1 -b 2 --save --module rjv3`

然后查看输出，如果显示认证成功，即可尝试能否正常上网。

其中命令中使用各参数解释如下：

`-n 网卡名` 这里网卡名称为你WAN口名称，可以在网络->接口中找到WAN口对应的网卡名

`-b 参数`  后台运行方式： [默认0]
                                0 = 不后台
                                1 = 后台运行，关闭输出
                                2 = 后台运行，输出到当前控制台
                                3 = 后台运行，输出到日志文件
                                
确定网络正常之后，设置minieap为开机运行，把上述命令输入到系统->启动项->本地启动脚本。为了提高稳定性，建议开启定时重启。

## 拓展

可以探索更多R2S的连接方式。当我回到家里之后，发现忘记了家里路由器的登陆密码，又不想破坏家里原有的网络，所以选择让R2S以旁路网关的方式接入网络。当设备以默认方式连接AP时，享受原有正常网络；当指定网关到R2S时，可以享受R2S提供的多姿多彩的网络服务。

网络拓扑如下:

<img src="https://user-images.githubusercontent.com/25294404/180906960-47319f4f-5aa0-412c-b5bb-a78562c61301.png" width="60%">

原本主路由不需要进行任何配置（实际上我也没办法去配置hhh，忘记密码只能够强行重置）。使用一根网线，连接两个路由器的**LAN口**（注意都是LAN口，R2S的WAN口空着）。

进入到R2S OpenWrt的接口->LAN中，把IPv4地址修改成与主路由同一网段的IP，将网关指定为主路由的IP（默认会自动配置）。进入DHCP服务器设置，选择忽略此接口。一切就绪。

手机或者电脑连接家里WiFi时，流量不会经过R2S。打开WiFi设置，将IP设置从DHCP改成静态，并且指定网关到R2S IP (此例中为192.168.0.123)，即可使用R2S的服务。






