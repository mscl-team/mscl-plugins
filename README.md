# MSCL · 月之海启动器

MSCL（Moon Sea Client Launcher）是一款插件化的 Minecraft Java 版启动器。主程序 `mscl-app.exe` 是宿主（Harness），页面、资源、启动流程、下载源、设置项等都以插件形式挂载，可按职责拆细、启停、替换和扩展。

## 功能

- **实例与版本**：多版本安装、版本隔离、游戏文件校验与修复
- **账号**：离线登录与正版登录
- **资源中心**：Mod / 光影 / 材质包 / 数据包 / 整合包 / 服务端插件 / 启动器插件 / 皮肤
  - 按类型选择安装目标：实例、本地目录、服务器 / 多人游戏目录、启动器本身
  - 来源支持本地导入、URL 下载、MC百科 / MineBBS 等正规渠道
- **下载**：官方 / BMCLAPI / 自定义镜像，并发下载、失败重试、完整性校验
- **插件体系**：Harness Plugin DSL v1，安装向导可勾选默认插件（默认全选）
- **更新**：GitHub Release 检测 + markdown 更新日志弹窗 + 安全替换与失败回滚
- **备份还原**：按内容勾选设置、插件、实例元数据、账号与日志，还原前自动保护当前数据
- **其他**：3D 皮肤预览、主题切换、日志中心、自定义窗口与右键菜单管控

## 仓库分工

| 仓库 | 用途 |
|------|------|
| `mscl-team/mscl`（本仓库） | 启动器本体、安装 / 卸载程序、CI 与自更新源 |
| `mscl-team/mscl-plugins` | 插件仓库：默认插件库与第三方插件分发，见 [mscl-plugins/README.md](mscl-plugins/README.md) |

## 快速开始

### 用户安装

