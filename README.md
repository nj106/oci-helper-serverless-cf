# OCI Helper Serverless（部署到 Cloudflare Workers）
> 原项目：https://github.com/Yohann0617/oci-helper

一键把 **OCI 抢机 / 实例管理面板**部署到你自己的 Cloudflare 账号。

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/Yohann0617/oci-helper-serverless-cf)

## 特点

- **一键部署**：Deploy 按钮会自动 Fork 仓库，向导里点一下“新建数据库”即可创建 D1，无需提前去控制台建库。
- 数据库表结构由 Worker 在**首次请求时自动创建**，无需执行任何 SQL 命令。
- 只需要一个 Worker + 一个 D1 数据库，部署后即可通过浏览器使用。

## 部署前必读

- 点击 Deploy 按钮后，Cloudflare 会把本仓库**复制一份到你自己的 GitHub 账号**（Fork，无法关闭），随后在该副本上完成构建和部署。
- 部署向导会要求你在 D1 绑定那一栏点“新建数据库”并填一个名称（例如 `oci-helper-serverless`）。确认后 Cloudflare 会自动创建该库，并把真实的 `database_id` 写回你账号下的仓库副本，不需要提前去控制台建库。
- 如果向导里没出现 D1 这一项、或部署报 `database_id` 相关错误，直接看下方「方式二」手动补一个库即可，不影响使用。

## 方式一：Deploy 按钮（自动 Fork + 向导新建 D1）（推荐🏆）

1. 点击页面顶部的 **Deploy to Cloudflare** 按钮，分别授权 GitHub 和 Cloudflare。
2. 向导会列出要创建的资源：Worker 名称默认 `oci-helper-serverless`（可改）；D1 绑定为 `DB`，在它那一项点 **新建数据库 / Create new**，名称填 `oci-helper-serverless`（可自定义，仅作标识）。向导会自动创建数据库并把 `database_id` 注入你账号下的仓库副本，全程不用离开向导。
3. 点击部署，等待构建完成。
4. 部署完成后，去 Worker → **设置 → 变量与 Secrets** 添加 `AUTH_PASSWORD`、`AUTH_SECRET`、`KEY_SECRET`（见下方「部署后必做」）。
5. 打开 Worker 域名（如 `https://oci-helper-serverless.<你的子域>.workers.dev`）登录，首次请求会自动建表。

> 以后更新：到你 Fork 的仓库点 **Sync fork → Update branch**，Cloudflare 会自动重新部署。

## 方式二：Fork + 手动创建 D1 + 连接 GitHub（可选）

<details>
    <summary> ☜ Read more 👨‍💻</summary>

适用于向导里没有新建数据库、或者你想自己控制数据库的情况：

1. **Fork 仓库**：打开 https://github.com/Yohann0617/oci-helper-serverless-cf ，点右上角 **Fork → Create fork**。
2. **创建 D1 数据库**：Cloudflare Dashboard → **存储和数据库 → D1 SQL 数据库 → 创建**，名称填 `oci-helper-serverless`（可自定义，仅作标识），创建后复制页面上的 **Database ID**。
3. **填写 database_id**：在你 Fork 的仓库里编辑 `wrangler.toml`，把占位符换成第 2 步拿到的 ID，然后提交：
   ```toml
   [[d1_databases]]
   binding = "DB"
   database_name = "oci-helper-serverless"   # 与第 2 步创建的库名一致即可
   database_id = "粘贴第 2 步的 Database ID"
   migrations_dir = "migrations"
   ```
4. **连接 GitHub 部署**：Workers & Pages → **创建 → 连接 GitHub**，授权后选择**你自己的 Fork**。构建设置保持默认即可（构建命令留空，部署命令为 `npx wrangler deploy` 或 `npm run deploy`），保存并部署。
5. **设置变量并登录**：添加 `AUTH_PASSWORD`、`AUTH_SECRET`、`KEY_SECRET` 后打开 Worker 域名登录。

</details>

## 方式三：本机命令行部署（可选）

<details>
    <summary> ☜ Read more 👨‍💻</summary>

如果你想在自己机器上部署：

```bash
git clone https://github.com/Yohann0617/oci-helper-serverless-cf.git
cd oci-helper-serverless-cf
npm install -g wrangler
wrangler login
wrangler d1 create oci-helper-serverless   # 把返回的 database_id 填进 wrangler.toml
wrangler deploy
```

</details>

## 部署后必做（安全🔒）

