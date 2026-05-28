---
id: 6a169c6d6f03d
name: NodeJS
tags: []
updated_at: 2026-05-28T09:19:09.817612Z
---

## 版本管理

| 工具 | 安装 | 用法 |
|--|----------|------|
| [nvm](https://github.com/nvm-sh/nvm) | `curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh \| bash` | `nvm install 22` |
| [fnm](https://github.com/Schniz/fnm) | `curl -fsSL https://fnm.vercel.app/install \| bash` | 更快的 nvm 替代品 |
| [volta](https://volta.sh) | `curl https://get.volta.sh \| bash` | 按项目锁定版本 |

```bash
nvm install --lts          # 安装最新 LTS 版本
nvm alias default 22       # 设置默认版本
node -v && npm -v
```

## npm 常用命令

```bash
npm init -y                 # 生成 package.json
npm install <pkg>           # 添加依赖
npm install -D <pkg>        # 添加开发依赖
npm install -g <pkg>        # 全局安装
npm uninstall <pkg>
npm update
npm outdated                # 查看过时依赖
npm audit                   # 安全检查
npm audit fix
npm ci                      # 从 lockfile 全新安装（适合 CI）
npm run <script>
npm exec <pkg>              # 无需安装直接运行（类 npx）
npx <pkg>                   # 无需安装直接运行
```

## package.json 关键字段

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "main": "index.js",
  "type": "module",
  "scripts": {
    "start": "node index.js",
    "dev": "node --watch index.js",
    "test": "node --test",
    "build": "tsc"
  },
  "engines": { "node": ">=20" },
  "dependencies": {},
  "devDependencies": {}
}
```

> `"type": "module"` 让整个包使用 ESM（`import`/`export`）语法。

## ESM vs CommonJS

| | ESM | CommonJS |
|--|-----|----------|
| 语法 | `import` / `export` | `require()` / `module.exports` |
| 文件扩展名 | `.mjs` 或 `"type":"module"` | `.cjs` 或默认 |
| 顶层 await | ✅ | ❌ |
| Tree-shaking | ✅ | ❌ |

## 内置模块（Node 18+）

```js
import { readFile, writeFile } from 'node:fs/promises';
import { join, dirname } from 'node:path';
import { fileURLToPath } from 'node:url';
import { createServer } from 'node:http';
import { env } from 'node:process';

// ESM 中等价于 __dirname
const __dirname = dirname(fileURLToPath(import.meta.url));
```

## 实用内置功能（Node 20+）

| 功能 | API |
|------|-----|
| 测试运行器 | `node --test` / `import { test } from 'node:test'` |
| 监听模式 | `node --watch index.js` |
| 加载 `.env` | `node --env-file=.env index.js` |
| Fetch | `fetch()`（全局，无需导入）|
| Web Streams | `ReadableStream`、`WritableStream`（全局）|
| 加密 | `crypto.randomUUID()`（全局）|
| 权限控制 | `--allow-fs-read`、`--allow-env`（实验性）|

## npm Workspaces（Monorepo）

```json
// 根目录 package.json
{
  "workspaces": ["packages/*", "apps/*"]
}
```

```bash
npm install                       # 安装所有工作区依赖
npm run build -w packages/ui      # 在指定工作区运行脚本
npm run test --workspaces         # 在所有工作区运行
```

## 常用代码模式

```js
// 优雅退出
process.on('SIGTERM', async () => {
  await server.close();
  await db.disconnect();
  process.exit(0);
});

// 捕获未处理的 Promise 错误
process.on('unhandledRejection', (err) => {
  console.error(err);
  process.exit(1);
});
```

## 常用生态包

| 分类 | 推荐包 |
|------|----------------|
| Web 框架 | express, fastify, hono, koa |
| HTTP 客户端 | undici（内置）, axios, got |
| 数据校验 | zod, joi, yup |
| ORM/数据库 | prisma, drizzle, pg, mongoose |
| 认证 | passport, lucia, better-auth |
| 任务队列 | bullmq, pg-boss |
| 测试 | vitest, jest, node:test |
| 打包工具 | esbuild, rollup, tsup |
| 类型检查 | typescript, tsx |
| 代码规范 | eslint, prettier, biome |
| 环境变量 | dotenv, @t3-oss/env-core |

## npm 替代工具

| 工具 | 说明 |
|------|----------------|
| `pnpm` | 节省磁盘空间，速度快，依赖严格隔离 |
| `yarn` (berry) | plug-n-play，零安装模式 |
| `bun` | JS 运行时 + 包管理器，极快 |

```bash
# pnpm
pnpm add <pkg>
pnpm dlx <pkg>        # 类似 npx

# bun
bun add <pkg>
bun run <script>
bunx <pkg>
```

## 调试

```bash
node --inspect index.js          # 开启 DevTools 调试
node --inspect-brk index.js     # 在第一行暂停
# 然后打开 chrome://inspect
```

```bash
# 性能分析
node --prof index.js
node --prof-process isolate-*.log > profile.txt
```

## `.npmrc` 常用配置

```ini
save-exact=true          # 锁定精确版本号
engine-strict=true       # 强制 engines 字段检查
legacy-peer-deps=false
fund=false
audit-level=high
```

---

## 发布 npm 包：`@oh-my/ui` 发布流程

> 使用 [`just`](https://github.com/casey/just) 作为任务运行器（类似 Makefile）。
> 先安装：`cargo install just` 或 `brew install just`。

### 流程总览

```
登录 → 检查内容 → 本地测试 → 升版本 → 发布
```

### 各命令说明

| 命令 | 作用 |
|------|----------|
| `just whoami` | 检查是否已登录 npm（未登录先执行 `npm login`）|
| `just view` | 查看当前 npm 上已发布的版本号 |
| `just pack` | **干跑**：列出哪些文件会被打包，不生成文件 |
| `just pack-real` | 生成真实的 `.tgz` 压缩包，用于本地测试 |
| `just link-into ../other-app` | 把本地包安装进另一个项目，验证实际效果 |
| `just bump patch` | 升级补丁版本（1.0.0 → 1.0.1），也支持 `minor`/`major` |
| `just publish` | 发布到 npm（需要已登录且是 `@oh-my` 组织成员）|
| `just release` | 一键完成：升版本 + 发布 |
| `just init-scope` | **仅首次**：在 npm 创建 `@oh-my` 组织（免费公开包）|

### 首次发布完整步骤

```bash
# 1. 登录 npm
npm login

# 2. （仅一次）创建 @oh-my 组织
just init-scope

# 3. 检查打包内容是否正确
just pack

# 4. 本地联调测试（可选）
just pack-real
just link-into ../my-consumer-app

# 5. 升版本并发布
just release          # 默认升 patch 版本
just release minor    # 升 minor 版本
```

### 关键前提

- **npm 账号**：须在 [npmjs.com](https://www.npmjs.com) 注册
- **组织成员**：须加入 `@oh-my` 组织，或自己是创建者
- **`--access public`**：scoped 包（`@scope/pkg`）默认私有，需显式声明为公开
- **git 标签**：`just bump` 会自动创建 git tag（需在 git 仓库中）

---

## 在 Flask 项目中集成 `@dennisl0731/oh-my-ui`

> 以 `oh-my-clipboard` 为例，展示如何通过 npm 消费共享 CSS/JS 包，
> 替换原先手动 vendor 的副本，并说明后续如何升级。
>
> **场景**：Python Flask 应用，无 JS 构建系统，但需要 npm 发布包中的 warm-paper 设计 token。

### 快速上手

```sh
just deps      # → npm install --omit=dev
just build-up  # → docker compose up -d --build
```

Flask 将包文件挂载在 `/omi/ui/*`（如 `/omi/ui/css/omi.css`），
实际路径解析自 `node_modules/@dennisl0731/oh-my-ui/`。

### 1. 包提供的内容

`@dennisl0731/oh-my-ui` 通过 `package.json` 的 `exports` 字段暴露三个入口：

| 导出 | 实际文件 | 说明 |
|------|----------|------|
| `./omi.css` | `css/omi.css` | 设计 token（`--omi-*`）+ 基础样式 + 组件 |
| `./nav.js` | `js/nav.js` | 顶部导航渲染器，自动添加 `body.service-<name>` |
| `./tailwind-preset` | `tailwind-preset.js` | Tailwind 预设（仅用于 Tailwind 项目）|

对 Flask 静态文件服务而言，只需关注 `omi.css` 和 `nav.js`。

### 2. 需要修改的四个文件

**`package.json`（新建）** — 仅声明依赖，无构建脚本：

```json
{
  "name": "oh-my-clipboard",
  "version": "0.1.0",
  "private": true,
  "dependencies": {
    "@dennisl0731/oh-my-ui": "^0.2.0"
  }
}
```

**`.gitignore`** — 排除 `node_modules/`：

```
node_modules/
__pycache__/
*.pyc
data/
.env
```

**`app.py`** — 按优先级查找包目录，路由提供静态文件：

```python
_HERE = os.path.dirname(__file__)
_UI_CANDIDATES = [
    os.environ.get('OH_MY_UI_DIR', ''),                              # 显式指定
    '/opt/oh-my-ui',                                                 # 容器路径（软链接）
    os.path.join(_HERE, 'node_modules', '@dennisl0731', 'oh-my-ui'),# 本地开发回退
]
OH_MY_UI_DIR = next(
    (c for c in _UI_CANDIDATES if c and os.path.isfile(os.path.join(c, 'css', 'omi.css'))),
    None,
)

@app.route('/omi/ui/<path:fname>')
def omi_ui(fname):
    if not OH_MY_UI_DIR:
        return 'oh-my-ui not installed', 404
    return send_from_directory(OH_MY_UI_DIR, fname)
```

模板中引用方式（升级时记得更新 `?v=` 参数以强制浏览器重新请求）：

```html
<link rel="stylesheet" href="/omi/ui/css/omi.css?v=20260528a">
<script src="/omi/ui/js/nav.js?v=20260512a" data-service="clipboard"></script>
```

**`Dockerfile`** — 在镜像构建阶段执行 `npm install`：

```dockerfile
FROM python:3.11-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl git openssh-client nodejs npm \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY package.json /opt/oh-my-ui-pkg/package.json
RUN cd /opt/oh-my-ui-pkg && npm install --omit=dev --no-fund --no-audit \
    && ln -s /opt/oh-my-ui-pkg/node_modules/@dennisl0731/oh-my-ui /opt/oh-my-ui
ENV OH_MY_UI_DIR=/opt/oh-my-ui

COPY app.py shared/ routes/ templates/ static/ ./
```

> **为什么用软链接**：`/opt/oh-my-ui` 路径短且稳定，升级包版本时无需改 `OH_MY_UI_DIR`。

**`justfile`** — 本地开发一键化：

```just
deps:
    npm install --omit=dev --no-fund --no-audit

dev: deps
    UPLOAD_FOLDER=$(pwd)/data/uploads \
    SETTINGS_DB=$(pwd)/data/oh-my-clipboard.db \
    PORT=5007 python app.py

build-up:
    cd .. && docker compose up -d --build oh-my-clipboard
```

### 3. `service-*` 配色系统与陷阱

`nav.js` 读取 `<script>` 标签的 `data-service` 属性，向 `<body>` 添加 `service-<name>` 类名。
`omi.css` 为固定几个服务预定义了配色：

```css
body.service-image { --omi-accent: #c08a4a; }  /* 赭红 */
body.service-slide { --omi-accent: #5a8da8; }  /* 冷蓝 */
body.service-sage  { --omi-accent: #8fbc8f; }  /* 鼠尾草绿 */
/* 默认（无类名）: --omi-accent: #8a7042;        土棕 */
```

**陷阱**：若 `data-service="clipboard"` 而包内没有对应的 `body.service-clipboard` 块，则继承默认配色。0.1.0 是鼠尾草绿，0.2.0 是土棕——升级时配色可能悄悄变化。

**三种应对方式：**

1. **接受默认** — 最省事，包的默认色已经过设计。
2. **借用已有配色** — 改 `data-service` 为 `image` / `slide` / `sage`（注意影响导航逻辑）。
3. **自定义 token**（推荐）— 在模板 `<style>` 中覆盖：

```css
body.service-clipboard {
  --omi-accent:        #6b7d8f;
  --omi-accent-deep:   #4f6273;
  --omi-accent-deeper: #354858;
  --omi-accent-ring:   rgba(107, 125, 143, 0.28);
  --omi-tint-a:        rgba(170, 188, 205, 0.45);
  --omi-tint-b:        rgba(195, 205, 218, 0.30);
}
```

> 始终覆盖 **token**（`--omi-accent`），不要直接改组件 CSS——这样能在包升级后依然生效。

### 4. 保留 `nav.js` 自定义行为

从 vendor 版本切换到发布包时，两处行为可能丢失：

1. 包默认 `services[]` 中没有的服务条目（如 `clipboard`）
2. vendor 版本额外添加的品牌文字点击跳转

通过 `window.__OMI_NAV__` 注入服务列表（须在 `nav.js` 加载前设置）：

```html
<script>
  window.__OMI_NAV__ = [
    { id: 'image',     name: 'image',     port: 5006, label: '🎨 image' },
    { id: 'clipboard', name: 'clipboard', port: 5007, label: '📋 clipboard' },
    { id: 'slide',     name: 'slide',     port: 5008, label: '🖼 slide' },
    { id: 'skill',     name: 'skill',     port: 5009, label: '🧠 skill' },
  ];
  document.addEventListener('DOMContentLoaded', () => {
    const brand = document.querySelector('.omi-brand');
    if (brand && !brand.closest('a')) {
      brand.style.cursor = 'pointer';
      brand.addEventListener('click', () => { location.href = '/'; });
    }
  });
</script>
<script src="/omi/ui/js/nav.js?v=..." data-service="clipboard"></script>
```

### 5. 升级到新版本

```sh
# 1. 修改 package.json 中的版本号
sed -i 's/"@dennisl0731\/oh-my-ui": ".*"/"@dennisl0731\/oh-my-ui": "^0.3.0"/' package.json

# 2. 更新 node_modules
just deps

# 3. 确认配色变化（重点检查 colorway 块和默认 accent）
grep -nE "^body\.service-" node_modules/@dennisl0731/oh-my-ui/css/omi.css
grep -A1 '^:root' node_modules/@dennisl0731/oh-my-ui/css/omi.css | grep -- '--omi-accent:'

# 4. 更新模板中 /omi/ui/* 引用的 ?v= 参数

# 5. 重新构建并启动容器
just build-up
```

部署后验证：

```sh
curl -fsS http://localhost:5007/omi/ui/css/omi.css | wc -l
curl -fsS http://localhost:5007/omi/ui/js/nav.js  | wc -l
```

行数与 `node_modules/` 内文件不符，说明浏览器在缓存，或容器软链接已过期。

### 6. 踩过的坑

| 问题 | 解决 |
|------|------|
| `exports` 字段阻止直接 `require('…/package.json')` | 用 `fs.readFileSync` 直接读文件路径 |
| 容器以 root 创建的 SQLite 文件，本地 Python 无法写入 | 本地验证用 `SETTINGS_DB=/tmp/test.db` |
| `omi.css` 没有深色主题 | 在模板 `<style>` 里加 `body.theme-dark` 覆盖，JS 切换类名 |
| 直接修改 `node_modules/` | 禁止——`npm install` 会覆盖，所有覆盖只写在模板 `<style>` 中 |

### 7. 文件结构一览

```
oh-my-clipboard/
├── package.json              ← 声明 @dennisl0731/oh-my-ui
├── package-lock.json         ← npm install 自动生成
├── .gitignore                ← 排除 node_modules/
├── node_modules/             ← 不入 git；由 npm install 创建
│   └── @dennisl0731/oh-my-ui/{css/omi.css, js/nav.js, ...}
├── Dockerfile                ← 执行 npm install + 软链接到 /opt/oh-my-ui
├── justfile                  ← deps / dev / build-up
├── app.py                    ← /omi/ui/<path> 路由服务 OH_MY_UI_DIR
└── templates/
    ├── clipboard.html        ← 引用 /omi/ui/css/omi.css 和 nav.js
    └── humanizer.html        ← 同上
```

整个集成只涉及三个粘合文件（`package.json`、`Dockerfile`、`app.py`）、
一个 gitignore、一个 DX 文件（`justfile`），以及升级时更新模板的 `?v=` 参数或自定义配色 token。
