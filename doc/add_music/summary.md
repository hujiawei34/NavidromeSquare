# 问题
使用navidrome部署在云服务器，但是云服务器的硬盘空间很小，购买云存储贵且不长久，想要使用本地的硬盘中的音乐，但是本地网络是家庭宽带，没有公网ip，本文档指导如何在只有一个带公网ip的云服务器上，使用本地PC的音乐
# 解决办法
## 物料
本地PC: 1000M家庭宽带+WIN10+WSL2+ubuntu
云服务器：aliyun服务器 新用户优惠的68/年套餐： 2v/2G/200M的国内服务器

## 限制
本地PC要与云服务器的带宽足够大，我这个是1000M家庭宽带+200M的aliyun服务器（68/年）
## 思路
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
## 操作步骤
本地pc安装
```bash
apt install autossh open-ssh  
iperf3 -s 
autossh -N -R 10022:localhost:22 -R 15201:localhost:5201 root@8.138.131.24 
```

云服务器执行

```bash
apt install sshfs iperf3
ssh -p 10022 root@localhost
# 注意这边输入本地PC中的wsl的密码
iperf3 -c localhost -p 15201
# 测试下载（其实是本地的上传）速度有20M/s
sshfs -p 10022 root@localhost:/mnt/d/Music /mnt/music
ll /mnt/music
```
### 设置本地后台服务
因为本地wsl 的autossh 一旦断连，云服务器上就要再次执行`sshfs -p 10022 root@localhost:/mnt/d/Music /mnt/music` 来挂载，比较麻烦，所以先在本地将autossh设置成systemctl 的service
当前先不设置云服务器的automount

1. 准备免密登录（必须）

本地：
```bash
ssh-keygen -t ed25519
ssh-copy-id ubuntu@AWS公网IP
```

确保：
```bash
ssh ubuntu@AWS公网IP
```

不需要密码。

2. 创建 systemd 服务
```bash
sudo nano /etc/systemd/system/autossh-reverse.service
```


内容（可直接用）：
```ini
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
```

3️⃣ 启动并设为开机自启
```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable --now autossh-reverse
```

验证：
```bash
systemctl status autossh-reverse


```

```bash
ssh aliyun
ll /mnt/music
#如果没有音乐文件，需要重新挂载
ssh -p 10022 localhost
ll /mnt/d/music/
exit
#以下命令要在aliyun上执行
sshfs -p 10022 root@localhost:/mnt/d/Music/music /mnt/music
```