# 使用用本地音乐
---------------

我有一个aws 服务器， 有固定公网IP，但是这个aws的硬盘只有40G，我想做一个navidrome服务端，但是aws空间不足够存放音乐文件，我的本地PC有很大硬盘空间，如何将navidrome可以访问本地硬盘的音乐文件？本地是windows+wsl2+ubuntu+docker的环境，aws 是ubuntu

这个需求的本质一句话就能拆穿：Navidrome 跑在 AWS，但音乐数据不一定非要“住”在 AWS 硬盘上。
只要能把你本地的大硬盘“伪装成”AWS 上的一个目录，Navidrome 就会乖乖吃歌。

下面给你可落地、工程味最重的几种方案，按「稳定性 + 易用性」排序。

⭐ 推荐方案 1：SSHFS（最简单、最像本地盘）

思路
把你本地 PC 的音乐目录，通过 SSH 挂载到 AWS 的某个目录。

AWS:/mnt/music  —— 实际数据在 ——  本地PC:D:\Music


Navidrome 完全无感知。

架构
Navidrome (AWS)
      │
   /mnt/music   ← SSHFS →  本地 Windows / WSL2 磁盘

本地（WSL2 Ubuntu）操作

假设音乐在 D:\Music，WSL2 里一般是：

/mnt/d/Music


安装并启动 SSH 服务：

sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh


确认能从 AWS ssh 进你本地：

ssh user@你家公网IP -p 22


没公网 IP？那你之前搞过 WireGuard / sing-box，对你来说不是问题。

AWS 上挂载
sudo apt install -y sshfs
sudo mkdir -p /mnt/music
sudo sshfs user@你家IP:/mnt/d/Music /mnt/music \
  -o reconnect,ServerAliveInterval=15,ServerAliveCountMax=3


测试：

ls /mnt/music

Navidrome Docker 指向这个目录
volumes:
  - /mnt/music:/music:ro


优点

🧠 思维负担最低

🧩 不改 Navidrome

💾 本地盘多大都行

注意

本地断网 = AWS 播不了歌

首次扫描慢（但能忍）

⭐ 推荐方案 2：NFS（稳定性比 SSHFS 高）

更“正统”的远程文件系统，适合长期跑。

本地（WSL2 Ubuntu）
sudo apt install -y nfs-kernel-server


编辑：

sudo nano /etc/exports


加：

/mnt/d/Music *(ro,sync,no_subtree_check)

sudo exportfs -rav
sudo systemctl restart nfs-server

AWS 挂载
sudo apt install -y nfs-common
sudo mount -t nfs 本地IP:/mnt/d/Music /mnt/music


优点

比 SSHFS 稳定

CPU 占用低

缺点

配置比 SSHFS 稍麻烦

需要网络稳定

⚙️ 方案 3：rclone + HTTP / WebDAV（更“云化”）

你本地用 rclone serve 一个音乐服务，AWS 通过 HTTP 访问。

rclone serve http /mnt/d/Music --addr :8080


AWS 上：

用 rclone mount

或 Navidrome 指向 WebDAV（需中转）

优点

穿 NAT 方便

可加鉴权

缺点

比文件系统方案复杂

性能略差

❌ 不太推荐但可行

把音乐传到 AWS EBS / EFS（钱烧得快）

实时 rsync 同步（不适合频繁改动）

S3（Navidrome 不原生友好）

我给你的工程师级结论

首选 SSHFS
简单、直觉、能跑、出问题也好排查
对你这种 already 玩 AWS + VPN 的人，属于“无脑解”

长期 7×24 播放 → NFS

如果你愿意，我可以直接帮你：

给你一份 Navidrome + SSHFS systemd 自动重连方案

或画一个 你现有 WireGuard 拓扑下最稳的挂载方式

系统是混沌的，但文件系统可以很诚实。
# 没有公网ip
----------------------------------

但是实际上我现在没有合作wireguard，现在我使用的是clash verge，开启tune模式，但是我当前 的问题是我本地PC没有公网IP，要怎么办？

这就触到网络世界的硬骨头了：你想让 AWS 主动“看见”你家里的电脑，但你家电脑藏在 NAT 和运营商后面。
Clash TUN 再猛，也只是出站代理，不解决入站可达性。所以现在的问题不是 Navidrome，是反向连通性。

结论先给出，再解释宇宙运行原理。

没有公网 IP，本地 ≠ 服务器
你必须让「本地主动连 AWS」，而不是反过来

下面是真正可行的三条路，按靠谱程度排序。

⭐ 方案 1（强烈推荐）：反向 SSH 隧道（不需要公网 IP）

这是工程师最爱的“地下通道”。

核心思想
本地PC ──主动 SSH──▶ AWS
AWS 通过这个已建立的连接访问本地

架构
Navidrome (AWS)
   │
   │ localhost:10022
   │
反向 SSH
   │
本地 PC :22
   │
