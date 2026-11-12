# 轻记：Windows 手动部署

此包部署到你自己的 Cloudflare 账号，不会使用或更新此前的 Sites 项目。系统启动后由 Cloudflare 持续托管，电脑可以关机。本地只用于构建和上传。无需 GitHub、域名或自己租服务器。

状态：已完成本地权限测试、构建和定时处理器测试，尚未使用你的 Cloudflare 账号实测发布。当前环境无法安装 Wrangler 或连接云端，因此命令和网络结果须在你的电脑执行验证。

## 1. 准备

- 安装 Node.js 24 LTS：https://nodejs.org/ ，安装后重新打开终端，运行 `node --version` 和 `npm.cmd --version`。本包的测试使用 Node 内置 SQLite，需要 Node 24。
- 注册或登录 Cloudflare：https://dash.cloudflare.com/ 。到 Storage & databases → R2 → Overview，按控制台提示开通 R2。R2 有免费用量并按量计费，不承诺永久免费；Cloudflare 可能要求完成订阅/支付资料设置。不要公开备份桶。
- 将整个 ZIP 解压到普通文件夹，例如 `D:\qingji`。确认此目录直接包含 package.json、wrangler.json、ui、worker、scripts、drizzle、tests，不要只复制 dist。
- 在该目录右键 → 在终端中打开，使用自己的 PowerShell。以下命令里的 `.cmd` 是为避免 Windows 的 npm.ps1 执行策略问题。

## 2. 安装部署工具并登录

逐条执行，看到失败就停止，不要继续后面的命令：

```powershell
npm.cmd install
npx.cmd wrangler login
```

第二条会打开浏览器，自己登录并授权 Cloudflare。它授权给你本机的官方 Wrangler；不要把 API Token 或登录凭据发到聊天里。

## 3. 创建数据库与备份桶

```powershell
npx.cmd wrangler d1 create qingji-db
```

保留输出中的 `database_id`（UUID）。如果提示同名数据库已存在，先核对控制台中该数据库是否属于本系统，不要随意删除或覆盖。不要重复建库。

```powershell
node scripts/configure-database.mjs
```

按提示粘贴刚才的 database_id，直接粘贴 UUID，不带引号。此脚本只修改本机 wrangler.json，不创建或清空数据库。

```powershell
npx.cmd wrangler r2 bucket create qingji-backups
```

默认资源名称在 wrangler.json 中，可在首次部署前更换 Worker 名、数据库名和桶名；更换后要确保创建命令和配置保持一致。若存在名为 qingji-manager 的其他 Worker，先改本包 Worker 名再部署，避免覆盖其他项目。

## 4. 创建云端数据表

```powershell
npx.cmd wrangler d1 migrations apply DB --remote
```

按提示确认。`--remote` 表示更新云端数据库；不要改成 `--local` 后误以为云端已配置好。此包只有首次建表迁移，后续修改必须追加迁移，不要编辑已应用的文件。

## 5. 测试、构建、上传并部署

```powershell
npm.cmd run build
npm.cmd test
npx.cmd wrangler deploy
```

每条命令成功后再执行下一条。最后一条会上传构建结果，并输出真实 HTTPS 地址，通常是 `https://qingji-manager.<你的子域>.workers.dev`。首次可能要求设置 workers.dev 子域，按 Wrangler 的提示操作。保存终端实际给出的地址。

此时管理员密钥尚未配置，登录接口会提示尚未配置；这不代表部署失败。

## 6. 安全初始化管理员和备份密钥

```powershell
node scripts/setup-secrets.mjs
```

脚本先检查云端密钥是否已存在，然后让你输入两次初始管理员密码（12–128 字符，输入会隐藏）。它生成密码哈希和独立 AES-256-GCM 备份密钥，经 stdin 上传到 Cloudflare Secrets，不把明文密码或密钥写入源码/文件。

用户名固定为 `admin`，密码使用你刚才自己设置的。此独立部署不沿用此前聊天里的 Sites 默认密码。第一次登录必须再修改密码。

这个脚本只用于全新安装。已有 BACKUP_KEY 或 ADMIN_INITIAL_HASH 时会拒绝再次初始化，避免轮换密钥造成历史备份无法解密。后续改密码请在网页右上角完成，勿删除或覆盖 BACKUP_KEY。备份密钥由 Cloudflare Secrets 保存，请保留这个 Worker；删除 Worker/密钥可能导致 R2 历史备份无法解密。

