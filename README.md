# DoH 转发服务（Cloudflare Workers / Pages）

接收客户端的 DoH（RFC 8484）请求，转发到任意上游 DoH 服务器，并原样返回结果。
同一份代码（`src/doh.js`）既可部署为 Workers，也可部署为 Pages Functions。

## 目录结构

```
doh-proxy/
├── src/doh.js              # 核心转发逻辑
├── src/worker.js           # Workers 入口
├── functions/[[path]].js   # Pages Functions 入口
├── public/index.html       # Pages 静态首页
├── wrangler.toml
└── package.json
```

## 用法

| 路径 | 说明 |
|---|---|
| `/dns-query` | 使用默认上游（环境变量 `UPSTREAM`，默认 cloudflare） |
| `/dns-query/google` | 指定预设上游 |
| `/<TOKEN>/dns-query[/上游]` | 设置了 `TOKEN` 后的访问路径 |

内置上游：`cloudflare`、`google`、`quad9`、`adguard`、`opendns`、`alidns`、`dnspod`。
可用 `EXTRA_UPSTREAMS="名称=https://xxx/dns-query,..."` 追加任意自定义上游，
`UPSTREAM` 也可以直接填完整的 https URL。

支持 GET（`?dns=` base64url）和 POST（`application/dns-message`），并带 CORS 头。

## 部署为 Workers

```bash
npm install
npx wrangler login
npx wrangler deploy
# 可选：设置访问令牌
npx wrangler secret put TOKEN
```

部署后 DoH 地址：`https://doh-proxy.<你的子域>.workers.dev/dns-query`

## 部署为 Pages

```bash
npm install
npx wrangler pages deploy public --project-name=doh-proxy
```

环境变量在 Dashboard → Pages 项目 → Settings → Variables and Secrets 中设置。
也可以连接 Git 仓库自动部署：构建命令留空，输出目录填 `public`，`functions/` 目录会自动识别。

DoH 地址：`https://doh-proxy.pages.dev/dns-query`

## 设置访问令牌

DoH 客户端（浏览器、系统、手机）通常无法自定义请求头，所以令牌放在 URL 路径里。

1. 生成一个随机令牌：
   ```bash
   openssl rand -hex 16
   ```
2. 设置到环境变量 `TOKEN`：
   - Workers：`npx wrangler secret put TOKEN`
   - Pages：`npx wrangler pages secret put TOKEN --project-name=doh-proxy`，或在 Dashboard → Settings → Variables and Secrets 里添加 Secret
3. 重新部署后，DoH 地址变为：
   ```
   https://你的域名/<TOKEN>/dns-query
   https://你的域名/<TOKEN>/dns-query/google   # 指定上游
   ```

要点：
- 设置后，不带正确令牌的任何请求（包括首页）都返回 404，不暴露服务存在。
- `TOKEN` 支持多个，逗号分隔，如 `phone-abc,laptop-xyz`，可给不同设备分配、单独吊销（删掉对应项再更新 secret 即可）。
- 令牌比较使用常量时间算法。
- 令牌出现在 URL 中，请只在 HTTPS 下使用，且不要公开分享地址。

## 测试

```bash
# GET（wire format，查询 example.com A 记录）
curl -H 'accept: application/dns-message' \
  'https://你的域名/dns-query?dns=AAABAAABAAAAAAAAB2V4YW1wbGUDY29tAAABAAE' --output - | xxd | head

# JSON 格式（上游支持时）
curl -H 'accept: application/dns-json' \
  'https://你的域名/dns-query/google?name=example.com&type=A'
```

在浏览器中使用：Firefox → 设置 → 网络设置 → 启用基于 HTTPS 的 DNS → 自定义，填入上面的地址即可。

## 注意

- 公开部署且不设 `TOKEN` 时，任何人都能使用你的转发服务，消耗你的请求额度（Workers 免费版每天 10 万次）。建议设置 `TOKEN`。
- Cloudflare 官方条款对用途有限制，请避免把它当作大规模公共 DNS 服务运营。
