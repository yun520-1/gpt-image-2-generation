---
name: gpt-image-2-generation
version: "1.1.0"
description: 通用的对话生图技能。先发现已配置的 OpenAI 兼容 API，再验证是否支持 gpt-image-2，随后生成图片并保存为本地文件。适合所有 AI 使用，不绑定 HeartFlow。
author: HeartFlow
---

# gpt-image-2 生图技能

## 适用场景

当你需要：
- 在对话中直接生成图片
- 先发现已配置好的 OpenAI 兼容 API
- 检查 API 是否支持 `gpt-image-2`
- 生成图片并保存为本地文件
- 作为任何 AI Agent 的通用生图模块

## 安全原则

- 不在技能文档中保存真实密钥
- 不提供跳过 SSL 证书验证的示例
- 优先使用环境变量注入 API Key
- 如果接口不可用，先检查网络、证书、额度与配置，再重试

## 1. 发现已配置 API

优先从环境变量读取：
- `OPENAI_BASE_URL`
- `OPENAI_API_KEY`
- `CLAWTO_BASE_URL`
- `CLAWTO_API_KEY`

如果没有环境变量，也可以由宿主 Agent 自己从配置文件注入。

示例：

```bash
python3 - <<'PY'
import os
candidates = [
    ('OPENAI_BASE_URL', 'OPENAI_API_KEY'),
    ('CLAWTO_BASE_URL', 'CLAWTO_API_KEY'),
]
for base_var, key_var in candidates:
    base = os.getenv(base_var)
    key = os.getenv(key_var)
    if base and key:
        print(f'{base_var}={base}')
        print(f'{key_var}=SET')
        break
else:
    print('no_api_found')
PY
```

## 2. 验证是否支持 gpt-image-2

对发现的 API 调用 `/v1/models`：

```bash
python3 - <<'PY'
import os, json, urllib.request, ssl
base = os.environ['API_BASE_URL'].rstrip('/')
key = os.environ['API_KEY']
url = f'{base}/models'
req = urllib.request.Request(url, headers={'Authorization': f'Bearer {key}'})
ctx = ssl.create_default_context()
with urllib.request.urlopen(req, context=ctx, timeout=30) as r:
    data = json.loads(r.read().decode('utf-8', 'ignore'))
    models = [m.get('id') for m in data.get('data', [])]
    print('has_gpt_image_2=', 'gpt-image-2' in models)
    print(models)
PY
```

如果列表里包含 `gpt-image-2`，再继续下一步。

## 3. 调用 gpt-image-2

```bash
python3 - <<'PY'
import os, json, urllib.request, ssl
base = os.environ['API_BASE_URL'].rstrip('/')
key = os.environ['API_KEY']
api = f'{base}/images/generations'
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

## 4. 保存图片

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

## 示例提示词

```text
a stylish female livestream host in a professional streaming studio, smiling, modern lighting, realistic, safe and tasteful, no suggestive pose
```

可按场景替换为：
- 专业直播间
- 产品展示
- 访谈主持
- 商业宣传图
- 宇宙女神主题

## 验证标准

- 能发现一个已配置 API
- `/v1/models` 返回 200
- `gpt-image-2` 存在于模型列表中
- `/v1/images/generations` 返回 200
- 响应里存在 `data[0].b64_json`
- 图片成功写入本地文件

## 备注

不要在技能中保存真实密钥。所有 key 必须由环境变量或宿主配置注入。
