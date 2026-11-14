# 云端一次性编译 Aspia 3.3.5 定制客户端

**本机零安装**：编译在 GitHub Actions 的 Windows runner 上跑，机器用完即销毁。改功能、打补丁、出 `.exe` 全在云端，你只需要一个 GitHub 账号。

```
cloud-build\
  .github\workflows\build-client.yml              # 云端流水线（已修好，见下）
  patches\default_credentials.patch              # 默认凭据自动登录
  patches\ui_tabs_hide_management.patch          # 第 1 批 UI：列表 tab 显隐 + tab 固定显示 IP
  .gitattributes                                 # 强制 patch / yml 用 LF 签出
  一步一步操作指南.md                            # 从建仓库到拿 exe 的完整命令
```

> **第一步请看 `一步一步操作指南.md`**。本目录已初始化成 git 仓库（`main` 分支 + 首笔提交），你只需要建远端仓库、关联、push、点按钮。

上级目录的 **`UI定制方案-8项需求.md`** 是本次要改的 8 项界面功能的逐条源码级方案；那 8 项后续也是做成 patch 放进 `patches\` 编译。`ui_tabs_hide_management.patch` 是其中最容易的第 1 批（需求 2 / 6 / 8），可以拿它先验证整条云端链路。

---

## 1. 已修好的三处「必失败」问题

| # | 问题 | 修法 |
|---|---|---|
| 1 | `.gitmodules` 里 vcpkg4aspia 是 **SSH** 地址 `git@github.com:dchapyshev/vcpkg4aspia.git`，托管 runner 无 SSH key；原脚本先 `git config submodule.*.url https://...` 再 `git submodule sync`，**sync 会把 HTTPS 覆盖回 SSH** | 在任何 git 操作前设 `git config --global 'url.https://github.com/.insteadOf' 'git@github.com:'`，在 URL 解析层重写，sync 也覆盖不掉 |
| 2 | 原 `checkout` 用 `repository: ${{ github.repository }}` + `ref: v3.3.5`，如果你建的是空仓库/没同步 tag，直接「找不到 ref」失败 | 固定 `repository: dchapyshev/aspia` + `ref: v3.3.5`，与你的仓库状态完全无关 |
| 3 | Windows runner 的 Git 默认 `core.autocrlf=true`，源码被签出成 CRLF，而 patch 是 LF → `git apply` 报 "patch does not apply" | 全局设 `core.autocrlf false`，checkout 后再 `git checkout-index -a -f` 用 index 里的 LF 重建工作树 |

另外补了两点提高成功率：

- **缓存立即保存**：原来靠 `actions/cache` 的 post-step，job 被 6 小时超时杀掉时它根本不执行，所以"重跑续编"是空的。现在 configure 之后用 `actions/cache/save` 显式保存，key 每次唯一 + 前缀 `restore-keys` 形成滚动缓存，失败/中止的运行也能贡献已编好的包。
- **configure 后删 `vcpkg4aspia\buildtrees` / `packages`**：Qt 的编译中间产物能吃掉几十 GB，而装好的头文件/库在 `builds/local-win-x64/vcpkg_installed`，删掉不影响后续 build。托管 runner 只有 **14 GB SSD**，这一步很关键。

### 第二批修复（本次，都属于「不修就白跑」）

| # | 问题 | 修法 |
|---|---|---|
| 4 | `Run workflow` 按钮**只在默认分支**上出现；原文档让 push 到 `custom`，而新建空仓库的默认分支是 `main` → Actions 页根本找不到按钮 | 本地仓库直接用 `main`；删掉 `push` 触发器，改成纯 `workflow_dispatch` |
| 5 | 新仓库 `GITHUB_TOKEN` 默认只读，`actions/cache/save@v4` 需要 `actions: write` → 缓存保存失败，每次重跑都得从头编 Qt | workflow 顶层显式 `permissions: contents: read, actions: write` |
| 6 | helper 仓库的 checkout 发生在设 `core.autocrlf=false` **之前**，Windows runner 会把 `.patch` 签出成 CRLF，`git apply` 解析不了 | 加 `.gitattributes` 强制 `*.patch text eol=lf`，apply 前再做一次 CRLF→LF 规范化（UTF-8 无 BOM） |

---

## 2. 使用步骤（照着点）

> **完整命令逐条见 `一步一步操作指南.md`。** 下面只是骨架。
>
> 若本机 `github.com` 主站打不开（**当前实测就是如此**：主站 8/8 超时，`api.github.com` 3/3 正常），直接走上级目录 `cloud-api.py` 的 REST API 方案，见操作指南「备选路径 C」——不用网页、不用 `git push`。

1. 在 GitHub 上 **新建一个完全空的仓库**（推荐 Public，Actions 免费；Private 每月 2000 分钟，一次 6h 就吃掉 360 分钟）。名字随意，比如 `aspia-build`。**不要**勾选 Add README / .gitignore / license。
2. 本目录已是 git 仓库（`main` 分支 + 首笔提交），关联并推送：
   ```
   cd /d/softa/aspia-3.3.5/cloud-build
   git remote add origin https://github.com/<你的用户名>/aspia-build.git
   git push -u origin main
   ```
   分支必须是 **`main`（默认分支）**：`Run workflow` 按钮只在默认分支上出现。也可用网页端 *Add file → Create new file* 逐个粘 3 个文件（workflow + 2 个 patch），但长补丁容易粘坏。
