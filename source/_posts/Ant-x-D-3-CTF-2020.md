---
title: Ant x D^3 CTF 2021
date: 2021-03-10 19:55:05
tags: CTF
categories:
- CTF
---
JavaScript 真好玩。

<!-- more -->

# Web
## 8-bit Pub
用邮件附件的方式可以实现任意读取文件，支持 `[{key: "X-Key-Name", value: "val1"}, {key: "X-Key-Name", value: "val2"}]` 格式的自定义header。

由于使用了 `shvl`，可以实现原型链污染。
```js
const shvl = require('shvl');
var obj = {}
console.log("Before : " + obj.isAdmin);
shvl.set(obj, 'constructor.prototype.isAdmin', true);
console.log("After : " + obj.isAdmin);
```

# Misc
## Robust
{% asset_img upload_2cbbaa133a5219b9d6b67380028c8184.png %}
一组 QUIC/HTTP3 数据包，解密后导出为 JSON，可以用 Python 脚本进行解码
```python
import json
import binascii
import pylsqpack, pprint

decoder_srv = pylsqpack.Decoder(0x1000, 0x100)
decoder_cli = pylsqpack.Decoder(0x1000, 0x100)

pcap = json.load(open("h3.json", "r", encoding="utf-8"))

streams = {}

for packet in pcap[5:]:
  layers = packet['_source']['layers']
  src_port = int(layers['udp']['udp.srcport'])
  stream_id = int(layers['quic']['quic.frame']['quic.stream.stream_id'])
  frame_type = int(layers['http3']['http3.frame_type'])

  stream = streams.get(stream_id, None)
  if stream is None:
    stream = {
      'req_headers': {},
      'res_headers': {},
      'res_data': {}
    }
    streams[stream_id] = stream
  
  if frame_type == 1:
    header_payload = binascii.unhexlify(layers['http3']['http3.frame_payload'].replace(':', ""))
    if src_port == 8443:
      headers = decoder_srv.feed_header(stream_id, header_payload)[1]
      for k, v in headers:
        stream['res_headers'][k.decode('ascii')] = v.decode('ascii')
    else:
      headers = decoder_cli.feed_header(stream_id, header_payload)[1]
      for k, v in headers:
        stream['req_headers'][k.decode('ascii')] = v.decode('ascii')
  if frame_type == 0 and src_port == 8443:
    payload = binascii.unhexlify(layers['http3']['http3.frame_payload'].replace(':', ""))
    pkn = int(layers['quic']['quic.short']['quic.packet_number'])
    stream['res_data'][pkn] = payload

  
for i in streams:
  stream = streams[i]
  path = "files/" + stream['req_headers'][':path'].replace("/", "_")
  parts = list(stream['res_data'].keys())
  parts.sort()
  with open(path, "wb") as f:
    for part_i in parts:
      f.write(stream['res_data'][part_i])
```
解码后可以得到一个加密的 HLS 码流，播放列表中也指定了解密用的密钥，下载到同一文件夹下后直接用 `ffmpeg` 拼接即可。（注：由于 `ffmpeg` 的安全设置，可能不能直接加载密钥，可将密钥改名 `key.mp4` 后进行处理，不影响解密操作）。

解码后的音频是原神同人曲“让风告诉你”，频谱图疑似包含数据。
{% asset_img upload_81d70b9a8d05b5873509253d57a753a8.png %}

根据 Hint，这段数据是用 `libquiet` 编码的，自行编译一份这个应用，使用 `ultrasonic` 模式解码出一段 Base64，内含一段文本和一个与该曲目网易云 ID 相同的 JSON 文件。通过网易云 API 可以获得 CRC 相同的文件，已知明文攻击解出剩余内容，为一段台本，里面有零宽字符隐写。
{% asset_img upload_40f97ad862de6dbfb84ef48441bf27bc.png %}

随便找一个字符库比较全的在线网站提取，发现内容以 `< >` 包围，Base85 解码即可。