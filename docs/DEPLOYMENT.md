# supervisord 安装指南

supervisord 是用 Go 重写、兼容 Python supervisord 配置的进程管理工具，自带 HTTPS 网页控制台，用来在 Linux 服务器上启动、停止、守护你的程序并查看日志。安装文件是加密的，**安装前请先向维护者要安装密码**（本文件里没有密码）。

## 安装前准备

- 一台使用 systemd 的 Linux 服务器。推荐 Debian 12 / 13、Ubuntu 24.04；已实测 Debian 12、Debian 13、Ubuntu 23.04、23.10、24.04。
- 架构：x86_64（amd64）、ARM64（aarch64）或 ARMv7。
- 用 root 登录，或者先执行 `sudo -i`。
- 需要 `curl`（或 `wget`）和 `openssl`。Debian、Ubuntu 一般都自带；缺 `openssl` 时脚本会自动用 apt 安装。

## 一键安装

在服务器终端逐行执行：

```bash
curl -fL --proto '=https' -o install.sh https://github.com/nickdodo1088/supervisord-releases/releases/latest/download/install.sh
sudo sh install.sh
```

1. 提示“请输入安装密码”时，输入维护者给你的密码，输完按回车。输入时屏幕上不显示任何字符，这是正常的。
2. 提示“面板端口”时，直接回车使用默认的 **8002**，或者输入 1024 到 65535 之间的其他端口。端口被占用时会让你换一个。
3. 脚本自动下载、校验、解密并安装，结束时显示：

   ```
   访问地址：https://<IP>:8002/<安全入口>
   用户名：  admin_xxxxxxxx
   密码：    xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   证书指纹：<64 位十六进制>
   ```

   **密码只显示这一次**，请马上保存。密码输错时脚本会让你重新输入，最多 3 次；三次都不对就停止，不会安装任何东西。

4. 用浏览器打开访问地址。证书是安装时自动生成的自签名证书，浏览器会提示“不安全”：核对浏览器里看到的证书 SHA-256 指纹与上面显示的一致，再选择继续访问。然后用显示的用户名和密码登录。

安装密码只用来解密安装文件，和面板的登录密码不是一回事。

### 打不开面板？

- **只打开 `https://<IP>:<端口>/` 会显示 404。** 地址末尾的安全入口不能省略。这是为了让扫描器找不到登录页。
- 云服务器（阿里云、腾讯云、AWS 等）要在控制台的**安全组**里放行面板端口（TCP）。
- 服务器开着 ufw 或 firewalld 时，安装脚本会自动放行面板端口；用别的防火墙请自己放行。
- 显示的 IP 是服务器网卡上的地址。云服务器或在 NAT 后面的机器，请换成公网 IP。
- 查看服务状态：`systemctl status go-supervisord`；查看日志：`journalctl -u go-supervisord -n 50`。

## 常用安装参数

参数写在 `sudo sh install.sh` 后面，可以组合使用：

| 参数 | 作用 |
| --- | --- |
| `--port 9443` | 指定面板端口（默认 8002，不再询问） |
| `--entrance myentry` | 指定安全入口（默认随机 12 位）；`--entrance off` 表示不用安全入口 |
| `--bind 127.0.0.1` | 只在本机监听（默认 `0.0.0.0`，即所有网卡） |
| `--release-password-file ./pw.txt` | 从文件第一行读安装密码，不用手动输入 |
| `-y` | 不提问，没给的选项都用默认值 |
| `--no-firewall` | 不自动在 ufw / firewalld 放行端口 |
| `--lang en` | 提示改用英文 |

例如不想回答问题、全自动安装：

```bash
echo '<安装密码>' > ./pw.txt && chmod 600 ./pw.txt
sudo sh install.sh --release-password-file ./pw.txt --port 8002 -y
rm -f ./pw.txt
```

## 离线安装（服务器不能访问 GitHub）

1. 在能上网的机器上，从发布页 <https://github.com/nickdodo1088/supervisord-releases/releases/latest> 下载两个文件，放到服务器的同一个目录：
   - `install.sh`
   - `supervisord-<版本>-linux-amd64.enc`（ARM64 服务器下载 `arm64`，ARMv7 下载 `armv7`）
2. 在服务器上执行：

   ```bash
   sudo sh install.sh --package ./supervisord-<版本>-linux-amd64.enc
   ```

