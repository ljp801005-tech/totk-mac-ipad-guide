# 《塞尔达传说：王国之泪》在 Mac 与 iPad 上原生运行 · 完整实践教程

> 基于 Apple 芯片原生运行方案（MacNativeGames），实测环境：Mac mini M4 / 16GB / macOS 27.2 + iPad mini 7（A17 Pro）。
> 全程使用**自己导出的游戏文件**，不需要 Switch 实机在旁边。

---

## 目录

- [0. 这个方案是什么](#0-这个方案是什么)
- [1. 先决条件](#1-先决条件)
- [2. Mac 端：环境准备](#2-mac-端环境准备)
- [3. Mac 端：打包 App](#3-mac-端打包-app)
- [4. iPad 端：前置配置](#4-ipad-端前置配置)
- [5. iPad 端：打包 IPA](#5-ipad-端打包-ipa)
- [6. iPad 端：签名与安装](#6-ipad-端签名与安装)
- [7. iPad 端：首次启动](#7-ipad-端首次启动)
- [8. 7 天续签](#8-7-天续签)
- [9. 空间占用与清理](#9-空间占用与清理)
- [10. 疑难排查](#10-疑难排查)
- [11. 命令速查](#11-命令速查)

---

## 0. 这个方案是什么

不是传统意义上的模拟器（如 Ryujinx / Yuzu 系）。它把 Switch 游戏里的 ARM64 机器码**改写成能在 Apple 芯片上直接执行的代码**，并把 Switch 的图形接口（NVN）重写成 Metal。

实际差别：

| | 原生运行方案 | 传统模拟器 |
|---|---|---|
| 帧率（M4 / 1080p） | 84–105 FPS | 通常 30–60 FPS |
| 首次准备 | 打包 20–40 分钟 | 装上就能跑 |
| 通用性 | 只支持已适配的游戏 | 支持大量游戏 |

**本文只讲《王国之泪》这一款**（项目里适配最完整的一款，本体 1.0.0 + 更新 1.1.2 / 1.2.1 / 1.4.3，推荐 1.4.3）。

---

## 1. 先决条件

### 硬件与系统

| 项目 | 要求 | 说明 |
|---|---|---|
| 芯片 | Apple silicon（M1 及以上） | Intel Mac 不支持 |
| 内存 | 建议 16GB | 8GB 会明显吃力 |
| macOS | 26 或更高 | 需要 Metal 4 |
| 空闲磁盘（Mac） | 60GB 以上 | 工作目录 + 产物 |
| iPad（可选） | iPadOS 26+，剩余空间 20–25GB | A16 及以上实测可用 |

### 软件

| 软件 | 用途 | 安装方式 |
|---|---|---|
| Xcode 命令行工具 | 编译 | `xcode-select --install` |
| Homebrew | 依赖管理 | [brew.sh](https://brew.sh) |
| opus、libvpx | 音频/视频解码 | `brew install opus libvpx` |
| python@3.14 | 构建脚本运行环境 | `brew install python@3.14` |
| Xcode 完整版（仅 iPad） | 签名与设备部署 | App Store，10GB+ |
| xcodegen、cmake、ninja（仅 iPad） | iOS 工程生成与构建 | `brew install xcodegen cmake ninja` |

> ⚠️ 构建脚本把 Homebrew 路径硬编码为 `/opt/homebrew`，Apple 芯片默认就是这里；如果你装在别处，需要改脚本。

### 必须具备的文件（本项目不提供）

| 文件 | 说明 |
|---|---|
| 游戏本体 | **必须是 XCI 格式**（原因见 [坑 1](#坑-1nsp-格式缺系统字体会卡住)），建议 1.0.0 版 |
| 更新包 | NSP，1.4.3 最稳 |
| `prod.keys` | 用 Lockpick_RCM 从你自己的主机导出 |

合法性边界：**自己 dump 自己购买的游戏没问题，从网上下载的不行。** 本教程不提供、也不指向任何游戏文件或密钥来源。

---

## 2. Mac 端：环境准备

```sh
# 1) 命令行工具
xcode-select --install

# 2) Homebrew（若未安装，需输入登录密码）
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 3) 依赖
brew install opus libvpx python@3.14
```

验证：

```sh
/opt/homebrew/bin/python3.14 --version    # 应为 3.14.x
ls -d /opt/homebrew/opt/opus /opt/homebrew/opt/libvpx
```

拉仓库：

```sh
git clone --depth 1 https://github.com/ly05010419/MacNativeGames.git
cd MacNativeGames
```

仓库约 258MB。作者还提供了一个面向 AI 助手的安装 skill（`.claude/skills/install-zelda/`），可以让助手按步骤自动执行。

---

## 3. Mac 端：打包 App

### 3.1 先做体检

```sh
/opt/homebrew/bin/python3.14 -m tools.builder --cli --game totk \
  --game-file "/path/to/本体.xci" \
  --keys "/path/to/prod.keys" \
  --update "/path/to/更新-1.4.3.nsp" \
  --check-only
```

全部通过会打印 `Checks: ready.`；有任何 `BLOCK` 就先解决再往下走（省得跑 20 分钟后才发现）。

### 3.2 正式打包

```sh
mkdir -p ~/Applications
/opt/homebrew/bin/python3.14 build_app.py "/path/to/本体.xci" \
  -k "/path/to/prod.keys" \
  --update "/path/to/更新-1.4.3.nsp" \
  --work "$HOME/totk-build" \
  -o "$HOME/Applications/Tears of the Kingdom.app"
```

四个阶段与耗时（M4 实测，总计约 32 分钟）：

| 阶段 | 内容 | 耗时 |
|---|---|---|
| 1/4 code | 改写 sdk / subsdk0 / main | ~1 分钟 |
| 2/4 data | 解包校验 17GB RomFS、合并更新、提取系统字体 | ~3 分钟 |
| 3/4 shaders | 准备并为本机 GPU 编译着色器（本例 27929 个） | 6–12 分钟 |
| 4/4 app | 组装 App、macOS 校验 | 2–3 分钟 |

> **`--work` 必须显式指定**，且**不要指向 `~/Downloads`**（见 [坑 2](#坑-2downloads-目录写不进去)）。

### 3.3 运行

双击 App 即可。首次启动要 1–2 分钟准备着色器，之后约 12 秒。

- `1` 键：设置面板（图像 / 补帧 / 按键）
- `ESC`：退出全屏
- 手柄即插即用

**不要移动 App 的位置**：着色器缓存绑定路径，挪走后下次启动要重新编译。

---

## 4. iPad 端：前置配置

按顺序做完这四步，缺一步后面都会失败：

### 4.1 安装完整版 Xcode

App Store 搜 Xcode，10GB+。装完执行一次：

```sh
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
xcodebuild -downloadPlatform iOS
brew install xcodegen cmake ninja
```

### 4.2 登录 Apple ID（在 Mac 上，不是 iPad）

Xcode → 菜单栏 **Xcode → Settings… → Accounts → 左下角 + → Apple ID**，登录后能看到 **Personal Team**。

> 免费账号签名 **7 天有效**，到期重跑安装脚本续签，存档保留。

### 4.3 连接 iPad 并信任

数据线连 Mac，iPad 解锁，弹出「信任此电脑」点信任并输入密码。

```sh
xcrun devicectl list devices     # 应看到你的设备，State 为 connected
```

### 4.4 打开开发者模式

iPad 设置 → 隐私与安全性 → **拉到最底部** → 开发者模式 → 打开 → 重启 → 用密码确认。

> 如果找不到这个开关：它是**隐藏的**，要先连过一次 Xcode（或尝试安装一次开发签名的 App）才会出现。用设置首页搜索框直接搜「开发者」更快。

---

## 5. iPad 端：打包 IPA

iPad **必须用数据线连着**——打包时要为它的 GPU 预编译管线。

```sh
cd MacNativeGames
/opt/homebrew/bin/python3.14 build_app.py "/path/to/本体.xci" \
  --update "/path/to/更新-1.4.3.nsp" \
  -k "/path/to/prod.keys" \
  --work "$HOME/totk-build" \
  --platform ios --ios-game-data documents \
  -o "$HOME/Tears of the Kingdom.ipa"
```

实测：约 50 分钟，产物 IPA 仅 **200MB**（不含游戏数据）。

日志里关键两行：

```
pipelines: <你的 iPad> on the cable: applegpu_g16p    ← 认出设备并选对 GPU
979 个管线变体已为设备编译
```

> `--work` 指向已有工作目录时，前两个阶段（代码改写、数据解包）会复用，不重跑。

---

## 6. iPad 端：签名与安装

```sh
/opt/homebrew/bin/python3.14 games/totk/ios/tools/install_device.py \
  "$HOME/Tears of the Kingdom.ipa" \
  --game-data "$HOME/totk-build/totk/extracted/v143/romfs/romfs.bin"
```

多台设备时加 `--device "设备名"`。这一步做三件事：

1. 生成临时 Xcode 工程，让 Xcode 申请描述文件并签名
2. 把 200MB 的 App 装进 iPad
3. 把 **16GB 游戏数据分 16 块**（每块 1GiB，单文件大会断连）+ **2.1GB 着色器包**（约 13 万个文件）拷进 App 的 Documents

耗时取决于线材：10–30 分钟。**中途别锁屏、别拔线。**

成功标志：

```
installed. First time on this device: Settings > General > VPN & Device Management > trust your Apple ID.
```

> 这一步是整个流程最容易卡住的地方，两个高频坑见 [坑 4](#坑-4证书有效但-xcodebuild-报-no-accounts) 和 [坑 5](#坑-5证书存在但显示无效)。

---

## 7. iPad 端：首次启动

1. iPad 设置 → 通用 → VPN与设备管理 → 信任你的 Apple ID 证书
2. 点开 **Tears of the Kingdom**
3. 首次启动约 3 分钟（校验数据 + 导入着色器），之后约 8 秒

已知限制：

- 渲染为**手持模式 960×540 放大到 720p**，最高 60 帧
- **不能边玩边编译着色器**：没覆盖到的场景可能缺贴图，需连回 Mac 重新跑安装脚本补
- 被动散热，长时间高负载会降频

---

## 8. 7 天续签

免费 Apple ID 签名 7 天后失效（App 打不开）。续签只需重跑一次安装命令，**不用重新打包**：

```sh
/opt/homebrew/bin/python3.14 games/totk/ios/tools/install_device.py \
  "$HOME/Tears of the Kingdom.ipa" \
  --game-data "$HOME/totk-build/totk/extracted/v143/romfs/romfs.bin"
```

脚本会检查设备上已有的内容：

- 游戏数据已存在（`romfs.sha256` 一致）→ **跳过 16GB 拷贝**
- 着色器已存在（`shader-seed.txt` 一致）→ **跳过 2.1GB 拷贝**

所以续签通常 3–5 分钟。**存档不受影响**（存在 App 的 Documents 里）。

---

## 9. 空间占用与清理

| 位置 | 占用 |
|---|---|
| `~/Applications/Tears of the Kingdom.app` | 19GB |
| 工作目录（`--work` 指定处） | 约 61GB（含解包数据、管线、中间件） |
| IPA | 200MB |
| 仓库 | 258MB |
| iPad | 约 20GB |

### 清理建议

**能立刻删的**（都是可再生产物，删了只影响重新打包的时间）：

```
<work>/totk/build/{data-bundles,ios-pipelines,mac-pipelines,shader-seed,ios-derived,ios-payload,ios-deps}
<work>/totk/ios
<work>/totk/extracted/v0                 # 1.0.0 基础版解包，已被合并版取代
<ipa 同目录>/Tears of the Kingdom-shader-seed
<ipa 同目录>/Tears of the Kingdom.signed
~/Library/Caches/MacNativeGames/ios-signing
```

**必须保留的**：

- `Tears of the Kingdom.ipa`（续签要用）
- `<work>/totk/extracted/v143/romfs/romfs.bin` + `index.json`（重装游戏数据要用）
- 已装好的 `.app`

按上面清单清理，实测从 64GB 降到 17.5GB，续签功能完全不受影响。

---

## 10. 疑难排查

### 坑 1：NSP 格式缺系统字体，会卡住

**现象**：`--check-only` 报缺 shared fonts。

**原因**：游戏需要 6 个系统字体（`.bfttf`），它们来自主机的系统固件。**XCI 卡带镜像自带一个系统更新分区**，字体在里面；**NSP 数字版没有这个分区**，游戏 romfs 里也不会有。

**解决**（三选一）：

1. 换 XCI 格式的本体（最省事，脚本会自动优先选 XCI）
2. 提供固件文件夹（装满 `.nca`，SystemData 标题 `0100000000000810`~`0814`），用 `--firmware` 指过去
3. 用已提取的字体目录 `--shared-fonts`

### 坑 2：Downloads 目录写不进去

**现象**：打包刚开始就失败，日志提示无法创建工作目录。

**原因**：macOS 对 `~/Downloads`、`~/Desktop`、`~/Documents` 有隐私保护（TCC），未授权的进程无法写入。

**解决**：把 `--work` 指到别的位置，例如 `$HOME/totk-build` 或任意项目目录。同一磁盘内游戏数据会以克隆方式占用，不会额外翻倍。

### 坑 3：批量删除被限制（特定环境）

**现象**：着色器阶段报删除失败退出。

**原因**：若通过受限的执行环境（如 AI 助手沙箱、某些 MDM 策略）运行，一次批量删除超过一定数量文件会被拦。脚本要清理 200+ 个临时文件就会失败。

**解决**：在**自己的终端**里跑，不受此限制。已完成的阶段会复用，不会重跑。

### 坑 4：证书有效，但 xcodebuild 报 No Accounts

**现象**：

```
error: No Accounts: Add a new account in Accounts settings.
error: No profiles for 'xxx' were found
```

iPad 上**看不到游戏图标**。

**关键判断**：这不是证书问题。检查钥匙串里有没有账号会话凭据：

```sh
security find-generic-password -s "Xcode-Token" 2>&1
```

返回 `could not be found` 就说明 **Xcode 里 Apple ID 没有真正登录成功**（证书可能是之前登录过留下的，或者只创建了证书但账号没保留）。

**解决**：Xcode → Settings → Accounts，确认账号在列表里且下方显示 **Personal Team**；没有就删掉重新登录（需要双重验证码）。

### 坑 5：证书存在但显示无效

**现象**：

```
1) "Apple Development: xxx@xxx.com (XXXXXXXXXX)"
Matching identities: 1     Valid identities: 0
```

**原因**：证书链断了。开发证书由 Apple WWDR 中间证书签发，而系统里可能只有**已过期**的 WWDR G1（2023 年 2 月过期），缺 G3。

**诊断**：

```sh
security find-certificate -a -c "Worldwide Developer Relations" -p 2>/dev/null \
  | openssl x509 -noout -subject -enddate
```

**解决**：装 Apple 官方的 WWDR G3：

```sh
curl -fsSL -o /tmp/AppleWWDRCAG3.cer https://www.apple.com/certificateauthority/AppleWWDRCAG3.cer
security add-certificates -k ~/Library/Keychains/login.keychain-db /tmp/AppleWWDRCAG3.cer
security find-identity -v -p codesigning    # 应显示 1 valid identities found
```

### 坑 6：iPad 上找不到「开发者模式」

这是正常的：该开关默认隐藏，**连接过一次 Xcode 或尝试过一次开发签名安装后才会出现**。用设置首页搜索「开发者」直接跳转。

### 坑 7：装好了但点开闪退

iPad 设置 → 通用 → VPN与设备管理 → 信任你的 Apple ID。仍未解决则检查签名是否已过期（7 天）。

### 坑 8：游戏内频繁卡顿

新区域首次进入会编译着色器，属正常。第二次经过同一处就顺了。玩得越多缓存越全。

---

## 11. 命令速查

```sh
# ── Mac 端 ────────────────────────────────────────────────
# 体检
python3.14 -m tools.builder --cli --game totk \
  --game-file base.xci --keys prod.keys --update upd.nsp --check-only

# 打包 Mac App
python3.14 build_app.py base.xci -k prod.keys --update upd.nsp \
  --work ~/totk-build -o ~/Applications/"Tears of the Kingdom.app"

# 运行
open ~/Applications/"Tears of the Kingdom.app"

# ── iPad 端 ──────────────────────────────────────────────
# 确认设备
xcrun devicectl list devices

# 打包 IPA（iPad 需连数据线）
python3.14 build_app.py base.xci -k prod.keys --update upd.nsp \
  --work ~/totk-build --platform ios --ios-game-data documents \
  -o ~/"Tears of the Kingdom.ipa"

# 签名 + 安装 + 拷数据（首次 10–30 分钟，续签 3–5 分钟）
python3.14 games/totk/ios/tools/install_device.py ~/"Tears of the Kingdom.ipa" \
  --game-data ~/totk-build/totk/extracted/v143/romfs/romfs.bin

# ── 诊断 ─────────────────────────────────────────────────
security find-identity -v -p codesigning        # 签名身份是否有效
security find-generic-password -s "Xcode-Token" # Xcode 账号会话是否存在
xcrun devicectl list devices                    # iPad 是否连上
tail -20 ~/Library/Logs/MacNativeGames-build.log
```

---

## 参考

- MacNativeGames：<https://github.com/ly05010419/MacNativeGames>
- 同类项目 nxnative：<https://github.com/seedds/nxnative>
- Apple 证书颁发机构：<https://www.apple.com/certificateauthority/>

---

## 免责声明

本教程仅记录软件安装与配置过程，不包含、不提供任何游戏本体、更新包、密钥或固件。运行游戏所需的全部文件须由使用者从自己合法拥有的主机与游戏中导出。请遵守所在地法律及游戏软件许可协议。
