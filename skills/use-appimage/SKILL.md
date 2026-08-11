---
name: use-appimage
description: AppImage 的安装、运行、菜单集成与开机自启。适用于用户提到 AppImage、appimage、FUSE、libfuse2、.desktop、应用菜单集成、开机自启、appimage-extract、appimagelauncher 等场景
---

# AppImage 使用指南

AppImage 是 Linux 免安装绿色软件格式：一个文件 = 整个应用，不需要 root 安装，删除文件即卸载。

## 1. 基础用法

```bash
chmod +x 应用.AppImage     # 首次赋予执行权限
./应用.AppImage            # 运行
```

- 报 FUSE 错误：安装 `libfuse2`（Ubuntu 24.04+ 用 `apt install libfuse2t64`）
- 无法运行时可查依赖：`ldd 应用.AppImage | grep "not found"`
- 解包查看内容：`./应用.AppImage --appimage-extract`（生成 `squashfs-root/` 目录）

## 2. 文件管理规范

统一存放，集中管理：

```
~/Apps/                          # 所有 AppImage 存放处
└── Snipaste.AppImage
```

- 更新 = 下载新版本直接覆盖旧文件
- 不用了 = 直接删除文件，无残留垃圾

## 3. 应用菜单集成

每个 AppImage 建一个 .desktop 文件，使应用出现在系统菜单、支持图标：

```bash
mkdir -p ~/.local/share/applications ~/.local/share/icons/hicolor/256x256/apps
```

创建 `~/.local/share/applications/<应用名>.desktop`：

```ini
[Desktop Entry]
Type=Application
Name=Snipaste
Comment=Snip & Paste!
Exec="/home/gauss/Apps/Snipaste.AppImage"
Icon=Snipaste
Categories=Graphics;ImageProcessing;
Terminal=false
```

- `Icon=Snipaste` 对应 `~/.local/share/icons/hicolor/256x256/apps/Snipaste.png`
- 图标可从 AppImage 内提取：`./应用.AppImage --appimage-extract` 后复制里面的 .png
- 注册菜单数据库：`update-desktop-database ~/.local/share/applications`

## 4. 开机自启

在 `~/.config/autostart/` 下放同名 .desktop 文件即可（GNOME/KDE 均识别）：

```ini
[Desktop Entry]
Type=Application
Name=Snipaste
Exec="/home/gauss/Apps/Snipaste.AppImage"
Terminal=false
X-GNOME-Autostart-enabled=true
X-GNOME-Autostart-Delay=3
```

## 5. 自动化集成工具（可选）

```bash
sudo apt install appimagelauncher
```

双击 AppImage 时自动询问集成：注册到应用菜单、自动处理图标，无需手动建 .desktop。

## 6. 转换为 deb（不推荐，仅特殊需求）

AppImage 无法"转换"为 deb，只能拆包重打包：

```bash
sudo apt install appimage2deb
appimage2deb 应用.AppImage          # 生成 .deb
apt install ./应用_*.deb
```

或手动：`./应用.AppImage --appimage-extract` 解包后用 `dpkg-deb --build` 打包。
缺点：依赖和启动脚本需手动调整，更新时要重新转换，可能和系统库冲突。

