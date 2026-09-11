# 从源码构建与运行

[返回首页](../README.md) · [安装指南](INSTALL.md) · [工程说明](DEVELOPMENT.md)

本页面向**从源码安装、运行和验证 OpenWrite** 的用户与开发者。只使用已验收发布包的用户，
请直接看 [安装指南](INSTALL.md)；本页的源码路径与发布路径**安装同一份插件包**
（根包 `dsh-openwrite`），区别只在产物来自本机构建还是 Release 附件。

## 三种取源方式

| 方式 | 命令 | 需要本地克隆 | 需要 OpenWrite Python 源码 | 领域后端 |
|---|---|---|---|---|
| Release 包 / npm | `dsh plugin --profile web add -w dsh-openwrite@latest` | 否 | 否 | 受管理 Core（自动准备） |
| GitHub 源码 | `dsh plugin --profile web add -w github:LiPu-jpg/Openwrite#<ref>` | 否 | 否 | 受管理 Core（自动准备） |
| 本地克隆 | `npm ci` 后 `dsh plugin --profile web add -w "$PWD"` | 是 | 否 | 受管理 Core（自动准备） |
| 开发直连 | `scripts/install.sh` + `scripts/dev.sh` | 是 | **是** | 外部 Studio（`http://127.0.0.1:4567`） |

前三种是「安装」路径，本机构建后与 Release 行为一致，**不需要预装 Python，也不需要 Core 源码**：
bridge 由根 `cordis.patch.yml` 以 `mode: managed` 挂载，首次打开 OpenWrite 时按
`release/runtime-manifest.json` 下载固定版 uv、Python 3.12 与 Core wheel 到 `$DSH_HOME/openwrite/`。
第四种是保留的开发直连路径，需要 `uv` 和 Core 源码工作树，见「方式三」。

## 前置条件

| 依赖 | 版本 | 用途 |
|---|---|---|
| Node | **≥ 22.19.0**（验收基线 24） | 宿主 dsh 与全部插件构建 |
| npm | 随 Node | 依赖安装与构建 |
| pnpm | 9.15.9 | `dsh plugin add -w` 的 profile workspace 转发 |
| git | 任意 | 克隆仓库；`scripts/install.sh` 自动获取 dsh-dog |
| rsync | 任意 | `scripts/install.sh` 同步预设（仅方式三） |
| uv | 任意 | 仅方式三创建 Core 运行环境 |

macOS arm64/x64、Linux x64、Windows x64 为发布目标平台。仓库内的 `scripts/*.sh` 是 POSIX
脚本，Windows 请在 WSL 中执行，或只用 dsh CLI 命令（方式一、二的命令本身是跨平台的）。

```sh
node -v          # 必须 >= 22.19
npm -v
command -v pnpm rsync git uv
```

安装 pnpm（其余按需）：

```sh
npm install --global pnpm@9.15.9
```

## 方式一：本地克隆安装（推荐）

```sh
git clone https://github.com/LiPu-jpg/Openwrite.git
cd Openwrite
npm ci --no-audit --no-fund                     # 触发 prepare：按需构建三个插件
node_modules/.bin/dsh plugin --profile web add -w "$PWD"
node_modules/.bin/dsh web                       # 浏览器地址由宿主输出，默认 http://127.0.0.1:3080
```

要点：

- `npm ci` 会执行根 `prepare`（`scripts/prepare.mjs`）：对 `packages/openwrite-bridge`、
  `packages/studio-panel`、`vendor/dsh-dog` 依次 `npm ci` + 构建，并生成发布来源记录。
  `lib/` 在 `.gitignore` 中，**干净克隆必须构建**，否则插件入口不存在。
- 与 CI 完全等价的显式步骤（`--ignore-scripts` 时使用）：

  ```sh
  npm ci --ignore-scripts --no-audit --no-fund
  npm ci --prefix packages/openwrite-bridge --no-audit --no-fund
  npm ci --prefix packages/studio-panel --no-audit --no-fund
  npm ci --prefix vendor/dsh-dog --ignore-scripts --no-audit --no-fund
  node scripts/prepare.mjs
  ```

- `-w` 是 pnpm profile workspace 的参数，不能省略；只输入 `dsh` 会要求 `--profile`。
- 重复执行 `dsh plugin --profile web add -w "$PWD"` 是幂等的，可修复「依赖已在但 bundle
  未登记」或本地链接失效的状态。
