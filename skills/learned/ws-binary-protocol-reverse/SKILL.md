---
name: ws-binary-protocol-reverse
description: WebSocket 二进制协议逆向。Protobuf 手工编解码(无.proto)、AES-CBC 加密帧、Cookie->Token->WS 认证链、IM 消息收发 Python 复现。当目标使用 WebSocket 传输二进制/Protobuf 数据时使用。
---

# WebSocket 二进制协议逆向

执行时遵循 **reverse-auto-execute**：实际执行、反复测试、根据报错诊断重试。

## 触发场景

- 目标站点使用 WebSocket 传输二进制数据（ArrayBuffer/Uint8Array）
- 帧内容非 JSON 而是 Protobuf、自定义二进制格式
- WebSocket 帧有加密层（AES-CBC/AES-GCM 等）
- 需要脱离浏览器，用 Python/Node 直接收发消息

## 逆向流程

```
1. SunnyNet/DevTools 抓 WebSocket 帧 (二进制)
2. 分析帧头结构 (魔数/版本/长度字段)
3. 在 iframe/Webpack 中定位 SDK 模块 (Protobuf 定义、加密逻辑)
4. 还原 Protobuf 字段映射 (手工或 .proto)
5. 还原加密算法 (密钥来源、IV 生成)
6. Python 实现: 编解码 + 加解密 + 连接注册 + 心跳 + 收发
7. SunnyNet 抓配套 HTTP API (转移/状态等业务接口)
```

## 帧结构分析

### 典型二进制帧格式 (以 KLink 为例)

```
[Tag:2字节][Version:2字节][HeaderLen:4字节][PayloadLen:4字节][Header][Payload]
```

- **Tag**: 魔数标识，如 `0xABCD`
- **Version**: 协议版本
- **HeaderLen/PayloadLen**: 大端序 uint32
- **Header**: Protobuf 编码的 PacketHeader（含 appId/uid/加密模式/seqId/serviceToken）
- **Payload**: 加密后的 Protobuf 消息体

### 识别方法

1. DevTools -> Network -> WS -> 看 Binary Message
2. 前几字节有固定魔数 -> 帧头
3. JS 搜索魔数值（如 `0xABCD`、`43981`）定位编解码函数
4. 搜索 `struct.pack`/`DataView`/`setUint16`/`getUint32` 等关键词

## Protobuf 手工编解码 (无 .proto)

当目标使用 Protobuf 但无 .proto 文件时，手工实现编解码：

### 编码

```python
def encode_varint(value):
    result = bytearray()
    if value < 0:
        value = value + (1 << 64)
    while value > 0x7F:
        result.append((value & 0x7F) | 0x80)
        value >>= 7
    result.append(value & 0x7F)
    return bytes(result)

def encode_field(field_number, wire_type, data):
    tag = (field_number << 3) | wire_type
    return encode_varint(tag) + data

def encode_varint_field(fnum, value):
    if value == 0: return b''
    return encode_field(fnum, 0, encode_varint(value))

def encode_bytes_field(fnum, value):
    if not value: return b''
    if isinstance(value, str): value = value.encode('utf-8')
    return encode_field(fnum, 2, encode_varint(len(value)) + value)

def encode_message_field(fnum, msg_bytes):
    return encode_bytes_field(fnum, msg_bytes)
```

### 解码

```python
def decode_varint(data, offset=0):
    result, shift = 0, 0
    while offset < len(data):
        byte = data[offset]
        result |= (byte & 0x7F) << shift
        offset += 1
        if not (byte & 0x80): break
        shift += 7
    return result, offset

def decode_fields(data):
    fields = {}
    offset = 0
    while offset < len(data):
        tag, offset = decode_varint(data, offset)
        fnum, wtype = tag >> 3, tag & 0x07
        if wtype == 0:
            value, offset = decode_varint(data, offset)
            fields[fnum] = ('varint', value)
        elif wtype == 2:
            length, offset = decode_varint(data, offset)
            fields[fnum] = ('bytes', data[offset:offset+length])
            offset += length
        elif wtype == 1:
            fields[fnum] = ('64bit', data[offset:offset+8])
            offset += 8
        elif wtype == 5:
            fields[fnum] = ('32bit', data[offset:offset+4])
            offset += 4
    return fields
```

### 字段映射还原

1. **JS 搜索 Protobuf 定义**: 搜 `field_number`、`LABEL_OPTIONAL`、`TYPE_STRING` 等
2. **Webpack 模块提取**: 在 iframe console 中 `webpackChunk` 或 `__webpack_require__` 拿到模块
3. **对比二进制与 JS 字段定义**: 用 decode_fields 解码抓到的帧，和 JS 中的字段编号对照

## AES-CBC 加密帧

### 常见模式

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad, unpad
import os, base64

