<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [1. 概述](#1-%E6%A6%82%E8%BF%B0)
- [2. HTTP请求鉴权](#2-http%E8%AF%B7%E6%B1%82%E9%89%B4%E6%9D%83)
- [3. app 操作接口](#3-app-%E6%93%8D%E4%BD%9C%E6%8E%A5%E5%8F%A3)
  - [3.1 CreateApp](#31-createapp)
  - [3.2 GetApp](#32-getapp)
  - [3.3 DeleteApp](#33-deleteapp)
  - [3.4 UpdateApp](#34-updateapp)
- [4. room 操作接口](#4-room-%E6%93%8D%E4%BD%9C%E6%8E%A5%E5%8F%A3)
  - [4.1 ListUser](#41-listuser)
  - [4.2 KickUser](#42-kickuser)
  - [4.3 ListActiveRoom](#43-listactiveroom)
- [5. RoomToken 的计算](#5-roomtoken-%E7%9A%84%E8%AE%A1%E7%AE%97)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


# 1. 概述

Qiniu RTC Server API 为七牛实时音视频云提供权限验证和房间管理功能，API 均采用 REST 接口。



# 2. HTTP请求鉴权

Qiniu RTC Server API 通过 Qiniu Authorization 方式进行鉴权，每个房间管理 HTTP 请求头部需增加一个 Authorization 字段：

```
Authorization: "<QiniuToken>"
```

**QiniuToken**: 管理凭证，用于鉴权。

使用七牛颁发的`AccessKey`和`SecretKey`，对本次 http 请求的信息进行签名，生成管理凭证。签名的原始数据包括http请求的 `Method`, `Path`, `RawQuery`, `Content-Type` 及 `Body` 等信息，这些信息的获取方法取决于具体所用的编程语言，建议参照七牛提供的 SDK 代码。
计算过程及伪代码如下：

```

// 1.构造待签名的 Data

// 添加 Method 和 Path
data = "<Method> <Path>"

// 添加 Query，如果Query不存在或者为空，则跳过此步
if "<RawQuery>" != "" {
        data += "?<RawQuery>"
}

// 添加 Host
data += "\nHost: <Host>"

// 添加 Content-Type，如果Content-Type不存在或者为空，则跳过此步
if "<Content-Type>" != "" {
        data += "\nContent-Type: <Content-Type>"
}

// 添加回车
data += "\n\n"

// 添加 Body， 如果Content-Length, Content-Type和Body任意一个不存在或者为空，则跳过此步；如果Content-Type为application/octet-stream，也跳过此步
bodyOK := "<Content-Length>" != "" && "<Body>" != ""
contentTypeOK := "<Content-Type>" != "" && "<Content-Type>" != "application/octet-stream"
if bodyOK && contentTypeOK {
        data += "<Body>"
}

// 2. 计算 HMAC-SHA1 签名，并对签名结果做 URL 安全的 Base64 编码
sign = hmac_sha1(data, "Your_Secret_Key")
encodedSign = urlsafe_base64_encode(sign)  

// 3. 将 Qiniu 标识与 AccessKey、encodedSign 拼接得到管理凭证
<QiniuToken> = "Qiniu " + "Your_Access_Key" + ":" + encodedSign

```



# 3. app 操作接口

app 是客户在七牛实时音视频云的业务配置集合。一个 app 帐号下的连麦房间（频道）拥有独立的命名空间，并且这些房间的可配置功能选项遵从 app 的配置，例如人数限制、是否允许抢流、合流、旁路直播等。一个客户允许创建多个不同的 app ，以实现不同的配置需求；不同 app 下面所属的同名房间并不互通。

## 3.1 CreateApp

```
Host rtc.qiniuapi.com
POST /v3/apps
Authorization: qiniu mac
Content-Type: application/json
{
    "hub": "<Hub>",
    "title": "<Title>",
    "maxUsers": <MaxUsers>,
    "noAutoCloseRoom": <NoAutoCloseRoom>,
    "noAutoCreateRoom": <NoAutoCreateRoom>,
    "noAutoKickUser": <NoAutoKickUser>
}
```

**Hub**: 绑定的直播 hub，可选，使用此 hub 的资源进行推流等业务功能，hub 与 app 必须属于同一个七牛账户。

**Title**: app 的名称，可选，注意，Title 不是唯一标识，重复 create 动作将生成多个 app。

**MaxUsers**: int 类型，可选，连麦房间支持的最大在线人数。

**NoAutoCloseRoom**: bool 类型，可选，禁止自动关闭房间。默认为 false ，即用户退出房间后，房间会被主动清理释放。

**NoAutoCreateRoom**: bool 类型，可选，禁止自动创建房间。默认为 false ，即不需要主动调用接口创建即可加入房间。

**NoAutoKickUser**: bool 类型，可选，禁止自动踢人（抢流）。默认为 false ，即同一个身份的 client (app/room/user) ，新的连麦请求可以成功，旧连接被关闭。

```
200 OK 
{
    "appId": "<AppID>",
    "hub": "<Hub>",
    "title": "<Title>",
    "maxUsers": <MaxUsers>,
    "noAutoCloseRoom": <NoAutoCloseRoom>,
    "noAutoCreateRoom": <NoAutoCreateRoom>,
    "noAutoKickUser": <NoAutoKickUser>,
    "createdAt": <CreatedAt>,
    "updatedAt": <UpdatedAt>
}
616
{
    "error": "hub not match"
}
```

**AppID**: app 的唯一标识。

**Hub**: 绑定的直播 hub，使用此 hub 的资源进行推流等业务功能，hub 与 app 必须属于同一个七牛账户。

**Title**: app 的名称，注意，Title不是唯一标识。

**MaxUsers**: int 类型，连麦房间支持的最大在线人数。

**NoAutoCloseRoom**: bool 类型，禁止自动关闭房间。

**NoAutoCreateRoom**: bool 类型，禁止自动创建房间。

**NoAutoKickUser**: bool 类型，禁止自动踢人。

**CreatedAt**: time 类型，app 创建的时间。

**UpdatedAt**: time 类型，app 更新的时间。

## 3.2 GetApp

```
Host rtc.qiniuapi.com
GET /v3/apps/<AppID> 
Authorization: qiniu mac
```

**AppID**: app 的唯一标识，创建的时候由系统生成。

```
200 OK 
{
    "appId": "<AppID>",
    "hub": "<Hub>",
    "title": "<Title>",
    "maxUsers": <MaxUsers>,
    "noAutoCloseRoom": <NoAutoCloseRoom>,
    "noAutoCreateRoom": <NoAutoCreateRoom>,
    "noAutoKickUser": <NoAutoKickUser>,
    "mergePublishRtmp": {
        "audioOnly": <AudioOnly>,
        "height": <OutputHeight>,
        "width": <OutputHeight>,
        "fps": <OutputFps>,
        "kbps": <OutputKbps>,
        "url": "<URL>",
        "streamTitle": "<StreamTitle>"
    },
    "createdAt": <CreatedAt>,
    "updatedAt": <UpdatedAt>
}

612
{
    "error": "app not found"
}
```

**AppID**: app 的唯一标识。

**UID**: 客户的七牛帐号。

**Hub**: 绑定的直播 hub，使用此 hub 的资源进行推流等业务功能，hub 与 app 必须属于同一个七牛账户。

**Title**: app 的名称，注意，Title不是唯一标识。

**MaxUsers**: int 类型，连麦房间支持的最大在线人数。

**NoAutoCloseRoom**: bool 类型，禁止自动关闭房间。

**NoAutoCreateRoom**: bool 类型，禁止自动创建房间。

**NoAutoKickUser**: bool 类型，禁止自动踢人。

**MergePublishRtmp**: 连麦合流转推 RTMP 的配置。

**CreatedAt**: time 类型，app 创建的时间。

**UpdatedAt**: time 类型，app 更新的时间。

## 3.3 DeleteApp

```
Host rtc.qiniuapi.com
DELETE /v3/apps/<AppID> 
Authorization: qiniu mac
```

**AppID**: app 的唯一标识，创建的时候由系统生成。

```
200 OK

612
{
    "error": "app not found"
}
```

## 3.4 UpdateApp

```
Host rtc.qiniuapi.com
Post /v3/apps/<AppID> 
Authorization: qiniu mac
{
    "hub": "<Hub>",
    "title": "<Title>",
    "maxUsers": <MaxUsers>,
    "noAutoCloseRoom": <NoAutoCloseRoom>,
    "noAutoCreateRoom": <NoAutoCreateRoom>,
    "noAutoKickUser": <NoAutoKickUser>,
    "mergePublishRtmp": {
        "enable": <Enable>,
        "audioOnly": <AudioOnly>,
        "height": <OutputHeight>,
        "width": <OutputHeight>,
        "fps": <OutputFps>,
        "kbps": <OutputKbps>,
        "url": "<URL>",
        "streamTitle": "<StreamTitle>"
    }
}
```

**AppID**: app 的唯一标识，创建的时候由系统生成。

**Title**: app 的名称， 可选。

**Hub**: 绑定的直播 hub，可选，用于合流后 rtmp 推流。

**MaxUsers**: int 类型，可选，连麦房间支持的最大在线人数。

**NoAutoCloseRoom**: bool 指针类型，可选，true 表示禁止自动关闭房间。

**NoAutoCreateRoom**: bool 指针指型，可选，true 表示禁止自动创建房间。

**NoAutoKickUser**: bool 类型，可选，禁止自动踢人。

**MergePublishRtmp**: 连麦合流转推 RTMP 的配置，可选择。其详细配置包括如下

- **Enable**: 布尔类型，用于开启和关闭所有房间的合流功能。
- **AudioOnly**: 布尔类型，可选，指定是否只合成音频。
- **Height**, **Width**: int64，可选，指定合流输出的高和宽，默认为 640 x 480。
- **OutputFps**: int64，可选，指定合流输出的帧率，默认为 25 fps 。
- **OutputKbps**: int64，可选，指定合流输出的码率，默认为 1000 。
- **URL**: 合流后转推旁路直播的地址，可选，支持魔法变量配置按照连麦房间号生成不同的推流地址。如果是转推到七牛直播云，不建议使用该配置。
- **StreamTitle**: 转推七牛直播云的流名，可选，支持魔法变量配置按照连麦房间号生成不同的流名。例如，配置 Hub 为 `qn-zhibo` ，配置 StreamTitle 为 `$(roomName)` ，则房间 meeting-001 的合流将会被转推到 rtmp://pili-publish.qn-zhibo.***.com/qn-zhibo/meeting-001地址。详细配置细则，请咨询七牛技术支持。

```
200 OK 
{
    "appId": "<AppID>",
    "hub": "<Hub>",
    "title": "<Title>",
    "maxUsers": <MaxUsers>,
    "noAutoCloseRoom": <NoAutoCloseRoom>,
    "noAutoCreateRoom": <NoAutoCreateRoom>,
    "noAutoKickUser": <NoAutoKickUser>,
    "mergePublishRtmp": {
        "enable": <Enable>,
        "audioOnly": <AudioOnly>,
        "height": <OutputHeight>,
        "width": <OutputHeight>,
        "fps": <OutputFps>,
        "kbps": <OutputKbps>,
        "url": "<URL>",
        "streamTitle": "<StreamTitle>"
    },
    "createdAt": <CreatedAt>,
    "updatedAt": <UpdatedAt>
}

612
{
    "error": "app not found"
}
616
{
    "error": "hub not match"
}
```

**AppID**: app 的唯一标识。

**UID**: 客户的七牛帐号。

**Hub**: 绑定的直播 hub，使用此 hub 的资源进行推流等业务功能，hub 与 app 必须属于同一个七牛账户。

**Title**: app 的名称，注意，Title不是唯一标识。

**MaxUsers**: int 类型，连麦房间支持的最大在线人数。

**NoAutoCloseRoom**: bool 类型，禁止自动关闭房间。

**NoAutoCreateRoom**: bool 类型，禁止自动创建房间。

**NoAutoKickUser**: bool 类型，禁止自动踢人。

**MergePublishRtmp**: 连麦合流转推 RTMP 的配置。

**CreatedAt**: time 类型，app 创建的时间。

**UpdatedAt**: time 类型，app 更新的时间。

# 4. room 操作接口

一次连麦过程，多个终端音视频通信行为的管理是通过 room 进行的。一个连麦房间必然属于某一个 app。房间无需主动创建或删除，用户直接使用客户端 sdk 指定某个 app 和 room 进行连麦即可加入房间。通过接口可以查询 app 下所有的活跃房间，也可以对某一个房间做相关的业务操作。

## 4.1 ListUser

```
Host rtc.qiniuapi.com
GET /v3/apps/<AppID>/rooms/<RoomName>/users
Authorization: qiniu mac
```

**AppID**: 连麦房间所属的 app 。

**RoomName**: 操作所查询的连麦房间。

```
200 OK
{
    "users": [
        {
            "userId": "<UserID>"
        },
    ]
}
612
{
    "error": "app not found"
}
```

**UserID**: 连麦房间里在线的用户，如果无在线用户，则返回空数组。

## 4.2 KickUser

```
Host rtc.qiniuapi.com
DELETE /v3/apps/<AppID>/rooms/<RoomName>/users/<UserID>
Authorization: qiniu mac
```

**AppID**: 连麦房间所属的 app 。

**RoomName**: 连麦房间。

**UserID**: 操作所剔除的用户。

```
200 OK
612
{
    "error": "app not found"
}
612
{
    "error": "user not found"
}
615
{
    "error": "room not active"
}
```

## 4.3 ListActiveRoom

```
Host rtc.qiniuapi.com
GET /v3/apps/<AppID>/rooms?prefix=<RoomNamePrefix>&offset=<Offset>&limit=<Limit>
Authorization: qiniu mac
```

**AppID**: 连麦房间所属的 app 。

**RoomNamePrefix**: 所查询房间名的前缀索引，可以为空。

**Offset**: int 类型，分页查询的位移标记。

**Limit**: int 类型，此次查询的最大长度。

```
200 OK
{
    "end": <IsEnd>,
    "offset": <Offset>,
    "rooms": [
            "<RoomName>",
            ...
    ]
}
612
{
    "error": "app not found"
}
```

**IsEnd**: bool 类型，分页查询是否已经查完所有房间。

**Offset**: int 类型，下次分页查询使用的位移标记。

**RoomName**: 当前活跃的房间名。

# 5. RoomToken 的计算

连麦用户终端通过房间管理鉴权获取七牛连麦服务，该鉴权包含了房间名称、用户ID、用户权限、有效时间等信息，需要通过客户的业务服务器使用七牛颁发的AccessKey和SecretKey进行签算并分发给手机APP。手机端SDK以拟定的用户ID身份连接服务器，加入该房间进行视频会议。若用户ID或房间与token内的签算信息不符，则无法通过鉴权加入房间。

计算方法：

```
// 1. 定义房间管理凭证，并对凭证字符做URL安全的Base64编码
roomAccess = {
    "appId": "<AppID>"
    "roomName": "<RoomName>",
    "userId": "<UserID>",
    "expireAt": <ExpireAt>,
    "permission": "<Permission>"
}
roomAccessString = json_to_string(roomAccess) 
encodedRoomAccess = urlsafe_base64_encode(roomAccessString)

// 2. 计算HMAC-SHA1签名，并对签名结果做URL安全的Base64编码
sign = hmac_sha1(encodedRoomAccess, <SecretKey>)
encodedSign = urlsafe_base64_encode(sign)

// 3. 将AccessKey与以上两者拼接得到房间鉴权
roomToken = "<AccessKey>" + ":" + encodedSign + ":" + encodedRoomAccess
```

**AppID**: 房间所属帐号的 app 。

**RoomName**: 房间名称，需满足规格 `^[a-zA-Z0-9_-]{3,64}$`

**UserID**: 请求加入房间的用户 ID，需满足规格 `^[a-zA-Z0-9_-]{3,50}$`

**ExpireAt**: int64 类型，鉴权的有效时间，传入以秒为单位的64位Unix绝对时间，token 将在该时间后失效。

**Permission**: 该用户的房间管理权限，"admin" 或 "user"，默认为 "user" 。当权限角色为 "admin" 时，拥有将其他用户移除出房间等特权.

