# ADB 环境初始化

在设备上完成一次环境准备：安装 adb、连接 adbd、部署 YADB。

## 参数

- SSH 连接参数：同 `SKILL.md`（`SSH_HOST` / `SSH_PORT` / `SSH_USER` / `SSH_PASS`）
- 技能目录：本文件所在目录（含 `yadb` 二进制），实际路径 `/AstrBot/data/skills/adb-control`
- 设备端 YADB 路径：`/sdcard/yadb`（固定，勿改）

## 1. 环境检查

通过 SSH 执行（把 `COMMANDS` 换为下方检查脚本）：

```bash
sshpass -p "$SSH_PASS" ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=8 -p "$SSH_PORT" "$SSH_USER"@"$SSH_HOST" \
  'export PATH=/data/data/com.termux/files/usr/bin:$PATH; command -v adb || echo ADB_MISSING; adb devices | grep emulator-5554 || echo ADBD_NOT_CONNECTED; ls /sdcard/yadb 2>/dev/null || echo YADB_MISSING'
```

根据输出决定后续步骤：

| 输出 | 处理 |
|------|------|
| `ADB_MISSING` | 执行第 2 步 |
| `ADBD_NOT_CONNECTED` | 执行第 3 步 |
| `YADB_MISSING` | 执行第 4 步 |

## 2. 安装 adb（Termux 内）

```bash
pkg update -y && pkg install android-tools -y
```

## 3. 连接 adbd

```bash
adb connect 127.0.0.1:5555
```

失败则重启 adb 服务再连：

```bash
adb kill-server && adb start-server && adb connect 127.0.0.1:5555
```

## 4. 部署 YADB

在技能目录 `/AstrBot/data/skills/adb-control` 下执行（`./yadb` 是技能目录内附带的二进制）：

```bash
sshpass -p "$SSH_PASS" scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -P "$SSH_PORT" ./yadb "$SSH_USER"@"$SSH_HOST":/sdcard/yadb
adb -s emulator-5554 shell "chmod 644 /sdcard/yadb"
```

## 5. 验证部署

```bash
adb -s emulator-5554 shell "app_process -Djava.class.path=/sdcard/yadb /data/local/tmp com.ysbing.yadb.Main -readClipboard"
```

- 返回剪贴板内容或空白行 → 部署成功 ✅
- 返回 `Invalid argument` / `Aborted` → 部署失败，检查 YADB 文件是否完整、路径是否正确

> 注意：YADB v1.1.2 不支持 `-help` 参数（会报 `Invalid argument`），请用 `-readClipboard` 验证。
