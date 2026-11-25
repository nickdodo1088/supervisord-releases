# supervisord 安装指南

supervisord 是用 Go 重写、兼容 Python supervisord 配置的进程管理工具，自带 HTTPS 网页控制台，用来在 Linux 服务器上启动、停止、守护你的程序并查看日志。安装文件是加密的，**安装前请先向维护者要安装密码**（本文件里没有密码）。

## 安装前准备

- 一台使用 systemd 的 Linux 服务器，系统和 CPU 架构见下面的“支持的系统”。
- 用 root 登录，或者先执行 `sudo -i`。
- 需要 `curl`（或 `wget`）和 `openssl`。Debian、Ubuntu 一般都自带；缺 `openssl` 时脚本会自动用 apt 安装，其他系统请先自己装好。

### 支持的系统

推荐 **Debian 12 / 13** 或 **Ubuntu 24.04 LTS**。下表是 v0.9.1 实际安装测试过的组合，每项都验证了下载、解密、启动服务、通过安全入口登录和程序隔离：

| 系统 | 已实测的 CPU 架构 |
| --- | --- |
| Debian 13 | x86_64、ARM64、ARMv7 |
| Debian 12 | x86_64、ARM64、ARMv7 |
| Ubuntu 24.04 LTS | x86_64、ARM64、ARMv7 |
| Ubuntu 23.04 | x86_64、ARM64（已停止维护，建议换成 24.04） |
| Ubuntu 23.10 | x86_64（已停止维护，建议换成 24.04） |

- 从 v0.9.8 起，发布页暂时只提供 x86_64（`amd64`）安装包，ARM64 和 ARMv7 暂停发布：在 ARM 机器上运行新版的 `install.sh` 会说明原因并停下，什么都不改。ARM 机器可以继续使用 v0.9.7：下载 v0.9.7 的 `install.sh`，加 `--no-self-update` 运行。表里的 ARM 组合是 v0.9.7 及以前实测的。
- 安装脚本按 `uname -m` 自动选择安装包：`x86_64` 用 `amd64`，`aarch64` 用 `arm64`，`armv7l` 用 `armv7`。程序是静态编译的，不依赖系统的 glibc，所以表里没有的组合（例如 Ubuntu 23.10 的 ARM64、64 位树莓派 OS）预计同样可用，只是没有逐一测试。
- 其他使用 systemd 的发行版（例如 Rocky Linux、AlmaLinux）没有实测。
- 不支持：32 位 x86、ARMv6（初代树莓派、树莓派 Zero / Zero W）、没有 systemd 的环境（例如普通 Docker 容器）。

内核版本不影响安装和面板本身，只决定被管理程序的隔离有多严格：

| 内核 | 常见系统 | 程序隔离 |
| --- | --- | --- |
| 6.12 及以上 | Debian 13 | 全部生效：读不到面板的配置、私钥和日志，不能跟踪面板和其他程序，也不能给它们发信号 |
| 5.19 到 6.11 | Debian 12（原装 6.1）、Ubuntu 24.04（原装 6.8） | 除了“不能发信号”以外都生效 |
| 低于 5.19 | 较老的系统 | 程序能读到面板的配置；面板日志里会给出警告 |

想在 Debian 12 或 Ubuntu 24.04 上得到完整隔离，可以换用 6.12 及以上的内核（Debian 12 用 bookworm-backports 里的内核，Ubuntu 24.04 用 HWE 内核），换内核需要重启服务器。

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
| `--restart` | 升级时直接重启服务（所有程序都会重启一次），不做热升级 |
| `--no-restart` | 正在运行的版本不能热升级时不自动重启，新版本等下次重启服务时生效 |
| `--no-self-update` | 不自动换成最新版的 `install.sh`，安装这个脚本对应的版本 |
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
   - `supervisord-<版本>-linux-amd64.enc`（v0.9.7 及以前另有 ARM64 的 `arm64` 和 ARMv7 的 `armv7`）
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

所有文件都在 `/opt/supervisord` 下：

