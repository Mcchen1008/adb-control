---
name: adb-control
description: 通过 SSH + ADB 远程控制 Android 设备。支持截图、触控、按键、中文输入（YADB）、剪贴板、UI 布局分析。
---

# ADB 远程控制

## 初始化

首次使用前，先执行同目录下 `install.md` 完成环境初始化（安装 adb、连接 adbd、部署 YADB）。技能目录内已附带 `yadb` 二进制，可直接上传部署。

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
sshpass -p "$SSH_PASS" scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -P "$SSH_PORT" "$SSH_USER"@"$SSH_HOST":/sdcard/screen.png ./screen.png
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
# 设备端截图
adb -s emulator-5554 shell "screencap -p /sdcard/screen.png"
# 拉回本地（用「拉取文件」模板）
sshpass -p "$SSH_PASS" scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -P "$SSH_PORT" "$SSH_USER"@"$SSH_HOST":/sdcard/screen.png ./screen.png

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
# 原生 uiautomator
adb -s emulator-5554 shell "uiautomator dump /sdcard/ui.xml && cat /sdcard/ui.xml"
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

## 注意事项

- YADB 必须通过 `adb shell` 运行，不能直接在 SSH shell 中执行（shell 用户才有事件注入权限，直接跑会 `Aborted`）
- YADB 路径固定为 `/sdcard/yadb`，不要移动（`/data/local/tmp` 重启清空；Termux 私有目录 shell 用户无权读）
- 中文参数在双引号中直接传递（UTF-8），避免嵌套单引号
- 输入中文前确保输入框已获得焦点；输入框有残留文本时先清空
- TV 输入法（搜狗/天赐）是 Surface 绘制，uiautomator 看不到候选词，输入中文优先用 YADB
- 命令无效果时先排查：`dumpsys window | grep mCurrentFocus` 与 `dumpsys input_method | grep -E "mCurMethodId|mInputShown"`
