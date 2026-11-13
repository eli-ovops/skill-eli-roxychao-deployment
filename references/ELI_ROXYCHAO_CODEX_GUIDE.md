# eli.roxychao.com - Codex 服务器工作说明

基础设施取证版本：2026-07-24  
治理修订：2026-08-09

## 1. 目标与身份

本文件供 Eli 的 Codex 在维护 `eli.roxychao.com` 时使用。

## 0. 临时预览分级与硬门禁来源

按项目 `site.yaml` 和当前授权把工作分为 `static`、`code`、`admin-demo`、`data-migration`。`static` 仅处理静态输出与 `static-current`；`code` 仅处理不含数据面的 release 与 `current`；`admin-demo` 仅在隔离预览数据上做受影响后台流程 focused smoke；`data-migration` 指共享 SQLite/WAL/SHM、media、schema 或数据写入。低级授权不外推到高级，且任何级别都不授权客户生产、其他站点、账号/权限变更、删除、Server Owner 确认替代或服务器配置。

新增硬阻断必须引用本指南的具体节、全局 AGENTS §1.3、项目 `site.yaml` 或 DS-002B 的具体条款，并陈述当前分级和事实；无条款与事实支持的项目只记录为风险或待确认。正常依赖路径始终是精确 `packageManager`、匹配 lockfile 和 `corepack pnpm install --frozen-lockfile`（或锁文件对应命令）。网络不可用时，只有用户或实时契约明确要求，才可使用来源、版本、SHA 和范围均可核验的离线闭包；不得默认构造离线工具链。

- 域名：`https://eli.roxychao.com`
- 服务器：广州 `[REDACTED_HOST]`
- SSH 用户：`eli`
- SSH 端口：`22`
- 登录方式：仅 SSH 公钥，不支持密码登录
- 允许的服务器来源 IP：`[REDACTED_HOST]`、`[REDACTED_HOST]`
- 连接 identity 必须由当前授权或本机受保护配置实时确认。本文不登记实际 identity 文件名或路径；未完成该确认前，禁止尝试连接或发布。
- 普通连接命令模板（仅在上述确认后填写）：

```bash
ssh -i "<受保护配置中已确认的实际 identity 路径>" \
  -o IdentitiesOnly=yes \
  eli@[REDACTED_HOST]
```

服务器已登记的公钥摘要：

- Key type：`ED25519`
- SHA256 fingerprint：`SHA256:uZYtDh9a1Ps2o780IVKmyV20ix9cJnVgByvC1MoLrrQ`
- 限制：只允许上述两个来源 IP；禁止 agent forwarding、X11 forwarding、SSH 端口转发和 user rc
- Shell：允许普通命令与 PTY；没有 forced command

`[REDACTED_HOST]` 是局域网地址，公网服务器不会把它识别为 SSH 来源。`127.0.0.1:7897` 是 Eli 电脑上的本地代理端口；只有 SSH 客户端确实通过该代理连接时，服务器才会看到代理公网出口，而不会看到 `127.0.0.1`。

不得读取、输出、复制或提交任何私钥、密码、token、cookie、环境变量值或数据库业务内容。

## 2. 站点运行结构

```text
/srv/eli-site
├── workspace                 Git 源码工作区
├── incoming                  待处理输入
├── staging                   Next/Payload 构建暂存区
├── current                   当前 Next 后端 release 的符号链接
├── releases                  Next/Payload 独立 release
├── static-staging            静态前端构建暂存区
├── static-current            当前静态前端 release 的符号链接
├── static-releases           静态前端独立 release
├── logs                      本站发布日志
├── static-logs               本站静态发布日志
├── state                     本站状态文件
└── shared
    ├── data/site.sqlite      本站 SQLite
    ├── media                 Payload media/uploads
    ├── cache                 共享构建缓存
    └── backups               数据与 media 备份
```

以下运行信息是 2026-07-24 的基础设施取证记录，不自动代表当前值；每次任务仍须先执行第 5 节实时只读 preflight：

- 应用：静态前台 + Next.js/Payload 后端
- systemd unit：`eli-site.service`
- 运行用户：`eli:eli`
- Working directory：`/srv/eli-site/current`
- 启动入口：`/usr/bin/node server.js`
- 内部监听：`127.0.0.1:3140`
- 环境配置：`/etc/eli-site/eli-site.env`
- 公网入口由 Caddy 处理，但 Eli 无权修改 Caddy、DNS 或 SSL

