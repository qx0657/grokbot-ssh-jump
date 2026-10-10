# 云电脑没有公网 SSH，也能登录，也能当节点

云电脑自己拨出来，把 22 挂在跳板的回环 `2222` 上。笔记本经这条隧道登录 `root`；另一条会话把云电脑的 `7890` 拉到本机 `17890`。保活有三层，每一层都有漏下去的情况。

地址、指纹和文件路径没有写进来。名字用的是笔记本、跳板、云电脑。

排版页在 [`share.html`](share.html)，克隆后用浏览器打开。

## 云电脑把 22 挂到跳板回环

云电脑上的 sshd 只听 `127.0.0.1:22`。出站是通的，所以由它主动连跳板，做一条反向转发：

```bash
ssh -N -o ExitOnForwardFailure=yes \
  -o ServerAliveInterval=15 -o ServerAliveCountMax=3 \
  -R 127.0.0.1:2222:127.0.0.1:22 jump
```

跳板因此只在自己的回环上听 `2222`，对端是云电脑的 `22`。`2222` 只出现在这个回环上。防火墙上不为它开公网，转发也不绑到 `0.0.0.0`。

无人值守时加上 `BatchMode` 和 `ConnectTimeout`。拨出失败就退出，交给后面的监督进程重来。

## 第二跳才是登录，用户是 root

笔记本先用跳板那把钥匙连上跳板的 `22`。第二跳才进云电脑：用户是 `root`，钥匙是另一把，并核对云电脑上固定的那把主机密钥。配置里的 `127.0.0.1:2222` 写的是跳板自己的回环，这一跳发生在跳板上。

拨出用第三把钥匙，只放在云电脑上，用来连跳板。三把钥匙各管一段。

```sshconfig
Host jump
  HostName 跳板的公网地址
  User root
  IdentityFile ~/.ssh/id_ed25519_to_jump
  IdentitiesOnly yes

Host cloud
  HostName 127.0.0.1
  Port 2222
  User root
  ProxyJump jump
  IdentityFile ~/.ssh/id_ed25519_to_cloud
  IdentitiesOnly yes
```

![三台机器和回环上的反向隧道](images/01-login.png)

`2222` 是这张图里唯一涂了强调色的门。公网可以到跳板的 `22`，到 `2222` 的那一笔在边界前停下。

## 节点是 box 的另一条会话

同一个 `2222` 上再开一条会话，用户是 `box`。这条会话留在转发上：笔记本的 `127.0.0.1:17890` 接到云电脑的 `127.0.0.1:7890`。`7890` 是云电脑上的 mixed 入口，出站 direct，出口地址属于云电脑，流量记在云电脑上。

代理客户端里它是一个 SOCKS，指向 `127.0.0.1:17890`，UDP 关掉。订阅文件里没有这一条。跳板自己的订阅端口在 `443`，流量留在跳板里，到边界就停。

```sshconfig
Host cloud-proxy
  HostName 127.0.0.1
  Port 2222
  User box
  ProxyJump jump
  IdentityFile ~/.ssh/id_ed25519_to_cloud
  IdentitiesOnly yes
  LocalForward 127.0.0.1:17890 127.0.0.1:7890
  ExitOnForwardFailure yes
  BatchMode yes
  RequestTTY no
  ServerAliveInterval 30
  ServerAliveCountMax 6
```

测连通用 `curl --socks5-hostname`。域名在云电脑上解析，本机即使指定 IPv4，结果里仍可能出现云电脑的 IPv6。这段转发没写 `ConnectTimeout` 时，第二跳卡住会留下一个活着的进程，`17890` 却没有在听；这时 root 登录往往还在。要重启的是笔记本上的这段转发。

![同一条隧道分成登录和本地代理](images/02-node.png)

强调色只落在 `17890` 和 `7890`。`443` 和这条隧道住在同一台跳板上，中间那一笔走到边界就停。

## 保活三层，本机进程只管 17890

隧道会断。三层各补一种断法。拨出侧的保活是 15 秒 × 3，笔记本这段转发是 30 秒 × 6，跳板 sshd 是 30 秒 × 2。三处数字各管各的连接。

监督进程大约每 20 秒看一次云端 sshd、固定主机密钥、root 的钥匙、`7890`，以及那条 `-R`。断了就重拨。`2222` 被占住时，只结束跳板上听在回环 `2222` 的那一个会话，跳板自己的 sshd 继续留着。

第二层每 5 分钟跑一次。启动文件缺了才补回去，再把监督进程拉起来。文件还在、会话已经死了的时候，这一层补不上。

第三层在跳板的 sshd 上。30 秒一次，连续 2 次没有响应就拆连接。对端还在回保活包时，它留着这条连接。

跳板的 sshd 整个停了，三层都拨不进去。这时打开商家面板的 VNC。笔记本上另有一个常驻进程，只把 `ssh -N cloud-proxy` 拉起来，保住本机 `17890`。云电脑和跳板的隧道由上面三层负责。

![三层保活和最后剩下的风险](images/03-keepalive.png)

强调色只在监督进程上。最底下的虚线是剩下的风险，再往下那一条虚线框在三层之外。
