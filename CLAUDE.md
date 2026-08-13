# 项目配置

- 工作目录：/home/project
- 语言：使用中文回答
- Agent 是项目生命周期 owner：先识别并尊重已有项目的技术栈、包管理器和启动方式，禁止因技术栈不同而重新初始化
- 仅当目录中没有任何已有项目标识时：无可信服务端逻辑时默认创建 Vite + React + TypeScript + Tailwind CSS 前端应用。Supabase 本身可作为应用后端；已启用 `supabase` 时，普通 CRUD、登录认证、存储、实时订阅及 Supabase 提供的 CRUD API 使用前端 + Supabase，提到普通 CRUD API/接口不触发全栈。只有必须由可信服务端执行的自定义 API、Webhook、支付回调、定时任务、服务端密钥、私密第三方 API 等逻辑才创建全栈应用（后端默认 Node/Express，用户明确指定 Python 时才用 Python；后端固定 8000、前端固定 5173）。❌ 不生成独立后端服务（无法预览）
- 运行时只支持镜像已预装的运行时（Node、Python）。你以普通用户身份运行，**无法** `apt-get`/`sudo` 安装系统级运行时；需要其他运行时（Go/Java 等）时，向用户说明暂不支持并建议用 Node/Python 替代
- 依赖安装只允许使用项目级包管理器（npm/pip/uv 等），安装到项目目录内，禁止修改系统目录
- 依赖安装命令必须以前台方式运行，禁止后台安装依赖；只有确认退出码为 0 后才能创建或修改 `.lingo/runtime.json`。依赖安装未完成或失败时，禁止创建或修改 `.lingo/runtime.json`，先修复并重新安装成功
- 沙箱预览的 Python 依赖必须对 ServiceManager 托管进程可见。仓库已声明 uv、poetry 或现有虚拟环境时沿用其项目环境；否则普通 requirements 项目必须创建项目内 `.venv`，使用 `python3 -m venv .venv && PIP_USER=0 .venv/bin/python -m pip install -r requirements.txt`，并让 runtime `startCommand` 使用 `.venv/bin/python`。禁止只把依赖安装到 `/home/user/.local` 后使用裸 `python3` 启动

## 运行时契约 `.lingo/runtime.json`

- 启动或恢复应用前必须写入 `.lingo/runtime.json`，声明每个需要常驻运行的服务。**使用以下当前多服务格式**：

  ```json
  {
    "schemaVersion": 1,
    "projectType": "vite|fullstack|other",
    "services": [
      {
        "name": "web",
        "role": "frontend",
        "cwd": "frontend",
        "port": 5173,
        "startCommand": "npx vite --port 5173 --host 0.0.0.0 --strictPort",
        "preview": true
      },
      {
        "name": "api",
        "role": "backend",
        "cwd": "backend",
        "port": 8000,
        "startCommand": "node --watch server.js"
      }
    ],
    "checkCommand": "npm run check"
  }
  ```

- 单前端应用只声明一个 `preview:true` 的 service；全栈应用只标前端一个对外预览入口（后端不对外暴露）
- 导入项目的预览端口服从平台预览契约而不是仓库默认值：Vite 前端预览固定使用 `5173`，`startCommand` 必须显式传入 `--port 5173 --host 0.0.0.0 --strictPort`，不得沿用仓库中的 `3000`；纯后端预览使用 `8000`
- 契约规则（违反将被判定为无效并要求你修复）：
  - `cwd`：相对项目根目录的安全相对路径（如 `"."`/`"frontend"`/`"backend"`），禁止绝对路径或 `..` 越界
  - `port`：必须是 1024～65535，且不得使用保留端口 **9001 / 5000**；各 service 端口不得重复；应用必须监听 `0.0.0.0`
  - `preview`：有对外端口的应用**必须且只能有一个** service 设 `preview:true`；纯 worker（无端口）应用则不允许 `preview`
  - `startCommand`：必须是可直接执行、非交互、前台常驻的启动命令；**禁止使用 `&` 后台化、禁止 `nohup`/`disown`**——进程由平台（ServiceManager）托管，你只负责声明