- 需要 headless profile 时，对 `--profile headless` 再执行一次。DoG 只在 web 挂载。

启动后：打开宿主输出的地址，点击侧栏 **OpenWrite**，等待环境准备完成（首次会下载 uv、
Python 和依赖），再选择作品目录。安装与环境自检不调用模型；写章和审稿需要在
「任务 → 模型」配置服务商。

## 方式二：不克隆，直接安装 GitHub 源码

```sh
dsh plugin --profile web add -w github:LiPu-jpg/Openwrite#v0.2.8
dsh web
```

- 使用 dsh 官方 GitHub source 机制；`prepare` 在安装时自行构建三个插件，不访问相邻工作区。
- 需要 Git 与构建依赖；普通用户优先选择已验收 Release 包。
- 若 pnpm 按本机策略阻止构建，按它显示的指引**仅批准本包**后重试；安装脚本不放宽构建授权。
- 源码安装同样必须通过发布验收；不能用本机已编译目录代替。

## 方式三：开发直连（legacy Studio 后端）

用于修改 Core Python 侧、或需要独立 Studio 进程（`http://127.0.0.1:4567`）的场景。

前置：Core 源码工作树（`native-core` 分支），默认取相邻目录 `../OpenWrite`。

```sh
# 1) 构建插件 + 安装预设 + 挂载 profile（幂等）
bash scripts/install.sh
# 2) 启动 Studio + dsh web
OPENWRITE_DIR=/path/to/OpenWrite bash scripts/dev.sh --project "$HOME/my_novel"
```

`scripts/install.sh` 的六个步骤：项目内 `npm ci` → 构建 bridge → 构建 panel → 安装
`presets/openwrite` 到 `$DSH_HOME/.agent-presets/` → 用 `--dump-config` 初始化 web/headless
profile（不调用模型）→ 把插件装进 profile 并写入 `dog:` 兜底配置。任一步失败返回非零。

`scripts/dev.sh` 在缺少 `.venv/bin/openwrite` 时用 `uv venv .venv --python 3.12` +
`uv pip install -e <Core 源码>` 创建运行时，然后由 `scripts/dev-supervisor.mjs` 同时托管
Studio 与其后的 `dsh web`（端口占用直接拒绝启动，Studio 30 秒健康检查，退出时清理两端进程组）。
该路径下 bridge 使用默认外部地址 `http://127.0.0.1:4567`，因此**必须**有 Studio 在运行。

相关开关：`DSH_DOG_DIR`（复用已有 dsh-dog 工作树）、`DSH_DOG_AUTO_INSTALL=0`（不装 DoG）、
`DSH_DOG_REF`、`DSH_DOG_WORKSPACE_ROOT`。自定义 `STUDIO_PORT` 时，必须同时把 dsh 中
bridge 的 `baseUrl` 指向同一端口。不要把 `dev.sh` 与常驻 daemon 绑到同一端口。

## 构建与验证

```sh
npm run build                    # 三个插件（bridge / panel / dsh-dog）一次构建
npm run check:plugin             # 自包含门禁：构建、doctor、生命周期、预设、smoke、组件测试
npm run doctor -- --profiles     # 只读核对本机 web/headless 是否登记标准 bundle
npm run test:managed             # 受管理运行时与工具策略回归
OPENWRITE_ROOT=/path/to/OpenWrite npm run check   # 追加跨仓库 contracts 校验
```

`check:plugin` 不需要服务、小说或模型凭据，也是 GitHub Actions `plugin-check.yml` 的门禁。
`doctor --profiles` 会对 **web 与 headless 两个** profile 各查一次：只装了 web 时，
headless 目录尚未初始化会报一条 `ENOENT .../profiles/headless/package.json`，这是未使用
profile 的正常结果，不是安装失败。只跑 web 时忽略该条；需要 headless 就先补：

```sh
node_modules/.bin/dsh --profile headless --dump-config
node_modules/.bin/dsh plugin --profile headless add -w "$PWD"
```

真实系统另跑集成与浏览器验收：

```sh
scripts/verify.sh                # 需已启动 dev.sh 的 Studio + dsh web（可用 DSH_URL/STUDIO_URL 覆盖）
npm run test:e2e                 # Playwright；首次需安装浏览器，跳过不算通过
npm run test:dog                 # 需已有 .venv：Python/TS DoG 产物一致性
```

