# Rin 项目 Cloudflare 部署完整指南

本指南将手把手教你如何在 Cloudflare 上部署 Rin 博客项目。

## 前置要求

1. 一个 Cloudflare 账号（免费版即可）
2. 一个域名（需要将 DNS 解析到 Cloudflare）
3. 一个 GitHub 账号（用于 OAuth 登录）
4. 本地安装 Node.js（推荐 18+ 版本）和 Bun
5. 安装 Wrangler CLI 工具

## 第一步：准备 Cloudflare 账号和域名

### 1.1 注册 Cloudflare 账号
1. 访问 https://dash.cloudflare.com/sign-up
2. 注册一个免费账号

### 1.2 添加域名到 Cloudflare
1. 登录 Cloudflare Dashboard
2. 点击 "添加站点"
3. 输入你的域名（例如：example.com）
4. 选择免费计划
5. 按照提示修改域名的 DNS 服务器到 Cloudflare 提供的地址
6. 等待 DNS 生效（通常需要几分钟到几小时）

## 第二步：安装依赖和工具

### 2.1 安装 Bun（如果还没安装）
```bash
curl -fsSL https://bun.sh/install | bash
```

### 2.2 安装 Wrangler CLI
```bash
npm install -g wrangler
# 或者使用 bun
bun install -g wrangler
```

### 2.3 登录 Wrangler
```bash
wrangler login
```
这会打开浏览器，授权 Wrangler 访问你的 Cloudflare 账号。

### 2.4 克隆项目并安装依赖
```bash
cd /data2/zhanghuaao/project/Rin
bun install
```

## 第三步：创建 Cloudflare 资源

### 3.1 创建 D1 数据库
```bash
wrangler d1 create rin
```

执行后会输出类似这样的信息：
```
✅ Successfully created DB 'rin'!

[[d1_databases]]
binding = "DB"
database_name = "rin"
database_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

**重要：记下 `database_id`，后面会用到！**

### 3.2 创建 R2 存储桶（用于存储图片）
```bash
wrangler r2 bucket create rin-images
```

### 3.3 创建 R2 访问密钥（获取 S3_ACCESS_KEY_ID 和 S3_SECRET_ACCESS_KEY）

**详细步骤：**

1. 访问 Cloudflare Dashboard：https://dash.cloudflare.com/
2. 在左侧菜单中找到并点击 **"R2"**
3. 点击右上角的 **"管理 R2 API 令牌"** 按钮
4. 点击 **"创建 API 令牌"** 按钮
5. 配置令牌：
   - **令牌名称**：输入一个名称，例如 "rin-blog-token"
   - **权限**：选择 **"对象读和写"** 或 **"管理员读和写"**
   - **TTL（生存时间）**：可以选择 "永久" 或设置过期时间
   - **指定存储桶**（可选）：可以限制只访问 `rin-images` 存储桶
6. 点击 **"创建 API 令牌"**
7. **重要：立即复制并保存以下信息**（关闭后无法再查看 Secret Access Key）：
   ```
   Access Key ID: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   Secret Access Key: yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy
   ```
   - `Access Key ID` 就是 **S3_ACCESS_KEY_ID**
   - `Secret Access Key` 就是 **S3_SECRET_ACCESS_KEY**

**⚠️ 注意：**
- Secret Access Key 只会显示一次，请务必保存好
- 如果忘记了，需要删除旧令牌并创建新的
- 这两个密钥将在第五步中通过 `wrangler secret put` 命令设置

### 3.4 获取 R2 端点信息

**获取 S3_ENDPOINT：**

1. 在 Cloudflare Dashboard 的 R2 页面
2. 点击你创建的 `rin-images` 存储桶
3. 在右侧找到 **"S3 API"** 部分
4. 复制 **"S3 API 端点"**，格式类似：
   ```
   https://<account-id>.r2.cloudflarestorage.com
   ```
   这就是你的 **S3_ENDPOINT**

**配置 S3_ACCESS_HOST（图片公开访问地址）：**

你有两个选择：

**选项 1：使用 R2.dev 子域（简单，但有限制）**
1. 在存储桶设置中，找到 **"公开访问"** 部分
2. 点击 **"允许访问"** 启用 R2.dev 子域
3. 会生成一个类似 `https://pub-xxxxx.r2.dev` 的地址
4. 这个地址就是你的 **S3_ACCESS_HOST**