D:\Music

本地（WSL2 Ubuntu）执行
ssh -N -R 10022:localhost:22 ubuntu@AWS公网IP


含义翻译成白话：

-R 10022:localhost:22

在 AWS 上监听 10022

实际连到 你本地的 22 端口

保持这个 ssh 不断。

AWS 上验证
ssh -p 10022 user@localhost


如果能进你本地，宇宙已经对齐。

AWS 挂载本地音乐（SSHFS）
sudo sshfs -p 10022 root@localhost:/mnt/d/Music /mnt/music


Navidrome 指向 /mnt/music。

自动化（重要）

本地用 autossh：

sudo apt install -y autossh
autossh -M 0 -N -R 10022:localhost:22 ubuntu@AWS公网IP


断线自动重连。

优点

🚀 不需要公网 IP

🔐 安全（SSH）

🧠 网络结构清晰

注意

本地 PC 必须在线

AWS 安全组开放 10022（或仅本机）

⭐ 方案 2：Cloudflare Tunnel（优雅，但有点“云味”）

如果你不想碰 SSH。

思路

本地跑 cloudflared

把一个目录/服务暴露到 Cloudflare

AWS 通过 Cloudflare 访问

问题是：
Navidrome 不直接支持 Cloudflare 文件系统，你还得中转（WebDAV / HTTP）。

更适合 Web 服务，不是文件系统

⚙️ 方案 3：反向 WireGuard（你未来一定会走到这）

你现在没用 WG，但这正是它的用武之地。

本地 WG Client ──▶ AWS WG Server
AWS 看到你本地 = 内网主机


一旦连上：

SSHFS / NFS / rsync 都顺了

Clash 可以退休一部分

但这条路今天不用走，知道就好。

❌ 为什么 Clash TUN 没用？

因为：

TUN = 出站流量接管

不解决端口映射

不提供“被访问能力”

Clash 是车，不是隧道。

给你一个现实可跑的建议

今晚就能用的方案：
autossh + 反向 SSH + SSHFS

这是那种：

不改家庭网络

不碰路由器

不求人

但却非常工程化的解法。

如果你愿意，我可以下一步直接给你：

autossh + systemd 的完整 unit 文件

AWS 侧 mount 失败自动重试方案

或把 Navidrome + 这个挂载打包成一个 Docker Compose

网络的秘密在于：永远让最弱的一端先出手。
# 如何测试速度
-------

ssh -N -R 10022:localhost:22 ubuntu@AWS公网IP ---使用这种方式可以连接，但是如何测试这种方式连接后的带宽？

这个问题问得非常工程师 👍
反向 SSH 隧道能通 ≠ 性能靠谱，必须量化。

核心原则一句话：

别测“SSH 连接本身”，要测“通过这个隧道传数据的吞吐”

下面给你三种靠谱、可复现的测试方法，从“最干净”到“最贴近 Navidrome 实战”。

一、最标准：iperf3 走反向 SSH（推荐）

这是网络工程里测带宽的标尺。

1️⃣ 本地（WSL2）启动 iperf3 服务
sudo apt install -y iperf3
iperf3 -s


默认监听 5201。

2️⃣ 建立反向隧道（多加一个端口）
ssh -N \
  -R 10022:localhost:22 \
  -R 15201:localhost:5201 \
  ubuntu@AWS公网IP


意思：

AWS:15201 → 本地:5201（iperf3）

3️⃣ AWS 上测试带宽
iperf3 -c localhost -p 15201


你会看到类似：

[  5]   0.00-10.00  sec  120 MBytes  100 Mbits/sec


这就是：

AWS → 本地 → 反向 SSH 隧道 的真实吞吐

📌 这个数值，几乎就是 Navidrome 未来的上限。

二、更贴近文件系统：scp 实测（非常直观）
AWS → 本地（通过隧道）
scp -P 10022 user@localhost:/mnt/d/Music/test.flac /tmp/


看输出：

test.flac   100%   50MB  6.2MB/s


换算一下：

6.2 MB/s × 8 ≈ 50 Mbps


这是：

SSHFS / Navidrome 扫描

FLAC 实时播放

最贴近的现实速度。

三、SSHFS 实战测试（Navidrome 同款）
AWS 上
dd if=/mnt/music/big.flac of=/dev/null bs=4M status=progress


输出会实时显示：

1.2 GB copied, 12 s, 100 MB/s


这是：

Navidrome 真正读文件时能跑多快

关键认知（非常重要）
1️⃣ 反向 SSH ≠ 慢

慢的不是 -R，慢的是：

家庭上行带宽

Clash TUN 抢流量

SSH 加密 CPU 开销

2️⃣ 音乐其实吃得很少

哪怕：

FLAC 24bit / 96kHz

≈ 3–5 Mbps

你就算只有 20 Mbps 上行，也能：

多设备播放

秒开无缓冲