发布相关（仅维护者）：

```sh
node scripts/release-provenance.mjs
npm pack --ignore-scripts
node scripts/release-check.mjs
node scripts/release-smoke.mjs
```

## 环境变量

| 变量 | 默认 | 说明 |
|---|---|---|
| `DSH_HOME` | `~/.dsh` | 安装、`doctor`、启动三步必须使用**同一个值** |
| `OPENWRITE_DIR` | `../OpenWrite` | 方式三的 Core 源码；缺失时 `dev.sh` 直接报错 |
| `OPENWRITE_PROJECT` / `--project` | `~/my_novel` | 方式三 Studio 的 legacy 默认作品 |
| `STUDIO_PORT` | `4567` | 方式三 Studio 端口，需与 bridge `baseUrl` 一致 |
| `OPENWRITE_ROOT` | — | `npm run check` 的相邻 Core 契约路径 |
| `DSH_DOG_DIR` / `DSH_DOG_AUTO_INSTALL` / `DSH_DOG_REF` | 自动克隆 `v1.2.0` 到 `$DSH_HOME/extensions/dsh-dog` | DoG 来源控制 |
| `NO_PROXY` / `no_proxy` | `dev.sh` 自动设置 | 本地代理不得劫持 `127.0.0.1` |

## 修改源码后如何生效

本地源码安装的 profile 里，`node_modules/dsh-openwrite` 是**指向本仓库的符号链接**
（`dsh plugin ... add -w "$PWD"` 建立），所以改完代码**不需要重新安装**，只需按下表重新构建
并重启 dsh。宿主在启动时加载插件的 Node 模块和 `cordis.patch.yml`，运行中不会热加载。

| 改动位置 | 需要做什么 |
|---|---|
| 宿主逻辑：`packages/openwrite-bridge/src/**`、`packages/studio-panel/src/**` 非 client 部分、`plugin.mjs` | `npm run build`（或对应包的 `npm run build`）→ **重启 dsh** |
| 工作台界面：`packages/studio-panel/src/client/**`（产物 `lib/client.js`） | `npm run build` → **重启 dsh**；不重启时该资源仍按请求读磁盘，但 URL 的 `rev` 在启动时固定且响应为 `immutable` 长缓存，必须浏览器硬刷新才能拿到新代码 |
| `cordis.patch.yml`（挂载行、`mode`） | 重启 dsh；改了行列 id/包名后再跑一次 `plugin add` |
| `package.json` 的 `name` / `version` / `dsh` 字段 | `plugin add -w "$PWD"` + 重启 dsh |
| `presets/openwrite/**`（预设与技能） | 见下方预设校验说明；**必须**同时提升 `package.json` 版本或清理已安装预设 |
| Python Core（`native-core` 分支） | 方式一/二用固定 wheel，改 Core 源码**不生效**；要改 Core 用方式三的 editable 安装，改完重启 Studio（`dev.sh`）即可 |

预设目录带来源摘要校验：`plugin.mjs` 在启动时比对仓库 `presets/openwrite/` 的摘要与安装记录，
直接改源码里的预设文件会让下次启动报
`OpenWrite preset openwrite-<版本> was modified; copy it to a custom preset before reinstalling this version`。
两种处理方式：

```sh
# A. 提升根 package.json 的 version → 生成新的 openwrite-<新版本> 预设目录
# B. 或先移除已安装的版本预设，再重启 dsh（会按当前源码重新安装）
rm -rf ~/.dsh/.agent-presets/openwrite-<版本>
```

日常调整提示词/技能建议直接复制为自定义预设（目录名仍用 `openwrite-` 前缀）在界面里使用，
不要改仓库内官方预设。改完跑一遍 `npm run check:plugin` 做回归；它不需要模型凭据。

小提示：`plugin add` 可能给包内 `bin` 目标文件补上可执行位，`git status` 里出现
`scripts/maintenance.mjs` 之类的 mode 变更属正常现象，用 `chmod 644` 还原即可。

## 日常启动

装好之后每次启动只要两条命令（Node 24 用 keg-only 方式安装时，新终端必须先补 PATH）：

```sh
export PATH="/opt/homebrew/opt/node@24/bin:$PATH"   # 让 dsh/pnpm 用 Node ≥22.19
node_modules/.bin/dsh web                          # 默认 http://127.0.0.1:3080
```