2026-07-24 观察到的历史指向（仅作取证样例，不得作为当前 release）：

```text
/srv/eli-site/current
  -> /srv/eli-site/releases/20260711-eli-version-b-static-v2

/srv/eli-site/static-current
  -> /srv/eli-site/static-releases/20260718-origin-vector-scheme3-l0-v1
```

现有完整 Next/Payload 源码 release：

```text
/srv/eli-site/releases/payload-full-20260711-v1
```

该 release 在 2026-07-24 取证时包含 Payload `3.86.0`、Next.js `16.2.6`、SQLite adapter、源码、migration、lockfile 和 standalone 构建产物。它是历史参考，不代表当前版本或已经上线。

该历史 release 没有声明 `packageManager` 版本。重新构建前必须从项目 README、CI 或已验证构建记录确认 pnpm 版本；不得直接使用服务器当前全局 pnpm 11 改写 lockfile。

实际指向必须每次实时确认：

```bash
readlink -f /srv/eli-site/current
readlink -f /srv/eli-site/static-current
```

SQLite 的实际活动路径仍应结合 `/etc/eli-site/eli-site.env` 和当前源码确认。可以读写本站 env，但不得把其内容打印到 Codex 对话、日志、Git 或制品中。

## 3. Eli 已具备的权限

Eli 可以在本站范围内：

- clone、pull 和管理 `/srv/eli-site/workspace` 内的 Git 工作副本
- 修改 Next.js 前端和 Payload 后端代码
- 按项目 lockfile 安装依赖并执行构建
- 管理 `releases/current` 和 `static-releases/static-current`
- 读写本站 SQLite、WAL、SHM 和 migration 文件
- 读写本站 media/uploads、env、日志、缓存和暂存目录
- 查看本站服务状态和日志
- 启动、停止、重启 `eli-site.service`

服务状态无需 sudo：

```bash
systemctl status eli-site.service --no-pager
systemctl is-active eli-site.service
systemctl is-enabled eli-site.service
journalctl -u eli-site.service --no-pager
```

仅以下三个提权命令被精确允许：

```bash
sudo /usr/bin/systemctl start eli-site.service
sudo /usr/bin/systemctl stop eli-site.service
sudo /usr/bin/systemctl restart eli-site.service
```

不得给这些命令增加额外参数，也不得替换为其他 unit。

## 4. 明确禁止的范围

Codex 不得操作：

- root、`ubuntu` 或其他账号
- 通用 sudo shell、任意 unit 或任意 `systemctl`
- Docker socket、容器和镜像
- Caddy、DNS、SSL、UFW、云防火墙、Fail2Ban 或 SSH 配置
- 系统包升级、全局 Node/Corepack/pnpm 升级或全局运行时替换
- 其他站点、其他用户 home、authorized_keys、数据库、media、env、日志或服务
- 未经确认的生产数据删除、media 清理或 release 清理

本站已取消自建发布器、发布包装器和发布锁。不得创建、恢复或调用这类机制。正常使用 SSH、Git、项目构建命令、原子符号链接切换和本站 systemd 权限。

修改整个首页为 Next.js SSR 全量反向代理仍需运维修改全局 Caddy；Eli 不得自行绕过该边界。

## 5. Codex 开始工作前的检查

每次开始修改、migration 或发布前执行只读检查：

```bash
whoami
hostname
date
readlink -f /srv/eli-site/current
readlink -f /srv/eli-site/static-current
systemctl status eli-site.service --no-pager
df -h /
find /srv/eli-site/workspace -maxdepth 2 -name .git -type d -print
ps -eo pid,user,etime,args | \
  grep -E '[n]ext build|[n]pm.*build|[p]npm.*build|[p]ayload.*migrat'
```

进入具体源码仓库后继续检查：

```bash
git status --short
git branch --show-current
git rev-parse HEAD
ls pnpm-lock.yaml package-lock.json yarn.lock 2>/dev/null
node -p "require('./package.json').packageManager || ''"
```

包管理器规则：