3️⃣ 真正的瓶颈不是“播放”，是：

Navidrome 首次全库扫描

大量小文件 stat / readdir

这点上：

SSHFS < NFS < WireGuard + NFS

但都能用。

给你一个结论型答案

用 iperf3 + 反向端口
测出来的 Mbps
就是你这套方案的“物理极限”

只要：

≥ 20 Mbps：放心用

≥ 50 Mbps：体验接近本地

≥ 100 Mbps：你已经赢了

如果你愿意，下一步我可以帮你：

调 SSHFS 参数（cache / readahead）

或算一笔账：你这条链路最多能同时播几首 FLAC

网络不是玄学，测出来它就老实了。
# 自动挂载设置
-----
ok，使用autossh + sshfs可行，我想将本地autossh 和云服务器上的sshfs 做成自动化执行，重启不会丢失的配置， 要怎么做？

很好，这一步已经从“能跑”迈进“像服务一样活着”。
目标很清晰：断电、重启、网络抖动，都能自己爬起来。这正是 systemd 的舞台。

下面给你一套生产级、最少魔法、最好排障的方案：

本地：autossh 常驻反向隧道

AWS：sshfs 开机自动挂载 + 掉线自动重连
两边互不耦合，但能一起复活。

一、本地 PC（WSL2 Ubuntu）：autossh → systemd
1️⃣ 准备免密登录（必须）

本地：

ssh-keygen -t ed25519
ssh-copy-id ubuntu@AWS公网IP


确保：

ssh ubuntu@AWS公网IP


不需要密码。

2️⃣ 创建 systemd 服务
sudo nano /etc/systemd/system/autossh-reverse.service


内容（可直接用）：

[Unit]
Description=AutoSSH reverse tunnel to AWS
After=network-online.target
Wants=network-online.target

[Service]
Environment="AUTOSSH_GATETIME=0"
ExecStart=/usr/bin/autossh \
  -M 0 \
  -N \
  -o "ServerAliveInterval=15" \
  -o "ServerAliveCountMax=3" \
  -o "ExitOnForwardFailure=yes" \
  -R 10022:localhost:22 \
  ubuntu@AWS公网IP
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target

3️⃣ 启动并设为开机自启
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable --now autossh-reverse


验证：

systemctl status autossh-reverse

二、AWS（Ubuntu）：sshfs → systemd automount

重点思想：

不用 mount 单元直接挂载，而是用 automount
用到的时候才连，断了还能再拉。

1️⃣ 安装 sshfs
sudo apt install -y sshfs
sudo mkdir -p /mnt/music

2️⃣ 创建 mount 单元
sudo nano /etc/systemd/system/mnt-music.mount


内容：

[Unit]
Description=Mount music via SSHFS
After=network-online.target
Wants=network-online.target

[Mount]
What=user@localhost:/mnt/d/Music
Where=/mnt/music
Type=fuse.sshfs
Options=_netdev,ro,allow_other,IdentityFile=/home/ubuntu/.ssh/id_ed25519,Port=10022,reconnect,ServerAliveInterval=15,ServerAliveCountMax=3

[Install]
WantedBy=multi-user.target


⚠️ 注意：

user 是你本地 WSL 的用户名

key 路径要真实存在

本地音乐路径按你实际改

3️⃣ 配 automount（关键）
sudo nano /etc/systemd/system/mnt-music.automount


内容：

[Unit]
Description=Automount music directory

[Automount]
Where=/mnt/music

[Install]
WantedBy=multi-user.target

4️⃣ 启用
sudo systemctl daemon-reload
sudo systemctl enable --now mnt-music.automount


测试：

ls /mnt/music

三、Navidrome（Docker）小坑提醒
Docker Compose 推荐写法
volumes:
  - /mnt/music:/music:ro


并且：

depends_on:
  - nothing


不要让 Navidrome 等 mount
systemd automount 会在第一次访问时拉起。

四、重启级测试（必做）
顺序测试

1️⃣ 重启本地 PC
2️⃣ 确认 autossh 起：

systemctl status autossh-reverse


3️⃣ AWS 重启：

reboot


4️⃣ 登录 AWS：

ls /mnt/music


如果能看到文件，你这套系统已经“有生命”。

五、工程师视角的几个关键判断

autossh ≠ 守护进程 → systemd 才是

sshfs ≠ 稳定挂载 → automount 才稳

不要用 /etc/fstab（你会后悔）

最后的结论

你现在这套架构是：

本地主动出站 → AWS 常驻入口 → 文件系统级透明访问 → Docker 服务消费

这已经是：

没公网 IP

没改路由器

成本 ≈ 0

能做到的最优解之一。

如果你想继续打磨，我可以下一步帮你：

给 sshfs 上缓存 / read-ahead 调优

或把 Navidrome 扫描和挂载失败做成健康检查

或设计“本地离线时自动切换 AWS 小缓存库”

系统开始像系统了。