- 宿主会打印带 token 的登录 URL，用它打开页面；`--no-open` 可阻止自动开浏览器，
  `--port 3081` 可换端口。
- 首次点击侧栏 **OpenWrite** 才准备写作环境（下载 uv、Python 与 Core wheel 到
  `$DSH_HOME/openwrite/`），等待期间可取消或重试。
- 确认已经起来：`curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:3080/` 返回
  `401`（需要登录令牌）即正常；`lsof -nP -iTCP:3080 -sTCP:LISTEN` 可看监听进程。
- 方式三的等价启动是 `OPENWRITE_DIR=/path/to/OpenWrite bash scripts/dev.sh`（也可
  `npm run dev`），它会先起 Studio 再起 dsh。

## 停止、升级、回退与卸载（源码安装）

`dsh web` 是**前台进程**：在跑它的终端按 `Ctrl-C` 即停止，受管理 Python Core 后端随 dsh
进程一起退出，不需要单独停（本插件不是常驻服务，也不会开机自启）。终端已关但端口被占时：

```sh
lsof -ti:3080 | xargs kill          # 按端口清理；必要时加 -9
pgrep -fl managed_runtime           # 无输出说明后端没有残留
```

方式三的 `dev.sh` 由 supervisor 托管，`Ctrl-C`/SIGTERM 会清理 Studio 与 dsh 两端进程组。

```sh
git pull                                        # 或切到目标 tag
npm ci --no-audit --no-fund
node_modules/.bin/dsh plugin --profile web add -w "$PWD"   # 重跑即修复登记与链接
node_modules/.bin/dsh plugin --profile web remove -w dsh-openwrite
```

升级前先停止 dsh。官方预设按版本 ID 安装（`openwrite-<版本>`）；同名目录被手工改过时，
插件保留文件并报冲突，先把官方预设复制为 `openwrite-你的名称` 再改。作品、配置、凭据、
会话默认保留；只清下载缓存时删除 `$DSH_HOME/openwrite/cache/`，不要删除整个
`$DSH_HOME/openwrite/`。

## 常见问题

| 现象 | 处理 |
|---|---|
| `FAIL Node >= 22.19.0: found 16.x` | 切换到 Node 22/24 后重新 `npm ci` 与构建 |
| `npm ci` 在 `@deepseek-ai/dsh-subprocess-local` 报 `TypeError: (intermediate value).resolve is not a function`（`ensure-spawn-helper.mjs` 的 `import.meta.resolve`） | Node 过低：`import.meta.resolve` 自 Node 20.6 才有。改用 Node ≥22.19 后重跑 `npm ci`，不要改这个 postinstall 脚本 |
| 启动报找不到 `packages/*/lib/index.js` | 运行 `node scripts/prepare.mjs`（或 `npm run build`） |
| `npm ci` / `pnpm` 仍以旧 Node 运行 | 脚本按 PATH 解析 `node`；必须让 Node ≥22.19 排在 PATH 最前（keg-only 安装如 `node@24` 需手动前置 `/opt/homebrew/opt/node@24/bin`） |
| `缺少命令: pnpm` / `rsync` | `npm install --global pnpm@9.15.9`；macOS 自带 rsync，Linux 装 `rsync` 包 |
| pnpm 阻止本包构建 | 按提示只批准本包，再重试；不要全局放宽构建授权 |
| `duplicate loader entry id: openwrite-bridge` | 停止 dsh 后 `node package/scripts/maintenance.mjs migrate --profile web --apply`，再重新安装 |
| `Studio 端口 4567 无法使用` | 释放端口或改 `STUDIO_PORT`，并同步 bridge `baseUrl` |
| Studio 30 秒健康检查失败 | 确认 `.venv/bin/openwrite` 存在且 `uv` 可用；必要时重建 venv |
| 本地请求被代理拦截 | 确保 `NO_PROXY=127.0.0.1,localhost` |
| `preset ... already exists without an ownership marker` | 把该目录改名为自定义预设，再重装 |
| 写章/审稿报模型未配置 | 在「任务 → 模型」配置服务商；安装自检不需要模型 |
| 源码改动没生效 | 先重新构建再重启 dsh；界面改动还需硬刷新浏览器。见「修改源码后如何生效」 |

不要修改用户 `node_modules` 里的 JS 来掩盖重复加载，也不要跳过 `plugin add` 直接手工登记
bundle。密钥不要写入对话、作品目录或 Git 仓库。