- 优先遵循 `package.json#packageManager`、项目 lockfile、README 和 CI 配置。
- 服务器当前能够运行 pnpm 11，不代表项目应升级到 pnpm 11。
- 有 `pnpm-lock.yaml` 时不得改用 npm；有 `package-lock.json` 时不得改用 pnpm。
- 不得因 Corepack 提示而擅自升级项目 lockfile、Node、Payload 或 Next.js。
- PAX/xattr/AppleDouble 仅在实际待发布归档的字节流、headers/清单及解包结果中检查；不得因此扫描或清理本地展开树、依赖缓存或不在归档中的文件。

停止条件：

- 存在其他正在运行的 build、migration、deploy 或 release switch
- Git 状态包含无法确认归属的修改
- 当前 SQLite 路径、media 路径或 env 使用方式无法确认
- 操作需要 Caddy、DNS、SSL、Docker、系统包或其他站点权限
- migration 备份未完成，或新 release 构建/健康检查失败

遇到停止条件时先报告，不扩大权限或绕过限制。

## 6. 推荐发布流程

### 6.1 静态前端发布

先在 `/srv/eli-site/workspace` 的真实 Git 仓库中完成修改、测试和构建。以下示例假设输出目录为 `dist`；如项目实际使用 `out` 或其他目录，应按项目配置替换。

```bash
cd /srv/eli-site/workspace/<repo>

# packageManager 必须由项目声明或由已验证构建记录确认。
package_manager="$(node -p "require('./package.json').packageManager || ''")"
case "$package_manager" in
  pnpm@*) ;;
  *) echo "未确认项目 pnpm 版本，停止"; exit 1 ;;
esac

# Corepack 按项目 packageManager 选择版本，不得使用全局 pnpm 升级项目。
corepack pnpm install --frozen-lockfile
corepack pnpm build

test -f dist/index.html

release_id="$(date -u +%Y%m%dT%H%M%SZ)-static"
static_release_dir="/srv/eli-site/static-releases/$release_id"
previous_static="$(readlink -f /srv/eli-site/static-current)"

mkdir "$static_release_dir"
rsync -a dist/ "$static_release_dir/"
test -f "$static_release_dir/index.html"
```

原子切换静态版本：

```bash
static_link_tmp="/srv/eli-site/.static-current.$release_id"
ln -s "$static_release_dir" "$static_link_tmp"
mv -Tf "$static_link_tmp" /srv/eli-site/static-current
```

静态切换不需要重启 `eli-site.service`。不得删除 `$previous_static`，它是即时回滚点。

如交付使用 macOS 归档，核验对象仅为该归档：列出 headers/清单，扫描归档内 PAX/xattr/AppleDouble，完成受控解包与 SHA-256 比对；外层目录属性、缓存或展开依赖树不构成阻断或清理范围。

### 6.2 Next/Payload 前后端发布

使用独立 staging 和 release，不在 `current` 上直接构建：

```bash
source_dir="/srv/eli-site/workspace/<repo>"
release_id="$(date -u +%Y%m%dT%H%M%SZ)-next-payload"
build_dir="/srv/eli-site/staging/$release_id"
release_dir="/srv/eli-site/releases/$release_id"
previous_release="$(readlink -f /srv/eli-site/current)"

mkdir "$build_dir"
rsync -a --exclude .git --exclude node_modules --exclude .next \
  "$source_dir/" "$build_dir/"
cd "$build_dir"

set -a
. /etc/eli-site/eli-site.env
set +a
```

按项目锁文件安装、生成类型并构建：

```bash
if [ -f pnpm-lock.yaml ]; then
  package_manager="$(node -p "require('./package.json').packageManager || ''")"
  case "$package_manager" in
    pnpm@*) ;;
    *) echo "未确认项目 pnpm 版本，停止"; exit 1 ;;
  esac
  corepack pnpm install --frozen-lockfile
  corepack pnpm payload generate:types
  corepack pnpm build
elif [ -f package-lock.json ]; then
  package_manager="$(node -p "require('./package.json').packageManager || ''")"
  case "$package_manager" in
    npm@*) ;;
    *) echo "未确认项目 npm 版本，停止"; exit 1 ;;
  esac
  npm ci
  npm run build
elif [ -f yarn.lock ]; then
  package_manager="$(node -p "require('./package.json').packageManager || ''")"
  case "$package_manager" in
    yarn@*) ;;
    *) echo "未确认项目 Yarn 版本，停止"; exit 1 ;;
  esac
  corepack yarn install --immutable
  corepack yarn build
else
  echo "未找到受支持的 lockfile，停止并确认项目构建方式"
  exit 1
fi
```

