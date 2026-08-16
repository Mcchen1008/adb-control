---
name: adb-control
description: 通过 SSH + ADB 远程控制 Android 设备。支持截图、触控、按键、中文输入（YADB）、剪贴板、UI 布局分析。
---

# ADB 远程控制

## 初始化

首次使用前，先执行同目录下 `install.md` 完成环境初始化（安装 adb、连接 adbd、部署 YADB）。技能目录内已附带 `yadb` 二进制，可直接上传部署。

## 整体工作流程

每次操控设备按此顺序执行：

1. **连接**：SSH 进入设备 Termux（`sshpass ssh ...`，参数见下）
2. **确认 adb**：`adb connect 127.0.0.1:5555`（首次或断线时执行），确认 `emulator-5554` 在线（`adb devices`）
3. **执行操作**：通过 `adb -s emulator-5554 shell "COMMAND"` 发指令——按键、点击、截图、启动应用
4. **中文输入**：需要输入中文时用 YADB：`adb -s emulator-5554 shell "app_process -Djava.class.path=/sdcard/yadb /data/local/tmp com.ysbing.yadb.Main -keyboard 中文"`
5. **确认结果**：截图拉回本地查看（`screencap` + scp），或 `uiautomator dump` / `-layout` 找元素坐标后点击
6. **防卡死**：操作后必须验证状态（焦点/画面/activity 是否变化）；**无变化时禁止重复按确认**，按「防卡死操作协议」诊断与兜底

## 连接配置

执行任何操作前先填写连接参数（必填，从用户处获取或沿用会话中已知值）：

```bash
SSH_HOST=""          # 设备 IP
SSH_PORT=22          # sshd 端口
SSH_USER=""          # Termux 用户
SSH_PASS=""          # sshd 密码
ADB_DEV="emulator-5554"
```

## 命令模板

所有设备操作都通过 SSH 进入设备的 Termux，再用 adb 执行。

**执行 adb 命令（通用模板）：**

```bash
sshpass -p "$SSH_PASS" ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=8 -p "$SSH_PORT" "$SSH_USER"@"$SSH_HOST" \
  'export PATH=/data/data/com.termux/files/usr/bin:$PATH; adb -s emulator-5554 shell "COMMAND"'
```

**拉取文件（截图/日志回传）：**

```bash
sshpass -p "$SSH_PASS" scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -P "$SSH_PORT" "$SSH_USER"@"$SSH_HOST":/data/local/tmp/screen.png ./screen.png
```

## 操作命令

以下 `adb` 命令均需套用上方「执行 adb 命令」模板（替换 `COMMAND` 部分）。

### 验证连接

```bash
adb devices
adb -s emulator-5554 shell "getprop ro.product.model"
```

### 截图

```bash
# 设备端截图（临时文件放 /data/local/tmp，重启自动清空）
adb -s emulator-5554 shell "screencap -p /data/local/tmp/screen.png"
# 拉回本地（用「拉取文件」模板）
sshpass -p "$SSH_PASS" scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -P "$SSH_PORT" "$SSH_USER"@"$SSH_HOST":/data/local/tmp/screen.png ./screen.png

# YADB 强制截图（应用阻止截图时）
adb -s emulator-5554 shell "app_process -Djava.class.path=/sdcard/yadb /data/local/tmp com.ysbing.yadb.Main -screenshot"
```

### 中文输入（YADB）

```bash
adb -s emulator-5554 shell "app_process -Djava.class.path=/sdcard/yadb /data/local/tmp com.ysbing.yadb.Main -keyboard 你好世界"
```

### 剪贴板

```bash
# 读取
adb -s emulator-5554 shell "app_process -Djava.class.path=/sdcard/yadb /data/local/tmp com.ysbing.yadb.Main -readClipboard"
# 写入
adb -s emulator-5554 shell "app_process -Djava.class.path=/sdcard/yadb /data/local/tmp com.ysbing.yadb.Main -writeClipboard 要复制的文本"
```

### UI 布局

```bash
# YADB 布局 dump（uiautomator 失效时用）
adb -s emulator-5554 shell "app_process -Djava.class.path=/sdcard/yadb /data/local/tmp com.ysbing.yadb.Main -layout"
# 原生 uiautomator（临时文件放 /data/local/tmp，重启自动清空）
adb -s emulator-5554 shell "uiautomator dump /data/local/tmp/ui.xml && cat /data/local/tmp/ui.xml"
```

### 按键与触摸

```bash
adb -s emulator-5554 shell input keyevent KEYCODE_HOME         # 主页
adb -s emulator-5554 shell input keyevent KEYCODE_BACK         # 返回
adb -s emulator-5554 shell input keyevent KEYCODE_DPAD_UP      # 上 / DOWN / LEFT / RIGHT
adb -s emulator-5554 shell input keyevent KEYCODE_DPAD_CENTER  # 确认
adb -s emulator-5554 shell input keyevent KEYCODE_ENTER        # 回车
adb -s emulator-5554 shell input tap 960 540                   # 点击 (x, y)
adb -s emulator-5554 shell "app_process -Djava.class.path=/sdcard/yadb /data/local/tmp com.ysbing.yadb.Main -touch 500 500 2000"  # 长按 500,500 2000ms
```

### 启动应用

```bash
# 按包名（启动入口 Activity）
adb -s emulator-5554 shell monkey -p com.example.app -c android.intent.category.LAUNCHER 1
# 通过 URL（浏览器打开）
adb -s emulator-5554 shell am start -a android.intent.action.VIEW -d "https://example.com"
```

### 辅助命令