服务器的 OpenSSL 太旧（低于 1.1.1）时，先在别的 Linux 机器上解密，不需要 root，也不会安装任何东西：

```bash
sh install.sh --decrypt ./supervisord-<版本>-linux-amd64.enc
```

得到 `supervisord-<版本>-linux-amd64`，把它和 `install.sh` 复制到服务器，执行：

```bash
sudo sh install.sh --binary ./supervisord-<版本>-linux-amd64
```

## 装好以后

| 位置 | 内容 |
| --- | --- |
| `/usr/local/bin/supervisord` | 程序本身 |
| `/etc/supervisord/supervisord.yaml` | 配置文件：账号、密码哈希、证书、安全入口和要管理的程序 |
| `/etc/supervisord/supervisord.log` | 面板自己的日志 |
| `go-supervisord.service` | systemd 服务，开机自动启动 |

服务以专用普通账号 `supervisord` 运行，不是 root，没有任何特权；每个被管理的程序都在自己的沙箱里，读不到面板的配置和私钥。

## 添加要管理的程序

用编辑器打开 `/etc/supervisord/supervisord.yaml`，在文件末尾加上，例如：

```yaml
programs:
  my-app:
    command: /srv/my-app/my-app --port 9000
    directory: /srv/my-app
    autostart: true
    autorestart: true
    stdout_logfile: /srv/my-app/logs/stdout.log
    stderr_logfile: /srv/my-app/logs/stderr.log
```

然后：

1. 把程序目录交给服务账号：`chown -R supervisord:supervisord /srv/my-app`。
2. 服务的沙箱默认不能写系统目录。程序需要写入的目录要放行：执行 `systemctl edit go-supervisord`，写入下面两行后保存：

   ```ini
   [Service]
   ReadWritePaths=/srv/my-app
   ```

3. 执行 `systemctl restart go-supervisord`，或者在网页控制台点“重新加载配置”。

配置文件开头和末尾有带注释的示例；`environment`（环境变量）、`autorestart`、`stopsignal` 等写法与 Python supervisord 相同。

## 常见操作

```bash
systemctl status go-supervisord          # 查看面板运行状态
systemctl restart go-supervisord         # 重启面板（会按配置优雅停止再启动所有程序）
journalctl -u go-supervisord -n 50       # 查看最近的面板日志
supervisord -c /etc/supervisord/supervisord.yaml ctl status   # 在命令行查看程序状态
grep entrance /etc/supervisord/supervisord.yaml               # 忘了安全入口
```

**忘了登录密码**：重新生成一个（旧密码立即失效），然后重启：

```bash
runuser -u supervisord -- /usr/local/bin/supervisord init --output /etc/supervisord/supervisord.yaml --reset-password
systemctl restart go-supervisord
```

**更换安全入口**（6 到 64 位字母、数字、`-` 或 `_`）：

```bash
runuser -u supervisord -- /usr/local/bin/supervisord init --output /etc/supervisord/supervisord.yaml --entrance 新入口
systemctl restart go-supervisord
```

**更换端口**：把配置文件里 `inet_http_server` 下的 `port: 0.0.0.0:8002` 改成新端口（1024 以上），然后 `systemctl restart go-supervisord`，并在防火墙和安全组放行新端口。

## 升级

在已经安装的服务器上，再执行一次同样的安装命令就是升级：

```bash
curl -fL --proto '=https' -o install.sh https://github.com/nickdodo1088/supervisord-releases/releases/latest/download/install.sh
sudo sh install.sh
```

升级只替换程序，配置、账号、密码、证书和安全入口都保持不变，不会再问端口。从很早的旧版本升级时，原来没有安全入口的安装仍然没有入口，地址不变。

## 卸载

```bash
systemctl disable --now go-supervisord
rm -f /etc/systemd/system/go-supervisord.service /usr/local/bin/supervisord
systemctl daemon-reload
# 下面两行会删除配置（含账号、证书、程序列表），确认不再需要再执行
rm -rf /etc/supervisord
userdel supervisord
```

## 安全说明

- `install.sh` 里写着本版本每个安装文件的 SHA-256。它先校验下载的加密文件，再解密，解密后再校验一次；文件被篡改或密码不对都会停止，不会安装任何东西。
- 安装过程只从 GitHub 下载本版本的加密安装包，不下载其他脚本，不上报任何信息。
- 面板只支持 HTTPS；不经过安全入口的访问一律返回 404；登录有失败次数限制。
- 安装密码请不要公开传播。
