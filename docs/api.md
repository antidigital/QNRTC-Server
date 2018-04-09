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
- [4. RoomToken 的计算](#4-roomtoken-%E7%9A%84%E8%AE%A1%E7%AE%97)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


# 1. 概述

Qiniu RTC Server API 提供为 Qiniu 连麦 SDK 提供了权限验证和房间管理功能，API 均采用 REST 接口。



# 2. HTTP请求鉴权

Qiniu RTC Server API 通过 Qiniu Authorization 方式进行鉴权，每个房间管理HTTP 请求头部需增加一个 Authorization 字段：

```
Authorization: "<QiniuToken>"
```

**QiniuToken**: 管理凭证，用于鉴权。

使用七牛颁发的`AccessKey`和`SecretKey`，对本次http请求的信息进行签名，生成管理凭证。签名的原始数据包括http请求的`Method`, `Path`, `RawQuery`, `Content-Type`及`Body`等信息，这些信息的获取方法取决于具体所用的编程语言，建议参照七牛提供的SDK代码。
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

## 3.1 CreateApp

```
POST /v3/apps
Authorization: qiniu mac
Content-Type: application/json
{
    "hub": "<Hub>",
    "title": "<Title>",
    "maxUsers": <MaxUsers>,
    "noAutoCloseRoom": <NoAutoCloseRoom>,
    "noAutoCreateRoom": <NoAutoCreateRoom>
}
```

**Hub**: 绑定的直播 hub，可选，使用此 hub 的资源进行推流等业务功能，hub 与 app 必须属于同一个七牛账户。

**Title**: app 的名称，可选，注意，Title不是唯一标识，重复 create 动作将生成多个 app。

**MaxUsers**: int 类型，可选，连麦房间支持的最大在线人数。

**NoAutoCloseRoom**: bool 类型，可选，禁止自动关闭房间。

**NoAutoCreateRoom**: bool 类型，可选，禁止自动创建房间。

```
200 OK 
{
    "appId": "<AppID>",
    "hub": "<Hub>",
    "title": "<Title>",
    "maxUsers": <MaxUsers>,
    "noAutoCloseRoom": <NoAutoCloseRoom>,
    "noAutoCreateRoom": <NoAutoCreateRoom>,
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

**CreatedAt**: time 类型，app 创建的时间。

**UpdatedAt**: time 类型，app 更新的时间。

## 3.2 GetApp

```
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
    "mergePublishRtmp": {
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

**MergePublishRtmp**: 连麦合流转推 RTMP 的配置。

**CreatedAt**: time 类型，app 创建的时间。

**UpdatedAt**: time 类型，app 更新的时间。

## 3.3 DeleteApp

```
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
Post /v3/apps/<AppID> 
Authorization: qiniu mac
{
    "hub": "<Hub>",
    "title": "<Title>",
    "maxUsers": <MaxUsers>,
    "noAutoCloseRoom": <NoAutoCloseRoom>,
    "noAutoCreateRoom": <NoAutoCreateRoom>,
    "mergePublishRtmp": {
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

**MergePublishRtmp**: 连麦合流转推 RTMP 的配置，可选择。可以显示的直接配置转推 URL，也可以通过指定 streamTitle 推流至 App 所绑定的七牛 Hub，支持魔法变量的配置。

```
200 OK 
{
    "appId": "<AppID>",
    "hub": "<Hub>",
    "title": "<Title>",
    "maxUsers": <MaxUsers>,
    "noAutoCloseRoom": <NoAutoCloseRoom>,
    "noAutoCreateRoom": <NoAutoCreateRoom>,
    "mergePublishRtmp": {
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

**MergePublishRtmp**: 连麦合流转推 RTMP 的配置。

**CreatedAt**: time 类型，app 创建的时间。

**UpdatedAt**: time 类型，app 更新的时间。

# 4. room 操作接口

## 4.1 ListUser

```

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

**AppID**: 房间所属帐号的 app。

**RoomName**: 房间名称，需满足规格`^[a-zA-Z0-9_-]{3,64}$`

**UserID**: 请求加入房间的用户ID，需满足规格`^[a-zA-Z0-9_-]{3,50}$`

**ExpireAt**: int64类型，鉴权的有效时间，传入以秒为单位的64位Unix绝对时间，token将在该时间后失效。

**Permission**: 该用户的房间管理权限，"admin" 或 "user"，默认为 "user" 。当权限角色为 "admin" 时，拥有将其他用户移除出房间等特权.

