# 安装、更新与卸载

安装已发布的版本；可下载产物、SHA-256 和实际平台结果以 [GitHub Release](https://github.com/LiPu-jpg/Openwrite/releases/latest) 附件为准。验收方法见 [验收报告](RELEASE_ACCEPTANCE.md)。

## 标准安装

本次锁定 dsh `0.1.2-rc.1`，Node 24 为完整验收基线；Node 22.19 / 26 做兼容检查。安装不自动升级 dsh。先执行 `dsh --version` 核对宿主。

安装命令：

```sh
dsh plugin --profile web add -w dsh-openwrite@latest
dsh web
```

`@latest` 只在执行命令时解析 npm 最新发布版本；安装后不会后台自动升级。

需要固定版本时，例如 npm 的 `dsh-openwrite@0.2.8` 与 GitHub Release 的 v0.2.8 `.tgz` 为同一份已验收产物，包含预编译插件、编辑器资源与 Core wheel。也可直接安装 Release：

```sh
dsh plugin --profile web add -w https://github.com/LiPu-jpg/Openwrite/releases/download/v0.2.8/dsh-openwrite-0.2.8.tgz
```

`-w` 是 pnpm profile workspace 的安装参数。只输入 `dsh` 会要求 `--profile`；启动浏览器使用 `dsh web`。浏览器地址由宿主输出，不固定为 3080。

点击侧栏 OpenWrite，等待环境准备完成。下载和安装进度来自同一个运行时状态，支持取消与重试。选择已有作品目录，或选择空目录后在创作工作台初始化。入口创建新创作会话，已有会话、作品和预设不被改写。点击「创作」进入编辑器；无需发送第一条消息。

Git 是可选的作品归档能力。未安装 Git 或 Git 初始化失败时，小说文件仍会保留，工作台显示自动 checkpoint 不可用；正常编辑和模型创作可以继续。

dsh 对话模型在宿主中配置。小说生成、评审与测试模型在「任务 → 模型」配置；安装健康检查不调用这些模型。

## 安装影响

本轮原生验收覆盖 macOS 15、Ubuntu 和 Windows GitHub 原生运行器。
依赖包要求 Apple Silicon Mac 至少 macOS 14，Intel Mac 至少 macOS 13；
其他系统小版本的覆盖范围以 Release 报告为准。

- 插件装在指定 profile。bridge、panel、DoG、技能、编辑器资源和 Core wheel 位于同一包。
- `$DSH_HOME/openwrite/` 保存下载缓存、Python 环境、管理记录、隔离配置与凭据；未指定 DSH_HOME 时使用 dsh 默认目录。
- 作品保存在作者选定目录。受管理 Core 仅监听回环地址，端口自动分配，实例认证只在宿主与 Core 之间传递。
- 只启动、停止本插件创建的进程。不修改系统 Python、全局 PATH 或 Shell 配置。
- 首次准备需要连接 GitHub 和 Python 包索引。版本及校验值见包内 `release/runtime-manifest.json`；依赖见 `requirements.lock`。

## 从旧本地安装迁移

旧开发安装会同时留下 `@dsh-novel/openwrite-bridge`、`@dsh-novel/studio-panel`、`@dsh-external/dsh-dog` 的 bundle 登记和 `dependencies`。只取消 bundle 而不走宿主 `dsh plugin remove` 时，随后的 `plugin add` 会按依赖重新登记，启动时报 `duplicate loader entry id: openwrite-bridge`。

执行顺序（dsh 必须已停止；需要本机 `dsh` 与 `pnpm`）：

1. 下载并解开新 Release `.tgz`。
2. 预览：检查 bundle **和** dependencies，不改文件。
3. 应用：先备份，再对仍在依赖中的旧包执行宿主 `dsh plugin --profile <name> remove -w <旧包名>`；失败则恢复备份。
4. 再执行下面的标准 `plugin add` 安装新包。
5. 启动 `dsh web`。不要修改用户 `node_modules` 源码来掩盖重复加载。

```sh
node package/scripts/maintenance.mjs migrate --profile web
node package/scripts/maintenance.mjs migrate --profile web --apply
dsh plugin --profile web add -w dsh-openwrite@latest
dsh web
```

备份写入 `$DSH_HOME/openwrite/migration-backups/`，包含 profile 的 `package.json` / lock、宿主 settings 和预设目录副本。不删除其他插件、用户配置、凭据、作品和历史会话。使用过旧 headless profile 的用户，再对 `--profile headless` 执行同样的预览与应用。

预设按来源区分：带 `.openwrite-managed.json` 的是包装管理的官方版本预设（目录名为 `openwrite-<版本>`）；没有该标记、或内容与包装校验不一致的，视为作者所有，迁移不会删除。本机若仍有旧名 `openwrite` 默认预设，会保留；已有会话继续使用当时绑定的预设，不会被改写。需要改官方预设时，先复制为 `openwrite-你的名称`。历史会话不会自动切换到新版本预设。

旧全局模型配置不会被受管理环境自动读取；作者可在新环境中重新配置，也可停机备份后迁移相应配置文件。不要把凭据复制进作品或 Git 仓库。不要手工改用户 `node_modules` 里的 JS 来消除重复 loader。

## 更新与回退

停止 dsh，对照新 Release 的精确宿主兼容版本，再执行：

```sh
dsh plugin --profile web add -w dsh-openwrite@latest
dsh web
```

`@latest` 不升级 dsh，也不改变已安装版本直到再次执行命令。环境安装使用临时目录，校验并成功启动后才替换活动记录；失败保留旧环境和配置。取消或失败后可从入口重试。

回退时，停止 dsh，使用精确版本（例如 `dsh plugin --profile web add -w dsh-openwrite@0.2.8`）或重新安装上一份已验收 `.tgz`，再启动。官方预设采用版本 ID（如 `openwrite-0-2-1`），不会覆盖用户预设。请先把官方预设复制为 `openwrite-你的名称` 再修改；对被直接修改的同版本官方预设，插件保留文件并报告冲突。

## 停止服务

`dsh web` 在前台运行，在它的终端按 `Ctrl-C` 即停止；受管理写作后端随 dsh 进程一起退出，
无需单独关闭，也不是开机自启服务。终端已关但端口仍被占用时，按端口清理
（`lsof -ti:3080 | xargs kill`），再用 `pgrep -fl managed_runtime` 确认后端没有残留。

## 卸载

停止 dsh 后执行：

```sh
dsh plugin --profile web remove -w dsh-openwrite
```

正常停止时卸载本插件服务、界面、连接与进程。作品、配置、凭据、会话和缓存默认保留。关闭全部 dsh 进程后，可单独删除 `$DSH_HOME/openwrite/cache/` 清理下载缓存；不要删除整个目录，其中也有配置与管理记录。异常强制退出留下的版本预设可保留排查，重装同版本并正常退出会清理未修改副本。

## 自定义预设和高级配置

自定义预设从官方版本复制，目录 ID 使用 `openwrite-` 前缀以启用创作界面。小说工具行是 `dsh-openwrite/tools`，DoG 工具行是 `dsh-openwrite/dog-tools`；不要再次注册全局 bridge 或 DoG 引擎。工具集合随会话预设保持稳定，不随界面切换。

小说工具支持 `presentation: native`（默认）、`ptc` 或 `both`。只读策略和计划模式独立约束副作用。模型服务商已接收的请求可能无法撤回：取消后停止新调度和重试，保存已完成结果与用量；费用未知时显示未知，不计为零。

bridge 的高级 `mode: external` / `baseUrl` 配置继续支持外部 Core，不接管外部进程。默认 `managed` 不需要配置端口。旧显式地址配置仍按外部模式解释。

## Embedding（检索向量）

「资料 → 检索」的向量模式需要 Embedding 档案，在「任务 → 模型」的 **Embedding** 页签配置。

**本地（推荐，不需要 API Key）**：插件自带 FastEmbed 运行时，首次加载会从 Hugging Face
下载 ONNX 模型（`bge-small-zh-v1.5` 约 182 MB，已缓存后加载约 0.4 秒）。

| 字段 | 值 |
|---|---|
| Embedding 协议适配器 | `local`（也接受 `fastembed`、`on-device`） |
| Embedding Model ID | `BAAI/bge-small-zh-v1.5`（默认值，中文句子向量 512 维） |
| Embedding Base URL | 留空 |
| Dimension | `512` |
| Max tokens | `512` |
| API Key | 留空 |

**本地 OpenAI 兼容服务**（LM Studio、Ollama、vLLM 的 embedding 模型等）：协议适配器选
`openai`，Base URL 填该服务的 `/v1` 地址，Model ID 填服务里真实的 embedding 模型名，
**Dimension 必须等于该模型的真实输出维度**（如 `nomic-embed-text` 768、`bge-m3` 1024），
API Key 需要非空。只提供聊天模型的端点通常没有 `/v1/embeddings` 路由，会返回
`{"detail":"Not Found"}`，不能当 Embedding 服务用。

**云端 API**：「openai」+ Base URL + 模型名（如 `text-embedding-3-small`）+ 对应维度 + API Key。

保存后点「Embedding 测试」验证；维度不一致会报「Embedding 实际维度为 X，配置值为 Y」。切换
Embedding 档案会改变检索索引记录的模型与维度指纹，索引会被判为需要重建。

本地模型的缓存与离线：

- 默认缓存在系统临时目录（macOS 为 `$TMPDIR/fastembed_cache`），会被系统清理。Core 子进程的
  环境变量是白名单，不接收 `OPENWRITE_FASTEMBED_CACHE_DIR`，但会继承 `TMPDIR`，所以要用固定
  缓存位置时改为启动 dsh 时覆盖：`TMPDIR="$HOME/.cache/openwrite-tmp" dsh web`。
- 首次加载仍需要能访问 Hugging Face；受限网络可用 `HTTPS_PROXY`/`HTTP_PROXY`（这两个在白名单内）。

## Chat 连接测试与思考型模型

「任务 → 模型」的 Chat 页签「连接测试」用固定提示词（「这是连接测试。请只回复 OK。」）
加 **`max_tokens=32`、temperature 0** 请求一次，要求返回的 `content` 非空，否则报
`MODEL_TEST_EMPTY_RESPONSE`：连接测试失败：模型返回空内容，请调大最大输出后重试。

对**思考型模型**（Qwen3 系、DeepSeek-R1 系等），32 个 token 可能全部被推理消耗，正文为空：

| 实测请求（vLLM 上的 Qwen3 系 27B） | 结果 |
|---|---|
| `max_tokens=32`（等于连接测试） | `finish_reason=length`、`content=null`、`reasoning_tokens=32` ❌ |
| `max_tokens=512` | `content="OK"`、`reasoning_tokens=41` ✅ |
| `max_tokens=32` + `chat_template_kwargs.enable_thinking=false` | `content="OK"`、`reasoning_tokens=0` ✅ |
| `max_tokens=32` + `thinking={"type":"disabled"}`（档案的 `thinking_modes` 会发这个） | 无效，仍被推理占满 ❌ |
| 提示词加 `/no_think` | 无效 ❌ |

因此这类模型的对策是：

1. **服务端关闭思考**（唯一可靠方式）：vLLM 启动时加
   `--default-chat-template-kwargs '{"enable_thinking": false}'`，或让服务默认不思考。
   OpenWrite 目前不能从 UI 发送任意 `extra_body`，所以只能在服务端设置。
2. **无视这条测试继续使用**：真正的写章、评审用档案里的 `max_output_tokens`（可到上万），
   不受 32 token 限制；但要留意小预算的内部调用同样可能拿到空内容。
3. **另建一个非思考模型档案**做连通性测试，把任务路由指向真正要用的模型。

档案里的 `thinking_modes` 只覆盖 `chapter_write` / `review` / `revision` 三个操作，取值
`enabled` / `disabled` / `omit`，发送的是 `thinking: {"type": ...}`；只对识别该字段的供应商
有效，上面那台 vLLM 会忽略它。

### 对创作的影响

正文写作本身受影响很小：写章调用的预算远大于推理开销（`agent/writer.py` 用
`max_tokens=max(16384, 目标字数×2)`，辅助调用 4096/8192，连接测试 32 是唯一的小预算）。

Core 明确把「只返回推理」「输出被截断」「返回空内容」当作可恢复错误，并各自有恢复路径：

| 错误码 | 触发条件 | 恢复方式 |
|---|---|---|
| `MODEL_REASONING_ONLY` | 有推理内容但没有最终答案 | 编排器用档案 `max_output_tokens` 重试，并追加「压缩内部推理，立即从最终内容开始输出」 |
| `MODEL_OUTPUT_TRUNCATED` | `finish_reason` 为 `length`/`max_tokens`/`incomplete` | 工具循环把预算提到 `min(当前×2, context_tokens÷2)`；写章与评审按章节/评审域拆分重试 |
| `MODEL_EMPTY_RESPONSE` | 无内容且无推理 | 同上重试；连接测试直接报错 |

实际代价是**重试**：更慢，且重试预算可能一次涨到档案的 `max_output_tokens`，费用与耗时都会上升；
工具参数被推理截断时还会先报 `MALFORMED_TOOL_ARGUMENTS`。

另外，请求里的 `max_tokens` 会原样取档案值（`llm/client.py`：未显式指定时用
`config.max_tokens`）。把 `max_output_tokens` 填成 130000 时，长篇小说后段提示词变长，容易顶到
服务端的 `max_model_len`（上面那台是 262144）而被拒绝或提前截断，建议控制在 32768–65536。

## GitHub 源码安装

源码安装使用 dsh 官方 GitHub source 机制：`dsh plugin --profile web add -w github:LiPu-jpg/Openwrite#v0.2.8`。仓库 `prepare` 自行构建三个插件，Core wheel 随固定提交提供，不访问相邻工作区。源码安装需要 Git 和构建依赖；建议普通用户优先使用已验收 Release 包。

如果 pnpm 按本机策略阻止构建，按它显示的构建审批指引仅批准本包，再重试。安装脚本不放宽构建授权。源码安装也必须在发布验收中通过，不能用本机已编译目录代替。

本地克隆、从源码构建、开发直连启动和验证命令，见 [从源码构建与运行](SOURCE_RUN.md)。

## 诊断

界面「OpenWrite → 诊断与帮助」显示实时状态与可恢复错误。解包后执行 `node package/scripts/maintenance.mjs doctor --profile web` 检查标准包登记、**bundle 与 dependencies 中的旧包**、以及环境是否准备；不打印凭据或调用模型。若仍报告 leftover 旧包，先停止 dsh 再执行 `migrate --apply`。

受管理后端随 **dsh 进程** 运行：首次打开时准备，就绪后若进程异常退出会有限次自动恢复（带退避），启动失败也按同一上限继续重试。连续失败后显示脱敏的退出码/信号并提供手动重试；恢复失败不会把 dsh 一起退出。取消准备、退出 dsh 或卸载插件不会把后端拉起来，也不会留下未处理错误。恢复只重新连接写作服务，不会重放写作、评审或其他付费请求。本插件不是开机常驻服务。

小说工具的只读 / 计划模式 / Workspace 绑定是 **dsh 策略与桥接检查**，不是操作系统沙箱。受管理 Python 与安装子进程只继承启动、代理和证书所需环境变量，不把无关 API Key 传给子进程。

反馈请提供宿主版本、系统与架构、Release 版本、安装阶段及脱敏错误。来源、许可证和校验值见 Release 附件及包内 `release/`。不要提交 API Key、浏览器登录 URL 或作品全文。
