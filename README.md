# supervisord

用 Go 重写、兼容 Python supervisord 配置的进程管理工具，自带 HTTPS 网页控制台：在 Linux 服务器上启动、停止、守护你的程序，并在浏览器里查看状态和日志。

[新手安装指南](docs/DEPLOYMENT.md) · [下载最新版本](https://github.com/nickdodo1088/supervisord-releases/releases/latest) · [English](#english)

## 第一次安装

准备一台使用 systemd 的 Linux 服务器（推荐 Debian 12 / 13、Ubuntu 24.04；支持 x86_64、ARM64、ARMv7，实测过的组合见[支持的系统](docs/DEPLOYMENT.md#支持的系统)），以 root 登录；sudo 用户先执行 `sudo -i`。向维护者索取**安装密码**，用于解密安装文件。

在服务器终端逐行执行：

```bash
curl -fL --proto '=https' -o install.sh https://github.com/nickdodo1088/supervisord-releases/releases/latest/download/install.sh
sudo sh install.sh
```

按提示输入安装密码（输入时不显示，属于正常情况），再输入面板端口（直接回车使用 8002）。脚本完成下载、校验、解密和安装，结束时显示访问地址、用户名和登录密码，请马上保存。

默认使用 **HTTPS 自签名证书、端口 8002 和随机安全入口**，访问地址形如 `https://<IP>:8002/<安全入口>`，末尾的安全入口不能省略。云服务器需在安全组放行显示的 TCP 端口；浏览器首次访问会提示证书不受信任，核对安装时显示的证书指纹后继续即可。

离线安装、自定义参数、添加程序、升级和卸载，见[新手安装指南](docs/DEPLOYMENT.md)。

## 升级

重新下载并执行上面的 `install.sh`，它会识别现有安装并升级程序；配置、账号、密码、证书和安全入口保持不变。

## 发布文件

| 文件 | 用途 |
| --- | --- |
| `install.sh` | 安装与升级入口，内置本版本所有文件的 SHA-256 |
| `supervisord-<版本>-linux-amd64.enc` | x86_64 服务器的加密程序 |
| `supervisord-<版本>-linux-arm64.enc` | ARM64 服务器的加密程序 |
| `supervisord-<版本>-linux-armv7.enc` | ARMv7 设备的加密程序 |
| `checksums.txt` | 上面各文件的 SHA-256，供手动核对 |

## English

supervisord is a Go process manager compatible with Python supervisord configuration, with an HTTPS web console. Release files are encrypted; ask the maintainer for the installation password. On a Linux server with systemd (x86_64, ARM64 or ARMv7; tested on Debian 12 and 13 and Ubuntu 23.04 to 24.04), as root:

```bash
curl -fL --proto '=https' -o install.sh https://github.com/nickdodo1088/supervisord-releases/releases/latest/download/install.sh
sudo sh install.sh --lang en
```

Enter the installation password and the panel port (Enter keeps 8002). The installer prints the URL `https://<IP>:<port>/<entrance>`, the user name and the password, which is shown only once. The panel answers 404 without the entrance at the end of the URL. Run the same commands again to upgrade.