| 位置 | 内容 |
| --- | --- |
| `/opt/supervisord/bin/supervisord` | 程序本身；上一个版本是同目录的 `supervisord.prev`。属于 root，服务自己改不了 |
| `/opt/supervisord/etc/supervisord.yaml` | 配置文件：账号、密码哈希、证书、安全入口和要管理的程序 |
| `/opt/supervisord/logs/supervisord.log` | 面板自己的日志；程序的日志也可以写在这个目录 |
| `/opt/supervisord/data` | 服务账号的家目录 |
| `/etc/systemd/system/go-supervisord.service` | systemd 服务，开机自动启动。systemd 只从 `/etc/systemd/system` 加载服务，所以只有它不在 `/opt` 下 |

服务以专用普通账号 `supervisord` 运行，不是 root，没有任何特权；每个被管理的程序都在自己的沙箱里，读不到面板的配置和私钥。

## 添加要管理的程序

用编辑器打开 `/opt/supervisord/etc/supervisord.yaml`，在文件末尾加上，例如：

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
2. 再运行一次 `sudo sh install.sh`。服务的沙箱默认不能写系统目录，脚本会把程序要写入、并且属于 `supervisord` 的目录（这里是 `/srv/my-app` 和 `/srv/my-app/logs`）自动加进沙箱的可写目录，不用自己改 systemd 设置。
3. 执行 `systemctl restart go-supervisord`，沙箱设置在重启时生效。

程序还要写别的位置时，执行 `systemctl edit go-supervisord`，在 `[Service]` 下加 `ReadWritePaths=<目录>`。

配置文件开头和末尾有带注释的示例；`environment`（环境变量）、`autorestart`、`stopsignal` 等写法与 Python supervisord 相同。

## 代码同步：推送代码后自动更新网站

网站（PHP、HTML、前端编译结果）可以交给 supervisord 自动从 Git 仓库更新，取代循环执行 `git pull` 的脚本：远端为准，白名单以外多出来的文件（例如被上传的木马）每次检查都会删除，文件属于你指定的用户和组，失败在控制台显示原因，随时回滚。在控制台“代码同步”页面点“新增同步任务”，按向导生成配置、复制到配置文件，再在服务器上以 root 执行一次 `sudo /opt/supervisord/bin/supervisord deploy approve`。完整说明见 [代码同步](CODE_SYNC.md)。

## 常见操作

```bash
systemctl status go-supervisord          # 查看面板运行状态
systemctl restart go-supervisord         # 重启面板（会按配置优雅停止再启动所有程序）
journalctl -u go-supervisord -n 50       # 查看最近的面板日志
/opt/supervisord/bin/supervisord ctl status       # 在命令行查看程序状态
grep entrance /opt/supervisord/etc/supervisord.yaml   # 忘了安全入口
```

**忘了登录密码**：重新生成一个，用打印出的新密码直接登录。控制台一秒内就会用上新密码，旧密码和所有已登录的浏览器同时失效；不需要重启，程序不受影响（v0.9.8 起；更早的版本还要执行 `systemctl restart go-supervisord`）：

```bash
runuser -u supervisord -- /opt/supervisord/bin/supervisord init --output /opt/supervisord/etc/supervisord.yaml --reset-password
```

**更换安全入口**（6 到 64 位字母、数字、`-` 或 `_`）：

```bash
runuser -u supervisord -- /opt/supervisord/bin/supervisord init --output /opt/supervisord/etc/supervisord.yaml --entrance 新入口
systemctl restart go-supervisord
```

**更换端口**：把配置文件里 `inet_http_server` 下的 `port: 0.0.0.0:8002` 改成新端口（1024 以上），然后 `systemctl restart go-supervisord`，并在防火墙和安全组放行新端口。

## 升级

在已经安装的服务器上，再执行一次安装命令就是升级，之后不需要手动做任何事：

```bash
curl -fL --proto '=https' -o install.sh https://github.com/nickdodo1088/supervisord-releases/releases/latest/download/install.sh
sudo sh install.sh
```

