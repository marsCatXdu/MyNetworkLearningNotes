# wpa_supplicant

内容主要翻译、删改自： [wpa_supplicant](https://w1.fi/cgit/hostap/tree/wpa_supplicant/README) 及 https://wireless.wiki.kernel.org/en/users/documentation/wpa_supplicant

如果在阅读本文时怀疑少了什么东西，请查阅原文。本文根据我的个人理解与需要进行了大量的删改



Supplicant 是一个网络认证（authentication）中的术语，指向另一个设备或服务器请求进行身份认证的设备或软件。

WPA supplicant 是支持 WPA （Wi-Fi Protected Access，一种无线网络安全协议）的 supplicant 软件。WPA supplicant 能够和认证服务器进行通信，以获取连接安全 Wi-Fi 网络的证书和密钥。

MLME 是 MAC 层子层的 Management Entity （Medium Access Control sublayer Management Entity），是一个 MAC 子层的组件，有一系列用于管理无线网络的功能，例如认证、关联、取消关联、扫描等。MLME 和驱动以及硬件进行通信，完成这些功能。MLME 是 mac80211 的一部分

SME（System Management Entity）是对无线网络进行较高层管理的组件，管理诸如配置、安全、漫游等功能。SME 通过与 MLME 等通信来实现这些功能

## wpa_supplicant

wpa_supplicant 是 WPA Supplicant 组件的一个实现，运行在客户端站点上。其使用 WPA Authenticator 实现了 WPA 密钥协商以及使用认证服务器（Authentication Server）的 EAP 身份认证。此外，wpa_supplicant 也控制 wlan 驱动的漫游以及 802.11 认证及关联功能。

wpa_supplicant 是作为一个在后台运行的守护进程设计的，其目标是作为一个后台组件对无线连接进行控制和管理。wpa_supplicant 能够支持独立的前端运行，比如 wpa_supplicant 自身包含的文本前端 wpa_cli 。

使用 WPA 关联 AP 时的步骤如下：

- wpa_supplicant 请求内核驱动扫描附近的 BSS
- wpa_supplicant 根据自己的配置文件选择一个 BSS
- wpa_supplicant 请求内核驱动与该 BSS 关联
- 如果使用 WPA-EAP ：集成的 IEEE 802.1X Supplicant 与认证服务器完成 EAP 认证（由 AP 中的 Authenticator 代理进行）
- 如果使用 WPA-EAP：从 IEEE 802.1X Supplicant 收到主密钥（master key）
- 如果使用 WPA-PSK：wpa_supplicant 使用 PSK 作为主会话密钥（master session key）
- wpa_supplicant 与验证者（Authenticator，即 AP）完成 WPA 四步握手和组密钥握手
- wpa_supplicant 为单播与广播配置加密密钥
- 即可开始发送与接收正常的数据包

## 用法

通常使用下面的命令即可。这会让 wpa_supplicant 进程在后台运行

```
wpa_supplicant -B -c/etc/wpa_supplicant.conf -iwlan
```

> -B ： 以 daemon 形式在后台运行。-c 指定配置文件。-i 指定使用的接口名称

如果想要调试问题，可以用下面的命令来让它在前台运行

```
wpa_supplicant -c/etc/wpa_supplicant.conf -iwlan0 -d
```

> -d ：增加 debug 输出。-dd 可以输出更加详细的日志信息

wpa_supplicant 可以通过多次运行的方式启动多个接口，也可以通过传入列表选项的方式启动多个接口。每个接口用 -N 选项进行分割。下面的例子使用一个命令启动了两个接口

```
wpa_supplicant \
        -c wpa1.conf -i wlan0 -D nl80211 -N \
        -c wpa2.conf -i wlan1 -D wext
```

如果 wpa_supplicant 要运行的接口未知或暂时不存在，wpa_supplicant 会等接口出现时再匹配运行。

## 配置文件

wpa_supplicant 使用文本文件进行配置，该配置文件中列出了所有接受连接的网络和安全策略，也包含预分享密钥。一个配置文件可以包含多个网络，wpa_supplicant 会按照顺序，综合网络安全等级（更偏向于选择 WPA/WPA2）、信号强度来选择最佳的网络。下面给出几个例子

1）WPA-Persional （PSK）家庭网络，使用 EAP-TLS 的 WPA-Enterprise 工作网络

```
# 允许所有在 'wheel' 用户组中的用户使用前端程序 (比如 wpa_cli) 
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=wheel
#
# 家庭网络; 允许使用所有的有效密文
network={
	ssid="home"
	scan_ssid=1
	key_mgmt=WPA-PSK
	psk="very secret passphrase"
}
#
# 工作网络; 使用带有 WPA 的 EAP-TLS; 仅允许 CCMP 和 TKIP 密文
network={
	ssid="work"
	scan_ssid=1
	key_mgmt=WPA-EAP
	pairwise=CCMP TKIP
	group=CCMP TKIP
	eap=TLS
	identity="user@example.com"
	ca_cert="/etc/cert/ca.pem"
	client_cert="/etc/cert/user.pem"
	private_key="/etc/cert/user.prv"
	private_key_passwd="password"
}
```

2）使用 old peaplabel （例如 Funk Odyssey and SBR, Meetinghouse Aegis, Interlink RAD-Series ） RADIUS 服务器的 WPA-RADIUS/EAP-PEAP/MSCHAPv2

```
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=wheel
network={
	ssid="example"
	scan_ssid=1
	key_mgmt=WPA-EAP
	eap=PEAP
	identity="user@example.com"
	password="foobar"
	ca_cert="/etc/cert/ca.pem"
	phase1="peaplabel=0"
	phase2="auth=MSCHAPV2"
}
```

3）包含用于非加密通信的匿名身份的 EAP-TTLS/EAP-MD5-Challenge 配置文件。实际身份只会在使用 TLS 加密通道时使用

```
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=wheel
network={
	ssid="example"
	scan_ssid=1
	key_mgmt=WPA-EAP
	eap=TTLS
	identity="user@example.com"
	anonymous_identity="anonymous@example.com"
	password="foobar"
	ca_cert="/etc/cert/ca.pem"
	phase2="auth=MD5"
}
```

4）使用动态 WEP 密钥（单播和广播都需要）的 IEEE 802.1X（也就是非 WPA）; 认证使用 EAP-TLS

```
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=wheel
network={
	ssid="1x-test"
	scan_ssid=1
	key_mgmt=IEEE8021X
	eap=TLS
	identity="user@example.com"
	ca_cert="/etc/cert/ca.pem"
	client_cert="/etc/cert/user.pem"
	private_key="/etc/cert/user.prv"
	private_key_passwd="password"
	eapol_flags=3
}
```

5）认证有线网络

```
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=wheel
ap_scan=0
network={
	key_mgmt=IEEE8021X
	eap=MD5
	identity="user"
	password="password"
	eapol_flags=0
}
```

## 动态添加接口以及不使用配置文件进行操作

wpa_supplicant 也可以在没有配置文件、也没有网络接口的情况下使用。这种情况下会工作一个全局（即一个 wpa_supplicant 进程）控制接口来添加和移除网络接口。然后可以通过这个控制接口来配置网络接口。下面的例子演示了如何在没有网络接口的情况下启动 wpa_supplicant 、如何添加网络接口并配置网络（SSID）：

```
# 在后台启动 wpa_supplicant
wpa_supplicant -g/var/run/wpa_supplicant-global -B

# 添加一个新的接口 (wlan0, 无配置文件, 驱动=nl80211, 启用控制接口)
wpa_cli -g/var/run/wpa_supplicant-global interface_add wlan0 \
	"" nl80211 /var/run/wpa_supplicant

# 为刚刚添加的网络接口配置网络
wpa_cli -iwlan0 add_network
wpa_cli -iwlan0 set_network 0 ssid '"test"'
wpa_cli -iwlan0 set_network 0 key_mgmt WPA-PSK
wpa_cli -iwlan0 set_network 0 psk '"12345678"'
wpa_cli -iwlan0 set_network 0 pairwise TKIP
wpa_cli -iwlan0 set_network 0 group TKIP
wpa_cli -iwlan0 set_network 0 proto WPA
wpa_cli -iwlan0 enable_network 0

# 到这里，新的网络接口就应该开始尝试使用 SSID test 关联 WPA-PSK 网络了

# 移除网络接口
wpa_cli -g/var/run/wpa_supplicant-global interface_remove wlan0
```





下面介绍 wpa_supplicant 中有关 Linux 的部分，更详细的说明请参阅 [wpa_supplicant 主页](http://w1.fi/wpa_supplicant/) 、详细的 [readme](https://w1.fi/cgit/hostap/tree/wpa_supplicant/README) 以及包含丰富注释的 [配置文件](https://w1.fi/cgit/hostap/tree/wpa_supplicant/wpa_supplicant.conf)

## 支持的 Linux 无线网卡 / 驱动

- 支持包含 WPA/WPA2 扩展的 [Linux Wireless Extensions](https://wireless.wiki.kernel.org/en/developers/documentation/wireless-extensions) v19 或更新版本的 Linux 驱动
- 所有的 Linux mac80211 驱动
- Prism2/2.5/3 （WPA 和 WPA2） Host AP 驱动
- 使用 Linuxant DriverLoader 加载的支持 WPA/WPA2 的 Windows NDIS 驱动
- Agere Systems Inc. Linux Driver （Hermes-I/Hermes--II 芯片组）（仅 WPA，不支持 WPA2）
- madwifi（Atheros ar521x）
- ATMEL AT76C5XXx
- Linux ndiswrapper
- Broadcom wl.o 驱动
- 有线以太网驱动

## 设置监管区域

可以通过在 wpa_supplicant 配置文件下面添加一行 ISO/IEC  3116 地区代码的方式来进行配置。下面的这一行示例可用于 0.6.7 版本

```
country=US
```

## 启用控制接口和 nl80211 驱动

```
-o<driver> 和 -O<ctrl>
```

上面这两个选项可以用于在添加接口时覆盖来自 dbus 或全局 ctl_interface 的参数。这可以用于在使用 NetworkManager 或 Connman 修改 wpa_supplicant 所用的驱动时，启用指定的控制接口。

wpa_cli 和 wpa_gui 也使用 wpa_supplicant 控制接口来控制 wpa_supplicant。GUI 应用通过 DBUS 服务文件来使用特定的参数运行 wpa_supplicant。你可以手动编辑该文件以使用新的功能

wpa_supplicant 服务文件通常存在下面的目录中

```
/usr/share/dbus-1/system-services/fi.epitest.hostap.WPASupplicant.service
```

文件内容通常是这样的

```
[D-BUS Service]
Name=fi.epitest.hostap.WPASupplicant
Exec=/sbin/wpa_supplicant -u -f /var/log/wpa_supplicant.log
User=root
```

将该文件修改为下面这样，可以让 wpa_cli 或 wpa_gui 在可用时使用 nl80211:

```
[D-BUS Service]
Name=fi.epitest.hostap.WPASupplicant
Exec=/sbin/wpa_supplicant -u -onl80211 -O/var/run/wpa_supplicant
User=root
```

## WPS 和 WEP

Wi-Fi Protected Setup （WPS，曾经的 Wi-Fi Simple Config, 或 WSC） 2.0 规范添加了不允许使用 WEP 的要求。按照该要求编译的 wpa_supplicant 会在 WPS 2.0 启用的情况下拒绝 WEP 网络。此外还应注意，WEP 和 WPS 1.0 的交互上也有一系列的问题，因为这种混用方式从来没有在 WFA 认证程序中测试过

## WPS 和隐藏 SSID

隐藏 SSID 无法和任何版本的 WPS 共用，该协议无法和隐藏 SSID 共用

## RSN 预认证（preauthentication）

请参阅 [hostapd 的 RSN 预认证文档](https://wireless.wiki.kernel.org/en/users/documentation/hostapd) 了解对于 AP 配置的要求。若 AP 是使用 RSN 预认证正确配置的，且使用了 WPA2 网络，则 wpa_supplicant 会默认启用该特性。

