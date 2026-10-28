---
name: use-linux-app
description: Linux 桌面应用的安装、桌面/Dock 集成与验收。适用于用户说"安装这个软件"、给一个 tar.gz/deb/自解压安装包、需要让应用出现在应用菜单或 Dock、需要验证 GUI 真的能跑（含 Wayland 无头/远程桌面环境）、排查图标重复或通用图标、清理卸载残留等场景
---

# Linux 桌面应用安装与验收

目标不是"端口起来了"或"进程活着"，而是：**装到位、菜单里能找到、图标对、真能干活**。下面每一步都给出可执行的验证手段，凡是无法验证的，要在结论里明说"没验证"。

## 1. 先看清包里有什么，再动手

```bash
curl -fsSLO <url>                    # 或 wget
tar tzf pkg.tar.gz | head -40        # 只看结构，先不解压
tar tzf pkg.tar.gz | wc -l
```

重点找这几类文件，它们决定了安装方式：

| 文件 | 含义 |
|---|---|
| `install.sh` / `uninstall.sh` | 官方安装器：负责路径、升级、菜单项、卸载，**优先用它** |
| `*.desktop` / `*.png` / `*.svg` | 已经带了桌面集成素材 |
| `AppRun` / `*.AppImage` | AppImage，转 `use-appimage` 技能 |
| `*.dll` + `libcoreclr.so` + `Filee` 这类组合 | 自包含 .NET，**不需要**装运行时 |
| `resources/app.asar` + `chrome-sandbox` | Electron |
| `libffmpeg.so` / `*.so` | 自带原生依赖，注意 glibc 版本 |

先读 `install.sh` 再执行（30 行就能看出它装到哪、改什么、能不能升级）：

- 装到 `~/.local/share/<app>/application` 还是 `/opt`
- 是否 `sudo`
- 是否写 `~/.local/bin`、`~/.local/share/applications`
- 是否有 `--upgrade`、事务/回滚、marker 文件

## 2. 安装

```bash
cd 解压目录 && ./install.sh          # 官方安装器
```

- **用户级应用不要 sudo**，装到 `~/.local/share` 才符合 XDG，也避免权限污染。
- 装完确认入口在 PATH 里：`command -v <cli>`，PATH 里没有 `~/.local/bin` 就提示用户加。
- 记下三个路径，后面写报告和卸载都要用：
  - 程序目录、CLI 符号链接、`.desktop` 文件位置。

## 3. 验收：必须真跑一次业务动作

只看 `--version` 不算验收。标准动作：

```bash
<cli> version
<cli> --help            # 或 formats / presets / list 之类
```

然后**造一个最小输入，跑真实转换/处理，并检查产物**：

```bash
# 造一个 64x64 测试图（没有 ImageMagick 也能用）
python3 - <<'PY'
import zlib, struct
w = h = 64
rows = b''.join(b'\x00' + bytes([(x*4) % 256 for x in range(w*3)]) for _ in range(h))
chunk = lambda t, d: struct.pack('>I', len(d)) + t + d + struct.pack('>I', zlib.crc32(t+d) & 0xffffffff)
png = b'\x89PNG\r\n\x1a\n' + chunk(b'IHDR', struct.pack('>IIBBBBB', w, h, 8, 2, 0, 0, 0)) \
    + chunk(b'IDAT', zlib.compress(rows)) + chunk(b'IEND', b'')
open('test.png', 'wb').write(png)
PY

<cli> convert test.png --to jpg -o out
file out/test.jpg        # 必须校验真实产物，而不是只看退出码
```

- 校验产物用 `file` / `identify` / `pdfinfo`，看**尺寸、格式、页数**对不对。
- 多测几条不同路径（不同目标格式、preset），能覆盖到不同引擎。
- 失败时先看日志：多数应用写在 `~/.config/<App>/logs/*.log`。

## 4. GUI 验证（Wayland / X11 都能做的办法）

先摸清会话类型：

```bash
echo "$XDG_SESSION_TYPE $XDG_CURRENT_DESKTOP DISPLAY=$DISPLAY WAYLAND_DISPLAY=$WAYLAND_DISPLAY"
```

启动 GUI 要脱离当前 shell，避免被会话回收：

```bash
(setsid ~/.local/share/<app>/application/<App> > /tmp/opencode/app.log 2>&1 &)
sleep 8
xwininfo -root -tree | grep -i <app>       # 看窗口是否真的建出来了
```

**截图优先级（按可靠性从高到低）**：

1. **窗口级截图**（首选，Wayland/XWayland 都能过）：
   ```bash
   import -window 0x3200018 /tmp/opencode/shot.png     # ImageMagick
   xwd -silent -id 0x3200018 -out w.xwd && convert w.xwd w.png
   ```
2. **整屏截图基本不可用**，别浪费时间：
   - `import -window root` / `xwd -root` → `BadMatch (X_GetImage)` 或 "资源暂时不可用"
   - `gdbus ... org.gnome.Shell.Screenshot` → `AccessDenied: Screenshot is not allowed`
   - `grim` 只适用于 wlroots，GNOME 下没有
3. 实在要整屏：只能让用户自己截，或装 `gnome-screenshot` 并接受 portal 授权弹窗。

**两个常见误判**：

- 窗口树里出现 `<App>` 但 `xwininfo -id <id>` 显示 `Map State: IsUnMapped` → 那是隐藏的辅助/幽灵窗口，不是主窗口。取 `IsViewable` 的那个。
- 同一应用出现多个窗口是正常的（主窗口 + 首次运行弹窗），分别截图确认内容，别把弹窗当崩溃。