从 v0.9.5 起，手里留着的 `install.sh` 是旧版本也可以直接运行：它会先从发布页取最新版的 `install.sh`，改用它来安装（加 `--no-self-update` 则安装这个脚本自己的版本）。v0.9.4 及更早的 `install.sh` 没有这个功能，用上面的命令重新下载一次。

升级只替换程序，配置、账号、密码、证书和安全入口都保持不变，不会再问端口。从很早的旧版本升级时，原来没有安全入口的安装仍然没有入口，地址不变。

**升级不会重启你的程序。** 服务正在运行时，脚本做热升级：面板原地换成新版本，所有程序继续运行，进程号和日志都不中断。结束时显示“已热升级到 vX：托管的程序没有重启”。网页控制台断开约一秒后自动重连，不用重新登录（从 v0.9.5 及更早的版本升级的那一次除外）。

不能热升级时，脚本自己处理：

- 正在运行的是 v0.9.1 或更早的版本（还没有热升级功能）：自动重启一次服务，所有程序重启一次。之后的升级都是热升级。
- 有程序正在启动或停止，或者配置正在重新加载：等一会儿再试，试几次都不行就自动重启一次服务。
- 自动重启后，脚本检查原来在运行的程序都重新运行起来，并且 10 秒内进程号不变。有程序起不来，就自动退回原来的版本：程序、systemd 设置和配置文件都恢复原样，服务照常运行，脚本列出起不来的程序并报错。
- 配置文件里改了某个程序但还没生效（或者配置文件加载不了）：**不重启**，因为重启会让这些修改一起生效。正在运行的面板和程序都不受影响。在网页控制台点“重新加载配置”让修改生效，或者撤销修改，再运行一次安装命令就会热升级。

不想在升级时自动重启，加 `--no-restart`：不能热升级时只替换程序文件，新版本等下次重启服务时生效。想直接重启服务，加 `--restart`。

上一个版本保存在 `/opt/supervisord/bin/supervisord.prev`，需要退回时把它改名回 `supervisord`，再重启服务。**退回前先删掉旧版本不认识的配置项。** 旧版本遇到不认识的配置项会拒绝加载整个配置文件（日志里是 `unknown key`），服务启动不了，所有程序都不会运行。例如 `allow_sctp` 是 v0.9.3 新增的，退回 v0.9.2 前要先从配置文件里删掉。

热升级实测过的组合：

| CPU 架构 | 系统 |
| --- | --- |
| x86_64 | Debian 12、Debian 13、Ubuntu 24.04 LTS |
| ARM64 | Debian 13 |
| ARMv7 | Ubuntu 24.04 LTS |

每个组合都用发布页的 v0.9.2 安装包实测过：
- 新版本之间热升级时，面板和每个程序的进程号都不变，日志不缺行，事件监听器、程序隔离和网页登录都正常。
- 配置有未生效的修改时拒绝升级，加 `--restart` 时照旧重启服务。
- 热升级后 `systemctl stop` 能停掉全部程序；面板进程被强制结束时，systemd 会清理掉全部程序。

v0.9.5 的安装脚本另外实测过：
- x86_64 的 Debian 12、Debian 13 和 Ubuntu 24.04 LTS：从发布页的 v0.9.0 升级时自动重启并检查程序；从以 root 运行的 v0.8.0 自动迁移；有不能安全迁移的程序时停下、什么都不改；迁移后有程序起不来时自动退回；旧的 `install.sh` 自动换成最新版（Debian 12 上换成的是发布页的 v0.9.5，并热升级）。
- 发布后只用发布页的文件，在 Debian 12、Debian 13 和 Ubuntu 24.04 LTS 的 x86_64、ARM64 和 ARMv7 上（共九种组合）：v0.9.4 热升级到 v0.9.5，面板和每个程序的进程号都不变，日志不缺行，再运行一次什么都不改；以 root 运行的 v0.8.0 由同一个 `install.sh` 自动迁移到 `supervisord`，程序照常运行并继续写入数据。