2026-07-24 取证时的 Next 配置使用 `output: 'standalone'`；当前配置必须从本次 workspace 实时确认。最终 release 必须满足 systemd 的启动契约：release 根目录存在 `server.js`，并包含匹配的 `.next/static`、`public` 和运行依赖。

如果构建只生成 `.next/standalone/server.js`，应按项目既有打包方式把完整 standalone 目录作为 release 根，并补齐静态资源；不要只复制一个 `server.js`：

```bash
runtime_dir="/srv/eli-site/staging/$release_id.runtime"
mkdir "$runtime_dir"
rsync -a .next/standalone/ "$runtime_dir/"
mkdir -p "$runtime_dir/.next"
rsync -a .next/static/ "$runtime_dir/.next/static/"
if [ -d public ]; then rsync -a public/ "$runtime_dir/public/"; fi

test -f "$runtime_dir/server.js"
test -d "$runtime_dir/.next/static"
mv "$runtime_dir" "$release_dir"
```

构建目录和 release 中不得复制生产 SQLite、media 或 env。持久化内容继续使用 `/srv/eli-site/shared` 和 `/etc/eli-site/eli-site.env`。

### 6.3 SQLite、Payload migration 与 media

env 已验证可由 Bash 读取，但不得输出值：

```bash
set -a
. /etc/eli-site/eli-site.env
set +a
```

migration 前创建 SQLite 官方在线备份和 media 快照：

```bash
backup_id="$(date -u +%Y%m%dT%H%M%SZ)-pre-migration"
backup_dir="/srv/eli-site/shared/backups/$backup_id"
mkdir -m 700 "$backup_dir"

sqlite3 /srv/eli-site/shared/data/site.sqlite \
  ".backup '$backup_dir/site.sqlite'"
rsync -a /srv/eli-site/shared/media/ "$backup_dir/media/"

test -s "$backup_dir/site.sqlite"
sqlite3 "$backup_dir/site.sqlite" 'PRAGMA integrity_check;'
```

`PRAGMA integrity_check` 预期返回 `ok`。不得把查询结果、业务表内容或 env 值发送到对话。

确认当前 release 对应的 migration 后，在新 build 目录中运行：

```bash
cd "$build_dir"
set -a
. /etc/eli-site/eli-site.env
set +a

corepack pnpm payload migrate:status
corepack pnpm payload migrate
```

破坏性 schema 变化应先停止本站服务，再执行 migration：

```bash
sudo /usr/bin/systemctl stop eli-site.service
# 运行已确认的 migration
sudo /usr/bin/systemctl start eli-site.service
```

数据库、WAL/SHM 与 media 必须作为同一发布的数据面判断。不得只回滚代码而忽略已执行的不可逆 migration，也不得把 media 清空后重新上传来代替一致性检查。

隔离测试源 DB 的 SHA 不变量只适用于该隔离副本的可控实验，不得套用于 live Payload 数据库。live `data-migration` 验收以 `PRAGMA quick_check`、`PRAGMA foreign_key_check`、预期 schema 与 migration ledger、经授权的数据差异核对、服务/应用运行健康及保留现场的回滚判断为准；不得输出业务表、env 值或把新写入覆盖为旧备份。

### 6.4 切换 current 与服务

构建、备份和必要 migration 全部成功后再切换：

```bash
release_link_tmp="/srv/eli-site/.current.$release_id"
ln -s "$release_dir" "$release_link_tmp"
mv -Tf "$release_link_tmp" /srv/eli-site/current
sudo /usr/bin/systemctl restart eli-site.service
```

不要删除 `$previous_release` 或 `$previous_static`。

## 7. 发布后验收

按分级执行相称验收：`static` 检查目标公开页、目标资源和阻断控制台错误；`code` 再检查受影响页面与必需 health；`admin-demo` 仅检查受影响登录/受保护入口、目标后台流程、相关 API/media 和阻断控制台错误；`data-migration` 另按 §6.3 核验数据库/ledger/授权差异与回滚判断。除非任务明确要求或有共享组件风险证据，后台预览不跑全路由或全角色矩阵。

服务侧：

