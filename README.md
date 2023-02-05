## 提供一个方便、稳定、简洁的微信聊天机器人api对接平台

## Whosecard WXBot - 微信聊天机器人api平台

特点:

* 使用pc版微信登录方案，稳定性高，支持异地登陆
* 后台可直接管理微信号登录状态，可一键扫码登录
* 本平台托管同步消息，状态维护，省去用户编写大量代码
* 支持websocket进行消息同步，使用简单方便
* 为常规功能接口（如发送消息）提供http api，灵活性高
* 后续可定制基础通用插件，如chatgpt等智能聊天功能

使用前须知:

* 本平台为付费工具，账号注册请联系qq 1628121385 添加好友时请注明:wxbot
* 目前只开放聊天相关接口，也只为这部分需求的用户提供，这样做的目的是避免本工具被用于高风险行为
* 建议使用实名后的微信小号登录，这样更稳定
* 在首次登录24小时内会掉线一次，再次通过历史登录成功后，可几个月不掉线，如果遇到掉线也可进入后台进行再次登录操作
* 目前只开放聊天相关接口，也只为这部分需求的用户提供，这样做的目的是避免本工具被用于高风险行为
* 登录成功后请不要高风险操作，比如频繁发送消息，否则容易被限制

术语解释:
* key: 代表账号，一个账号下可以创建多个token
* token: 可以理解为一个token对应一个设备，只能绑定登录一个微信号
* 微信号: 一般指wxid开头的一串字符，在代码里一般命名为UserName


## 对接流程

1. 注册账号key，账号注册请联系qq 1628121385 添加好友时请注明:wxbot
2. 登录后台 https://wxbot.whosecard.com
3. 首次使用请先创建1个token
4. 进入token管理页面，首次登录请点击【首次扫码登录】按钮
5. 手机端进行确认登录操作，登录成功后，就可以对接消息同步，以及调用发送消息等api功能了

⚠️ 在首次登录24小时内一般会掉线一次，再在后台点击【历史登录】

## 对接消息同步

目前可通过websocket进行消息同步，消息延时一般为数秒。


### WebSocket鉴权

使用WebSocket时，地址为```ws://api.whosecard.com:19095```

在你连接到WebSocket后，请先发送登录数据包，**否则服务器不会响应你的操作！**

#### 发送鉴权包
| 键值 | 解释 | 类型 | 示例 |
| --- | --- | --- | --- |
| app | 这是一个固定值，请不要随意更改 | String | ```wxbot``` |
| version | 这是一个固定值，请不要随意更改 | String | ```1.0``` |
| key | 账号Key（不是Token！是登录用的那个！） | String | / |

发送数据示例：
```Python
{
  "app": "wxbot",
  "version": "1.0",
  "key": "xxxxxxxxx"
}
```

#### 接收鉴权结果
| 键值 | 解释 | 类型 | 示例 |
| --- | --- | --- | --- |
| msg | 鉴权结果（中文消息内容） | String | / |
| retCode | 鉴权结果代码，```0```为成功，```1000```为鉴权失败，请根据此字段判断鉴权是否成功 | Int | / |

接收数据示例：
```Python
# 鉴权成功返回
{
  "type": "AuthCheckResult",
  "msg": "鉴权成功，本账号下所有已登录微信号的消息均会同步",
  "retCode": 0
}

# 鉴权失败返回
{
  "eventType": "AuthCheckResult",
  "msg": "鉴权失败，无效key",
  "retCode": 1000
}
```

#### WebSocket同步消息事件
当有以下事件发生时，服务端会通过WebSocket主动给您发送消息：

+ 收到个人/群聊聊天信息｜eventType=AddMsg

标准JSON数据包格式如下：
```Python
{
    "userInfo": {
      "wxid": "wxid_***", # 用户id
      "nickname": "用户昵称"
  	},
    "eventType": "AddMsg",  # 目前只开放了新增消息事件
    "event": {...},  # 消息类型，有很多种类，先以实际返回为准
}
```
如果一个key下绑定了多个token，可通过wxid和nickname来判断区分不同微信号。

以下为python代码示例，本地即可运行。

建议 Python >= 3.7

运行前先安装python依赖库
```
pip install websockets==10.4 --index-url https://mirrors.aliyun.com/pypi/simple/
```

```Python
#!/usr/bin/env python

import asyncio
import json
import logging
import time

import websockets

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(module)s - %(levelname)s - %(message)s')


async def main():
    while True:
        try:
            async with websockets.connect('ws://api.whosecard.com:19095') as websocket:
                client_info = {
                    'app': 'wxbot',
                    'version': '1.0',
                    'key': '这里填你的账号key，注意不是token',
                }
                await websocket.send(json.dumps(client_info))
                auth_result = await websocket.recv()  # 获取client_info鉴权结果，只有鉴权成功后才能同步
                auth_result = json.loads(auth_result)
                logging.info(f'鉴权结果: {auth_result}')

                if auth_result['retCode'] != 0:  # 鉴权失败直接退出，不要重试
                    return

                while True:
                    event = await websocket.recv()  # 这里会block直到有新消息
                    event = json.loads(event)
                    logging.info(f'收到新消息 {event}')
                    # TODO: 这里对消息进行处理，会同步账号key下所有token的消息
        except Exception:
            logging.exception('main exception.')
        time.sleep(10)


if __name__ == "__main__":
    asyncio.run(main())  # 需要python>=3.7
```

