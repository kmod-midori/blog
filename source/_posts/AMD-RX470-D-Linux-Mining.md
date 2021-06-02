---
title: AMD RX 470(D) Linux 挖矿
date: 2021-03-10 16:12:03
tags: Linux
---
打不过矿老板怎么办？那当然是自己加入了！

本文记载了将两年前 300 元购入的 XFX AMD RX 470D 投入锻炼的全过程，因为要兼顾日用，系统是 Manjaro。

<!-- more -->

## 固件 & 开核
### 取得 VBIOS
Polaris 系列显卡的 VBIOS 都是可以互刷的，前提是内存型号和大小一致，于是我们可以把 RX 470D 刷成 RX 470，解锁更高性能。

在 https://www.techpowerup.com/vgabios/ 这个页面上搜索厂商和显卡型号，选一个和自己的显卡最接近的，我选择了[这个](https://www.techpowerup.com/vgabios/187196/xfx-rx470-4096-160913)。注意核对显存厂商，自己的显卡的信息可以从 GPU-Z 取得，或手动 dump 原始 VBIOS 后查看。
```
Limits
  TDP: 85 W
  TDC Power: 76 A
  Battery Power: 87 W
  Small Power Power: 87 W
  Max. Power Limit: 87 W
  Max. Temp: 90°C
Memory Support
  4096 MB, GDDR5, Elpida EDW4032BABG
```
### 刷写 VBIOS
Linux下的 `amdvbflash` 刷写工具可以从[这里](https://www.techpowerup.com/download/ati-atiflash/)取得。

备份当前固件：
```shell
sudo amdvbflash -s 0 vbios.rom
```

刷写新固件：
```shell
sudo amdvbflash -f -p 0 new_vbios.rom
```
刷写成功之后，重启主机。

## 解锁功耗 & 超频
参考 [Arch Wiki 上的超频教程](https://wiki.archlinux.org/title/AMDGPU#Overclocking)，添加内核参数并使用 [amdgpu-clocks](https://aur.archlinux.org/packages/amdgpu-clocks-git/) 自动应用设置。

以下是我个人的超频设置，每张卡都可能不一样，挖不同的币种的时候也可能需要微调，建议参考挖矿软件的官方文档进行调整。
```
# /etc/default/amdgpu-custom-states.card0
# Set custom GPU states 5 & 6 & 7:
OD_SCLK:
3:       1150MHz        900mV
4:       1150MHz        900mV
5:       1150MHz        900mV
6:       1150MHz        900mV
7:       1150MHz        900mV
# Set custom memory states 2:
OD_MCLK:
1:       1950MHz        900mV

FORCE_PERF_LEVEL: manual
FORCE_SCLK: 7
```

## OpenCL
Arch/Manjaro 系统中有三种 OpenCL Runtime：

* `opencl-mesa` 最常用的开源驱动
* `opencl-amd` 来自 AMD 的闭源驱动，从 Ubuntu deb 解包，可以从 AUR 获得
* `rocm-opencl-runtime` ROCm 开源驱动，可以从 AUR 或者 arch4edu 获得

三种驱动原理上可互相替换，但不同的挖矿软件可能存在各种兼容性问题，目前测试下来效果最好的是 `opencl-amd`，推荐使用。

## 挖矿
推荐使用 [teamredminer](https://github.com/todxx/teamredminer)，提供了很多实用的功能，性能也很不错。矿池请自行选择。

有部分算法可能需要用到 `amdmemtweak`，建议自行参考资料。

### KawPow (RVN)
整体功耗要求较高（可能会达到功耗墙），但由于币值比较低，每天都可以达到起付额。

### Etchash (ETC)
比较类似旧的 ETH 算法，目前币值较高，大约一周左右可以达到起付。

### Autolykos2 (Ergo)
对硬件整体要求不高（似乎是 1.5G VRAM 即可），功耗也在可接受范围内，但是这币实在是太山寨了，不是很好找到矿池和交易所。

可以使用 https://www.gate.io/ 和 https://www.666pool.cn/pool2/ 进行交易和挖矿。

以下是个人的 Ergo 配置，加入了 API 和风扇控制相关的参数，控制风扇需要 root 权限。
```bash
./amdmemtweak -i 0 --ref 20
./teamredminer -a autolykos2 -o stratum+tcp://ergo.666pool.cn:9556 -u <ADDR>.rig0 -p x --fan_control=68 --watchdog_script --api_listen=127.0.0.1:4028 --dev_location=cn
```

可以把这个脚本用 systemd 启动，防止设备意外重启后不继续运行。

## 监控
对于各种有关参数，可以用 [sampler](https://github.com/sqshq/sampler) 在命令行进行监控。

```yaml
# sampler.yml
runcharts:
  - title: Ergo Balance
    position: [[0, 12], [40, 20]]
    rate-ms: 60000
    legend:
        enabled: true
        details: true
    scale: 10
    items:
      - label: ERG
        sample: curl https://www.666pool.cn/pool2/main/ERGO/<ADDR>
            -ks | sed -n 438p | awk '{print $1}'
  - title: Ergo Rate
    position: [[40, 12], [40, 20]]
    rate-ms: 5000
    legend:
        enabled: true
        details: true
    scale: 2
    items:
      - label: Ergo (kH/s)
        sample: "echo '{\"command\": \"summary\"}' | nc 127.0.0.1 4028 | jq '.SUMMARY[0].\"KHS 30s\"'"
  - title: Temps
    position: [[0, 0], [80, 12]]
    rate-ms: 1000
    legend:
        enabled: true
        details: true
    scale: 2
    items:
      - label: GPU
        sample: sensors -j amdgpu-pci-2d00 | jq '."amdgpu-pci-2d00".edge.temp1_input'
      - label: CPU
        sample: "sensors -j nct6797-isa-0a20 | jq '.\"nct6797-isa-0a20\".\"SMBUSMASTER 0\".temp7_input'"
            
  - title: Fans
    rate-ms: 1000
    legend:
        enabled: true
        details: true
    scale: 2
    items:
      - label: GPU
        sample: sensors -j amdgpu-pci-2d00 | jq '."amdgpu-pci-2d00".fan1.fan1_input'
      - label: CPU
        sample: sensors -j nct6797-isa-0a20 | jq '."nct6797-isa-0a20".fan2.fan2_input'
textboxes:
  - title: Ergo Accepted
    position: [[0, 32], [80, 8]]
    rate-ms: 10000
    sample: "echo '{\"command\": \"summary\"}' | nc 127.0.0.1 4028 | jq '.SUMMARY[0].Accepted'"
```