3. 打开仓库的 **Actions** 页 → 左侧选 `build custom Aspia client (v3.3.5)` → 右侧 **Run workflow**。`patch_file` 输入框支持**逗号分隔多个补丁**，按填写顺序依次应用：
   - `none` —— 编**原版 3.3.5** 做基线测试，强烈建议第一次先跑这个验证 Qt 静态编译能过；
   - `default_credentials.patch` —— 默认凭据自动登录（默认值）；
   - `ui_tabs_hide_management.patch` —— 第 1 批 UI 改动；
   - `default_credentials.patch,ui_tabs_hide_management.patch` —— **两个都要**（已验证可叠加）；
   - 你自己改的 UI：把 patch 放进 `patches\` 再填文件名即可。
4. 等它跑完（首次几小时，见第 4 节）。页面底部 **Artifacts** 下载 `aspia-client-3.3.5-custom`，里面是 `aspia_client.exe`。
5. exe 是静态链接单文件，单独拷到任意可写目录即可双击。若用了默认凭据补丁：首次手工输一次账号密码，同目录生成 `aspia_default_login.ini`，之后免输；`ACCESS_DENIED` 时会自动弹框重输并存新的。

> 只有**手动触发**，push 不会自动开跑，避免误启动一次 6 小时构建。改了 patch 想重编，就再点一次 Run workflow。

---

## 3. 换成你自己的功能改动

两条路，任选：

- **推荐（改动可控）**：把改动做成 `git diff` patch 放进 `patches\`，`patch_file` 填新文件名。改完一版 push 一次、跑一次，构建日志就是编译验证。多个 patch 可以叠，但注意应用顺序。
- **直接改源码**：fork 官方仓库 → 基于 `v3.3.5` 建 `custom` 分支 → 直接提交改动的 `.cc/.h/.ui` → 把 workflow 里 `Checkout Aspia v3.3.5` 那步的 `repository:` 改成你的 fork、`ref:` 改成 `custom`，并给 `patch_file` 填 `none`。

`UI定制方案-8项需求.md` 第 3 节建议按 3 批出 patch，每批单独编一次，报错好定位。第一次构建日志贴给我，我按报错改 patch。

---

## 4. 现实风险（务必先看）

**能编，但不保证第一次就过。** 瓶颈不在工具链，而在 **Qt 6 必须静态全编译**：

- 顶层 `CMakeLists.txt:79-82` 无条件 `find_package(Qt6DeviceDiscoverySupportPrivate / Qt6FbSupportPrivate CONFIG REQUIRED)`，WIN32 还要 `Qt6EntryPointPrivate`。这些是 Qt **内部私有模块**，官方预编译 Qt 安装包里没有，所以不能用 `install-qt-action` 装个二进制 Qt 绕过，只能让 vcpkg 用 `x64-windows-static` 从源码编 `qtbase + qtsvg + qttools` 和另外 20 个依赖。
- GitHub 托管 Windows runner：**4 核 / 16 GB RAM / 14 GB SSD**，job 硬上限 **6 小时**（`timeout-minutes: 360`）。
- 旁证：官方自己的 `.github/workflows/windows.yml` 用 `runs-on: win11`（**自托管 runner**），不是托管机。我们只编一个 `x64 Release aspia_client` target，负载比官方矩阵（x86/x64 × debug/release + 全套测试 + MSI 打包）小得多，但官方的选择说明这套构建对托管机并不轻松。

**按概率排的排查顺序**：

| 现象 | 处理 |
|---|---|
| 超时 / 卡在 vcpkg 编 Qt | **直接重跑一次**，滚动缓存会命中已编好的包；通常第二、三次能续过 |
| `No space left on device` | 已有「configure 后删 buildtrees」步骤；仍不够就在 workflow 开头加一步删 runner 预装大件（`C:\Program Files (x86)\Android`、`C:\hostedtoolcache\windows\Java_*` 等，别删 VS/SDK），或改用付费 larger runner |
| `git apply --check failed` | 多半是补丁基于别的版本；确认 patch 是针对 `v3.3.5` tag 生成的（现有 `default_credentials.patch` 已在 v3.3.5 tag 上 `git apply --check` 通过，5 个文件全部干净） |
| 编译报错（Qt 头文件 / 成员名） | 把日志发我改 patch，这是云端迭代的正常环节 |
| submodule 拉取失败 | 确认没有把 `Rewrite GitHub SSH URLs to HTTPS` 那步删掉 |

---

## 5. 为什么不用 WSL

WSL 里能编，但产物是 **Linux 版**（ELF + Qt for Linux），不是 `.exe`：

- 官方 `local-win-*` 预设绑定 `x64-windows-static`（MSVC 专用 triplet）、`CMAKE_MSVC_RUNTIME_LIBRARY`，代码里有 `qt_windows.h` / `wincrypt.h`，顶层 WIN32 分支还 `find_package(WDK REQUIRED)`；
- WSL 只能用 `local-linux-x64`，出来靠 WSLg 跑 GUI，剪贴板 / 输入法 / 全屏体验很差；
- MinGW 交叉编译官方不支持，Qt 6 静态 + Win32/WDK 头这套走不通。

「随装随删」的思路是对的，只是载体应该是云端的 Windows runner。

---

## 6. 如果根本不需要改功能

不用编译，直接：

1. 官方 MSI 已经下好并校验过：`D:\softa\aspia-3.3.5\aspia-client-3.3.5-x86_64.msi`；
2. 绿色单文件也已解好：`D:\softa\aspia-3.3.5\portable\aspia_client.exe`（`3.3.5.8622`，静态链接、无 Qt/MSVC 运行库依赖）；
3. 用它自带的 *Credentials + 地址簿 Use saved credentials* 实现免输，详见 `README-中文说明.md` 第 2 节。

> 注意：3.3.5 客户端连不上 2.6.5 主机。你的 6 台若仍是 2.6.5，要么保持用 `D:\softa\aspia-3.0.37\portable\aspia_client.exe`（通吃 2.x/3.x），要么逐台升级 Host。详见 `D:\softa\aspia-3.0.37\版本兼容说明.md`。