v0.9.6 发布后只用发布页的文件，在 x86_64 的 Debian 12、Debian 13 和 Ubuntu 24.04 LTS 上实测过：
- v0.9.5 热升级到 v0.9.6，面板和每个程序的进程号都不变，日志不缺行，网页换成新界面；这一次网页需要重新登录。
- 之后 `systemctl reload`、网页“重新加载配置”、再运行一次 `install.sh` 和热升级，已登录的网页都不用重新登录。
- 用 v0.9.5 的 `install.sh` 加 `--no-self-update` 退回 v0.9.5，再用最新的 `install.sh` 升回 v0.9.6，两次都是热升级，进程号不变。

v0.9.7 的安装脚本发布前在 x86_64 的 Debian 12、Debian 13 和 Ubuntu 24.04 LTS 上实测过：
- 发布页的 v0.9.6 自动搬到 `/opt/supervisord`：程序重启一次后都正常运行，面板日志和用 `%(here)s` 写的程序日志都跟着搬过去，账号、密码和安全入口不变，旧位置什么都不留；之后再运行一次什么都不改，热升级时进程号都不变。
- 配置里写了旧路径、`systemctl edit` 的设置里写了旧路径、配置有还没生效的修改、加了 `--no-restart`：都停下，什么都不改。
- 搬完后有程序起不来：全部搬回原来的位置，v0.9.6 和所有程序照常运行，日志一行不少。
- 以 root 运行的 v0.8.0：一次完成账号迁移和搬家，程序以 `supervisord` 账号运行并继续写入数据。
- 新安装：`/opt` 以外只有 systemd 服务文件。

v0.9.7 发布后只用发布页的文件，在同样三个系统上又测过一遍，结果相同：
- 发布页的 v0.9.6 直接再运行它自己的 `install.sh`（不加参数），自动换成最新版并搬到 `/opt/supervisord`；之后热升级时进程号都不变，网页登录保持。
- 不能搬的几种情况都停下、什么都不改；搬完有程序起不来时全部搬回 v0.9.6。
- 全新安装和以 root 运行的 v0.8.0 都正常，`/opt` 以外只有 systemd 服务文件。

v0.9.8 发布后只用发布页的文件，在 x86_64 的 Debian 12、Debian 13 和 Ubuntu 24.04 LTS 上实测过：
- 发布页的 v0.9.7 热升级到 v0.9.8：面板和每个程序的进程号都不变，日志不缺行，网页不用重新登录，新控制台的体检、最近动态和安全页都能用；再运行一次什么都不改，配置文件一个字节都不变。
- 用 v0.9.7 的 `install.sh` 加 `--no-self-update` 退回 v0.9.7，再用最新的 `install.sh` 升回 v0.9.8，两次都是热升级，进程号不变，网页登录保持。
- 全新安装：配置读取 `conf.d/`，`/opt` 以外只有 systemd 服务文件，服务账号没有任何 capability，沙箱评分 OK；不带安全入口的地址返回没有正文的 404，安全入口返回没有正文的 302。
- 发布页的 v0.9.6 再运行它自己的 `install.sh`，自动换成最新版并搬到 `/opt/supervisord`，程序照常运行，旧位置什么都不留。

v0.9.9 发布后只用发布页的文件，在 x86_64 的 Debian 12、Debian 13 和 Ubuntu 24.04 LTS 上实测过：
- 发布页的 v0.9.8 热升级到 v0.9.9：面板和每个程序的进程号都不变，日志不缺行，网页不用重新登录；再运行一次什么都不改，配置文件一个字节都不变。用 v0.9.8 的 `install.sh` 加 `--no-self-update` 退回 v0.9.8 再升回来，两次都是热升级，进程号不变，网页登录保持。
- 全新安装、发布页的 v0.9.6 自动换成最新版并搬到 `/opt/supervisord`：都正常，`/opt` 以外只有 systemd 服务文件。
- 在 4 核的虚拟机里，用 60 个连接对面板各打 10 秒快速重置、MadeYouReset、SETTINGS 洪水和 PING 洪水：面板最多占 82% 的单核 CPU，内存不超过 112 MB；同时从另一个地址访问面板，每次都在 0.31 秒内返回。快速重置的来源被拒绝并记进日志，30 秒后自动恢复，服务和程序都没有重启。同样的虚拟机里，v0.9.8 在 PING 洪水下要占 2 个多核。
- 登录前请求体超过 16 KB 返回 413、地址超过 4 KB 返回 414，登录后同样长的地址正常返回；网页防火墙照常拦截 SQL 注入。

