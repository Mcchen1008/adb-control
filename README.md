# adb-control

通过 SSH + ADB 远程控制 Android 设备（电视 / 盒子 / 手机）的脚本与文档集合，以 AstrBot 技能（`SKILL.md`）的形式组织。

支持截图、触控、遥控按键、应用启动、UI 布局分析，以及**中文输入**（基于 [YADB](https://github.com/ysbing/YADB) 的 `app_process` 文本注入，无需安装 APK、无需切换输入法）。

---

## ✨ 功能特性

| 功能 | 说明 |
|------|------|
| 📸 截图 | `screencap` 截图 + scp 回传；应用阻止截图时可改用 YADB 强制截图 |
| ⌨️ 中文输入 | YADB `-keyboard` 注入任意 Unicode 文本，绕过 `adb input text` 不支持中文的限制 |
| 📋 剪贴板 | 读写设备剪贴板（YADB `-readClipboard` / `-writeClipboard`） |
| 🎮 遥控 | 主页 / 返回 / 方向键 / 确认 / 回车 / 音量等 keyevent，点击与长按 |
| 📱 应用管理 | 按包名启动应用、通过 URL 唤起浏览器、列出已安装包 |
| 🔍 布局分析 | uiautomator 失效时可用 YADB `-layout` 抓取 UI 层级与坐标 |
| 🧩 零 APK | 唯一需要部署的组件是 14KB 的 `yadb` 二进制（放 `/sdcard` 即可） |

---

## 📦 目录结构

```
adb-control/
├── SKILL.md          # AstrBot 技能主文件（操作手册）
├── install.md        # 环境初始化指南（安装 adb / 连接 adbd / 部署 YADB）
├── yadb              # YADB 二进制（v1.1.2，来自 ysbing/YADB，LGPL-3.0）
├── LICENSE           # 本仓库自身许可证（MIT）
└── LICENSE.yadb      # YADB 的 LGPL-3.0 许可证副本（第三方组件）
```

---

## 🚀 快速开始

### 1. 环境要求（设备端）

- Android 设备已安装 [Termux](https://github.com/termux/termux-app)，且已运行 `sshd`
- Termux 内已安装 `android-tools`（提供 adb），设备 adbd 可本地连接
- 设备可被 SSH 访问（记录 IP、端口、用户名、密码）

### 2. 部署 YADB（一次性）

```bash
# 从本仓库上传 yadb 到设备
scp -P $SSH_PORT ./yadb "$SSH_USER@$SSH_HOST:/sdcard/yadb"
```

详细初始化步骤见 [install.md](install.md)。

### 3. 使用

```bash
SSH_HOST="<设备 IP>"     # 自行填写
SSH_PORT=22
SSH_USER="<Termux 用户>"
SSH_PASS="<sshd 密码>"

# 示例：中文输入（聚焦输入框后执行）
sshpass -p "$SSH_PASS" ssh -o StrictHostKeyChecking=no -p "$SSH_PORT" "$SSH_USER"@"$SSH_HOST" \
  'export PATH=/data/data/com.termux/files/usr/bin:$PATH; adb -s emulator-5554 shell "app_process -Djava.class.path=/sdcard/yadb /data/local/tmp com.ysbing.yadb.Main -keyboard 你好世界"'

# 示例：截图回传
sshpass -p "$SSH_PASS" ssh -o StrictHostKeyChecking=no -p "$SSH_PORT" "$SSH_USER"@"$SSH_HOST" \
  'export PATH=/data/data/com.termux/files/usr/bin:$PATH; adb -s emulator-5554 shell "screencap -p /sdcard/screen.png"'
sshpass -p "$SSH_PASS" scp -o StrictHostKeyChecking=no -P "$SSH_PORT" "$SSH_USER"@"$SSH_HOST":/sdcard/screen.png ./screen.png
```

完整命令手册见 [SKILL.md](SKILL.md)。

---

## 🧪 测试说明

- ✅ 本仓库**仅在 AstrBot 环境中测试通过**（测试设备：创维 Skyworth 7T861_A23 / Android 10 / Termux + android-tools）
- ⚠️ **未在其他工具中测试**（如 Claude Desktop、Cline 等其他 Agent 框架或手动环境）
- 📌 核心机制（SSH + Termux adb + YADB app_process 注入）与具体框架无关，理论上可迁移到任何能执行 shell 命令的环境，但迁移后请自行验证

---

## 🧩 致谢与第三方组件：YADB

本仓库的**中文输入能力完全依赖 [YADB](https://github.com/ysbing/YADB)（作者 ysbing）**。

- 📦 仓库内 `yadb` 二进制来自 YADB 官方 Release（v1.1.2），**未做任何修改**
- 📜 YADB 采用 **GNU Lesser General Public License v3.0 (LGPL-3.0)**，许可证全文见 [LICENSE.yadb](LICENSE.yadb)
- 🔗 原始项目：<https://github.com/ysbing/YADB>
- 根据 LGPL-3.0 要求：分发本仓库时保留 YADB 的版权与许可声明，并注明其来源；如需修改 YADB 源码，修改版须继续以 LGPL-3.0 发布

核心用法（输入中文）：

```bash
adb -s emulator-5554 shell "app_process -Djava.class.path=/sdcard/yadb /data/local/tmp com.ysbing.yadb.Main -keyboard 任意中文"
```

---

## 📄 许可证

- **本仓库原创内容**（SKILL.md、install.md、README 等文档与脚本）：[MIT License](LICENSE)
- **第三方组件 yadb**：LGPL-3.0（见 [LICENSE.yadb](LICENSE.yadb)），是独立分发的未修改二进制，与本仓库 MIT 授权内容互不影响

---

## ⚠️ 免责声明

- 本工具仅用于**控制你拥有或有权操作的设备**
- 远程控制、屏幕录制、文本注入等功能请在**遵守当地法律法规**的前提下使用
- 请勿将本工具用于未经授权的设备监控、数据窃取或任何非法用途
- 使用本工具造成的任何后果由使用者自行承担
