---
title: Roome 室友小易 - 设备与通信协议
date: 2021-08-28 14:29:31
tags: 
- Hardware
- Multimedia
categories:
- Hardware
toc: true
---
给（厂商已经倒闭的）Roome 室友小易添加HTTP API、DLNA投放等更多功能，让它比原厂更强大。本节对设备的基础信息和通信协议进行了整理。

<!-- more -->

# 硬件信息
此处参考了[云中漫步的拆解文章](https://www.sohu.com/a/239062255_596101)，主要硬件信息如下：

| 组件            | 型号            | 说明                                              |
| --------------- | --------------- | ------------------------------------------------- |
| CPU             | Mstar MSV5263   | ARMv7 with NEON，无 SDK 和数据手册                |
| RAM             |                 | 128MiB                                            |
| Flash           | TC58NVG0S3HTA00 | 1Gb (128MiB), Raw NAND                            |
| Bluetooth Audio | JL AC1819AP     | 只具有音频功能，无法进行BLE通信，通过 AT 指令控制 |
| PIR             | BISS0001        | 热释电红外开关                                    |
| WiFi            | MT7601          | 仅 2.4 GHz，USB连接，性能一般                     |
| 数码管驱动      | FZH216          | GPIO 控制                                         |
| 触摸主控        | TP224B          |                                                   |
| 光线传感器      |                 |                                                   |

# 原厂系统
配网时或配网后，均可直接 Telnet 连接，用户名 `root`，无密码。
```
~ # export PATH="/bin:$PATH"

~ # mount
rootfs on / type rootfs (ro)
tmpfs on /dev type tmpfs (rw,nosuid,relatime,mode=755)
devpts on /dev/pts type devpts (rw,relatime,mode=600,ptmxmode=000)
proc on /proc type proc (rw,relatime)
sysfs on /sys type sysfs (rw,relatime)
tmpfs on /mnt/usb type tmpfs (rw,relatime,mode=755,gid=1000)
ubi0:SYSTEM on /system type ubifs (rw,nosuid,nodev,relatime)
ubi0:APP on /app type ubifs (rw,nosuid,nodev,relatime)
ubi1:CONFIG on /config type ubifs (rw,nosuid,nodev,relatime)
ubi1:DATA on /data type ubifs (rw,nosuid,nodev,relatime)
tmpfs on /tmp type tmpfs (rw,relatime)
tmpfs on /var type tmpfs (rw,relatime)
none on /config/adb_config type configfs (rw,relatime)
adb on /dev/usb-ffs/adb type functionfs (rw,relatime)

~ # cat /proc/mtd
dev:    size   erasesize  name
mtd0: 00060000 00020000 "IPL0"
mtd1: 00060000 00020000 "IPL1"
mtd2: 00060000 00020000 "IPL_CUST0"
mtd3: 00060000 00020000 "IPL_CUST1"
mtd4: 00140000 00020000 "UBOOT0"
mtd5: 00140000 00020000 "UBOOT1"
mtd6: 00560000 00020000 "OS"
mtd7: 00a80000 00020000 "OS_BAK"
mtd8: 00440000 00020000 "FACTORY"
mtd9: 00100000 00020000 "ENV"
mtd10: 032a0000 00020000 "UBI0"
mtd11: 03300000 00020000 "UBI1"

~ # uname -a
Linux (none) 4.9.27 #1 SMP PREEMPT Thu May 24 22:29:19 HKT 2018 armv7l GNU/Linux

~ # lsmod
mt7601Usta 1518871 1 - Live 0xbf074000 (O)
mtprealloc 2058 1 mt7601Usta, Live 0xbf070000 (PO)
cfg80211 219982 1 mt7601Usta, Live 0xbf02e000
firmware_class 10473 0 - Live 0xbf027000
hw_device_id 1544 0 - Live 0xbf023000 (O)
snd_aloop 12129 0 - Live 0xbf01c000
asix 21218 0 - Live 0xbf012000
usbnet 17735 1 asix, Live 0xbf009000
mii 3712 2 asix,usbnet, Live 0xbf005000
ffph_auth 4604 0 - Live 0xbf000000 
```

# 软件架构
部分基于 Android 的 Linux 系统，核心主进程为 `/app/bin/aiAudioMain`，加载 `/app/lib` 中的如下动态库：
* `libffmpeg.so` 几个 ffmpeg 库静态链接到一起
* `libhomi.so` 几乎所有的主要逻辑
* `libjansson.so` JSON 解析库
* `libmi.so` 设备驱动接口
* `libplayer.so` 音频播放逻辑
* `libtinyalsa.so` 底层音频驱动

## 配置文件
* `/config/config-clock-user.json` 设备配置文件
* `/data/wifi.conf` WiFi 配网信息（用户名及密码）

# 配网流程
连接设备发射的网络后，发送包含 SSID 和密码的 UDP 数据包即可配网，具体流程待补充。

# 设备连接流程
## HTTPS 认证，获取 Token
配网完成后，设备访问
```
https://io.myroome.com/authDevice?...
```
以获得 Access Token 以及后续 TCP 长连接的地址，所有的 HTTPS 请求均不验证证书，可直接劫持。
```json
{
    "access_token": "aaaaaaaaaaaaaaaa", // 至少 10 字节，后续连接时传入
    "ssid": "Test_SSID1234", // 至少 10 字节，没什么用
    "server": "127.0.0.100" // 至少 10 字节，所以不能使用 127.0.0.1，TCP 连接地址
}
```
## TCP 连接
设备通过 TCP 连接 `server:9898`，并使用 JSON 进行通信（无分隔符号，只能自己数花括号了）。

# 通信协议
## 基础结构
```json
{"k":[1, 2],"v":[0, {}]}
```
* `k` 命令或消息的类型，数值
* `v` 命令或消息对应的数据，任意 JSON
`k` 和 `v` 按顺序一一对应，可以一次传输多个

## 设备 - 服务器
```json
{"k": [10], "v": [0],"t":"12294ffb712edfe1489abcefbe89d82","c":200}
```
* `t` MD5 字符串，与消息中的 `k` 有关
* `c` 返回值，仅存在于对服务器请求的回复中，语义类似 HTTP

对服务器请求的回复均为整数（布尔类型也会转换成整数），`k` 与请求时的值相同。
### 2 - 心跳
```json
{"u": 1101,"l":2,"f":88719630,"s":57344}
```
## 服务器 - 设备
无其他扩展字段。此处仅列出部分指令，其余请自行反编译 `libhomi.so` 中的 `conn_process_command_request` 函数查看。

| `k` | `v`               | 返回值            | 说明               | 函数                          |
| --- | ----------------- | ----------------- | ------------------ | ----------------------------- |
| 9   |                   | Unix 时间戳（秒） | 获取系统时间       | `SysTimeGet`                  |
| 10  | Unix 时间戳（秒） |                   | 设置系统时间       | `SysTimeSet`                  |
| 163 |                   | bool              | 获取自动屏幕对比度 | `AutoScreenContrastEnableGet` |
| 164 | bool              |                   | 设置自动屏幕对比度 | `AutoScreenContrastEnableSet` |
| 165 |                   | int 0-100         | 获取屏幕对比度     | `ScreenContrastGet`           |
| 166 | int 0-100         |                   | 设置屏幕对比度     | `ScreenContrastSet`           |
| 167 |                   | bool              | 获取自动灯光亮度   | `AutoLampBrightnessEnableGet` |
| 168 | bool              |                   | 设置自动灯光亮度   | `AutoLampBrightnessEnableSet` |
| 169 |                   | int 0-100         | 获取灯光亮度       | `LampBrightnessGet`           |
| 170 | int 0-100         |                   | 设置灯光亮度       | `LampBrightnessSet`           |
| 181 |                   | bool              | 获取灯光开关       | `LampOnGet`                   |
| 182 | bool              |                   | 设置灯光开关       | `LampOnGet`                   |

### 202 - 播放 URL
```json
{"url": "url", "time": "string_unused", "keep": 0}
```
仅 URL 参数被实际使用（长度至少 20 字节），但是其他的也需要提供，开始播放后不提供远程控制功能，如需停止可传入20字节的非 URL 字符串。