```bash
# 列出已安装包
adb -s emulator-5554 shell pm list packages | grep -i 关键词
# 查看前台应用
adb -s emulator-5554 shell dumpsys window | grep mCurrentFocus
# 查看屏幕状态
adb -s emulator-5554 shell dumpsys power | grep -E "mWakefulness|Display Power"
# 清空输入框（两条分开执行，避免嵌套转义问题）
adb -s emulator-5554 shell "input keyevent --meta 4096 29"   # Ctrl+A 全选
adb -s emulator-5554 shell "input keyevent KEYCODE_DEL"      # 删除
```

## 防卡死操作协议（阉割系统兼容）

> 部分安卓阉割系统（TV 定制 ROM）功能不完整：应用可能**打不开**，或**打开了但选择无效果**，尤其系统自带的对话框（权限/ANR/存储提示）按钮经常"点了没反应"。AI 一旦卡住会反复按确认键空转。**禁止盲目连按，先按本协议来。**

### 核心原则：一次操作一次验证

任何 `keyevent` / `tap` / `am start` 之后：

1. 等待 1~2 秒让界面响应
2. 重新抓状态快照（见下）
3. **对比操作前后**：焦点变了 / 画面变了 / activity 变了 → 生效，继续下一步
4. **连续 2 次操作都无变化 → 立即停止点击，转「诊断」**，绝不来第 3 次

### 状态快照（操作前后各抓一次用于对比）

```bash
# ① 当前焦点窗口（最常用）
adb -s emulator-5554 shell "dumpsys window | grep mCurrentFocus"
# ② 前台 Activity（判断应用是否真的切过去了）
adb -s emulator-5554 shell "dumpsys activity activities | grep mResumedActivity"
# ③ 画面指纹（截图后取 md5，两次一致 = 画面没变）
adb -s emulator-5554 shell "screencap -p /data/local/tmp/shot.png && md5sum /data/local/tmp/shot.png"
# ④ 是否弹了系统对话框（ANR / 权限 / 存储提示）
adb -s emulator-5554 shell "dumpsys window windows | grep -iE 'dialog|anr|permission'"
adb -s emulator-5554 shell "uiautomator dump /data/local/tmp/ui.xml && cat /data/local/tmp/ui.xml"
```

### 操作无效果时的诊断顺序（按序排查，不要重按）

1. **看焦点**：`mCurrentFocus` 还在原地吗？——在 → 按键可能被系统吞了
2. **看画面**：截图 md5 前后是否一致？——一致 → 界面确实没响应
3. **看对话框**：是否被系统对话框挡住？（`-iE 'dialog|anr|permission'`）
4. **看应用**：目标包是否真的在前台？（`mResumedActivity`）
5. **看输入法**：`dumpsys input_method | grep -E "mCurMethodId|mInputShown"`

### 兜底恢复序列（标准操作无效时按序尝试，每步后重新验证）

```bash
# ① 返回一次（可能只是焦点丢了）
adb -s emulator-5554 shell "input keyevent KEYCODE_BACK"
# ② 回主页，重新进应用（绕开卡死的界面）
adb -s emulator-5554 shell "input keyevent KEYCODE_HOME"
adb -s emulator-5554 shell "monkey -p 包名 -c android.intent.category.LAUNCHER 1"
# ③ 应用打不开时，用 am start 直接指定入口 Activity（先用 resolve-activity 查）
adb -s emulator-5554 shell "cmd package resolve-activity --brief 包名"
adb -s emulator-5554 shell "am start -n 包名/入口Activity"
# ④ 仍不行：强停后重开（清掉坏状态）
adb -s emulator-5554 shell "am force-stop 包名"
adb -s emulator-5554 shell "monkey -p 包名 -c android.intent.category.LAUNCHER 1"
```

### 阉割系统已知坑

- **按键被吞**：部分 ROM 的 `KEYCODE_DPAD_CENTER` 在个别控件上无效，改用 `KEYCODE_ENTER` 或 `input tap x y` 直接点击
- **应用"打不开"**：`monkey` 启动失败是常态，改用 `am start -n 包名/Activity`；入口 Activity 用 `cmd package resolve-activity --brief 包名` 查询
- **对话框按钮看不见**：TV 对话框常为 Surface 绘制，uiautomator 无节点；用截图对比 + 估算按钮坐标 tap
- **权限弹窗"点了没反应"**：可能被桌面系统拦截，先 `KEYCODE_HOME` 再重进，或直接 `pm grant 包名 权限名` 绕过弹窗
- **点击无效果**：先确认屏幕是否亮着（`dumpsys power | grep -E "mWakefulness|Display Power"`），TV 待机时点击会被吞

## 注意事项

- YADB 必须通过 `adb shell` 运行，不能直接在 SSH shell 中执行（shell 用户才有事件注入权限，直接跑会 `Aborted`）
- YADB 路径固定为 `/sdcard/yadb`，不要移动（`/data/local/tmp` 重启清空；Termux 私有目录 shell 用户无权读）
- 临时文件（截图 `screen.png`、UI dump `ui.xml` 等）统一放设备端 `/data/local/tmp`，重启自动清空、不留垃圾；拉回本地后即用即删
- 中文参数在双引号中直接传递（UTF-8），避免嵌套单引号
- 输入中文前确保输入框已获得焦点；输入框有残留文本时先清空
- TV 输入法（搜狗/天赐）是 Surface 绘制，uiautomator 看不到候选词，输入中文优先用 YADB
- 命令无效果时先排查：`dumpsys window | grep mCurrentFocus` 与 `dumpsys input_method | grep -E "mCurMethodId|mInputShown"`
