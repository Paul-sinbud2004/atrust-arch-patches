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
| 系统 libcurl 基于 OpenSSL 3, 与自带 curl/OpenSSL 1.1 混用 | 登录后每 35 秒左右守护进程 SIGSEGV, systemd 反复重启 → 隔几分钟掉线一次; `systemctl` 显示 `core-dump` | drop-in 注入 `LD_LIBRARY_PATH` 优先使用自带 curl, 并补上 `libcurl.so.4` 名字, 进程内不再出现第二套 curl/OpenSSL |
| aTrustAgent 不响应 SIGTERM | 每次 stop/重启都要等 `TimeoutStopSec` 超时再 SIGKILL, 慢 5~8 秒且 `systemctl` 报 `Failed with result 'timeout'` | drop-in 主动收尾(TERM → 2s → KILL 整个进程家族), stop 约 2 秒完成, `SuccessExitStatus=SIGKILL` 消除报错 |
| 官方单元 `KillMode=process` | 关机时 systemd-shutdown 长时间等待 aTrustAgent, `/home`、`/tmp` 无法卸载 | drop-in 改 `KillMode=control-group` + 超时 6s + ExecStopPost 精确清孤儿(不杀托盘) |
| 依赖 `dmidecode` 未声明 | 环境自检每 60 秒报 `can't find dmidecode cmd`, 设备信息缺失 | 加入 `depends` |
| deb 自带 libstdc++/libxcb 与系统冲突 | 托盘崩溃/黑窗 | prepare() 剔除, 用系统库 |
| nodejs 兼容性 | bsod_checker 报错 | 覆盖 `bsod_checker.js`(来自上游 atrust-bin) |

## 与上游 atrust-bin 的其他区别

- 保留 deb 自带 Electron, 不依赖系统 electron9(上游依赖系统全局 Electron 9)
- 无 `chown ${USER}:${USER}` 步骤

## 文件说明

- `PKGBUILD` — AUR 包定义(deb 从深信服官方 CDN 拉取, 不转发二进制)
- `sangfor-atrust-bin.install` — pacman 钩子, 复刻 postinst + 会话类型声明
- `atrust-daemon-shutdown.conf` — systemd drop-in, 修复关机卡住与 stop 超时报错
- `atrust-daemon-libs.conf` — systemd drop-in, 修复 OpenSSL 混用导致的周期崩溃/掉线
- `atrust-daemon-path.conf` + `atrust-loginctl` — `Type=tty` 会话兼容层
- `atrust-launcher.sh` — `/usr/bin/atrust` 启动器
- `bsod_checker.js` — nodejs 兼容性修复(源自上游 atrust-bin)

## 许可

补丁与打包脚本按 Arch 通用惯例分发; aTrust 软件本体归深信服(Sangfor)所有,
由 PKGBUILD 在构建时从官方 CDN 下载。