**选项 2：使用自定义域名（推荐）**
1. 在存储桶设置中，找到 **"自定义域"** 部分
2. 点击 **"连接域"**
3. 输入你的子域名，例如：`img.your-domain.com`
4. Cloudflare 会自动创建 DNS 记录
5. 使用 `https://img.your-domain.com` 作为你的 **S3_ACCESS_HOST**

**示例配置：**
```toml
S3_ENDPOINT = "https://abc123def456.r2.cloudflarestorage.com"
S3_ACCESS_HOST = "https://img.yourdomain.com"  # 或 https://pub-xxxxx.r2.dev
S3_BUCKET = "rin-images"
```

## 第四步：配置 GitHub OAuth（获取 GITHUB_CLIENT_ID 和 GITHUB_CLIENT_SECRET）

### 4.1 创建 GitHub OAuth 应用

**详细步骤：**

1. **登录 GitHub** 并访问开发者设置页面：
   - 直接访问：https://github.com/settings/developers
   - 或者：GitHub 右上角头像 → Settings → 左侧菜单最下方 "Developer settings"

2. **创建 OAuth 应用**：
   - 点击左侧 **"OAuth Apps"**
   - 点击右上角 **"New OAuth App"** 按钮（或 "Register a new application"）

3. **填写应用信息**：
   ```
   Application name: Rin Blog
   （应用名称，可以自定义，例如：我的博客、My Rin Blog 等）
   
   Homepage URL: https://your-domain.com
   （你的博客域名，例如：https://blog.example.com）
   
   Application description: （可选）
   （应用描述，可以写：My personal blog powered by Rin）
   
   Authorization callback URL: https://your-domain.com/api/oauth
   （回调地址，非常重要！格式必须是：你的域名/api/oauth）
   ```

4. **点击 "Register application"** 按钮

5. **获取 Client ID**：
   - 创建成功后，会自动跳转到应用详情页
   - 页面上会显示 **"Client ID"**
   - 复制这个 Client ID，这就是 **GITHUB_CLIENT_ID**
   ```
   Client ID: Iv1.a1b2c3d4e5f6g7h8
   ```

6. **生成 Client Secret**：
   - 在同一页面，找到 **"Client secrets"** 部分
   - 点击 **"Generate a new client secret"** 按钮
   - 可能需要输入 GitHub 密码或进行二次验证
   - 生成后会显示 **Client Secret**（只显示一次！）
   - 立即复制保存，这就是 **GITHUB_CLIENT_SECRET**
   ```
   Client Secret: a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0
   ```

**⚠️ 重要注意事项：**

- **Client Secret 只显示一次**，刷新页面后就看不到了
- 如果忘记保存，需要重新生成新的 Secret（旧的会失效）
- **回调 URL 必须准确**，格式：`https://your-domain.com/api/oauth`
- 如果是本地开发，可以先用 `http://localhost:5173/api/oauth`，部署后再修改
- 部署后如果修改了域名，记得回来更新 OAuth 应用的 URL

**示例配置：**
```
GITHUB_CLIENT_ID: Iv1.a1b2c3d4e5f6g7h8
GITHUB_CLIENT_SECRET: a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0
Homepage URL: https://blog.example.com
Callback URL: https://blog.example.com/api/oauth
```

## 第五步：配置项目

### 5.1 复制配置文件
```bash
cp wrangler.example.toml wrangler.toml
```

### 5.2 编辑 wrangler.toml
打开 `wrangler.toml` 文件，修改以下内容：

