---
title: Roome 室友小易 - FFmpeg、Hosts、Hooking
date: 2021-09-04 13:42:16
tags: 
- Hardware
- Multimedia
categories:
- Hardware
toc: true
---
给（厂商已经倒闭的）Roome 室友小易添加HTTP API、DLNA投放等更多功能，让它比原厂更强大。本节对设备中的主程序以及一些依赖库进行了修改，修正了一些原厂的 Bug 以及缺失的功能。

<!-- more -->

# 设备使用的工具链
设备上的 `glibc` 版本为 `2.23`，经过测试，使用 `gcc-linaro-6.5.0-2018.12-x86_64_arm-linux-gnueabihf` [工具链](http://releases.linaro.org/components/toolchain/binaries/6.5-2018.12/arm-linux-gnueabihf/)可以编译出可用的可执行文件和动态库。


# 重新编译 FFmpeg 3.3.4
原厂的 FFmpeg 存在以下的问题：
* 支持的封装格式和编码格式十分有限（封装只支持 `mp3`、`aac` 等极少数格式），不能支持大多数的 DLNA 使用场景（特别是直播）
* HTTPS 无法使用（暂未解决）
* 未开启架构特定的优化，播放部分直播流时卡顿，甚至可能导致整个系统无响应

设备上的 `libffmpeg.so` 是一个由多个静态库组合而成的“缝合怪”，查找了一下资料，在 Android 上也有进行类似的操作，这里参考了编译 JNI 的资料进行合并。除开启架构优化之外，这里还添加了对一部分视频（仅音频部分）和额外音频格式的解析支持，并去掉了无法使用的 HTTPS 组件。

```bash
#!/bin/sh
T_PREFIX=$HOME/devtools/gcc-linaro-6.5.0-2018.12-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-

./configure \
    --prefix=/usr \
    --enable-static \
    --enable-gpl \
    --enable-pic \
    --enable-nonfree \
    --disable-doc --disable-programs --disable-everything --disable-swscale --disable-postproc --disable-debug \
    --enable-decoder=aac,aac_fixed,aac_latm,vorbis,mp3,mp3adu,mp3adufloat,mp3float,mp3on4,mp3on4float,opus,pcm_alaw,pcm_f16le \
    --enable-decoder=pcm_f24le,pcm_f32be,pcm_f32le,pcm_f64be,pcm_f64le,pcm_mulaw,pcm_s16be,pcm_s16be_planar,pcm_s16le,pcm_s16le_planar \
    --enable-decoder=pcm_s24be,pcm_s24daud,pcm_s24le,pcm_s24le_planar,pcm_s32be,pcm_s32le,pcm_s32le_plana,pcm_s64be,pcm_s64le,pcm_s8 \
    --enable-decoder=pcm_s8_planar,pcm_u16be,pcm_u16le,pcm_u24be,pcm_u24le,pcm_u32be,pcm_u32le,pcm_u8,flac \
    --enable-parser=aac,aac_latm,flac,mpegaudio,opus,vorbis \
    --enable-demuxer=aac,ogg,mp3,wav,mpegts,hls,dash,flv \
    --enable-protocol=http,httpproxy,file,cache,async \
    --enable-bsf=aac_adtstoasc \
    --enable-cross-compile \
    --cross-prefix=$T_PREFIX \
    --extra-cflags="-march=armv7-a -mtune=cortex-a7 -mfpu=neon-vfpv4 -mfloat-abi=hard" \
    --enable-neon --arch=arm --cpu=armv7-a --target-os=linux

make -j16

# 这里对编译后的静态库进行了合并，方便后面生成完整的 libffmpeg.so
${T_PREFIX}ar -M << EOF
CREATE libffmpeg.a
ADDLIB libswresample/libswresample.a
ADDLIB libavcodec/libavcodec.a
ADDLIB libavformat/libavformat.a
ADDLIB libavutil/libavutil.a
ADDLIB libavfilter/libavfilter.a
ADDLIB libavdevice/libavdevice.a
SAVE
END
EOF

# 顺序不可改变，`--no-undefined` 确保链接完全
${T_PREFIX}gcc -shared -o libffmpeg.so -lc -lm -lpthread -Wl,--whole-archive -Wl,--no-undefined libffmpeg.a -Wl,--no-whole-archive
# 去除多余符号，否则设备上放不下
${T_PREFIX}strip libffmpeg.so
```

完成后，可以使用如下命令将导出的符号排序后输出到文件，以方便和原始符号对比，找出缺失或多出来的组件。

```bash
objdump -T libffmpeg.so | cut -d ' ' -f 17 | sort > sym-new
```

# `/` 是只读的？没问题
这台设备上的 `/{,bin,etc}` 等目录都是直接放在根文件系统下的（`/app` 分区也不够大），而这个根文件系统是一个 RamDisk，也就是对其中的内容进行的更改并不会在重启时保留。
## Bind Mount
对 Android 上的 Magisk 有了解的同学可能听说过 Bind Mount 这一名称，实际上就是利用 Linux 的 VFS 功能把文件或目录挂载到另外的位置，“覆盖”掉原有的内容，例如，可以通过以下方法在启动时“覆盖”掉原有的 `libffmpeg.so`：
```bash
mount --bind /data/app/lib/libffmpeg.so /app/lib/libffmpeg.so
```

## 创建目标挂载点
Bind Mount 有一个小问题：对于不存在的目标，并不能进行挂载（因为没有这个挂载点），这时可以临时把 `/` 挂载为可读写，创建对应的文件，再返回只读，对 `/` 的更改在重启前将一直有效。
```bash
mount -o remount,rw /
touch /etc/nsswitch.conf
mount --bind /data/nsswitch.conf /etc/nsswitch.conf
mount -o remount,ro /
```

# nsswitch.conf(5) 与 Hosts
`glibc` 使用 `/etc/nsswitch.conf` 文件控制对各类数据库（如域名、用户、用户组、密码等）的查找优先级，如果该文件或对应的子项缺失，则会使用默认值，多数发行版都自带了与默认值不同的配置文件，而在这个设备上并没有，此时 `hosts` 数据库查询的顺序为 DNS 优先，查询无结果才读取 `/etc/hosts`。由于厂家的域名仍未过期（只是后端服务停止），这就导致我们无法正确地覆盖解析，只有创建并重写 `/etc/nsswitch.conf` 了。
```
# /data/nsswitch.conf
passwd: files
group: files
shadow: files
hosts: files dns
```
并添加以下内容到 `/system/etc/init.sh`
```bash
mount -o remount,rw /
touch /etc/nsswitch.conf
mount --bind /data/nsswitch.conf /etc/nsswitch.conf
mount -o remount,ro /
```
Hosts 也可使用上述的 Bind Mount 方式直接覆盖（因为这个文件已经存在）

# 增加新命令 - Hooking
背景知识：[Correct usage of `LD_PRELOAD` for hooking `libc` functions](https://tbrindus.ca/correct-ld-preload-hooking-libc/)

TODO