## 7. 核验功能和每日备份

1. 打开刚才的 HTTPS 地址，以 admin 和自设初始密码登录，完成首次改密，再重新登录。
2. 管理员 → 账号管理 → 创建两个测试成员。分别登录并改密，各提交不同记录，确认成员只看到自己记录。
3. 管理员 → 全量数据管理：确认能查看两位成员记录、编辑和导出 CSV。
4. 展开全量数据管理底部的“数据库备份” → 立即创建全量备份。成功后应显示备份时间和记录数；下载一份 JSON，检查包含 users、records 等表。JSON 包含敏感账号哈希，请私下保管。
5. Cloudflare 控制台 → Workers & Pages → qingji-manager → Settings → Triggers → Cron Triggers：核对 `0 19 * * *`。Cloudflare 的 cron 使用 UTC，对应北京时间每日 03:00。触发器配置传播可能需要最多约 15 分钟。
6. 下一次定时执行后核对网页备份时间；Cloudflare Logs 应出现 `daily_backup_completed`。也可运行 `npx.cmd wrangler tail` 查看新日志。配置存在不等于实际执行成功，须看时间和日志确认。电脑关闭、网页关闭不影响云端触发。

备份以加密对象存于私有 R2；管理员下载时服务器解密。备份为完整逻辑快照，包含所有应用表和迁移状态及建表 SQL。保留全部备份版本，未设置自动清理，会随时间累计占用存储。当前没有一键恢复按钮；恢复前需校验备份并在维护窗口操作，恢复账号后应清除 sessions，不能直接盲目覆盖生产数据。

## 8. 日常修改与再次发布

- 普通成员最多 4 个，加 admin 合计最多 5 个；停用也计入账号总数。成员不开放自助注册。
- 记录编辑、CSV 导出、账号停用、成员密码重置都在网页操作。
- 改文字/表单：ui/app.js；改样式：ui/style.css；改后端规则：worker/server.mjs；改 cron：wrangler.json。涉及字段或数据表时要同步后端校验并追加 SQL 迁移。
- 后续不需要再创建数据库、创建 R2 或运行 setup-secrets。普通代码更新依次执行：

```powershell
npm.cmd run build
npm.cmd test
npx.cmd wrangler deploy
```

如果新增迁移，在发布新代码前运行 `npx.cmd wrangler d1 migrations apply DB --remote`。保留 DB 的 database_id、R2 桶名和全部 Secrets，就会继续使用已有数据。发布代码不要求删除数据库。

## 常见问题

- npm 安装或 Wrangler 连不上：先用浏览器确认能访问 Cloudflare 和 npm。若你本机需要代理，确保代理正在运行；可仅为当前终端设置 `$env:HTTPS_PROXY = 'http://127.0.0.1:7892'`，端口应以你自己的代理实际配置为准。不要关闭证书校验或修改全局防火墙来处理。用户终端与 ChatGPT 执行沙箱相互独立，此步骤需在你自己的终端进行。
- Wrangler 提示开通 R2：到 Cloudflare 控制台完成 R2 订阅开通后再建桶，不要改成公开桶。
- database_id 无效：重新核对 Cloudflare D1 的 UUID，再运行 configure-database；不要使用 Sites 的 project_id。
- `no such table`：检查 DB 指向正确数据库，确认迁移命令带了 --remote。
- `503 管理员尚未配置`：完成第 6 步并确认上传成功；不要把明文密码写到 wrangler.json。
- `1102` / CPU time limit：查看 Cloudflare 日志与套餐限制。免费 Workers 的 CPU 配额有限，密码哈希或较大备份可能超限；如需变更套餐，先了解其价格，不能保证免费套餐足够。
- workers.dev 无法访问：先核对终端地址、部署状态和当地网络。workers.dev 在不同网络的可达性不同；必要时在 Cloudflare 控制台绑定你自己的域名，不能保证默认域名在所有地区可达。
- 已有密钥错误：初始化脚本会保护已有密钥，不能靠重跑脚本重置账号。进入网页改密码；若无法登录，先保留备份和现有密钥，再单独处理账号恢复。

## 官方参考

- https://nodejs.org/
- https://developers.cloudflare.com/workers/wrangler/commands/workers/
- https://developers.cloudflare.com/d1/wrangler-commands/
- https://developers.cloudflare.com/r2/get-started/
- https://developers.cloudflare.com/workers/configuration/cron-triggers/
- https://developers.cloudflare.com/workers/platform/limits/
