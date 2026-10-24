# fanout 部署运行手册

本文档说明如何把本项目（含本地改动）完整部署到一台 Linux 服务器并运行起来。

---

## 1. 服务器要求

| 项目 | 要求 |
|---|---|
| 系统 | Linux（systemd 或 OpenRC），x86_64 / arm64 |
| 权限 | root（要创建 netns、改 iptables） |
| 内核能力 | 必须放开 `/dev/net/tun`。**LXC 容器常见没给**：`ls /dev/net/tun` 不存在且 `mknod` 报 Operation not permitted 的机器用不了 |
| 依赖 | `openvpn`、`curl`、`openssl`、`tar`、`iproute2`、`iptables`（install.sh 会自动装） |
| 内存 | 每条隧道一个 openvpn + netns，20 槽位约几百 MB |
| 架构 | x86_64 → amd64，aarch64/arm64 → arm64 |

Alpine 先装 bash：`apk add bash curl`。

---

## 2. 三种部署路径

### 路径 A：一键脚本（自己仓库的发布版）

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/cnhkpro/fanout/main/install.sh)
```

自动识别架构下载预编译二进制、装依赖、装服务、开机自启。装的是 `cnhkpro/fanout` 仓库 latest release 的版本。

### 路径 B：源码传到服务器上编译（推荐，含本地改动）

适合本地改过代码、服务器能装 Go 的场景：

```bash
# 本地：把整个项目目录传到服务器（git push 后 clone，或直接 rsync）
rsync -avz --exclude=.git ./ root@<服务器IP>:/opt/fanout/

# 服务器上：
apt-get install -y golang-go        # 或 dnf install golang，需 Go 1.21+
cd /opt/fanout
bash install.sh
```

install.sh 检测到当前目录有 `main.go` 且有 `go` 命令时，自动走源码编译分支：
`go build -trimpath -ldflags "-s -w" -o /usr/local/bin/fanout .`，后续步骤（Xray、转发、服务）与一键安装完全相同。

### 路径 C：本地交叉编译，只传二进制（服务器不装 Go）

本地有 Go 时最干净的方式：

```bash
# 本地（Mac 交叉编译 Linux 版，按服务器架构二选一）
GOOS=linux GOARCH=amd64 go build -trimpath -ldflags "-s -w" -o fanout-linux .
# 或 GOOS=linux GOARCH=arm64