```bash
systemctl is-active eli-site.service
systemctl status eli-site.service --no-pager
journalctl -u eli-site.service --since "-5 minutes" --no-pager

curl -fsS -o /dev/null -w 'internal=%{http_code}\n' \
  http://127.0.0.1:3140/
```

公网侧：

```bash
curl -fsS -o /dev/null -w 'home=%{http_code}\n' \
  https://eli.roxychao.com/

curl -sS -o /dev/null -w 'admin=%{http_code}\n' \
  https://eli.roxychao.com/admin

curl -sS -o /dev/null -w 'api=%{http_code}\n' \
  https://eli.roxychao.com/api

curl -fsS -o /dev/null -w 'media=%{http_code}\n' \
  https://eli.roxychao.com/media/<known-file>
```

还应在真实浏览器中检查：

- 首页、主要导航和关键页面正常
- 桌面和移动视图没有阻断性错位
- 静态资源与 Payload media 正常加载
- `/admin` 能进入预期登录或受保护流程
- API 返回符合当前应用契约的状态，不是意外 404/5xx
- 浏览器控制台无阻断性错误

2026-07-24 的历史验收观察为：公网首页是静态 `Prototype`，`/admin`、`/api`、`/graphql` 为 404。该观察不得当作当前基线；当前值只能由本次只读 preflight 与发布后验收产生。权限已经就绪不等于 Next/Payload 应用已经部署。

HTTP 200、构建成功、PID 或一张截图不能单独构成完整验收。

## 8. 回滚

静态前端失败时：

```bash
static_rollback_tmp="/srv/eli-site/.static-current.rollback"
ln -s "$previous_static" "$static_rollback_tmp"
mv -Tf "$static_rollback_tmp" /srv/eli-site/static-current
```

Next/Payload 启动或公网验收失败时：

```bash
rollback_link_tmp="/srv/eli-site/.current.rollback"
ln -s "$previous_release" "$rollback_link_tmp"
mv -Tf "$rollback_link_tmp" /srv/eli-site/current
sudo /usr/bin/systemctl restart eli-site.service
systemctl is-active eli-site.service
```

随后重新检查公网首页、关键资源、admin、API 和 media。

如果已经执行数据库 migration：

- 只有确认旧代码兼容新 schema 时，才能只回滚代码。
- 必须先判断 migration 后是否已有新增或修改数据。只要存在新写入，禁止用旧 SQLite 备份或旧 media 快照覆盖；立即停止扩展操作、保留故障现场，并制定差异合并、补偿或前滚方案。
- 仅在确认没有新写入且旧代码不兼容新 schema 时，才可按当前专项授权停止本站服务并使用发布前已验证的 SQLite 备份恢复；media 必须回到与数据库一致的快照。
- 恢复前保留故障现场副本，不在失败状态下连续试错生产。

## 9. 首次登录验收

首次登录验收只能在当前任务明确授权、实际 identity 已由本机受保护配置确认且来源网络满足本指南条件后执行；缺一项即停止，不得尝试连接。满足门禁后，Eli 使用匹配 identity 完成一次真实登录：

```bash
ssh -i "<受保护配置中已确认的实际 identity 路径>" \
  -o IdentitiesOnly=yes \
  eli@[REDACTED_HOST]
```

登录后执行：

```bash
whoami
test -w /srv/eli-site/workspace && echo workspace-write-ok
test -w /srv/eli-site/current && echo current-write-ok
test -w /srv/eli-site/releases && echo releases-write-ok
test -w /srv/eli-site/static-current && echo static-current-write-ok
test -w /srv/eli-site/static-releases && echo static-releases-write-ok
test -w /srv/eli-site/shared/data/site.sqlite && echo database-write-ok
test -w /srv/eli-site/shared/media && echo media-write-ok
test -w /etc/eli-site/eli-site.env && echo env-write-ok
systemctl status eli-site.service --no-pager
sudo -n -l
```

预期：

- `whoami` 返回 `eli`
- 上述本站写权限检查均通过
- 无 forced command，能够获得普通 Shell/PTY
- `sudo -n -l` 只显示 `eli-site.service` 的 start、stop、restart
- 不能访问其他站点、Docker、Caddy、UFW 或 root shell

如果 SSH 登录失败，记录客户端公网出口、登录时间和错误摘要交给服务器 owner。不得发送私钥内容；不得为排障猜测用户名、重复触发 Fail2Ban 或扩大服务器权限。
