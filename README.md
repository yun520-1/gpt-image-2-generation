# gpt-image-2-generation

通用的对话生图技能仓库，面向所有支持 OpenAI 兼容接口的 AI Agent。

## 功能

- 先发现已配置的 OpenAI 兼容 API
- 验证接口是否支持 `gpt-image-2`
- 使用 `gpt-image-2` 生成图片
- 支持把返回的 `b64_json` 解码保存为本地图片
- 适合对话生图、产品图、插画、概念图等场景

## 适用场景

- 在自然语言对话中直接生成图片
- 作为任意 AI Agent 的通用生图模块
- 对话驱动的图像生成、迭代、保存与分发

## 安全原则

- 不保存真实 API Key
- 使用环境变量注入密钥
- 不提供跳过 SSL 证书验证的示例
- 优先保证链路可验证、可复用、可审计

## 快速开始

### 1. 设置环境变量

```bash
export OPENAI_BASE_URL="https://api.clawto.link/v1"
export OPENAI_API_KEY="your-api-key"
```

如果使用其他兼容 API，也可以设置：

```bash
export CLAWTO_BASE_URL="https://api.clawto.link/v1"
export CLAWTO_API_KEY="your-api-key"
```

### 2. 检查是否支持 gpt-image-2

```bash
python3 - <<'PY'
import os, json, urllib.request, ssl
base = os.getenv('OPENAI_BASE_URL') or os.getenv('CLAWTO_BASE_URL')
key = os.getenv('OPENAI_API_KEY') or os.getenv('CLAWTO_API_KEY')
if not base or not key:
    raise SystemExit('missing base url or api key')
req = urllib.request.Request(base.rstrip('/') + '/models', headers={'Authorization': f'Bearer {key}'})
ctx = ssl.create_default_context()
with urllib.request.urlopen(req, context=ctx, timeout=30) as r:
    data = json.loads(r.read().decode('utf-8', 'ignore'))
    models = [m.get('id') for m in data.get('data', [])]
    print('gpt-image-2' in models)
    print(models)
PY
```

### 3. 生成图片

```bash
python3 - <<'PY'
import os, json, urllib.request, ssl
base = os.getenv('OPENAI_BASE_URL') or os.getenv('CLAWTO_BASE_URL')
key = os.getenv('OPENAI_API_KEY') or os.getenv('CLAWTO_API_KEY')
if not base or not key:
    raise SystemExit('missing base url or api key')
api = base.rstrip('/') + '/images/generations'
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

### 4. 保存图片

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

## 示例提示词

- 一位时尚女主播在专业直播间，微笑，自然长发，现代灯光，写实风格
- 宇宙女神主题，长发，宏伟背景，电影级光影，安全、自然、不露骨
- 商业宣传图风格的主播肖像，清晰、精致、专业

## 返回格式

成功时，接口通常会返回 JSON，其中：
- `data[0].b64_json` 为图片内容

## 常见问题

### 证书错误
优先检查系统证书链、代理和网络环境，不要通过跳过证书校验规避。

### 请求超时
`gpt-image-2` 可能生成较慢，建议把超时提高到 180 秒或更长，并允许重试。

### 模型不在列表中
如果 `/v1/models` 没有 `gpt-image-2`，说明当前 API 网关不支持该模型，应切换到支持该模型的兼容 endpoint。

## 许可证

MIT
