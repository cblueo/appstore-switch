# appstore-switch

在 macOS 上用终端一键切换 App Store 账号（例如国区 / 美区 / 土区）。

密码保存在系统钥匙串（Keychain），不写进配置文件。脚本通过 AppleScript 操作本机 App Store 的 **Store** 菜单和登录弹窗，完成退出 / 登录。

## 环境要求

- macOS（在 macOS 26.3、App Store 3.0 上验证过）
- 终端或 `osascript` 已开启 **系统设置 → 隐私与安全性 → 辅助功能**
- 已用 `./appstore-switch add` 保存过对应别名

## 安装

把仓库克隆到本地，或直接复制 `appstore-switch` 脚本：

```bash
chmod +x appstore-switch
```

## 用法

```bash
./appstore-switch add <别名> <Apple ID>
./appstore-switch list
./appstore-switch remove <别名>
./appstore-switch <别名>
```

示例：

```bash
./appstore-switch add cn1 you@example.com
./appstore-switch add us you-us@example.com
./appstore-switch add tr you-tr@example.com

./appstore-switch list
./appstore-switch us
```

不必事先打开 App Store。脚本会启动应用，等到 **Store** 菜单里出现「登录」或「退出登录」后再操作。

若 Apple 弹出双重认证，请手动输入验证码。

## 配置与密码

| 内容 | 位置 |
|------|------|
| 别名 → Apple ID | 脚本同级的 `accounts.conf`，格式 `别名=Apple ID` |
| 密码 | macOS Keychain，服务名 `appstore-switch:<别名>` |

`accounts.conf` 首次运行会自动创建，权限为 `600`。仓库里提供的是 [`accounts.conf.example`](accounts.conf.example)。真实的 `accounts.conf` 含有 Apple ID，不要提交进版本库，也不要公开分享。

## 日志

切换过程会向 stderr 打印带 `[appstore-switch]` 前缀的进度，例如等待菜单、点击登录、填写账号。不会打印密码。

## 限制

- 依赖 App Store 当前菜单文案和登录弹窗结构（`sheet 1 of sheet 1`）。系统或 App Store 更新后可能失效。
- 仅中英文菜单名做了适配（`Store` / `商店`，`Sign In` / `登录` 等）。
- 不能代替 iOS，也不能自动下载 App。

菜单或弹窗变了时，按下面「给 Agent」一节在本机探测后再改。

## 给 Agent：脚本失效时怎么改

只改 `appstore-switch` 里切换账号的那段 AppleScript。不要改 `add` / `list` / `remove` 和钥匙串逻辑；不要先加 Swift 或新依赖；不要用固定 `delay` 代替轮询。

先探测、验证能填上，再改代码。用占位邮箱，不要读 `accounts.conf` 里的真实账号，也不要从钥匙串取真密码试登。

1. 确认终端或 `osascript` 已开辅助功能。
2. 列出 Store 菜单（中文环境把 `"Store"` 换成 `"商店"`）：

   ```bash
   osascript -e 'tell application "App Store" to activate' -e 'delay 1' \
     -e 'tell application "System Events" to tell process "App Store" to get name of every menu item of menu 1 of menu bar item "Store" of menu bar 1'
   ```

   把实际的「登录 / 退出登录」文案写回脚本里的 `signInItems`、`signOutItems`、`storeMenuNames`。

3. 点 Store → Sign In 打开弹窗后，核路径（当前是 `sheet 1 of sheet 1 of window 1`）：

   ```bash
   osascript -e 'tell application "System Events" to tell process "App Store" to tell sheet 1 of sheet 1 of window 1 to return (count of text fields) & " " & (count of buttons)'
   ```

   路径报错就改 `loginSheet`；`fields` 不是 1（账号页）或 2（密码页）就改 `text field` 序号。

4. 用占位邮箱测：填 `text field 1` → return → 应出现 2 个 text field；密码在 field 1，账号在 field 2。
5. 抽出 AppleScript 用 `osascript` 编译/试跑，确认没有 `-2741`。冷启动验证要先完全退出 App Store（Command+Q）再跑 `./appstore-switch <别名>`。
6. 就绪条件是轮询 UI：Store 菜单出现登录/退出项（约 30 秒）→ `window 1`（约 30 秒）→ `sheet 1 of sheet 1` 且有 `text field 1`（约 10 秒）。
7. 终端日志前缀是 `[appstore-switch]`。报「没有找到登录入口」先看菜单是否就绪；报「还没加载完成 / 窗口还没出现 / 弹窗还没出现」再调对应 wait 次数。

用户可以对 Agent 说：「App Store 菜单或登录弹窗变了」或「没打开 App Store 时找不到登录入口，按 README 给 Agent 一节先探测再改。」

## 致谢

登录弹窗的填写路径参考了 [tobemaster 用 AppleScript 切换 App Store 账号](https://gist.github.com/tobemaster56/15ee009cca6de05c10ff7fae0592c3cb) 的做法。

## License

[MIT](LICENSE)
