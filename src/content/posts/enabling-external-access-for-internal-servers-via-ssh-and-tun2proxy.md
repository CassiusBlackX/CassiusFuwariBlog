---
title: 通过 SSH 和 tun2proxy 实现内网服务器访问外部网络
published: 2026-04-23
description: '通过ssh -R把本机代理端口开放给内网服务器，并通过tun2proxy虚拟网卡把服务器所有非内网流量转发到本机，实现内网服务器的上网'
image: ''
tags: ["linux", "web"]
category: 'deployment'
draft: false 
lang: ''
---

# motivation
在使用服务器的场景中，很多时候服务器没办法正常/稳定连接上诸如github, crates.io之类的重要网站。
虽然有时候可以通过直接在服务器上部署一个代理，并通过`https_proxy`或者`all_proxy`之类的方式临时让服务器能够稳定连接上这些网站。
但是存在难以解决的问题:

1. 不是所有的应用都遵循环境变量`https_proxy`或者`all_proxy`
2. 服务器集群的路由器/防火墙可能会直接丢弃所有非白名单IP，导致根本不可能在服务器上部署代理
3. 本机到服务器的连接是单向的（没办法通过让服务器直接走本机的代理）

针对这样的情况，只能把本机的代理端口提供给服务器，通过ssh端口转发的方式，让服务器可以利用到本机的代理，解决了问题3和2

通过tun虚拟网卡的方式，把所有非内网、非SSH的流量都转发到服务器的代理端口（也即本机的代理端口），
这样不管应用是否遵循`https_proxy`之类的环境变量，都一定会经过本机的代理，解决了问题1。

# env setup
## ssh端口转发
通过命令
```sh
# server_available_port: 在服务器上用于做代理的端口，只要是没被占用的都可以
# local_proxy_port: 本机的代理监听的端口
# 其实两个端口都一致也不会有影响
ssh -R $server_available_port:localhost:$local_proxy_port user@server_ip
```
连接上服务器即可。

在服务器上使用命令
```sh
ss -tuln | grep $server_proxy_port
```
如果真的被监听了，说明端口转发成功

## tun2proxy的安装
直接使用`cargo install`就可以安装。

因为后续需要使用`sudo`来运行tun2proxy-bin，所以最好创建一个软链接，把`~/.cargo/bin/tun2proxy-bin`链接到`/usr/local/bin`或者其它在sudo下`$PATH`中的路径下

# 编写启动脚本tun2proxy
脚本sample如下
```sh
#!/usr/bin/env bash
# tun2proxy.sh
set -euo pipefail

ACTION="${1:-}"

TUN_DEV="tun0"
PROXY_URL="socks5://127.0.0.1:server_proxy_port"

# 这些地址/网段不进 tun，避免自环和断 SSH
BYPASS_LIST=(
  "your_proxy_server_real_ip"  # 防止出现回环
  "your_client_ssh_real_ip"  # 防止ssh也走localhost，ssh就断连了
  "192.168.100.0/24" # 其它内网IP
  "192.168.122.0/24"
)

UP_LOG="/tmp/tun2proxy-up.log"
PID_FILE="/tmp/tun2proxy.pid"
sudo -v

build_bypass_args() {
  local args=()
  local item
  for item in "${BYPASS_LIST[@]}"; do
    args+=( --bypass "$item" )
  done
  printf '%q ' "${args[@]}"
}

up() {
  echo "[1/4] Checking existing tun2proxy ..."
  if pgrep -af "tun2proxy-bin.*--tun ${TUN_DEV}" >/dev/null 2>&1; then
    echo "tun2proxy already appears to be running for ${TUN_DEV}"
    status
    return 0
  fi

  echo "[2/4] Removing stale ${TUN_DEV} if any ..."
  if ip link show "$TUN_DEV" >/dev/null 2>&1; then
    sudo ip link del "$TUN_DEV" 2>/dev/null || true
  fi

  echo "[3/4] Starting tun2proxy ..."
  # shellcheck disable=SC2046
  sudo tun2proxy-bin \
    --tun "$TUN_DEV" \
    --proxy "$PROXY_URL" \
    --dns virtual \
    --setup \
    $(build_bypass_args) \
    >"$UP_LOG" 2>&1 &

  # Get the PID of the background process
  echo $! > "$PID_FILE"
  
  sleep 2

  # 测试是否真的能连接成功，可以不用这一段
  echo "[4/4] Testing connectivity ..."
  if curl -4 --max-time 10 -I https://www.google.com >/dev/null 2>&1; then
    echo "tun2proxy is up. Test succeeded."
  else
    echo "tun2proxy started, but curl test failed."
    echo
    echo "Check these:"
    echo "  curl --socks5 127.0.0.1:10808 -I https://www.google.com"
    echo "  ip route"
    echo "  cat /etc/resolv.conf"
    echo "  pgrep -af tun2proxy-bin"
    exit 2
  fi

  echo
  status
}

down() {
  echo "[1/3] Stopping tun2proxy ..."
  if [ -f "$PID_FILE" ]; then
    sudo kill "$(cat "$PID_FILE")"
    rm "$PID_FILE"
    echo "tun2proxy stopped successfully."
  else
    echo "No tun2proxy PID file found, process might already be stopped."
  fi
  sleep 1

  echo "[2/3] Deleting ${TUN_DEV} ..."
  if ip link show "$TUN_DEV" >/dev/null 2>&1; then
    sudo ip link del "$TUN_DEV" 2>/dev/null || true
  fi

  echo "[3/3] Done."
  echo "Note: if tun2proxy --setup modified /etc/resolv.conf or routes, deleting ${TUN_DEV} usually clears tun routes."
  echo "If DNS still looks wrong, check: cat /etc/resolv.conf"
}

status() {
  echo "=== process ==="
  pgrep -af "tun2proxy-bin.*--tun ${TUN_DEV}" || echo "not running"

  echo
  echo "=== interface ==="
  ip a show "$TUN_DEV" 2>/dev/null || echo "${TUN_DEV} not found"

  echo
  echo "=== routes ==="
  ip route 

  echo
  echo "=== resolv.conf ==="
  cat /etc/resolv.conf || true
}

case "$ACTION" in
  up) up ;;
  down) down ;;
  status) status ;;
  *)
    echo "Usage: $0 {up|down|status}"
    exit 1
    ;;
esac
```

# 随地启动
如果`~/.local/bin`在当前用户的`$PATH`中，可以给脚本赋予执行权限后移动到该路径下，然后就可以在任何地方通过
```sh
tun2proxy.sh up
tun2proxy.sh down
tun2proxy.sh status
```
命令来快速的启动或关闭了。