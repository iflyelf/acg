# ACG 二次元风景图片 API

基于 Cloudflare Workers 的随机二次元风景图片 API。项目只有一个 JS 文件 `worker.js`，图片源、过滤逻辑、失败占位图和响应头全部内嵌，不依赖其他 JS 或静态资源文件。

- 仓库：<https://github.com/iflyelf/acg>
- 运行环境：Cloudflare Workers
- 图片源：Konachan
- 默认行为：每次请求随机返回一张风景图，不出现人物

## ✨ 特性

- 仅使用 Konachan，查询标签为 `scenic no_humans rating:safe`
- 额外过滤人物相关标签以及带 `watermark` 的图片
- Konachan API 直连被 Cloudflare 人机验证拦截时，自动通过 Jina Reader 读取同一份 Konachan JSON 元数据；图片本身仍然只来自 Konachan，并继续通过 `wsrv.nl` 缩放
- 直连和 Jina Reader 都不可达或返回空图池时，使用 `worker.js` 内嵌的最近 Konachan 图池兜底；图片仍只来自 Konachan，仍通过 `wsrv.nl` 缩放
- 每次请求随机选图，不做 Worker 端 HTTP 响应缓存
- 手机、平板、PC 自适应：
  - 手机：`720 × 1280`
  - 平板：`800 × 1200`
  - PC：`1920 × 1080`
- 支持 URL 自定义尺寸和方向
- 实时图池、内嵌图池和缩放服务都失败时，返回内嵌 PNG，图片中显示“图片获取失败 / 请稍后刷新重试”
- 所有响应均为 `Cache-Control: no-store`
- 支持 CORS、`HEAD` 和 `OPTIONS`

## 📁 文件结构

```text
acg/
├── README.md
└── worker.js
```

| 文件 | 说明 |
| --- | --- |
| `worker.js` | Worker 全部代码，包含 4 张 Base64 PNG 失败占位图 |
| `README.md` | 项目说明 |

`worker.js` 约 197 KB，其中包含 4 张内嵌 PNG 占位图和 84 条 Konachan 内嵌图池数据，符合 Cloudflare Workers 免费套餐的大小限制。

## 🔌 API

### 随机图片

```http
GET /
```

返回一张随机风景图片。图片的 Content-Type 取决于上游和缩放服务，通常为 `image/jpeg` 或 `image/webp`。

### 自定义尺寸

```http
GET /?w=400&h=600
GET /?w=1920&h=1080
```

`w` 和 `h` 会限制在 `200` 到 `3840` 之间。只传一个参数时，会根据方向或设备默认值推算另一个参数。

### 指定方向

```http
GET /?orientation=portrait
GET /?orientation=landscape
GET /?orientation=square
```

可选值：

- `portrait`：竖图
- `landscape`：横图
- `square`：方图

### JSON 元数据

```http
GET /?format=json
```

返回图片元数据，不会直接返回图片二进制：

```json
{
  "code": 200,
  "provider": "konachan",
  "pool": "live",
  "id": 123456,
  "width": 1920,
  "height": 1080,
  "originalWidth": 2560,
  "originalHeight": 1440,
  "url": "https://wsrv.nl/...",
  "original": "https://konachan.net/image/...",
  "page": "https://konachan.net/post/show/123456",
  "source": "https://www.pixiv.net/artworks/..."
}
```

`pool` 字段为 `live`（实时 API/Jina）或 `embedded`（内嵌图池）。

### 302 跳转

```http
GET /?format=redirect
```

返回 `302`，`Location` 指向实际图片地址，适合需要图片直链的场景。

### 健康检查

```http
GET /health
```

返回服务状态：

```json
{
  "ok": true,
  "time": "2026-10-20T00:00:00.000Z",
  "upstreams": ["konachan"]
}
```

## ⚙️ 参数

| 参数 | 取值 | 说明 |
| --- | --- | --- |
| `w` | `200` - `3840` | 目标宽度 |
| `h` | `200` - `3840` | 目标高度 |
| `orientation` | `portrait` / `landscape` / `square` | 目标方向 |
| `format` | `image` / `json` / `redirect` | 返回格式，默认 `image` |

尺寸识别优先级：

1. URL 中的 `w` / `h`
2. 浏览器 Client Hints（`Sec-CH-Viewport-Width`、`Sec-CH-Viewport-Height`、DPR）
3. Cloudflare `CF-Device-Type`
4. User-Agent
5. 默认 PC 尺寸 `1920 × 1080`

## 📱 自适应尺寸

| 设备 | 默认尺寸 | 方向 |
| --- | --- | --- |
| 手机 | `720 × 1280` | 竖图 |
| 平板 | `800 × 1200` | 竖图 |
| PC | `1920 × 1080` | 横图 |
| 方形 | `1080 × 1080` | 方图 |

排查手机尺寸时，可直接用 `?w=720&h=1280` 强制手机目标尺寸；`?format=json` 返回的 `width` / `height` 就是本次实际目标尺寸。

失败占位图同样按方向和比例选择：

