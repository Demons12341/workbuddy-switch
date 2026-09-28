# 本地定制说明（自分支 `local`）

这份文档记录**本人对上游仓库的本地改动**、以及**日常使用 / 打包 / 跟随上游更新**的完整流程。
上游仓库：`https://github.com/changexbc/workbuddy-switch`（就是本仓库的 `origin`）。

---

## 一、分支怎么摆的

| 分支 | 用途 | 说明 |
| --- | --- | --- |
| `main` | **只跟上游**，不要在上面改东西 | 跟踪 `origin/main`，与作者保持一致 |
| `local` | **我自己的改动都在这里** | 基于作者发布提交 `46abee9` + 一次本地提交 `24e63f6` |

关键要求：**永远在 `local` 上工作**。如果哪天误切回 `main`，改动会"消失"（其实在 `local` 里，切回来就看到了）。

```bash
git switch local        # 日常第一步：确认自己在 local
git branch -v           # 看看两个分支各在哪
```

---

## 二、本地改了哪些东西（2026-09-27）

| 主题 | 改动内容 | 主要文件 |
| --- | --- | --- |
| **模型限额 chip** | 把「插件宿主 agent 日志树」纳入限额扫描（此前只扫 exthost 树，真实 429 一条都查不到） | `crates/wb-switch-core/src/modules/limits.rs`、`src/pages/AccountsPage.tsx` |
| **VS Code 切换「关不掉」** | 优雅关闭改为向**可见窗口**发 `SC_CLOSE`；等待对象收窄为「带窗口的主进程」；失败写 `~/.wb-switch/error.log` | `crates/wb-switch-core/src/modules/vscode_ext.rs`、`process.rs` |
| **权限提醒链路** | 改读插件宿主日志的**状态行**（`running → waiting_user_input` / 反向）来判断"等待确认"，弃用"确认之后才写"的 `Dangerous command detected` | 新增 `vendor/agent-companion/crates/agent-studio-core/src/log_watch.rs`、`workspace_history.rs`；`adapters/*.rs`、`hub.rs`、`runtime/main.rs`、`collector/lib/codebuddy-ide.js`、`src/monitor/permission-check.ts`、`src/desktop/rail-controller.ts` |
| **提示音** | 宿主按告警类型播**不同的系统 wav**（需要确认/完成 = 感叹音，失败 = 错误音）——Windows 上通知自带的 `sound()` 不生效，必须自己播 | `vendor/agent-companion/crates/agent-studio-desktop/src/lib.rs` |
| **VS Code 唤起** | Windows 上支持从悬浮栏唤起 VS Code（目录定位 + 唤起应用） | `crates/agent-studio-desktop/src/session.rs` |

> 每一处改动的**根因、证据、验证结果**都记在 `.codebuddy/memory/2026-09-27.md`，跟随上游更新遇到冲突时，那份文件就是"补丁说明书"。

### 2026-09-28 追加改动

| 主题 | 改动内容 | 主要文件 |
| --- | --- | --- |
| **提示音默认开启** | `notifications` 默认值从 `desktop:false / sound:false` 改为 `desktop:true / sound:true`。原来宿主只有在 `desktop=true` 时才收得到告警（`sound` 又跟着告警一起下发），而「悬浮窗设置」页面**并不暴露这两个开关** ⇒ **任何全新安装的机器都不会有提示音，也不弹桌面通知**。 | `vendor/agent-companion/crates/agent-studio-core/src/settings.rs`、`vendor/agent-companion/src/settings-config.js` |

> 只影响**没有** `~/.agent-studio/settings.json` 的机器（全新安装）。已经存在该文件的机器保持原值：
> 把 `notifications.desktop` 和 `notifications.sound` 改成 `true` 后重启 App 即可（本机已按此写好）。
> 排查过程见 `.codebuddy/memory/2026-09-28.md`。

---

## 三、日常使用

### 方式 A：开发模式（不安装，最省事）

```powershell
npm run tauri dev
```

- 改代码会自动重编译；适合调试
- 缺点：依赖终端常驻、没有"开机自启"

### 方式 B：打包成正式安装包（推荐，像个正式软件）

```powershell
git switch local

npm ci                       # 装依赖（首次必须；package.json 没变可跳过）

npm run build:desktop        # 构建 companion 嵌入包 + 应用前端 + 复制资源
                             # = prepare build → npm run build → prepare copy

npm run tauri build -- --config '{"bundle":{"createUpdaterArtifacts":false}}'
```

产物：

```
src-tauri\target\release\bundle\msi\*.msi          ← 双击安装（推荐）
src-tauri\target\release\bundle\nsis\*-setup.exe   ← 或者这个
```

只想出个绿色 exe（更快，不用下载 WiX/NSIS）：

```powershell
npm run tauri build -- --no-bundle
# 产物：src-tauri\target\release\wb-switch-rust.exe
```

> 便携 exe 必须**整个 `release` 目录一起用**（companion 的 runtime、`companion/` 资源都在它旁边），单拷 exe 会在启动悬浮栏时报错。

### 关于 `--config '{"bundle":{"createUpdaterArtifacts":false}}'`（必须加）

- `tauri.conf.json` 里 `bundle.createUpdaterArtifacts = true` 时，打包会要求**签名私钥** `~/.wb-switch/wb-switch-updater.key`；这把私钥是作者的，本机没有 ⇒ 不加这个参数**打不出包**。
- 它只影响「**你打的包要不要产出更新产物**（`.sig` / `latest.json`）」，**不会**关掉 App 的**更新检查**：更新检查由 `plugins.updater`（`pubkey` + 指向作者 `releases/latest/download/latest*.json` 的 endpoints）决定，保持原样。
- 也就是说：**你仍然会收到「有新版本」的提示** ✅，但你的包不会对外发布更新。