首次使用必须配置以下变量（未设置 `KEY_SECRET` 时所有 API 将返回 503，不会以固定常量加密敏感数据）：
1. 在 Cloudflare 控制台 → Workers & Pages → 你的 Worker → **Settings → Variables and Secrets** 添加：
   - `AUTH_PASSWORD`：你的登录密码。未设置时 Worker 会生成随机初始密码并输出到日志；**修改该变量会使已签发 token 立即失效**。
   - `AUTH_SECRET`：一串随机长字符串（仅用于签名登录 token，与加密密钥相互独立）。
   - `KEY_SECRET`：另一串随机长字符串（**必填**，用于 HKDF 派生 AES-GCM 密钥加密 OCI 私钥、私钥口令、root 密码及系统密钥；**修改后已录入的敏感配置需重新添加**）。
2. 在「系统设置」页填写 Telegram Bot Token / Chat ID（可选，用于通知）。

> 提示：本项目没有默认口令。若未设置 `AUTH_PASSWORD`，请从 Worker 日志获取随机初始密码并尽快改为自己的密码。

<img width="898" height="555" alt="image" src="https://github.com/user-attachments/assets/9d4a62fd-55be-402d-bb38-b336ac8831a7" />

## 常见问题

- **部署向导里没看到 D1 / 没点“新建数据库”就继续了**：按「方式二」手动创建一个库，把 ID 填进你 Fork 仓库的 `wrangler.toml` 后重新部署。
- **构建报 `binding DB of type d1 must have a valid database_id`**：`wrangler.toml` 里还是占位符，按方式二第 3 步填入真实 ID。
- **部署成功但页面报错 / 提示数据库未初始化**：Worker 没有名为 `DB` 的 D1 绑定。请确认仓库 `wrangler.toml` 里的 `database_id` 已填真实 ID 后重新部署（在 Dashboard 手动加的绑定会在每次部署时被配置覆盖）。
- **之前把部署命令改成过 `--config wrangler.import.toml`**：该方式已移除，请改回 `npx wrangler deploy`，并在 `wrangler.toml` 里填好 `database_id`，否则每次部署都会丢掉 D1 绑定。
- **打不开登录页但不知道密码**：未设置 `AUTH_PASSWORD` 时到 Worker 实时日志查看随机初始密码，或直接设置该变量。
- **改了 `KEY_SECRET` 后原有 OCI 配置报解密失败**：这是预期行为，加密密钥已变，需重新录入私钥等敏感信息。
- **报 `Cannot apply new-sqlite-class migration ...` (10074)**：说明你的 Worker 里已有早期的 Durable Object 命名空间。本仓库配置已改用声明式 `exports`，正常情况下不会触发；若仍出现，请用最新仓库重新同步部署，或删除旧 Worker 后重建（D1 数据不受影响，`KEY_SECRET` 需沿用）。

## 需要哪些API权限
在 OCI 控制台创建一个 IAM 用户 + API Key，把 user OCID / tenancy OCID / fingerprint / 私钥填到系统设置。建议给该用户单独的组，策略最小化，例如：

```
Allow group <你的组> to manage instance-family in tenancy
Allow group <你的组> to manage volume-family in tenancy
Allow group <你的组> to manage virtual-network-family in tenancy
Allow group <你的组> to read identities in tenancy
Allow group <你的组> to inspect limits in tenancy
```

换 IP 任务需要 `manage vnic-attachment-family in tenancy`。配额查询功能需要 `inspect limits in tenancy`（上面策略已包含）。本项目不调用容量报告接口，无需 `compute-capacity-report-family` 权限。

## 后续更新

所有部署方式都基于**你 Fork 的仓库**，更新流程一致：

1. 在你 Fork 的仓库页面点 **Sync fork → Update branch**。
2. Cloudflare 会自动重新构建部署；若没有自动触发，就到 Worker 的「部署」页点一次 **Deploy**。
3. 若同步时 `wrangler.toml` 出现冲突，**保留你自己填的 `database_id`**，只合并源仓库的其他改动。

## 隐私与数据

- OCI 私钥、私钥口令、任务 root 密码及系统密钥均用 `KEY_SECRET` 经 HKDF-SHA256 派生的 AES-256-GCM 加密后存在**你自己的 D1 数据库**，本项目不收集任何数据；缺少 `KEY_SECRET` 时拒绝服务与写入。登录 token 含 `jti` 与口令版本，改密即失效。