v0.9.10 发布后只用发布页的文件，在 x86_64 的 Debian 12、Debian 13 和 Ubuntu 24.04 LTS 上实测过：
- 发布页的 v0.9.9 热升级到 v0.9.10：面板和每个程序的进程号都不变，日志不缺行，网页不用重新登录。再运行一次什么都不改，配置文件一个字节都不变。用 v0.9.9 的 `install.sh` 加 `--no-self-update` 退回 v0.9.9 再升回来，两次都是热升级，进程号不变，网页登录保持，代码同步照常更新。
- 代码同步，用虚拟机里的 SSH Git 仓库和以 www-data 运行的 nginx 实测：
  - 用 www-data 做网站用户的任务，批准时写明 nginx 以这个账号运行；`--yes` 不批准，在终端里回答“否”也不批准。
  - 用单独账号的任务批准后，第一次同步删掉了遗留的旧编译文件并列出文件名，`dist/config/` 保留。文件属于单独账号、组只读，nginx 通过组正常读取，自己写不进去。
  - 两次推送之间放进网站目录的木马文件，和藏在嵌套 Git 仓库里的文件，在一个检查周期（10 秒）内被删除，被改过的文件恢复。更新记录、任务警告和最近动态都写明了文件名；之后的检查不再重复恢复。
  - 新的编译换掉所有文件名后，旧文件全部删除，`dist/config/` 保留，没有误报。
- 全新安装：`/opt` 以外只有 systemd 服务文件，服务账号没有任何 capability，沙箱评分 OK；不带安全入口的地址返回没有正文的 404，安全入口返回没有正文的 302。

### 从 v0.9.6 及更早的版本升级：自动搬到 /opt

v0.9.6 及更早的版本装在 `/usr/local/bin/supervisord`、`/etc/supervisord` 和 `/var/lib/supervisord`。新的安装脚本发现这样的安装时，会一次性搬到 `/opt/supervisord`，全部自动完成：

1. 先检查。下面这些情况脚本会列出来并停下，什么都不改，服务和程序照常运行：
   - 配置文件（包括 include 的文件）里写了 `/etc/supervisord`、`/var/lib/supervisord` 或 `/usr/local/bin/supervisord` 这样的绝对路径，搬家后那里就没有了。用 `%(here)s`（配置文件所在的目录）写的路径会跟着一起搬，不受影响；
   - 用 `systemctl edit go-supervisord` 加的设置里写了这几个路径；
   - 配置文件里有还没生效的修改（搬家要重启一次服务，重启会让这些修改一起生效）；
   - 加了 `--no-restart` 或 `--no-start`；
   - `/opt/supervisord` 下已经有 `logs` 或 `data` 目录。

   处理好以后再运行一次安装命令。
2. 停止服务，把配置目录搬到 `/opt/supervisord/etc`，把 `/var/lib/supervisord` 搬到 `/opt/supervisord/data`，面板自己的日志搬到 `/opt/supervisord/logs`（配置里的 `logfile` 跟着改），服务账号的家目录改成 `/opt/supervisord/data`，换上新的 systemd 设置后启动。所有程序重启一次，只有这次搬家需要。
3. 检查原来在运行的程序都重新运行起来，并且进程号稳定。有程序起不来，就全部搬回原来的位置，恢复原来的程序、systemd 设置和配置文件，服务照常运行，脚本列出起不来的程序。
4. 成功后删掉 `/usr/local/bin/supervisord`，原来的版本保存为 `/opt/supervisord/bin/supervisord.prev`，`/opt` 以外不留任何文件（systemd 服务文件除外）。

账号、密码、证书和安全入口都不变，访问地址也不变。命令行以后用 `/opt/supervisord/bin/supervisord`。程序自己的脚本里如果写死了这几个旧路径，脚本查不到，需要自己改。

