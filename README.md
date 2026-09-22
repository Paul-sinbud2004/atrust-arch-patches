# sangfor-atrust-bin — 让 Sangfor aTrust 在 Arch Linux 上开箱即用

AUR 包: <https://aur.archlinux.org/packages/sangfor-atrust-bin>

```bash
yay -S sangfor-atrust-bin
atrust   # 或从应用菜单启动托盘
```

## 为什么已有 atrust-bin 还要这个包

[`atrust-bin`](https://aur.archlinux.org/packages/atrust-bin) 在不少 Arch 机器上
"装上了但跑不正常", 甚至安装直接失败。本包完整复刻官方 deb 的安装后初始化,
并修复了几处环境适配问题(全部不改动厂商二进制):

| 问题 | 现象 | 修复 |
|---|---|---|
| PKGBUILD `chown ${USER}:${USER}` | 安装报"无效的组"直接失败 | 不做 chown (本包文件统一 root 属主) |
| deb 的 postinst 未被复刻 | 装完托盘报"核心服务未启动" | `.install` 钩子补齐: `chmod 777 resources/app`、`.aTrust/var`、`dmidecode` 工具加权、`InstallLocation` + `eaio_packman` 注册、启用并启动 `aTrustDaemon` |
| xinit/startx 直启桌面 (logind 会话 `Type=tty`) | 登录用户检测失败, 核心服务拒绝以该用户运行, CA 证书无法导入 | 钩子向 `/etc/environment` 声明 `XDG_SESSION_TYPE=x11`(卸载即还原) + `loginctl` 兼容包装器兜底 |
| 官方单元 `KillMode=process` | 关机时 systemd-shutdown 长时间等待 aTrustAgent, `/home`、`/tmp` 无法卸载 | drop-in 改 `KillMode=control-group` + 超时 8s + ExecStopPost 精确清孤儿(不杀托盘) |
| deb 自带 libstdc++/libxcb 与系统冲突 | 托盘崩溃/黑窗 | prepare() 剔除, 用系统库 |
| nodejs 兼容性 | bsod_checker 报错 | 覆盖 `bsod_checker.js`(来自上游 atrust-bin) |

## 与上游 atrust-bin 的其他区别

- 保留 deb 自带 Electron, 不依赖系统 electron9(上游依赖系统全局 Electron 9)
- 无 `chown ${USER}:${USER}` 步骤

## 文件说明

- `PKGBUILD` — AUR 包定义(deb 从深信服官方 CDN 拉取, 不转发二进制)
- `sangfor-atrust-bin.install` — pacman 钩子, 复刻 postinst + 会话类型声明
- `atrust-daemon-shutdown.conf` — systemd drop-in, 修复关机卡住
- `atrust-daemon-path.conf` + `atrust-loginctl` — `Type=tty` 会话兼容层
- `atrust-launcher.sh` — `/usr/bin/atrust` 启动器
- `bsod_checker.js` — nodejs 兼容性修复(源自上游 atrust-bin)

## 许可

补丁与打包脚本按 Arch 通用惯例分发; aTrust 软件本体归深信服(Sangfor)所有,
由 PKGBUILD 在构建时从官方 CDN 下载。