```toml
#:schema node_modules/wrangler/config-schema.json
name = "rin-server"
main = "server/src/_worker.ts"
compatibility_date = "2024-05-29"
node_compat = true

[triggers]
crons = ["*/20 * * * *"]

[vars]
FRONTEND_URL = "https://your-domain.com"  # 改成你的域名
S3_FOLDER = "images/"
S3_CACHE_FOLDER = "cache/"
S3_REGION = "auto"
S3_ENDPOINT = "https://<account-id>.r2.cloudflarestorage.com"  # 你的 R2 端点
S3_ACCESS_HOST = "https://img.your-domain.com"  # 图片访问域名
S3_BUCKET = "rin-images"  # 你的 R2 存储桶名称
S3_FORCE_PATH_STYLE = "false"
WEBHOOK_URL = ""  # 可选：用于评论通知的 Webhook URL
RSS_TITLE = "Your Blog Title"  # 你的博客标题
RSS_DESCRIPTION = "Your Blog Description"  # 你的博客描述

[[d1_databases]]
binding = "DB"
database_name = "rin"
database_id = "你的数据库ID"  # 第三步创建的 D1 数据库 ID

[[r2_buckets]]
binding = "BUCKET"
bucket_name = "rin-images"
```

### 5.3 设置环境变量（密钥信息）

这些敏感信息不应该写在 `wrangler.toml` 中，而是使用 Cloudflare 的 Secrets 功能：

```bash
# 设置 GitHub OAuth 密钥（注意：使用 RIN_ 前缀）
wrangler secret put RIN_GITHUB_CLIENT_ID
# 粘贴你在第四步获取的 GitHub Client ID

wrangler secret put RIN_GITHUB_CLIENT_SECRET
# 粘贴你在第四步获取的 GitHub Client Secret

# 设置 R2 访问密钥
wrangler secret put S3_ACCESS_KEY_ID
# 粘贴你在第三步获取的 R2 Access Key ID

wrangler secret put S3_SECRET_ACCESS_KEY
# 粘贴你在第三步获取的 R2 Secret Access Key
```

**💡 为什么使用 `RIN_` 前缀？**

GitHub Actions 不允许环境变量以 `GITHUB_` 开头，所以项目使用 `RIN_GITHUB_CLIENT_ID` 和 `RIN_GITHUB_CLIENT_SECRET`。代码会优先使用带 `RIN_` 前缀的变量，如果没有则回退到不带前缀的版本。

**设置步骤说明：**

1. 在项目根目录执行上述命令
2. 每个命令执行后会提示你输入对应的值
3. 粘贴对应的密钥后按回车
4. 密钥会安全地存储在 Cloudflare 中，不会出现在代码里

## 第六步：初始化数据库

### 6.1 查找数据库迁移文件
```bash
ls server/drizzle/migrations/
```

### 6.2 执行数据库迁移
```bash
# 方法一：使用项目提供的迁移脚本
bun run cf-deploy

# 方法二：手动执行迁移（如果有 SQL 文件）
# 找到 server/drizzle/migrations/ 目录下的 SQL 文件
wrangler d1 execute rin --file=server/drizzle/migrations/0000_xxx.sql
```

## 第七步：构建和部署

### 7.1 构建项目
```bash
bun run b
```

这会构建前端（client）和后端（server）代码。

### 7.2 部署到 Cloudflare
```bash
# 部署 Workers（后端）
wrangler deploy

# 部署 Pages（前端）
cd client
wrangler pages deploy dist --project-name=rin-blog
```

**首次部署 Pages 时：**
- 会提示你创建一个新项目
- 输入项目名称（例如：rin-blog）
- 确认部署

### 7.3 配置 Pages 环境变量
1. 访问 Cloudflare Dashboard → Pages
2. 选择你的项目（rin-blog）
3. 进入 "设置" → "环境变量"
4. 添加以下变量：
   - `VITE_API_URL`: `https://rin-server.your-account.workers.dev`（你的 Worker 地址）

重新构建并部署：
```bash
cd client
bun run build
wrangler pages deploy dist --project-name=rin-blog
```

