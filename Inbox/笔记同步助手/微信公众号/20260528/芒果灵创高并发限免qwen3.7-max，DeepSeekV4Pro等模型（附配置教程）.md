---
author: LLMera
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzk0NTMyNzk3Nw==&mid=2247485061&idx=1&sn=b879d4c605b07b4922878af295bdfe24&chksm=c24abb8c89e9f6d04c069ae16d0d8a86509c2a14e0191552a9421b2461167d2ab1192ab3fcb7&mpshare=1&scene=1&srcid=0528Hax8bl5ZnYshng4OU83y&sharer_shareinfo=1b0c3aa30af93467291960463d8c1c25&sharer_shareinfo_first=1b0c3aa30af93467291960463d8c1c25#rd
saved: 2026-05-28 22:23:04
tags:
  - 笔记同步助手
id: f4026fb1-04d8-4e85-8761-8935f3bd932d
---

公众号名称：湖南师范大学22计科师范班团支部

作者名称：LLMera

发布时间：2026-05-28 18:17

## 平台概述

芒果灵创 API 面向业务系统、自动化工具和第三方应用，提供文本、图片、视频生成等能力。开发者可以通过标准化接口，将平台的 AIGC 能力集成到自有业务流程中，实现文生图、参考图生图、文本生成视频、图生视频、视频参考生成等能力。

## 使用指南

先访问官网创建账号：https://aigc.mgtv.com/develop。

然后创建key

![](https://relay-1.bijitongbu.site/p/c5d4295e97cd3d784136701913898308.png)

保存key，最后按照下文配置code agent就行了。

![](https://relay-1.bijitongbu.site/p/4a93977e1c05a4d3048e761d250a65f4.png)

## 文本模型

### 入口

Base URL:

```
https://aigc-llm.mgtv.com/
```

> 如果用CC-Switch配置Claude code的话，baseurl填写“https://aigc-llm.mgtv.com”就行。 如果用 CC-Switch配置codex的话，baseurl填写“https://aigc-llm.mgtv.com/v1”就行。

接口:

```
POST /v1/chat/completions
```

完整地址:

```
https://aigc-llm.mgtv.com/v1/chat/completions
```

### 支持模型与有效期

| 模型名称 | 单价 | RPM限流 | 可调用有效期 |
| --- | --- | --- | --- |
| qwen3.7-max | 限时免费 | 60 | 未知 |
| qwen3.6-plus | 限时免费 | 60 | 05-31 23:59 |
| qwen3.6-max-preview | 限时免费 | 60 | 05-31 23:59 |
| qwen3.6-flash | 限时免费 | 60 | 05-31 23:59 |
| qwen3.5-plus | 限时免费 | 60 | 05-31 23:59 |
| deepseek-v4-pro | 限时免费 | 60 | 05-31 23:59 |
| deepseek-v4-flash | 限时免费 | 60 | 05-31 23:59 |
| qwen3-vl-plus | 限时免费 | 60 | 05-31 23:59 |
| glm-5.1 | 限时免费 | 60 | 05-31 23:59 |
| glm-5 | 限时免费 | 60 | 05-31 23:59 |

### 4.3 调用示例

HTTP 非流式:

```
curl https://aigc-llm.mgtv.com/v1/chat/completions \
  -H "Authorization: Bearer $AIGC_SK" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3.6-flash",
    "messages": [
      {
        "role": "user",
        "content": "你好，请用一句话介绍你自己"
      }
    ]
  }'
```

Python 流式:

```
import json
import os
import requests

url = "https://aigc-llm.mgtv.com/v1/chat/completions"
headers = {
    "Authorization": f"Bearer {os.environ['AIGC_SK']}",
    "Content-Type": "application/json",
}
payload = {
    "model": "qwen3.6-flash",
    "stream": True,
    "messages": [
        {"role": "user", "content": "你好，请用一句话介绍你自己"}
    ],
}

with requests.post(url, headers=headers, json=payload, stream=True, timeout=60) as resp:
    resp.raise_for_status()
    for line in resp.iter_lines(decode_unicode=True):
        ifnot line:
            continue
        if line.startswith("data: "):
            data = line[len("data: "):]
            if data == "[DONE]":
                break
            print(json.loads(data), flush=True)
```

### 4.4 并发限制

当前按 SK 限制请求频率，默认 60 RPM，需要扩大请联系平台管理员或者运营人员；超出限制时返回 429，调用方需要等待后重试。建议按 1s、2s、4s、8s 做退避重试，并设置最大重试次数。

### 4.5 常见错误码

| HTTP 状态码 | 说明 | 处理建议 |
| --- | --- | --- |
| 400 | 请求参数错误 | 检查 model、messages、stream |
| 401 | SK 缺失或无效 | 检查 Authorization: Bearer |
| 403 | SK 不可用或无权限 | 检查 SK 状态和模型权限 |
| 404 | 模型不存在或接口不存在 | 检查模型名称和请求路径 |
| 429 | 请求超过频率限制 | 等待后重试 |
| 500 | 服务内部错误 | 稍后重试，持续失败请联系接口负责人 |
| 502 | 上游服务异常 | 稍后重试 |
| 503 | 服务暂不可用 | 稍后重试 |
| 504 | 请求超时 | 降低并发或稍后重试 |

错误响应示例:

```
{
  "error": {
    "message": "Rate limit exceeded",
    "type": "rate_limit_error",
    "code": "429"
  }
}
```

## cc-switch配置示例图

### claude code

![](https://relay-1.bijitongbu.site/p/c84c291c7517fea67b6d9b6aa38b0187.png)

## codex

![](https://relay-1.bijitongbu.site/p/724ee7438e98a2055565d9480ff77fc6.png)

---

Original LLMera 湖南师范大学22计科师范班团支部

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/93d0e83e_1779978180476?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzk0NTMyNzk3Nw%3D%3D%26mid%3D2247485061%26idx%3D1%26sn%3Db879d4c605b07b4922878af295bdfe24%26chksm%3Dc24abb8c89e9f6d04c069ae16d0d8a86509c2a14e0191552a9421b2461167d2ab1192ab3fcb7%26mpshare%3D1%26scene%3D1%26srcid%3D0528Hax8bl5ZnYshng4OU83y%26sharer_shareinfo%3D1b0c3aa30af93467291960463d8c1c25%26sharer_shareinfo_first%3D1b0c3aa30af93467291960463d8c1c25%23rd&s=obsidian)