截图后**自己看一眼**：标题、版本号、界面语言、是否有报错对话框。界面渲染出中文/英文文字即说明字体与 Skia 后端正常。

## 5. 桌面集成：应用菜单 + Dock 图标

`.desktop` 模板（用户级，不要 sudo）：

```ini
[Desktop Entry]
Type=Application
Name=<App>
Comment=<一句话说明>
Exec="/home/<user>/.local/share/<app>/application/<App>" %F
Icon=/home/<user>/.local/share/<app>/application/<app>.png
Terminal=false
Categories=Utility;
StartupWMClass=<App>
```

### 5.1 Dock 图标重复 / 显示通用齿轮的根因

Dock（GNOME Shell、Plasma）靠 **WM_CLASS ↔ .desktop 关联**决定"这个窗口属于哪个应用"：

- 关联不上 → 运行中的窗口被当成一个**没有启动器的新应用**，于是 Dock 里出现第二个图标，或者只有通用占位图标。
- 关联的桥梁就是 `StartupWMClass`，或者 `.desktop` 文件名与 WM_CLASS 完全同名。

查真实 WM_CLASS（不要猜）：

```bash
xprop -id <window-id> WM_CLASS          # 例：WM_CLASS(STRING) = "Filee", "Filee"
```

把 `StartupWMClass` 设成 **第二个字段（class）**，通常和 `.desktop` 的 `Name` 不同（`com.filee.app.desktop` vs `Filee`）。这是最容易被安装器漏掉的一步。

### 5.2 图标本身

- `Icon=` 用**绝对路径**最省事，且不会在卸载后留残渣。
- 用主题名（`Icon=filee`）则需要把 png 放进 `~/.local/share/icons/hicolor/<size>/apps/` 并 `gtk-update-icon-cache`；好处是缩放清晰，坏处是**官方卸载脚本不会删**，会留垃圾。自包含应用优先绝对路径。
- 图标至少 256×256（512 更好），否则 Dock 里发虚。

### 5.3 固定到 Dock

```bash
gsettings get org.gnome.shell favorite-apps
# 追加 'com.filee.app.desktop'，注意保持 GVariant 列表格式
```

### 5.4 验证桌面集成（不用重启会话）

```bash
desktop-file-validate ~/.local/share/applications/<id>.desktop   # 语法
update-desktop-database ~/.local/share/applications

python3 - <<'PY'
import gi; gi.require_version('Gio', '2.0')
from gi.repository import Gio
i = Gio.DesktopAppInfo.new('com.filee.app.desktop')
print(i.get_name(), '|', i.get_startup_wm_class(), '|', i.get_icon().to_string())
PY
```

三项都要对：名称、`StartupWMClass` 与 WM_CLASS 一致、图标路径存在。

### 5.5 安装器托管文件的注意事项

带 `X-<Vendor>-Managed=true` 或 `.filee-package` 这类 marker 的文件是**安装器资产**：

- 升级时会**重新生成** `.desktop`，你手打的 `StartupWMClass` 会被覆盖 → 升级后要重打，或在报告里写清楚。
- 卸载脚本只删带 marker 的文件，所以你额外放进 hicolor 的图标、改过的 favorites 都会留下 → 要么别加，要么在报告里列出"需要手动清理的项"。

## 6. 进程与信号：三个会把自己坑死的操作

1. **`pkill -f <路径>` 会匹配到自己这条命令**，把当前 shell 一起杀掉（表现为 `Killed by SIGTERM`、后续命令全丢）。用精确名：
   ```bash
   pkill -x Filee          # 匹配进程名，不匹配命令行
   pgrep -ax Filee
   ```
2. **单实例应用**第二次启动只是把参数交给已运行实例然后 `exit 0`，看起来"启动失败"其实是正常。日志里常见 `A later launch handed over 1 argument(s)`。判断是否真在跑：`pgrep -x <App>` + `xwininfo -root -tree`。
3. **优雅退出**优先用应用自带参数（`<App> --quit`），比 `kill -9` 安全，也能顺带验证退出路径。

## 7. 首次启动会自己下载东西

很多应用首启会拉引擎/组件（Pandoc、FFmpeg、模型、语言包）：

- 弹窗问"是否下载"时**不要替用户点"安装"**，除非明确要求；但要在报告里说明"还差什么、多大"。
- 有时它会**在后台默默装完**（日志里出现 `Installed engine component pandoc 3.12`），所以判断"是否装好"要看 `~/.config/<App>/logs`，别只靠界面截图。
- 有 `--quit` 就用它收尾，避免把半下载状态留在一个被杀死的进程里。

## 8. 收尾

- 删掉 `/tmp` 里的安装包和解压目录（几百 MB 很常见）。
- 告诉用户**怎么卸载**（官方 `uninstall.sh` 的路径），以及它会保留什么（配置、转换结果）。
- 报告固定包含五段：**装了什么版本 / 装到哪 / 怎么启动（GUI + CLI）/ 实测通过了什么 / 没验证或需要用户决定什么**。

## 9. 验收清单（可直接抄）

```
[ ] 包内容看过，安装器读过，知道装到哪
[ ] 安装成功，CLI 在 PATH
[ ] <cli> version / --help 正常
[ ] 真实业务动作跑通，产物用 file/identify 校验
[ ] GUI 窗口 IsViewable，窗口级截图确认界面正常
[ ] .desktop 通过 desktop-file-validate
[ ] StartupWMClass == WM_CLASS，Dock 不会出现双图标
[ ] 图标文件存在且 >=256px
[ ] 需要的话已固定到 Dock
[ ] 知道卸载命令，/tmp 已清理
```
