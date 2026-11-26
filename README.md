# IMREPO — Image Registry Studio

> DBX 插件：在同一个工作台里浏览、管理和检查 OCI / Harbor 容器镜像仓库。

[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![DBX Store](https://img.shields.io/badge/DBX%20Store-已上架-success)](#安装)
[![Platforms](https://img.shields.io/badge/platforms-windows%20%7C%20linux%20%7C%20macos%20(x64%2Farm64)-informational)](#安装)
[![Plugin API](https://img.shields.io/badge/DBX%20Host%20API-1-informational)](https://dbxio.com/cn/docs/plugin-development)

兼容 Harbor 和主流 OCI 仓库：项目与命名空间导航、Tag 重命名与删除、镜像层解析、Trivy 漏洞报告、
总览图表和多架构镜像展示。

## 安装

在 DBX 插件中心搜索 **IMREPO**（插件 ID `com.leavingrain.imrepo`）。商店包由 DBX Store 签名，
安装时自动验证，不需要开启「允许安装未签名开发包」。

<details>
<summary>手动安装 / 指定版本</summary>

从 [Releases](https://github.com/LeavingRain/dbx-plugin-imrepo/releases) 下载对应平台的 `.dbxp`：
`windows-x64` / `windows-arm64` / `linux-x64` / `linux-arm64` / `darwin-x64` / `darwin-arm64`，
然后在插件中心开启「允许安装未签名开发包」。

Release 里是未签名候选包（上架用的输入，见[发布](#发布)），内容与商店签名包相同。
适合商店版本暂不可用或需要固定版本时使用。
</details>

装好后新建连接，选择仓库类型，填写地址和认证方式。

Sidecar 是静态链接的 Go 二进制（`CGO_ENABLED=0`），不依赖目标机 libc。已在麒麟 V10（aarch64）、
openEuler 24.03（amd64）和 Windows 11 Enterprise 实体机上安装使用；CI 在 ubuntu、windows、macos
真实 runner 上各跑一遍完整测试。平台细节见 [docs/DESIGN.md](docs/DESIGN.md#平台支持)。

## 功能

| 模块 | 说明 |
|------|------|
| 连接与预设 | Harbor / Registry v2 / Docker Hub / GHCR / 阿里云 ACR / 腾讯云 TCR / AWS ECR / Azure ACR / Nexus / Artifactory / Quay |
| 认证 | Basic、Harbor Robot、Bearer/PAT、云厂商 AK/SK（AWS SigV4、阿里云 HMAC-SHA1、腾讯云 TC3）、免认证 |
| 仓库浏览 | Harbor 项目树、通用 `_catalog`、V2 命名空间树（按仓库名首段分组）；搜索同时匹配项目和仓库 |
| 项目总览 | 点项目文件夹列出该项目跨仓库聚合的镜像，含镜像数、总大小、仓库数和 Tag 数，按推送时间排序 |
| 总览图表 | 统计卡和 SVG 柱状图：Harbor 显示拉取最多的项目和存储分布，V2 显示命名空间存储分布 |
| 多架构展示 | 每个平台一个徽章（`linux/amd64`、`linux/arm64`…），V2 侧懒加载 |
| Tag 管理 | 列出 Tag、重命名（写新 Tag + 删原 Tag）、删除（OCI DELETE 或 Harbor 原生接口） |
| 上传镜像包 | 解析 `.tar` / `.tar.gz` 中的 docker save 归档或 OCI 布局，展示站点、项目、仓库、版本、格式、平台、层数和大小；目标固定为当前仓库，仅目标 Tag 可编辑；分块上传 |
| 下载镜像包 | 将 Tag 或 digest 导出为 `.tar` / `.tar.gz`：Docker 归档（`docker load` 可直接加载）或 OCI 布局（skopeo/podman）；多架构可选平台，OCI 可导出全部平台。桌面宿主写入保存对话框，网页宿主回退浏览器下载 |
| 删除仓库 / 项目 | 仅 Harbor。危险区列出将删除的仓库、镜像、Tag、大小和受保护 Tag，必须输入目标名；后端也会校验 |
| 镜像层解析 | Manifest → Index → Config，逐层指令和大小，带占比、最大层高亮和排序 |
| 漏洞报告 | Harbor/Trivy CVE 分级统计，按设置阈值判定是否达标，支持缓存、刷新和触发扫描 |
| 清理无 Tag 产物 | 扫描并列出无 Tag 产物（含大小和合计），勾选后逐个复核删除；受规则保护的条目不可选 |
| 设置 | 全局（用户、扫描器、关于）和项目级（可见性、配额、成员与角色、回收策略、扫描器），按连接保存，由后端强制执行 |
| 拉取命令 | 镜像地址、docker、nerdctl、crictl、podman、ctr、docker login，点击整行复制 |
| 加载体验 | 展开项目时后台预热前 6 个仓库；60 秒列表缓存和请求去重；慢响应不会覆盖后点的仓库 |
| AWS 私有 ECR | 仓库列表走 `ecr:DescribeRepositories`（ECR 不实现 `/v2/_catalog`），其余功能走标准 OCI；认证支持 AK/SK、Session Token、环境变量、`~/.aws` profile、ECS 任务角色和 EC2 实例角色 |
| 大文件传输 | 上传按 1 MiB 分块，下载按 4 MiB 分块并走宿主的二进制保存通道；临时文件放在插件数据目录的 `transfers/`，关闭对话框或完成后删除，异常退出由 24 小时清扫兜底 |
| Harbor 自动识别 | 类型选为 Registry v2 但服务器是 Harbor 时，探测 `/api/v2.0/ping` 后启用 Harbor 功能 |

## 仓库类型支持范围

插件按能力而非品牌区分：标准 OCI Distribution 接口的功能所有类型可用；绑定 Harbor REST API 的功能
只在 Harbor 上出现。不可用的按钮会置灰并说明原因。

| 功能 | Harbor | 通用 OCI v2<br>(Registry / Nexus / Artifactory / Quay / 云厂商托管) | 不可用的原因 |
|---|:---:|:---:|---|
| 仓库与镜像浏览 | ✅ | ✅ | — |
| 命名空间 / 项目树 | ✅ | ✅ | v2 无项目对象，按仓库名首段分组为命名空间 |
| 多架构展示 | ✅ | ✅ | 两者都按 tag 读 manifest（懒加载并缓存）；Harbor 列表接口不返回平台信息 |
| 每 tag 的 sha256 | ✅ | ✅ | v2 的 `tags/list` 不带 digest，与架构信息在同一次请求取回 |
| Tag 重命名 | ✅ | ✅ | OCI 没有 rename 原语，重命名 = 写新 Tag + 删原 Tag |
| Tag 删除 | ✅ | ✅ | 走 OCI `DELETE`（需仓库开启删除）；Harbor 用原生接口按 Tag 删 |
| 删除仓库 / 项目 | ✅ | ❌ | v2 协议没有仓库级删除接口，也没有项目对象；两者都是 Harbor 的 REST 资源 |
| 镜像层逐层解析 | ✅ | ✅ | — |
| 拉取命令生成 | ✅ | ✅ | — |
| 上传本地镜像包 | ✅ | ✅ | 走标准 OCI push；Harbor 的目标需写成「项目/仓库」 |
| 下载镜像包 | ✅ | ✅ | 走标准 OCI pull；由 sidecar 边读边打包 |
| 总览图表（存储分布） | ✅ | ✅ | — |
| 总览图表（拉取次数） | ✅ | ❌ | 数据来自 Harbor 审计日志，v2 无 |
| 漏洞报告 (CVE) | ✅ | ❌ | 需要扫描器 API（Harbor/Trivy），v2 协议无此接口 |
| 无 Tag 产物清理 | ✅ | ❌ | v2 的 `catalog` / `tags` 只暴露有 tag 的 manifest，无法枚举无 tag 的 |
| 新建项目 / 配额 / 成员 / 可见性 | ✅ | ❌ | 都是 Harbor 的 REST 资源，v2 无对应对象（命名空间是分组视图） |
| 操作日志 | ✅ | ❌ | Harbor 专有的审计日志 |
| 用户管理 | ✅ | ❌ | Harbor 的用户体系，v2 无 |

云厂商托管仓库（Docker Hub、GHCR、ACR、TCR、ECR）在协议上都是 OCI v2 加各自的认证方式，
能力范围与「通用 OCI v2」一列相同；差异在认证（AK/SK 换临时 Token）和托管商额外 API
（本插件未对接）。

## 安全

### 连接凭据的本地缓存

插件会缓存最后一次使用的连接密码，优先放在 DBX 宿主提供的插件数据目录
（`DBX_PLUGIN_DATA_DIR`，与 `settings.json` 同目录）：

| 部署方式 | 路径 |
|---|---|
| 宿主提供了插件数据目录（Docker / dbx-web 和新版桌面端） | `$DBX_PLUGIN_DATA_DIR/credentials.json`。该目录位于 DBX 数据目录内，备份或迁移时凭据和设置会一并保留 |
| 宿主未提供（旧版本宿主、独立运行） | Windows `%AppData%\imrepo-dbx-plugin\`；macOS `~/Library/Application Support/imrepo-dbx-plugin/`；Linux `~/.config/imrepo-dbx-plugin/`（或 `$XDG_CONFIG_HOME` 下同名路径） |

首次在提供数据目录的宿主上启动时，旧位置的 `settings.json` / `credentials.json` 会复制到新位置；
新位置已有同名文件则不覆盖，旧文件保留。

文件权限为 `0600`，但 base64 是编码不是加密：能读取该文件的本地进程或用户都能还原密码。
威胁模型与 `~/.docker/config.json` 相同。

缓存的原因：DBX 重连时可能不再下发连接密钥，没有缓存的话，成员管理、产物清理和漏洞报告等
需要认证的操作会全部失败。

建议：

- 不需要时删除该文件，插件下次需要时会重新写入。
- 此类连接使用最小权限账号，例如 Harbor robot account 或只读令牌，不要用管理员账号。
  浏览、镜像层、架构、漏洞报告和拉取命令只需要读权限。

### 其他

- 凭据不经 UI：工作台只持有 `connectionId`，密钥由宿主在生命周期请求中下发给 sidecar。
- 日志不含密钥：`stderr` 只输出诊断信息，不含密码或令牌。
- 删除是软删除：删除 Tag 或产物只解除引用，空间需注册库执行 GC 回收。插件提供 Harbor GC
  的查看和触发入口，见「设置 → 镜像回收 (GC)」。

## AI 助手工具

插件通过 MCP 向 DBX 内置 AI 助手（Agent 模式）提供工具，默认关闭。需要在
「插件中心 → 已安装 → 内置 AI 工具」中开启。

| 工具 | 作用 | 需确认 |
|---|---|:---:|
| `list_projects` | 列项目（Harbor）或命名空间（v2） | — |
| `list_repositories` | 列某项目下的仓库 | — |
| `list_tags` | 列 tag：digest / 大小 / 架构 / 推送时间（默认最近 50 个） | — |
| `image_info` | 单个镜像：digest、大小、平台、层数、推送时间 | — |
| `vulnerabilities` | CVE 汇总与阈值判定（仅 Harbor） | — |
| `create_project` | 新建项目（仅 Harbor） | ✅ |
| `set_project_public` | 切换项目可见性（仅 Harbor） | ✅ |
| `retag` | 重命名 tag | ✅ |
| `delete_tag` | 删除 tag | ✅ |
| `delete_repository` | 删除整个仓库，`confirm` 必须逐字重复仓库名（仅 Harbor） | ✅ |
| `delete_project` | 删除整个项目，`confirm` 必须逐字重复项目名（仅 Harbor） | ✅ |

### 安全边界

- 只读工具带 `readOnlyHint`，可直接执行；写操作每次由宿主弹出完整参数，用户选择允许一次或拒绝
  （5 分钟无响应视为拒绝）。
- 插件规则不因宿主确认而放宽。例如删除受保留策略保护的 tag，即使用户在宿主层允许，插件仍会拒绝
  并说明原因。工具调用的是工作台同一套后端函数，不是另一条路径。
- 凭据不经工具参数：连接信息由宿主在 `lifecycle` 中下发，模型看不到也无需提供密钥。
- 工具只在用户已打开的连接上工作，且 `external_tools` 关闭，不会出现在 Claude Code / Cursor
  等外部 MCP 客户端。

## 开发

依赖：Node.js 22+、Go 1.22+、Python 3（仅脚本用）。

```bash
npm install                 # 插件 CLI（自带与本版本匹配的 Go SDK）
npm run package             # 交叉编译六个平台 → dist/*.dbxp + release-candidates.json
npm run preview             # 生成验证台 .preview/preview.html
npm test                    # 全量：包门禁 + 10 套端到端 + 配色门禁 + 翻译门禁

npx dbx-plugin dev --path . --port 5190      # 本地调试宿主 → http://127.0.0.1:5190/
```

`CGO_ENABLED=0` 让 Sidecar 静态链接，与构建机 libc 无关，所以一条命令能出全部平台。
官方 CLI `dbx-plugin package .` 只能打当前平台的包（拒绝交叉编译），多平台由
`tools/build-packages.py` 负责：它按目标平台改写 manifest 的 `entrypoints.backend.executable`，
并给二进制加 0755。官方 CLI 的产物可用于验收：两者的 `manifest.json` 应逐字节一致。

### 测试

Sidecar 测试跑打包产物（`dist/` 中对应当前平台的 `.dbxp` 内二进制）；UI 测试用无头浏览器渲染真实
`ui/index.html`。`tools/_harness.py` 按主机选包、把配置目录隔离到临时目录，并查找 Chromium 系浏览器
（优先 `chrome-headless-shell`，可用 `CHROME_HEADLESS_SHELL` 指定；本机找不到会跳过，CI 找不到则失败）。

每个测试输出实际执行的断言数（`RESULT: ALL PASS (N checks)`）。

| 测试 | 断言数 | 验证的是 |
|---|---|---|
| `test-settings-e2e.py` | 193 | 设置持久化、校验与后端实际行为，V2 命名空间、总览和包布局 |
| `test-ui-e2e.py` | 217 | 界面行为与策略可见性：总览图表、架构徽章、连接切换、空项目侧栏、日志日期筛选、镜像包上传/下载 |
| `test-transfer-e2e.py` | 93 | 镜像包上传/下载：归档解析、推送请求、分块回退、平台选择、导出内容和错误路径 |
| `test-layers-e2e.py` | 23 | 逐层大小、指令和总量解析 |
| `test-cleanup-e2e.py` | 21 | 无 Tag 产物清理的安全规则 |
| `test-harbor-paging-e2e.py` | 35 | Harbor 列表分页、空列表、artifact 全量读取和总览扫描预算 |
| `test-logs-e2e.py` | 17 | 操作日志的日期范围过滤和分页 |
| `test-retag-e2e.py` | 17 | 重命名 Tag 的请求序列 |
| `test-ecr-e2e.py` | 26 | 私有 ECR 列表、SigV4 分页、Session Token、环境变量/profile、权限错误和无凭据路径 |
| `test-mcp-e2e.py` | 51 | MCP 工具的参数校验和绕过安全规则的拒绝路径 |
| `check-contrast.py` / `check-i18n.py` | 全量配色对 / 393 键×2 语言 | 配色对比度门禁、翻译门禁 |
| `check-packages.py` | 6 个目标 | 每个包二进制的格式、架构、签名和 manifest 路径是否与平台相符 |
| `check-browser-baseline.py` | 34 个特性 | 是否使用了新于 Chrome 109 的 CSS/JS 特性（DBX 最低引擎） |

## 目录结构

```
dbx-plugin-imrepo/
├── manifest.json        # 插件清单：connection-provider + workbench + backend（版本号在这里）
├── dbx-plugin.toml      # 构建配置（[backend] Go + [package].include）
├── assets/              # 插件图标
├── ui/                  # 工作台 UI（沙箱 iframe，classic script + window.IMREPO）
│   ├── index.html       # 按依赖顺序加载 js/*（不用 ES module，见 docs/DESIGN.md）
│   ├── styles.css       # 明/暗主题调色板（私有 --im-* 令牌）
│   └── js/              # 20 个模块：state / i18n / dom / format / rpc / render-* / main
├── backend/             # Go Sidecar
│   ├── main.go          # SDK 入口 + 身份解析（从包内 manifest 读 id/version）
│   ├── dispatch.go      # RPC 方法表
│   ├── connection.go    # 连接生命周期与宿主 payload 解析
│   ├── oci.go           # OCI Registry v2 客户端（含 token-challenge）
│   ├── harbor.go        # Harbor REST v2.0 客户端
│   ├── v2projects.go    # V2 命名空间视图（"项目" = 仓库名首段）
│   ├── cleanup.go / retag.go / layers.go    # 清理、重命名、镜像层解析
│   ├── transfer.go / oci_transfer.go        # 镜像包解析/推送/导出，流式 blob 读写
│   ├── scanner.go / settings.go             # CVE 报告与扫描器；设置与策略引擎
│   ├── registry.go / admin.go               # 项目生命周期；管理面板聚合
│   ├── session.go / types.go / pool.go      # 会话、连接解析、有界并发
│   └── credentials.go / auth.go / cloud.go  # 凭据兜底与各认证方式
├── tools/               # 开发脚本（Python，无第三方依赖）
│   ├── _harness.py      # 测试公共件（选包/隔离配置/找浏览器/断言计数）
│   ├── build-packages.py / make-preview.py / shoot-screenshots.py
│   └── check-packages.py / check-browser-baseline.py / check-contrast.py
│       / check-i18n.py / test-*-e2e.py
├── .preview/mock.js     # 验证台的 mock 桥（源文件，要提交）
├── docs/                # 设计文档
└── CHANGELOG.md
```

## 发布

已上架 **DBX 官方商店**（插件 ID `com.leavingrain.imrepo`）。商店条目和签名产物列表见
[`catalog/index.json`](https://dl.dbxio.com/catalog/index.json)。

几点约束：

- `.dbxp` 不提交进 Git（`dist/` 已忽略），只作为 Release 附件。Release 上是未签名候选包，
  商店分发的是签名后的副本。
- Release 附件上传后不可修改。字节变了（哪怕只是重新打包），候选里的 `sha256` / `size` 就对不上，
  签名会被拒绝，必须重发 Release 并更新候选元数据。
- 候选元数据里的 `id` / `version` / `publisher` 必须与包内 `manifest.json` 一致。

## 文档

- [`docs/DESIGN.md`](docs/DESIGN.md)：架构、后端 RPC 方法表和设计取舍，包括打包路径改写、
  主题令牌前缀、classic script、后端策略和清理无 Tag 的安全设计。
- [`CHANGELOG.md`](CHANGELOG.md)：版本变更。

## 许可

[Apache-2.0](LICENSE)。插件使用官方 SDK `github.com/t8y2/dbx/plugins/sdk/go/dbx-plugin-sdk`（同为 Apache-2.0）。