def aes_cbc_encrypt(plaintext, key):
    iv = os.urandom(16)
    cipher = AES.new(key, AES.MODE_CBC, iv)
    return iv + cipher.encrypt(pad(plaintext, AES.block_size))

def aes_cbc_decrypt(ciphertext, key):
    iv, data = ciphertext[:16], ciphertext[16:]
    cipher = AES.new(key, AES.MODE_CBC, iv)
    return unpad(cipher.decrypt(data), AES.block_size)
```

### 密钥来源

| 阶段 | 密钥 | 来源 |
|------|------|------|
| 注册(认证) | ssecurity | HTTP 接口返回，base64 编码的 AES key |
| 注册后 | sessionKey | 注册响应中服务端返回的会话密钥 |

JS 中搜索 `ssecurity`、`sessionKey`、`AES`、`encrypt`、`decrypt` 定位密钥逻辑。

## Cookie -> Token -> WebSocket 认证链

### 典型流程 (以快手客服为例)

```
1. 浏览器登录获取 Cookie (kshop.api_st 等)
2. HTTP GET /token/get  ->  返回 {imId, token, security}
3. WebSocket connect wss://klink-merchant.kwaizt.com/
4. 发送 Basic.Register (Protobuf, AES-CBC 加密, 用 ssecurity)
5. 收到注册响应 -> 提取 sessionKey + instanceId
6. 后续消息用 sessionKey 加密 (EncryptionMode=2)
7. 定时发送 KeepAlive 心跳 (50s 间隔)
```

### Python 实现要点

```python
import websockets, asyncio

async def connect_and_register():
    ws = await websockets.connect(WS_URL, origin="https://...", max_size=2**20)

    # 构建注册包: Header(Protobuf) + Payload(Protobuf, AES加密)
    register_data = encode_register_request(app_info, device_info, zt_info)
    upstream = encode_upstream_payload("Basic.Register", seq_id=1, payload_data=register_data)
    encrypted = aes_cbc_encrypt(upstream, ssecurity_key)
    header = encode_packet_header(uid=user_id, encryption_mode=1, service_token=token_bytes)
    frame = encode_frame(header, encrypted)

    await ws.send(frame)
    resp_raw = await ws.recv()
    # 解码响应 -> 提取 sessionKey
```

## 消息收发

### 发送文本消息

```python
# Message 的 Protobuf 字段:
# field 2: clientSeqId (int64)
# field 3: seqId (= clientTimestampMs)
# field 4: fromUser {appId, uid}
# field 6: toUser {appId, uid}
# field 7: title (str)
# field 8: contentType (0=文本)
# field 9: content (嵌套 Protobuf, field1=text)
# field 18: strTargetId (str, 目标用户ID)
# field 22: extra (业务扩展字段)
# field 27: clientTimestampMs
```

### 接收推送消息

- 推送命令: `Push.Message`、`Push.SyncMessage`、`Push.DataUpdate`
- payload 是 Protobuf，repeated Message (field 1) 包含多条消息
- 需要 `decode_fields_multi` 处理同一字段多次出现的情况
- 过滤自己发的消息: `fromUser.uid == self.user_id`

### 常见 errorCode

| errorCode | 说明 |
|-----------|------|
| 0 | 成功 |
| 10018 | 未登录 / Token 过期 |
| 24106 | 会话未激活（买家未发过消息） |
| 82002 | 消息格式错误 |

## 配套 HTTP API 抓取

IM 系统通常有配套的 HTTP 管理接口（转移客户、设置状态、查会话列表等），用 SunnyNet 抓取：

1. `sunnynet request_search` 按 URL 关键词搜索
2. `sunnynet request_get` 查看完整请求体/响应体
3. HTTP API 通常无加密，Cookie 鉴权 + JSON body

## Webpack 模块提取技巧 (iframe 中的 SDK)

很多 IM SDK 运行在 iframe 里，主页面无法直接访问：

1. **jshook 连 iframe**: DevTools 切换执行上下文到 iframe
2. **搜索模块 ID**: `webpackChunk.push([[999], {999: (m,e,r) => { window._r = r }}, ['999']])`
3. **提取目标模块**: `_r(60578)` 获取 Protobuf 定义模块
4. **分析字段**: 遍历模块导出的 message type，找到字段编号和类型

## 调试常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 注册失败 10018 | Cookie 过期或 serviceToken 编码错误 | 重新获取 token; token 用 latin-1 编码 |
| 发送返回 82002 | Protobuf 字段缺失或格式错误 | 对比浏览器发出的帧，逐字段比对 |
| 收不到推送 | 未处理 Push.Message 命令 | 同时处理 Push.Message 和 Push.SyncMessage |
| Windows 时间戳报错 | 毫秒/微秒级时间戳超出范围 | ts_ms/1000, 超大则 /1000000, try-except 兜底 |
| 心跳断开 | 心跳间隔过长或未发送 | 50s 间隔, KeepAlive 命令 |

Last updated: 2026-03-20