从 [Releases](https://github.com/mscl-team/mscl/releases) 下载 `MSCL_Setup_<version>.exe` 运行即可。安装器是流程式向导：许可协议 → 安装位置 → 默认插件勾选 → 确认安装 → 完成。安装目录内自带独立卸载器；控制面板的卸载入口也指向该卸载器。

### 源码开发

环境要求：Node.js 22、Rust 1.77.2+、Tauri 2 命令行工具（`cargo install tauri-cli`）。

```bash
npm ci                    # 安装前端依赖
npm run dev               # 浏览器预览模式（IPC 使用 mock，不依赖后端）
npm run build:installer   # 全量构建：前端 → 启动器 → 安装器 → release/ 产物
cargo test --manifest-path src-tauri/Cargo.toml
```

`npm run build:installer` 依次执行：

1. `npm run build`（tsc + vite）
2. `cargo tauri build --no-bundle`（启动器，产出 `src-tauri/target/release/mscl-app.exe`）
3. 复制 exe 与许可证到安装器 resources
4. 构建安装器前端
5. 编译安装器 / 卸载器，输出到 `release/MSCL_Setup_<version>.exe` 与 `release/MSCL_Uninstall_<version>.exe`

## 目录结构

```
src/                        启动器前端（React + TypeScript）
  lib/api.ts                IPC 封装与命令契约
  lib/plugins.ts            插件 DSL 类型与内置插件清单
  components/               通用组件（备份、更新弹窗、插件面板、3D 皮肤等）
  pages/                    各功能页面
src-tauri/                  启动器后端（Tauri 2 + Rust）
  src/commands/             按领域拆分的命令实现
  crates/mscl-core/         版本解析、下载、启动命令领域层
  crates/mscl-platform/     Java / 进程 / 平台适配层
installer/                  安装器与卸载器（同一 exe，按启动参数分流）
scripts/build-installer.mjs 全量构建脚本
PLUGIN_DSL.md               插件语法权威文档
plugins.schema.json         插件 DSL 机器可读 schema
```

## 插件体系

插件清单文件固定为 `plugin.json`，安装根目录为游戏数据目录下的 `plugins/`。详细语法见 [PLUGIN_DSL.md](PLUGIN_DSL.md)，机器可读枚举见 [plugins.schema.json](plugins.schema.json)。

- **category**：`core` 核心模块 / `launcher` 启动器插件 / `server` 服务端插件
- **layer**：`harness` 宿主内核 / `module` 功能模块 / `feature` 功能插件
- **capability / hook / extensionPoint**：能力声明、生命周期钩子、UI 与流程挂载位
- **安装来源**：内置清单、本地目录、GitHub 仓库
- **安装目标白名单**：`instance` / `local` / `server` / `multiplayer` / `launcher` 及已登记的自定义目录

内置插件清单在 `src/lib/plugins.ts`，安装器可勾选项来自 `OPTIONAL_DEFAULT_PLUGINS`。

## 更新机制

- 更新源固定为 `https://github.com/mscl-team/mscl`，可用环境变量 `MSCL_LAUNCHER_REPO` 覆盖（便于测试）。
- 启动时按设置检查更新；设置页也可手动检查。版本比较基于 `src-tauri/Cargo.toml` 的 `CARGO_PKG_VERSION` 与 release tag，只有 tag 数字更高才提示。
- 下载资产**精确匹配** `mscl-app.exe`，不会误选同 release 里的安装器或卸载器。
- 更新弹窗展示 release body 的 markdown 更新日志，提供“立即更新 / 暂不更新”。
- 立即更新会下载并校验新版 exe（MZ 头 + 最小大小），启动独立助手安全替换；失败时自动恢复 `.bak` 旧版。
- 插件库更新下载 zip 资产（缺失时使用 tag 的 source zip），先备份 `plugins/` 目录再替换，失败回滚。

## 发布 v0.1.0（用于测试更新）

### 1. 同步版本号

四处必须一致，当前均为 `0.1.0`：

| 文件 | 说明 |
|------|------|
| `package.json` | 前端版本 |
| `src-tauri/tauri.conf.json` | 启动器打包版本 |
| `src-tauri/Cargo.toml` | **更新比较基准**（`CARGO_PKG_VERSION`） |
| `installer/src-tauri/tauri.conf.json` | 安装器版本 |

### 2. 发布基线 v0.1.0

```bash
git tag -a v0.1.0 -m "MSCL 0.1.0"
git push origin v0.1.0
```

推送 tag 会触发 `.github/workflows/release.yml`：

- 在 `windows-latest` 上执行 `npm ci` 与 `npm run build:installer`
- 校验三个产物存在
- 从 `git log` 生成 release notes（即更新弹窗里的 markdown）
- 创建 Release 并上传：`mscl-app.exe`、`MSCL_Setup_0.1.0.exe`、`MSCL_Uninstall_0.1.0.exe`

在 Actions 页确认任务成功，再打开 Release 页确认三个资产齐全。

### 3. 测试启动器自更新

**同版本不会提示更新**，所以测试需要两个版本：

1. 安装刚发布的 v0.1.0，确认能正常启动。
2. 把上述四处版本号同步升到 `0.1.1`，提交并推送。
3. 打 tag 并推送：`git tag -a v0.1.1 -m "MSCL 0.1.1" && git push origin v0.1.1`。
4. 等待 Release 构建完成，确认资产里有新的 `mscl-app.exe`。
5. 回到运行中的 v0.1.0：设置页点“检查更新”，或开启自动更新后重启启动器。
6. 应弹出 v0.1.1 的更新日志弹窗 → 点“立即更新” → 启动器关闭，助手完成替换。
7. 重新打开启动器验证版本已更新，并检查安装目录没有残留 `mscl-app.exe.bak` 与 `updates/mscl-app-*.exe`。

验证失败回滚：手动把下载中的新版文件替换为损坏文件再触发更新，助手应恢复旧版并保持启动器可运行。

### 4. 测试插件库更新

见 [mscl-plugins/README.md](mscl-plugins/README.md) 的“发版与更新测试”一节。要点：设置中插件仓库地址填 `https://github.com/mscl-team/mscl-plugins`，插件库 release tag 需大于 `1.0.0` 才会被启动时检查识别。

## 团队协作

- Organization `mscl-team` 下建 Team（如 `Core-Dev`），对 `mscl` 仓库授予 Write 权限。
- `main` 分支开启保护：禁止直推、要求 Pull Request 与 Review。
- 插件相关改动提交到 `mscl-plugins` 仓库，主仓库只维护宿主与内置插件清单。