### 从以 root 运行的旧版本升级

v0.8.0 的服务以 root 运行。新的安装脚本发现这样的安装时，会一次性迁移到专用账号 `supervisord`，全部自动完成：

1. 先以 `supervisord` 账号加载配置做检查。下面这些情况不能安全地自动处理，脚本会列出来并停下，什么都不改，旧服务和程序照常以 root 运行：
   - 程序设置了 `user=` 别的账号（服务账号不能切换账号）；
   - 程序或它的目录在 `/home`、`/root` 下（服务看不到）；
   - 有程序在监听小于 1024 的端口（需要 root）；
   - 程序要写入的目录是系统目录（`/usr`、`/etc`、`/var/lib` 这类，或者软件包装的目录），或者属于别的账号；
   - 路径里有空格或特殊字符。

   处理好以后再运行一次安装命令。
2. 把配置目录，以及程序要写入的目录（`directory` 和日志所在的目录）交给 `supervisord` 账号，并加进 systemd 沙箱的可写目录。只改原来属于 root 的文件，不跟随符号链接、不跨文件系统，别的账号的文件不动。同时像上一节那样搬到 `/opt/supervisord`。
3. 重启一次服务（只有这次迁移需要），所有程序改以 `supervisord` 账号运行。
4. 检查原来在运行的程序都重新运行起来。有程序起不来（例如它还要写配置里没有写到的 root 目录），就自动退回：恢复原来的程序、systemd 设置和配置文件，服务照常以 root 运行，脚本列出起不来的程序。已经交给 `supervisord` 的目录不改回去，以 root 运行的旧版本照样能用。

## 卸载

```bash
systemctl disable --now go-supervisord
systemctl disable --now 'go-supervisord-deploy@*.timer' 'go-supervisord-deploy@*.path' 2>/dev/null
rm -f /etc/systemd/system/go-supervisord.service /etc/systemd/system/go-supervisord-deploy@.*
rm -rf /etc/systemd/system/go-supervisord.service.d /etc/systemd/system/go-supervisord-deploy@*.d
systemctl daemon-reload
# 下面两行会删除程序、配置（含账号、证书、程序列表）、日志和数据，确认不再需要再执行
rm -rf /opt/supervisord
userdel supervisord
```

## 安全说明

- `install.sh` 里写着本版本每个安装文件的 SHA-256。它先校验下载的加密文件，再解密，解密后再校验一次；文件被篡改或密码不对都会停止，不会安装任何东西。
- 安装过程只从 GitHub 下载本版本的加密安装包，不下载其他脚本，不上报任何信息。
- 面板只支持 HTTPS；不经过安全入口的访问一律返回 404；登录有失败次数限制。
- 面板的 HTTP/2 用 Go 标准库实现（不是 nginx，不受 NGINX2Root 那组 nginx 漏洞影响），并限制了每个连接的并发流、帧大小和接收缓冲区。对方把接收窗口一直设为 0、想占住内存和连接的攻击（HTTP/2 Bomb，CVE-2026-49975）也防住了：每个应答最多等 15 秒，不知道安全入口的访问不占用任何资源。面板不提供 HTTP/3。
- 快速重置（Rapid Reset）、MadeYouReset、SETTINGS/PING 洪水这类 HTTP/2 攻击也防住了：一个 IP 刚开就取消大量请求时，面板拒绝它 30 秒（日志里有 `refusing a management client` 一行）；每个 IP 每秒最多读 1 MB。登录前的请求只接受很小的输入，WAF 同时最多检查几个。实测这些攻击期间服务器只占不到一个核，内存不涨。
- 托管的程序默认不能创建 SCTP 套接字，即使某个程序被攻破，也无法借内核 SCTP 漏洞（CVE-2026-64564）提权。确实需要 SCTP 的程序设置 `allow_sctp: true`。服务器本身仍应升级内核；不用 SCTP 的服务器可以禁用 sctp 模块。
- 安装密码请不要公开传播。
