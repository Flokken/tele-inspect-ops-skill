---
AIGC:
  ContentProducer: '001191110102MAD55U9H0F10002'
  ContentPropagator: '001191110102MAD55U9H0F10002'
  Label: '1'
  ProduceID: '8e6a676b-3aed-4d32-abde-b0c279f1c47d'
  PropagateID: '8e6a676b-3aed-4d32-abde-b0c279f1c47d'
  ReservedCode1: 'af712fd1-d368-415f-9826-2e02d4c2374d'
  ReservedCode2: 'af712fd1-d368-415f-9826-2e02d4c2374d'
---

# IM 群机器人 Webhook 接口速查（imtwo.zdxlz.com）

向量微服务（IM）群机器人 webhook 接口。创建者在机器人详情页可看到该机器人特有的 webhook key。
**务必保护好 key，避免泄露。**

## 发送消息

```
POST https://imtwo.zdxlz.com/im-external/v1/webhook/send?key=KEY
Content-Type: application/json
```

支持四种消息类型：text / image / news / file。

### 文本 text
```json
{
    "type": "text",
    "textMsg": {
        "content": "合肥今日天气：29度，大部分多云，降雨概率：60%",
        "isMentioned": true,
        "mentionType": 2,
        "mentionedMobileList": ["12888888888", "12666666666"]
    }
}
```
- content 必填，UTF-8
- isMentioned：是否@（true/false）
- mentionType：1=@所有人，2=@部分人（isMentioned=true 时必填）
- mentionedMobileList：@人员手机号（mentionType=2 时必填）

### 图片 image
```json
{
    "type": "image",
    "imageMsg": { "fileId": "42b0719d4d1f44aaac94ab966a5e67c6" }
}
```
fileId 来自上传接口返回。

### 文件 file
```json
{
    "type": "file",
    "fileMsg": { "fileId": "fileId" }
}
```

### 图文 news
```json
{
    "type": "news",
    "news": {
        "info": {
            "title": "今年春节有好礼相送",
            "description": "点击图片领取礼品",
            "url": "https://www.baidu.com",
            "picUrl": "http://img.sccnn.com/bimg/341/34669.jpg"
        }
    }
}
```
- title 必填，≤128 字节（超长自动截断）
- description 可选，≤512 字节
- url 必填，点击跳转链接
- picUrl 可选，图片链接

## 上传附件

```
POST https://imtwo.zdxlz.com/im-external/v1/webhook/upload-attachment?key=KEY&type=TYPE
Content-Type: multipart/form-data
```

- key：机器人 key
- type：1=图片，2=文件
- file：文件二进制流
- 文件大小 ≤ 30M

返回：
```json
{
    "ok": true,
    "code": 200,
    "data": { "id": "ce34fd4a5b0043efa86c37b33e4da0ca", "name": "test.jpg", "type": ".jpg", "size": 318182 },
    "message": "成功"
}
```
data.id 即 fileId，供 send 接口引用。

## Python 调用示例（requests）

```python
import os
import requests

IM_HOST = "https://imtwo.zdxlz.com"
WEBHOOK_KEY = "你的机器人key"  # 注意保密

def send_text(content, key=WEBHOOK_KEY):
    url = f"{IM_HOST}/im-external/v1/webhook/send?key={key}"
    payload = {"type": "text", "textMsg": {"content": content}}
    resp = requests.post(url, json=payload, timeout=30)
    resp.raise_for_status()
    return resp.json()

def upload_file(file_path, key=WEBHOOK_KEY):
    """上传文件，返回 fileId。图片(type=1)/其他文件(type=2) 按扩展名自动判断。"""
    ext = os.path.splitext(file_path)[1].lower()
    ftype = 1 if ext in {".jpg", ".jpeg", ".png", ".gif", ".bmp", ".webp"} else 2
    url = f"{IM_HOST}/im-external/v1/webhook/upload-attachment?key={key}&type={ftype}"
    with open(file_path, "rb") as f:
        resp = requests.post(url, files={"file": (os.path.basename(file_path), f)}, timeout=60)
    resp.raise_for_status()
    return resp.json()["data"]["id"]

def send_file(file_id, key=WEBHOOK_KEY):
    url = f"{IM_HOST}/im-external/v1/webhook/send?key={key}"
    payload = {"type": "file", "fileMsg": {"fileId": file_id}}
    resp = requests.post(url, json=payload, timeout=30)
    resp.raise_for_status()
    return resp.json()
```

成功响应：`{"ok": true, "code": 200, "message": "成功"}`。

## 频率限制

消息发送有频率限制，批量推送时注意控制节奏（如逐条间隔发送），避免触发限流。