## HTTP API列表

#### 鉴权
所有请求都需要传账号key进行鉴权

#### 有效计次
接口只要返回cost=true，就表示请求有效，会计入有效请求次数，这通常在限速中会用到

#### 返回结果

* 所有接口均返回json格式，其中参数ok[true|false]表示是否请求成功
* 当返回ok=false时，可以参考返回的error字段（如果存在的话）以及retCode状态码
* 除非特殊指明，返回结果均为原始采集结果，字段含义请按字面意思理解

#### 状态码说明

状态码以retCode字段出现在返回结果中，是整数类型。

| 状态码 | 描述                                            |
| ------ | ----------------------------------------------- |
| 0    | 执行成功 |
| 1000    | 内部错误，一般情况下可直接重试 |
| 1001    | 参数错误，请修改参数后再重试 |
| 1002    | 请求资源不存在，该资源可能被平台删除/封禁，这种情况下请不要再重试 |
| 1003    | 内部处理较忙，请稍后重试，一般为内部请求资源暂时性不足，休息一会儿再重试 |
| 1004    | 余额/权限不足，请充值或联系管理员开通权限 |

#### 使用建议
由于接口调用有时候会遇到失败或者不稳定的场景，接口对接建议如下：
* 接口超时时间建议设置为30秒，如果超时时间设置太短，可能会出现用户提前断掉请求，但是服务端这边还在执行的场景
* 当接口返回cost字段为true时，不管有没有请求成功，都不要再重试了
* 当接口cost返回false，此时根据状态码判断是否需要重试
* 对于重试请求，建议中间sleep 1-3秒，也可以设置一个最大重试次数
* 接口均为POST方式请求，返回ok=true表示服务器处理成功，但实际业务是否成功请看result字段
* 使用本接口前建议先对接消息同步功能，因为消息里会包含UserName这类参数，能方便的获取群id/用户微信号
* 友情提示：群id的格式以@chatroom结尾

#### 返回示例
```Python
# 服务器处理成功
{
    "ok": true,
    "result": {***},
    "retCode": 0,
    "cost": true
}

# 服务器处理失败
{
    "ok": false,
    "retCode": 1001,
    "cost": false
}
```

#### 目前开放以下接口

##### 给指定群/用户发送文本消息

| 键值 | 解释 | 类型 | 示例 |
| --- | --- | --- | --- |
| Token | 微信号绑定token | String | / |
| UserName | 群id/对方微信号 | String | wxid_123 |
| Content | 文本内容 | String | / |

```
POST https://api.whosecard.com/wxapi/pc/message/sendText?key=***

Payload:
{
  "UserName": "wxid_123",
  "Content": "你好",
  "Token": "****"
}
```

##### 给指定群/用户发送图片消息

| 键值 | 解释 | 类型 | 示例 |
| --- | --- | --- | --- |
| Token | 微信号绑定token | String | / |
| UserName | 群id/对方微信号 | String | / |
| Base64Image | 图片base64编码文本 | String | / |

```
POST https://api.whosecard.com/wxapi/pc/message/sendImage?key=***

Payload:
{
  "UserName": "wxid_123",
  "Base64Image": "data:base64;image/png,xxxxxxxxxxxx",
  "Token": "****"
}
```

##### 给指定群/用户发送语音消息

| 键值 | 解释 | 类型 | 示例 |
| --- | --- | --- | --- |
| Token | 微信号绑定token | String | / |
| Type | 音频格式：AMR = 0, SPEEX = 1, MP3 = 2, WAVE = 3, SILK = 4 | Int | / |
| UserName | 群id/对方微信号 | String | / |
| DataFileBase64 | 音频文件BASE64 | String | / |
| VoiceTime | 音频时长，单位毫秒 | Int | / |

```
POST https://api.whosecard.com/wxapi/pc/message/sendText?key=***

Payload:
{
  "Type": 0,
  "UserName": "wxid_123",
  "DataFileBase64": "******",
  "VoiceTime": 2000,
  "Token": "****"
}
```

##### 给指定群/用户发送视频消息（请限制视频大小在3M内）

| 键值 | 解释 | 类型 | 示例 |
| --- | --- | --- | --- |
| Token | 微信号绑定token | String | / |
| UserName | 群id/对方微信号 | String | / |
| VideoFileBase64 | 视频文件BASE64 | String | / |
| ImageFileBase64 | 缩略图文件BASE64 | String | / |
| VideoTime | 视频时长，单位毫秒 | Int | / |

```
POST https://api.whosecard.com/wxapi/pc/message/sendText?key=***

Payload:
{
  "UserName": "wxid_123",
  "VideoFileBase64": "******",
  "ImageFileBase64": "data:base64;image/png,xxxxxxxxxxxx",
  "VideoTime": 10000,
  "Token": "****"
}
```

## 免责声明

请在合法范围内使用本程序。任何人因使用本程序造成的意外损失或恶意攻击，项目开发者对此概不负责，亦不承担任何法律责任。
