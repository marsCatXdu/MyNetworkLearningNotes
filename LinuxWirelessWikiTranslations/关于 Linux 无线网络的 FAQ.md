本文列出了一些关于 Linux 无线网络的最常见问题

## Q：系统对我的 xxx 型号设备是否有支持？

先查询一下设备的 ID 对：VID:PID ，把这个数对记下来

PCI 设备：

```
lspci -nn
```

USB 设备：

```
lsusb
```

然后到 Google ，把下面这个贴在搜索框中，用你刚刚查出来的 VID 和 PID 换掉下面的 VID 和 PID：

```
"VID PID" site:cateee.net/lkddb/
```

比如

```
"0cf3 9271" site:cateee.net/lkddb/
```

搜索结果会告诉你该用哪个驱动。如果并没有搜到结果，那可能就是不支持你这个设备了。如果你真的确定应该有驱动支持你的设备，那么可以试着手动修改驱动的代码把你自己的 ID 加上去，如果是 USB 设备，还可以用 `newid` sysfs node。



## Q：设备运行还是有问题

按照下面挨个试：

1. 安装最新的驱动 [Driver Backports Wiki (kernel.org)](https://backports.wiki.kernel.org/index.php/Main_Page)
2. 到 [驱动](https://wireless.wiki.kernel.org/en/users/drivers) 页面了解你的设备的限制以及额外需求
3. 仔细阅读 `dmesg` 的输出，认真看与固件（firmware）相关的消息
4. 使用 `rfkill list` 看看有没有与 [rfkill](https://wireless.wiki.kernel.org/en/users/documentation/rfkill) 相关的问题。这里面应该不能有任何和你的设备相关的 block。有些时候设备上可能会有一个硬件机械按钮，来切换控制 "hard" block
5. 试试直接使用 [iw](https://wireless.wiki.kernel.org/en/users/documentation/iw) 和 [wpa_supplicant](https://wireless.wiki.kernel.org/en/users/documentation/wpa_supplicant)，而不是 [NetworkManager](https://wireless.wiki.kernel.org/en/users/documentation/networkmanager) 之类的工具
6. 试试安装 [bleeding edge backports](http://drvbp1.linux-foundation.org/~mcgrof/rel-html/backports/) 



## Q：iwconfig 报错 xxxxx

直接使用 [iw](https://wireless.wiki.kernel.org/en/users/documentation/iw) 和 [wpa_supplicant](https://wireless.wiki.kernel.org/en/users/documentation/wpa_supplicant)

iwconfig 和内核进行通信的方式已经过时了，仅能使用兼容模式提供的有限支持，而且未来可能会被完全移除



## Q：iwconfig wlan0 模式 master 不工作

使用 [hostapd](https://wireless.wiki.kernel.org/en/users/documentation/hostapd) 和支持 AP 模式的驱动来运行 master/AP 模式。现代驱动已经不支持使用 iwconfig 设置 master 模式。



## Q：为什么不能用 channel X，已经运行过 iw reg set NN 了但还是不可用

每个网卡都要根据环境限制（包含对于信道、最大功率以及其他特殊标志的一组限制）相关的规定工作。Intel 网卡的相关限制由固件实现，Atheros 则在 EEPROM 中保存了一个注册区域码（regdomain code），该值会在驱动启动时读取，然后联系 [CRDA](https://wireless.wiki.kernel.org/en/developers/regulatory) 获取限制要求。



## Q：我从其他的某些地方安装了驱动，现在我这里显示了一个 ra0 interface 等等...

如果使用了直接由厂商提供的驱动（而非 Linux 上游或其他的兼容无线驱动）请直接联系厂商。这不归 linux-wireless 管