---

## 四、跟随上游更新（重点）

当你看到 App 提示「有新版本」时（**它就是你发现作者发版的信号**）：

1. **不要点「安装更新」** ❌ —— 那会把 App 换成作者的官方版，你的本地改动就没了
2. 回到源码目录执行：

```bash
git switch local
git fetch origin              # 只下载上游变化（只读，不需要推送权限）
git rebase origin/main        # 把上游更新垫到你的改动下面
```

3. 如果出现冲突：编辑冲突文件 → `git add -A` → `git rebase --continue`
   （不熟悉 `rebase` 的话，可用 `git merge origin/main` 代替，效果一样，只是多一个合并提交）
4. 冲突解决后 **重新打包**（见上一节），覆盖安装即可

> 只拉不推完全可行：`fetch` / `rebase` / `merge` 都是**只读**操作，不需要对作者仓库的写权限。
> 想给自己留备份（换机器、防丢），可以 fork 到自己账号后：
> `git remote add mine <你的fork地址>` → `git push mine local`（可选）。

---

## 五、已知的环境坑（都是本机实测）

| 现象 | 原因 / 处理 |
| --- | --- |
| `npm ci` 报 `[safe-delete][SAFE_DELETE_BULK_CONFIRM_REQUIRED]` | 某些 IDE 环境自带「批量删除确认」包装器。**在普通终端（PowerShell / Windows Terminal）里跑通常不会出现**；所以我这边（IDE 内）不能替你打包 |
| `vite build` 报删除 `dist/` 失败 | 同上（`emptyDir` 触发批量删除保护）。绕过办法：改用新输出目录，例如 `npx vite build --outDir dist2 --emptyOutDir false` |
| `git checkout -- <已暂存文件>` 无效 | 已暂存的文件要从 HEAD 恢复：`git restore --source=HEAD --staged --worktree <file>` |
| `vendor/agent-companion/dist-embed2/` 出现在 git 里 | 那是构建产物，已加入 `.gitignore`，不要提交；目录可手动删除 |
| 改了 `vendor/agent-companion/src/desktop/*` 但界面没变 | 悬浮栏加载的是**预构建资源包**（`companion/`），需要重新构建并覆盖（见第三节的 `npm run build:desktop`） |
| `npm ci` 后 vite 报 `Could not load index.html?html-proxy...`，或 rollup 报 `MODULE_NOT_FOUND`（`rollup/dist/native.js`） | **锁文件是作者在 macOS 上生成的**，`npm ci` 在 Windows 上会漏装平台原生包。补一条即可：`npm install --no-save @rollup/rollup-win32-x64-msvc`，然后 `node "node_modules\vite\bin\vite.js" build` |
| 隐藏的 cmd 里 `npx vite build` 报 `'vite' is not recognized` | 不要用 `npx` / `node_modules\.bin\vite.cmd`（shim 可能不存在）；直接 `node "node_modules\vite\bin\vite.js" build` |
| `linker link.exe not found` | 本机 Rust 是 `x86_64-pc-windows-msvc`，必须有 MSVC 链接器。已装 VS 2022 Build Tools 到 **`D:\VS2022`**；打包前先 `call "D:\VS2022\VC\Auxiliary\Build\vcvars64.bat"`（或把其 bin 加进 PATH） |
| `tauri build` 的 beforeBuildCommand 里跑 `build:desktop`，根 vite 报 `[vite:html-inline-proxy] ... No matching HTML proxy module found` | 只在「一条 cmd/npm 链里连着跑」时出现；**把 3 步分开跑就正常**：`node scripts/prepare-agent-companion.mjs build` → `npm run build` → `node scripts/prepare-agent-companion.mjs copy`，然后 `tauri build` 用一个把 `build.beforeBuildCommand` 改掉的 `--config` 文件（避免它重复跑前端）。注意 `copy` 不能漏，否则打出的包没有悬浮栏资源 |
| 打包时 `Downloading https://github.com/wixtoolset/...` 失败 `timeout: global` | msi 需要 WiX，从 GitHub 下不动。**只出 NSIS 安装包**：配置里 `"bundle":{"targets":["nsis"]}`；本机 NSIS 已有缓存（`%LOCALAPPDATA%\tauri\NSIS`），可直接用 |
| 打包进程莫名消失（日志停在编译中途） | 别用 `start` 起打包；用独立进程（PowerShell `Start-Process -WindowStyle Hidden`）跑，避免 shell 会话回收把子进程带走 |

---

## 六、提示音改在哪（想换音色时）

`vendor/agent-companion/crates/agent-studio-desktop/src/lib.rs` 的 `sound_alias()`：

| 场景 | 当前文件 |
| --- | --- |
| 需要确认（`wait` / `prolonged`） | `C:\Windows\Media\Windows Exclamation.wav` |
| 任务完成 | `C:\Windows\Media\Windows Exclamation.wav` |
| 任务失败（`error`） | `C:\Windows\Media\Windows Error.wav` |

可选的其他音源（都在 `C:\Windows\Media`）：`Windows Notify System Generic.wav`、`Windows Foreground.wav`、`Windows Message Nudge.wav`、`chimes.wav`、`Windows Notify Messaging.wav`。改完由 `tauri dev` / 重新打包生效。