- 平台依据此契约启动并持有进程；应用掉线（如沙箱重建）时会自动重放 `startCommand` 恢复。你不要留下后台进程，也不需要自行做进程守护
- 平台不会回写此文件：契约是你的单向声明，请一次写准确
- 创建或修改 runtime 契约后，必须等待 ServiceManager 拉起服务并轮询 `preview:true` 服务的根地址。只有获得 HTTP 2xx/3xx 且确认响应属于当前应用才算预览成功；连接失败、HTTP 404、端口上其他服务的响应、`idle` 或 `resumeRequired` 都必须继续排查并修正依赖环境、端口或 `startCommand`，禁止结束本轮或声称“平台稍后会拉起”
- 禁止使用旧的顶层 `previewPort` / `startCommand` 单服务格式；所有服务必须声明在 `services[]` 中
- 新建 Vite 项目使用 HashRouter；已有导入项目保持其原有路由模式

## 部署契约 `.lingo/deployment.json`

- 识别项目后同时维护 `.lingo/deployment.json`；技术栈、目录、依赖、端口或启动方式变化时必须同步更新。
- 固定格式：`{"schemaVersion":1,"runtime":"npm","port":8080,"applicationStart":"...","applicationStop":"..."}`。`runtime` 必须根据项目实际主运行时填写为 `npm`、`python`、`java` 或 `go`，禁止根据期望部署环境臆测；Node.js/npm/pnpm/yarn 项目统一填 `npm`。
- 生成前先读取 README 和实际项目清单（如 `package.json`、锁文件、`pyproject.toml`、`requirements.txt`），并检查现有生产部署入口（如 `Procfile`、`boot.sh`、Dockerfile 的 CMD/ENTRYPOINT、package scripts、Maven/Gradle 配置）。以仓库现有部署入口的启动语义为首要依据，README 次之，再做最小推断；复用项目声明的包管理器、依赖安装、构建和生产启动方式。禁止仅凭文件后缀猜测或无依据改用开发服务器；禁止根据框架惯例补写仓库未使用的命令。
- 先确定唯一的云上 HTTP 入口、端口、安装命令、构建命令和生产启动命令。只有仓库明确要求且部署必需时，才加入数据库迁移、翻译编译或种子数据等一次性步骤；不得臆造迁移、翻译编译、种子数据或把可选开发命令加入部署链路。若项目是库、CLI 或无法归约为单个 HTTP 服务，不要伪造可部署契约，应向用户说明当前无法直接部署及所缺条件。
- 各运行时必须沿用仓库的同一套工具链：Node 按锁文件选择 npm/pnpm/yarn 并使用仓库已有的生产脚本；Java 优先使用仓库自带的 Maven/Gradle Wrapper；Go 使用仓库实际 main package 构建出的二进制。Python 必须用同一个解释器或项目环境完成安装、初始化和启动：使用 `python3 -m pip` 安装，以 `python3 -m flask --app <入口>` 执行 Flask CLI，以 `python3 -m gunicorn`/`python3 -m uvicorn` 启动；仓库明确使用 venv、uv 或 poetry 时则全链路使用该项目环境，禁止混用裸 `pip` 与裸 `gunicorn`。
- `.lingo/runtime.json` 与部署契约的进程模型不同：runtime 的 `startCommand` 必须前台常驻；OOS 将 `applicationStart` 作为一次性部署 Hook 执行，长期服务必须后台拉起后正常退出，禁止让服务进程在前台阻塞 Hook。
- `port` 是云上 HTTP 服务端口，应用必须监听 `0.0.0.0` 和该端口。脚本从 Group 工作目录开始，Git 代码已位于 `code_deploy_application/`；先进入该目录，不要写 Application/Group 相关的绝对目录。
- ROS/OOS 模板已负责通用部署编排。`applicationStart` 只包含当前项目实际需要的依赖安装、构建和启动命令；除非 README 明确要求，否则不要生成系统运行时安装、系统包管理器探测、旧 PID 清理重试、部署重试或健康轮询。
- `applicationStart` 中所有可能失败的前台步骤（进入目录、安装依赖、构建、迁移、资源编译、启动前检查）必须用 `&&` 串联并成功后再启动服务，不得仅用换行或 `;` 连接；禁止把 `nohup`、端口环境变量或启动命令写成 `npm run build` 等构建命令的参数，例如禁止 `npm run build nohup ...`，禁止 `npm run build PORT=8080 nohup ...`。
- 长期服务必须使用 `cd code_deploy_application && <安装命令> && <构建命令> && { <可选环境变量> nohup <启动命令> >> /root/application.log 2>&1 & APP_PID=$!; echo "$APP_PID" > /root/application.pid; sleep 2; kill -0 "$APP_PID"; }` 结构，按项目实际需要省略不存在的安装、构建或环境变量步骤；需要 `PORT` 时把环境变量放在 `nohup` 前，禁止写成 `nohup PORT=8080 node ...`。`&` 只能后台化花括号内的服务命令，启动命令自身不得再次守护化、fork 到后台或使用 daemon 模式；禁止把进入目录、安装或构建所在的整个 `&&` 链后台化。`$!` 必须记录实际服务进程 PID。后台启动后只做这一次进程存活检查，用于让命令不存在或入口立即崩溃时令 Hook 失败；这不是健康轮询，不要增加重试循环。`&` 本身已是命令分隔符，禁止写成 `&;`。
- `applicationStop` 保持简洁且可重复执行：PID 文件存在时按 `/root/application.pid` 停止该应用进程并删除 PID 文件；文件不存在或进程已退出时也应成功。除非项目明确需要，否则不要增加等待循环或强制终止回退，且不得用 `exit` 提前终止部署流程。
- **任何形式的 `pkill -f` 都被校验器禁止**，即使带进程匹配条件也不允许；同时禁止删除工作目录之外的项目文件、写入凭证，或把 Token/密钥放入契约。


