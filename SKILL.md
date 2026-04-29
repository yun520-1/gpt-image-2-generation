---
name: gpt-image-2-generation
version: "1.0.0"
description: 独立的 gpt-image-2 生图技能，先验证接口可用性，再生成图片并保存为本地文件；所有密钥通过环境变量注入，不包含跳过证书验证的示例。
author: HeartFlow
---

# gpt-image-2 生图技能

## 适用场景

当你需要：
- 通过 `https://api.clawto.link/v1/images/generations` 调用 `gpt-image-2`
- 先验证 API 是否可用，再执行生图
- 把返回的 `b64_json` 解码成图片文件
- 形成一个**独立**的生图技能，与 HeartFlow 主体无关

## 安全原则

- **不在技能文档中保存真实密钥**
- **不提供禁用 SSL 证书验证的示例**
- **优先使用环境变量注入 API Key**
- **如果接口不可用，先检查网络、证书和额度，再重试**

## 1. 验证接口

先确认模型列表可用：

```bash
python3 - <<'PY'
import os, urllib.request, ssl
url = 'https://api.clawto.link/v1/models'
key = os.environ['CLAWTO_API_KEY']
req = urllib.request.Request(url, headers={'Authorization': f'Bearer {key}'})
ctx = ssl.create_default_context()
with urllib.request.urlopen(req, context=ctx, timeout=30) as r:
    print(r.status)
    print(r.read().decode('utf-8', 'ignore'))
PY
```

## 2. 调用 gpt-image-2

```bash
python3 - <<'PY'
import os, urllib.request, json, ssl
api = 'https://api.clawto.link/v1/images/generations'
key = os.environ['CLAWTO_API_KEY']
payload = json.dumps({
    'model': 'gpt-image-2',
    'prompt': 'a stylish female livestream host in a professional streaming studio, smiling, modern lighting, realistic, safe and tasteful, no suggestive pose',
    'size': '1024x1024'
}).encode()
req = urllib.request.Request(api, data=payload, headers={
    'Authorization': f'Bearer {key}',
    'Content-Type': 'application/json'
})
ctx = ssl.create_default_context()
with urllib.request.urlopen(req, context=ctx, timeout=180) as r:
    print(r.status)
    print(r.read().decode('utf-8', 'ignore'))
PY
```

## 3. 保存图片

如果返回里包含 `data[0].b64_json`，解码保存：

```bash
python3 - <<'PY'
import json, base64, pathlib
body = json.loads(open('response.json').read())
out = pathlib.Path('gpt_image_2.png')
out.write_bytes(base64.b64decode(body['data'][0]['b64_json']))
print(out)
PY
```

## 常见问题

### 证书错误
如果遇到证书错误，优先检查系统证书链、代理和网络环境，不要通过跳过证书校验规避。

### 请求超时
`gpt-image-2` 可能生成较慢，建议把超时提高到 180 秒或更长，并允许重试。

### 返回格式不是直接图片 URL
该链路可能返回 `b64_json`，先检查 JSON 里的 `data[0]`。

### 余额不足
如果出现 `insufficient balance`，说明不是接口问题，而是账户额度问题。

## 可复用提示词模板

```text
a stylish female livestream host in a professional streaming studio, smiling, modern lighting, realistic, safe and tasteful, no suggestive pose
```

可按场景替换为：
- 专业直播间
- 产品展示
- 访谈主持
- 商业宣传图

## 验证标准

- `/v1/models` 返回 200
- `/v1/images/generations` 返回 200
- 响应里存在 `data[0].b64_json`
- 图片成功写入本地文件

## 备注

不要在技能中保存真实密钥。所有 key 必须由环境变量或外部配置注入。