scp fanout-linux root@<服务器IP>:/usr/local/bin/fanout
```

服务器上首次安装仍需跑一遍 install.sh 完成依赖/服务/转发配置（可先用路径 A 装一版打底），之后**每次更新只需替换二进制再重启**：

```bash
ssh root@<服务器IP> 'chmod 755 /usr/local/bin/fanout && systemctl restart fanout'
```

---

## 3. install.sh 做了什么（排障时对照）

| 步骤 | 动作 |
|---|---|
| [1/6] 检查依赖 | 缺啥装啥，支持 apt/dnf/yum/pacman/apk/zypper |
| [2/6] 获取程序 | 源码目录+有 Go → 编译；否则下载 release 到 `/usr/local/bin/fanout` |
| [3/6] 准备 Xray | 有 3x-ui 或 xray-cf-lite 则跳过；否则下载 Xray 到 `/var/lib/fanout/bin/xray` |
| [4/6] 放行转发 | `sysctl net.ipv4.ip_forward=1` + iptables FORWARD 放行 `10.99.0.0/16` |
| [5/6] 安装服务 | `f.sh` → `/usr/local/bin/f`；`fanout.service` → systemd/OpenRC；写入 settings.json；enable + start |
| [6/6] 就绪 | 等程序生成口令和访问路径，打印管理地址 |

环境变量可调：`WEB_PORT`（首次安装端口，重装不覆盖已有 settings.json）、`WORK_DIR`、`REPO`。

---

## 4. 关键文件位置（`/var/lib/fanout/`）

| 文件 | 内容 | 生成时机 |
|---|---|---|
| `settings.json` | Web 端口、监听地址 | install.sh 写入 |
| `password` | 访问口令（随机） | 程序首次启动 |
| `basepath` | 访问路径（随机） | 程序首次启动 |
| `auth.secret` | 签名密钥（随机） | 程序首次启动 |
| `state.json` | 隧道状态（重启自动恢复，端口不变） | 运行时 |
| `nodes.json` | **节点池缓存**（含 .ovpn 配置） | 每次成功拉到节点列表后；vpngate 挂了靠它兜底 |
| `web.json` | 界面里改的设置 | 界面操作 |
| `fo1/`、`fo2/`… | 每条隧道的运行时目录（ovpn 配置、日志、auth 文件） | 建隧道时 |
| `bin/xray` | 自建模式的 Xray | install.sh |

删掉 `password` / `basepath` 后重启会重新随机生成（f 菜单第 8、9 项就是干这个的）。

---

## 5. 首次启动后

装完会打印：

```
管理界面  http://<IP>:8899/<随机路径>/
访问口令  <随机口令>
```

随时可查：

```bash
cat /var/lib/fanout/basepath
cat /var/lib/fanout/password
f info        # 一次看全
```

浏览器打开管理地址，输入口令登录。**路径不对一律 404**（不是连接失败），先确认路径末尾带 `/`。

防火墙/安全组要放行：
- Web 端口（默认 8899）
- 每个节点的入站端口（新建节点/出口时界面会显示）

---

## 6. 日常运维

```bash
f                # 管理菜单：启停/日志/隧道/改端口口令路径/更新/卸载
f log            # 跟踪日志（journalctl -u fanout -f）
f list           # 隧道列表
f restart        # 重启
systemctl status fanout
```

**更新自己改过的版本**（路径 C 的日常流程）：

```bash
# 本地
GOOS=linux GOARCH=amd64 go build -trimpath -ldflags "-s -w" -o fanout-linux .
scp fanout-linux root@<IP>:/usr/local/bin/fanout
# 服务器
ssh root@<IP> 'chmod 755 /usr/local/bin/fanout && systemctl restart fanout'
```

重启后 Web 秒级可用；节点列表在后台刷新，失败时自动沿用 `nodes.json` 缓存池。

---

## 7. 网络环境问题：vpngate 拉不通

程序依次尝试：直连 `https://www.vpngate.net/api/iphone/` → 反代 `https://p.xy.kg/vpngate`。国内/受限网络两者都可能失败，此时：

- **已有缓存池**：不影响使用，日志会提示"沿用现有节点池"。
- **全新机器**：界面上方会常驻"节点列表暂不可用"，需要先解决拉取：
  - 给服务挂出口代理（Go 默认读 `HTTPS_PROXY`）：编辑 `/etc/systemd/system/fanout.service`，在 `[Service]` 段加
    ```ini
    Environment=HTTPS_PROXY=http://127.0.0.1:7890
    ```
    然后 `systemctl daemon-reload && systemctl restart fanout`
  - 或自建反代后用环境变量替换内置反代：
    ```ini
    Environment=FANOUT_VPNGATE_MIRROR=https://你的worker地址/vpngate
    Environment=FANOUT_VPNGATE_MIRROR_KEY=你的key
    ```
- 手动验证连通性：
  ```bash
  curl -m 15 -o /dev/null -w '%{http_code}\n' https://www.vpngate.net/api/iphone/
  curl -m 15 -o /dev/null -w '%{http_code}\n' https://p.xy.kg/vpngate
  ```

---

## 8. 部署验证清单

```bash
systemctl is-active fanout                          # active
ss -tlnp | grep <web端口>                            # 监听中
cat /var/lib/fanout/basepath /var/lib/fanout/password
ls /var/lib/fanout/nodes.json                        # 节点池缓存已生成
curl -s http://127.0.0.1:<端口>/<路径>/api/nodes -o /dev/null -w '%{http_code}\n'   # 401（未登录属正常）
journalctl -u fanout -n 30 --no-pager                # 无持续报错
```

然后在界面里新建一个出口，成功建立隧道、SOCKS5 能出网即部署完成。

---

## 9. 常见坑

- **浏览器打不开管理界面**：先看端口是否监听、路径是否带尾斜杠；服务刚重启时 Web 约 1 秒内就绪，节点列表刷新不阻塞界面。
- **改路径/端口后"没反应"**：f 菜单改路径是重启生效，程序启动有几秒窗口；用 `f info` 拿当前真实地址。
- **AUTH_FAILED / 连不上**：志愿者节点正常损耗，会自动换候选重试（最多 6 个）。
- **没有 HTTPS**：公网使用建议前面套反代；订阅 token 是明文，别外泄，泄露就"换一串口令"。
- **LXC 无 tun**：换 KVM 机器，没有别的办法。