## 运行时环境变量

运行时已将以下环境变量注入到 `.env` 中：
- `VITE_APP_ID`

- 禁止创建、覆盖或替换 `.env`、`.env.local` 或任何环境文件为占位值。
- 禁止为上述变量写入 `your-api-key-here`、`your-app-id-here`、空字符串或伪造凭证。
- 运行时按技术栈对应的方式读取变量：
  - 前端（Vite）：`import.meta.env.<KEY>`（仅 `VITE_` 前缀的变量会暴露到浏览器）。如需类型声明，请创建或更新 `vite-env.d.ts`，禁止通过编辑 `.env` 来实现类型化。
  - 后端（Node）：`process.env.<KEY>`（通过框架或 dotenv 加载 `.env`）。
  - 后端（Python）：`os.environ["<KEY>"]` / `os.getenv("<KEY>")`（通过 python-dotenv 或框架配置加载 `.env`）。
- 密钥必须保留在服务端：禁止把仅后端使用的密钥暴露到浏览器包中，禁止在客户端代码中硬编码。
- 禁止打印、回显、记录或泄露这些变量的真实值。


## 数据库连接守卫

当前未启用 Supabase 或其他数据库连接 Skill。

- 禁止创建 Supabase SQL migrations、数据库表结构或 RLS policy。
- 如果用户需求需要真实数据库、登录账号、订单、购物车、打卡记录等业务持久化，必须先说明需要选择 Supabase/数据库连接后再继续，不要自行伪造数据库执行链路。
- 不要把 IndexedDB、localStorage、静态数组或 mock 数据描述成“数据库实现”或“真实持久化”；若只能做本地演示，必须明确它只是临时本地状态。