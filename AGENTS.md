# AGENTS.md

微信小程序 `.wxapkg` 解密与解包桌面工具。**Wails v2**（Go 后端） + **Vue 3 / TypeScript / PrimeVue**（前端），产物为 Windows x64 与 macOS universal 桌面应用。原项目已从 Wails3 降级回 Wails2，不要照 Wails3 的文档改。

仅用于学习研究，UI 与注释、错误信息一律用中文。

## 构建与验证

工具链：Go ≥ 1.26（见 `go.mod`，当前 `go 1.26.0`）、Wails v2.17.1、Node（CI 未锁定版本）。

```bash
# 首次准备（CLI 版本必须与 go.mod 中 wails/v2 的版本一致，不要用 @latest）
go install "github.com/wailsapp/wails/v2/cmd/wails@$(go list -m -f '{{.Version}}' github.com/wailsapp/wails/v2)"
cd frontend && npm install && cd ..

# 开发（热更新）
wails dev

# 构建，产物在 build/bin/
wails build -platform windows/amd64   # Windows x64
wails build -platform darwin/arm64    # macOS Apple Silicon

# 后端签名变更后必须重新生成前端绑定
wails generate module
```

- **wails CLI 版本坑**：CLI 版本必须与 `go.mod` 中 `wails/v2` 一致，不要用 `@latest`（`@latest` 曾跳到需要更高 Go 版本的 v2.17.1，直接让 CI 的 `Get Wails` 步骤挂掉）。CI 已改为自动从 `go.mod` 读版本并安装。
- 升级 wails 库往往连带要求更高 Go 版本（v2.17.x 要求 Go ≥ 1.26），升级前先在 B 上 `gvm install go1.xx.y --binary` 验证，否则 CI 会在装 CLI 阶段失败。
- 后端改动可用 `go vet ./...` / `go build ./...` 做最小校验；**仓库内没有测试**，不要自行新增测试文件或测试用例，除非明确要求。
- 前端只用 `npm run build`（等同于 `vite build --mode production`）验证能否编译；`npm run lint` 不存在（`package.json` 无 lint 脚本）。
- 本机（开发机 A）当前**没有 Go / Wails 工具链**；编译验证统一在服务器 B 上做（gvm + 默认 Go 1.26.11，`GOMODCACHE=$HOME/go/pkg/mod` 跨版本共享，工程位于 `~/workspace/wxapkg`）。不要擅自下载安装工具链或浏览器（`wails dev` 会拉起 webview/浏览器）。
- Linux 上无法 `wails build` 出 Windows/macOS 安装包（需 mingw / macOS SDK），但 `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build .` 与 `GOOS=darwin GOARCH=arm64 go build .` 可以验证编译面（`//go:embed all:frontend/dist` 需先造一个占位 `frontend/dist/index.html`）。
- 依赖管理：`wails.json` 与 `.github/workflows/release.yml` 都是 **npm + package-lock.json**（CI 用 npm cache）。虽然个人偏好 pnpm，但改这里会破坏 CI 缓存，动手前先问。

## 代码结构

```
main.go                    入口：embed frontend/dist，wails.Run，Bind AppService
app.go                     AppService 门面层（前端唯一调用面）
wechat/core.go             解包核心：Unpacker、wxapkg 二进制解析、并发落盘
wechat/util.go             扫描识别 wxId 目录、解密、路径大小、UUID
wechat/beautify.go         JSON/HTML/JS 美化
wechat/core_windows.go     平台实现：注册表 + v4 目录探测（//go:build windows）
wechat/core_macos.go      平台实现：Container 路径探测（//go:build darwin）
wechat/core_other.go       非 Win/mac 空实现
frontend/src/App.vue       主界面：列表、搜索、三个弹窗调度
frontend/src/components/   ScanDialog / UnpackDialog / SettingsDialog
frontend/src/entries/      entries.ts（事件名、类型）+ util.ts（toast、格式化）
frontend/wailsjs/          自动生成的前端绑定（已提交，禁止手改）
```

平台相关的路径探测通过 `wechat.Platform`（`platform` 接口，`newPlatform()` 按 build tag 选择实现）注入，新增平台只需加 `core_<os>.go`，不要在 `core.go` 里写 `runtime.GOOS` 分支。

