# gpt-image-2-generation

独立的 gpt-image-2 生图技能仓库。

## 功能

- 调用 OpenAI 兼容接口中的 `gpt-image-2`
- 先验证接口可用性，再执行生图
- 支持把返回的 `b64_json` 解码保存为本地图片
- 适合“对话生图”场景：在自然语言对话中直接生成图片

## 适用场景

- 对话里直接生成图片
- 直播/产品/插画/概念图生成
- 生成后保存为本地文件
- 作为其他 Agent 的可复用生图模块

## 安全原则

- 不在仓库中保存真实 API Key
- 使用环境变量注入密钥
- 不提供跳过 SSL 证书验证的示例
- 优先保证链路可验证、可复用、可审计

## 快速开始

### 1. 设置环境变量

```bash
export CLAWTO_API_KEY="your-api-key"
```

### 2. 验证模型列表

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

### 3. 调用 gpt-image-2 生成图片

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

## 返回格式

如果成功，接口通常返回 JSON，`data[0].b64_json` 包含图片内容。可以进一步解码保存：

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

## 许可证

MIT
