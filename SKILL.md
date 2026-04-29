---
name: gpt-image-2-generation
version: "1.2.1"
description: 通用的对话生图技能。自动发现不同 AI 宿主的 OpenAI 兼容配置，验证 gpt-image-2 能力，生成图片并保存为本地文件。
author: HeartFlow
---

# gpt-image-2 生图技能

## 适用场景

当你需要：
- 在对话中直接生成图片
- 自动发现可用的 OpenAI 兼容 API
- 检查 API 是否支持 `gpt-image-2`
- 生成图片并保存为本地文件
- 作为任何 AI Agent 的通用生图模块

## 安全原则

- 不在技能文档中保存真实密钥
- 不提供跳过 SSL 证书验证的示例
- 优先使用环境变量注入 API Key
- 如果接口不可用，先检查网络、证书、额度与配置，再重试
- 验证 `/v1/models` 与 `/v1/images/generations` 时必须使用同一个 base URL 和同一组凭证

## 实战发现

1. **同一网关上，模型列表和生图能力要分别验证**
   - 有的网关能返回模型列表，但图片生成仍然需要单独确认。
   - 先查 `/v1/models`，再查 `/v1/images/generations`，不要混用不同 endpoint 的配置。

2. **`gpt-image-2` 的返回通常是 `b64_json`**
   - 成功响应里常见 `data[0].b64_json`，需要本地解码保存。

3. **尺寸参数需要以实际网关为准**
   - 我们实测过 `2048x1024` 可用；若某些网关拒绝，再回退到 `1024x1024`。

4. **不同 AI / 不同宿主的配置来源可能不同**
   - 优先读取环境变量：`OPENAI_BASE_URL`、`OPENAI_API_KEY`、`CLAWTO_BASE_URL`、`CLAWTO_API_KEY`
   - 若宿主 AI 把配置写入 `config.yaml`，也应支持从其配置文件读取同名字段或等价字段，例如：
     - `model.base_url`
     - `model.api_key`
     - `auxiliary.vision.base_url`
     - `auxiliary.vision.api_key`
   - 目标是让技能在不同 AI 环境中都能自动发现可用 OpenAI 兼容接口，而不是绑定某一个宿主实现。

## 1. 发现可用 API

优先按以下顺序发现配置：

1. 环境变量
   - `OPENAI_BASE_URL`
   - `OPENAI_API_KEY`
   - `CLAWTO_BASE_URL`
   - `CLAWTO_API_KEY`

2. 宿主配置文件 `config.yaml`
   - `model.base_url`
   - `model.api_key`
   - `auxiliary.vision.base_url`
   - `auxiliary.vision.api_key`

3. 其它宿主约定的 OpenAI 兼容字段
   - 只要能映射成 `base_url + api_key`，即可用于验证与生成。

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

如果环境变量不可用，则读取宿主 `config.yaml`，把里面的 `base_url` 和 `api_key` 作为 API 发现结果。

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

### 大尺寸示例
如果网关支持，可以尝试：

```bash
size=2048x1024
```

如果返回 400 或尺寸不支持，先回退到 `1024x1024`。

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
优先检查系统证书链、代理和网络环境，不要通过跳过证书校验规避。

### 请求超时
`gpt-image-2` 可能生成较慢，建议把超时提高到 180 秒或更长，并允许重试。

### 返回格式不是直接图片 URL
该链路可能返回 `b64_json`，先检查 JSON 里的 `data[0]`。

### 余额不足
如果出现 `insufficient balance`，说明不是接口问题，而是账户额度问题。

### 401 Unauthorized
优先检查环境变量是否真正注入，base URL 和 API Key 是否来自同一套配置。

### 400 / 不支持模型
优先确认 `/v1/models` 和 `/v1/images/generations` 使用的是同一 gateway；不要把一个网关的模型列表和另一个网关的生图端点混用。

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

不要在技能中保存真实密钥。所有 key 必须由环境变量、宿主 `config.yaml` 或等价配置注入。