## 关键机制

- **推拉模式（push-pull）**：`AppService.items` 是 `sync.Map`。`UnpackWxapkgItem` 拿到 item 后存入 map 再后台解包，进度回调里 `EventsEmit(ctx, "unpack:progress-changed", uuid)` 只发 UUID；前端 `EventsOn` 收到后调 `GetWxapkgItem(uuid)` 拉最新状态。不要把整个 item 塞进事件（会破坏这条链路）。
- **绑定可见性**：Wails 只导出 Go 结构体的**导出字段**，`WxapkgItem` / `UnpackOptions` / `PathScanResult` 的字段名即前端 TS 字段名（首字母大写）。改字段必须重新 `wails generate module`，且要同步 `UnpackDialog.vue` 里 `reactive<UnpackOptions>` 的初值。
- **解包安全护栏**（`core.go` 的 `analyze`）：文件名去重（`a.txt` → `a-1.txt`）、目录穿越校验（`filepath.Abs` + 前缀检查）、`offset+size <= len(data)`（防止伪造索引导致切片越界 panic）、文件名长度 ≤1024、文件总数 ≤102400、魔数 `0xBE ... 0xED`。改动解析逻辑时这些检查不能放宽。
  - 历史坑：曾有 `item.size > 10*1024*1024` 的硬上限，误杀小游戏正常的 17MB `subpackages/script/game.js`，已移除。不要用「文件大小」代替范围内校验，小游戏代码包总上限 20MB、单文件不限。
- **加密文件**：未加密判定靠首字节 `0xBE` + 第 13 字节 `0xED`；密钥就是小程序 `wxid`（`^wx[0-9a-f]{16}$`），解密走 PBKDF2-SHA1 1000 轮 + AES-CBC + 尾部 XOR。落盘权限 `0600`。
- **并发**：`unpack` 固定 20 个 worker + 有缓冲需求的 channel；错误状态与进度更新都在 `u.locker` 保护下写 `item`，任何改并发结构的地方都要保持这把锁的语义。
- 输出目录偏好持久化在 localStorage key `wxapkg:output-directory:preference:v1`（`App.vue`）。

## 版本号

发布版本散落在三处，改版本要同时改：

- `main.go` → `var version = "v2.0.2"`（前端展示用，带 `v`）
- `wails.json` → `info.productVersion`（`2.0.2`，不带 `v`）
- `frontend/package.json` + `frontend/package-lock.json` → `version`（`2.0.2`，两处同步，否则 npm ci 可能报不一致）

发布由 `.github/workflows/release.yml` 触发：推送 `v*` tag → 矩阵构建 **Windows x64**（`windows/amd64`，`CGO_ENABLED=0` + UPX 压缩）与 **macOS Apple Silicon**（`darwin/arm64`，`MACOSX_DEPLOYMENT_TARGET=10.13`，`ditto` 打 zip）→ `softprops/action-gh-release` 建 release。构建目标只有这两个，**不构建 macOS x86_64 / universal**。**未经授权不要打 tag / 发版。**

## 约定

- 代码风格：Go 标准 `gofmt`；前端 2 空格缩进，`<script setup lang="ts">` + `defineModel` 组合式写法。
- 单文件体量：`App.vue`（543 行）、`UnpackDialog.vue`（508 行）已超内控线（组件 200 行 / 源码 300 行）。新增逻辑优先抽出子组件或 composable，不要再往这两个文件堆代码。
- 前端只与 `frontend/wailsjs/go/main/AppService` 和 `frontend/wailsjs/go/models` 交互；`vite-env.d.ts` 里的 `window.go` 声明是手工维护的影子类型，改 `AppService` 方法签名时一起改。
- 不提交配置文件、构建产物、`build/bin`、`frontend/dist`、`frontend/node_modules`（已在 `.gitignore`）。
- 提交信息用中文 + 语义化标签：`[✨ feat]` / `[🐞 fix]` / `[⚡️ perf]` / `[📚 docs]` / `[🎨 style]` / `[🔧 chore]` / `[🛠️ build]` / `[✅ test]`。