## 第八步：配置自定义域名

### 8.1 为 Pages 配置域名
1. 在 Cloudflare Dashboard → Pages → 你的项目
2. 进入 "自定义域"
3. 点击 "设置自定义域"
4. 输入你的域名（例如：blog.example.com 或 example.com）
5. Cloudflare 会自动创建 DNS 记录

### 8.2 为 Worker 配置路由（可选）
如果你想让 API 也使用自定义域名：
1. 在 Cloudflare Dashboard → Workers & Pages
2. 选择你的 Worker（rin-server）
3. 进入 "触发器" → "路由"
4. 添加路由：`your-domain.com/api/*`

### 8.3 为 R2 配置自定义域名（图片访问）
1. 在 Cloudflare Dashboard → R2
2. 选择你的存储桶（rin-images）
3. 进入 "设置" → "自定义域"
4. 添加域名（例如：img.your-domain.com）

## 第九步：验证部署

### 9.1 访问你的网站
打开浏览器，访问你配置的域名（例如：https://your-domain.com）

### 9.2 测试 GitHub 登录
1. 点击登录按钮
2. 使用 GitHub 账号授权
3. 第一个登录的用户将自动获得管理员权限

### 9.3 测试功能
- 尝试创建一篇文章
- 上传图片
- 发布文章
- 查看 RSS 订阅

## 第十步：后续更新

当你修改代码后，重新部署：

```bash
# 1. 构建项目
bun run b

# 2. 部署 Worker
wrangler deploy

# 3. 部署 Pages
cd client
wrangler pages deploy dist --project-name=rin-blog
```

## 常见问题

### Q1: 数据库迁移失败怎么办？
```bash
# 查看数据库列表
wrangler d1 list

# 查看数据库信息
wrangler d1 info rin

# 手动执行 SQL
wrangler d1 execute rin --command="SELECT * FROM sqlite_master WHERE type='table';"
```

### Q2: 图片上传失败？
- 检查 R2 存储桶是否创建成功
- 检查 S3 密钥是否正确设置
- 检查 `wrangler.toml` 中的 R2 配置

### Q3: GitHub OAuth 登录失败？
- 检查 GitHub OAuth 应用的回调 URL 是否正确
- 检查 Secrets 中的 `GITHUB_CLIENT_ID` 和 `GITHUB_CLIENT_SECRET` 是否正确
- 确保域名已经生效

### Q4: 如何查看 Worker 日志？
```bash
wrangler tail
```

### Q5: 如何重置数据库？
```bash
# 删除旧数据库
wrangler d1 delete rin

# 创建新数据库
wrangler d1 create rin

# 重新执行迁移
bun run cf-deploy
```

## 有用的命令

```bash
# 本地开发
bun run dev

# 查看 Worker 日志
wrangler tail

# 查看 D1 数据库
wrangler d1 execute rin --command="SELECT * FROM users;"

# 查看 R2 存储桶列表
wrangler r2 bucket list

# 查看 R2 存储桶内容
wrangler r2 object list rin-images

# 查看已设置的 Secrets
wrangler secret list
```

## 参考资源

- [Cloudflare Workers 文档](https://developers.cloudflare.com/workers/)
- [Cloudflare Pages 文档](https://developers.cloudflare.com/pages/)
- [Cloudflare D1 文档](https://developers.cloudflare.com/d1/)
- [Cloudflare R2 文档](https://developers.cloudflare.com/r2/)
- [Wrangler CLI 文档](https://developers.cloudflare.com/workers/wrangler/)
- [Rin 官方文档](https://docs.openrin.org)

## 总结

恭喜！你已经成功在 Cloudflare 上部署了 Rin 博客。现在你拥有一个：
- ✅ 无需服务器的博客系统
- ✅ 全球 CDN 加速
- ✅ 免费的数据库和存储
- ✅ 自动 HTTPS
- ✅ 无限流量（免费版有限制但足够个人使用）

享受你的博客吧！🎉