- 竖图：`720 × 1280` 或 `800 × 1200`
- 横图：`1920 × 1080`
- 方图：`1080 × 1080`

## 🧩 图池兜底顺序

Worker 按以下顺序获取图池：

1. 直接请求 Konachan `post.json`
2. 直连失败时，通过 Jina Reader 读取同一份 Konachan JSON
3. 以上都失败、超时或返回空图池时，使用 `worker.js` 内嵌的最近 Konachan 图池
4. 内嵌图片也无法通过 `wsrv.nl` 获取时，返回失败占位图

内嵌图池是最近一次从 Konachan 拉取并过滤的静态列表，不是第二图源，也不是 HTTP 响应缓存。图池中的图片仍然只来自 Konachan，仍通过 `wsrv.nl` 缩放；所有响应继续使用 `Cache-Control: no-store`。

## 🖼️ 上游失败时的兜底

当实时图池、内嵌图池和缩放服务全部失败时，默认图片模式会返回一张内嵌 PNG：

- HTTP 状态码：`200`
- `Content-Type: image/png`
- `X-Image-Fallback: 1`
- `X-Image-Pool: fallback`
- `X-Image-Provider: fallback`
- `X-Image-Id: fallback`
- `Cache-Control: no-store`
- 图片文字：`图片获取失败` / `请稍后刷新重试`

因此浏览器 `<img>` 不会把 502 JSON 当成图片显示。

`format=json` 和 `format=redirect` 不是图片响应；只有实时图池和内嵌图池都不可用时，才返回 `502` JSON：

```json
{
  "code": 502,
  "error": "UPSTREAM_UNAVAILABLE",
  "message": "No image available right now."
}
```

## 📋 响应头

| 响应头 | 说明 |
| --- | --- |
| `X-Image-Provider` | 图源，正常为 `konachan`，失败为 `fallback` |
| `X-Image-Id` | 图片 ID |
| `X-Image-Page` | 图片详情页 |
| `X-Image-Source` | 原始来源页 |
| `X-Image-Width` | 原图宽度 |
| `X-Image-Height` | 原图高度 |
| `X-Target-Width` | 本次请求的目标宽度 |
| `X-Target-Height` | 本次请求的目标高度 |
| `X-Image-Resized` | 是否经过缩放，`1` 或 `0` |
| `X-Image-Pool` | 图池来源：`live`、`embedded` 或 `fallback` |
| `X-Image-Fallback` | 是否使用失败占位图，`1` 或 `0` |

所有响应都会设置：

```text
Cache-Control: no-store, no-cache, must-revalidate, max-age=0
CDN-Cache-Control: no-store
Cloudflare-CDN-Cache-Control: no-store
```

## 🚀 部署

### 方式一：Cloudflare Dashboard

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. 进入 **Workers & Pages** → **Create Application** → **Create Worker**
3. 给 Worker 命名，例如 `acg`
4. 点击 **Edit Code**，删除默认代码
5. 粘贴 `worker.js` 的全部内容
6. 点击 **Save and Deploy**
7. 访问分配到的 `https://<worker-name>.<subdomain>.workers.dev`

### 方式二：Wrangler CLI

```bash
git clone https://github.com/iflyelf/acg.git
cd acg

npx wrangler@4 login
npx wrangler@4 deploy worker.js \
  --name acg \
  --compatibility-date 2026-10-20
```

部署完成后，Worker 地址类似：

```text
https://acg.<你的-workers-子域>.workers.dev
```

## 📝 使用示例

```html
<!-- 默认自适应尺寸 -->
<img src="https://acg.example.workers.dev/" alt="随机二次元风景" />

<!-- 固定 400 × 600 -->
<img src="https://acg.example.workers.dev/?w=400&h=600" alt="随机二次元风景" />

<!-- 随机竖图 -->
<img src="https://acg.example.workers.dev/?orientation=portrait" alt="随机二次元风景" />
```

```bash
# 获取图片元数据
curl 'https://acg.example.workers.dev/?format=json'

# 获取 302 图片直链
curl -I 'https://acg.example.workers.dev/?format=redirect'

# 健康检查
curl 'https://acg.example.workers.dev/health'
```

## ⚠️ 说明与限制

- 图源只有 Konachan，没有 Wallhaven 等其他图源兜底。
- 通过 `no_humans` 标签过滤人物，但标签由上传者填写，不能 100% 保证画面中绝对没有人。
- 每次请求都是独立随机选图，可能出现重复图片；没有持久化已发送图片 ID。
- Worker 端不缓存图池；实际图片可能被上游缩放服务缓存。
- Jina Reader 仅作为 Konachan 元数据的第三方读取兜底，可能受其限流或可用性影响；不会引入第二个图片源。
- Konachan、Jina Reader 和缩放服务都不可用时，会返回失败 PNG 或 502 JSON。
- 不要把 Cloudflare API Token、GitHub Token 等敏感信息写入仓库。

## 🔗 相关链接

- Konachan：<https://konachan.net/>
- Cloudflare Workers 文档：<https://developers.cloudflare.com/workers/>
- 项目仓库：<https://github.com/iflyelf/acg>
