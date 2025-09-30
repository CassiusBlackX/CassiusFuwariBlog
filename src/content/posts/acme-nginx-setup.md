---
title: acme & nginx setup on a new server
published: 2025-03-18
description: 'how to setup acme and nginx on a new server to deploy personal site'
image: ''
tags: ["linux", "network", "nginx", "acme.sh"]
category: 'deployment'
draft: false 
lang: ''
---
# 在vps上通过acme.sh自动申请证书，并用nginx作反向代理
本篇博客记录一下个人在自己的vps服务器上通过acme.sh获取证书，并配置nginx反向代理的过程。

## 准备
### 安装acme.sh
安装acme.sh非常简单，只要照着github上面的说明走就可以了。
```shell
curl https://get.acme.sh | sh -s email=my@example.com
```
一般来说，都是把acme.sh直接安装到当前用户的用户根目录下。
### 安装nginx
在Ubuntu上安装nginx直接通过apt就可以安装
```shell
sudo apt install nginx
```
### 域名
需要购买一个自己的域名，只要有了根域名后就可以在cloudflare上创建多个子域名，干各种事情了。

## 为自己的域名申请证书
假设在cloudflare上，为自己的域名解析添加了一个子域名test.example.com到自己的服务器xx.xx.xx.xx，类型为`A`，开启cloudflare代理。

### 创建一个随便的nginx配置文件
首先在`/etc/nginx/sites-available`中随便创建一个nginx的配置文件（真的随便创建一个就好，不影响后面的）
```nginx
# /etc/nginx/sites-available/test
server {
    listen 80;
    server_name test.example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
}
```
然后创建一个软链接，把这个文件链接到`/etc/nginx/sites-enabled`下面去
```shell
ln -s /etc/nginx/sites-available/test /etc/nginx/sites-enabled/test
```
接下来，最好检查一下创建的这个test配置文件是否有什么语法问题，使用以下命令来检查
```shell
# check if any syntax error in nginx configuration files
nginx -t
# reload nginx process
service reload nginx
```

### 使用acme.sh针对自己的域名和配置文件申请证书
使用以下命令来为域名test.example.com申请一个方便nginx使用的证书
```shell
acme.sh --issue --nginx -d test.example.com
```
如果输出中没有error，就说明申请成功。

## 修改nginx配置文件
目前只碰到过两种类型的网页。一种是诸如hexo博客这种的静态网页，直接会生成好文件。另一种就是像frps dashboard或者galene这样的直接在某一端口号上的进程。
两种网页的nginx配置稍微有点不同。
### 静态网页
```nginx
# HTTP 配置 - 监听 80 端口并重定向所有流量到 HTTPS
server {
    listen 80;
    server_name test.example.com;

    root /var/www/html;  # 默认生成的静态网页放在/var/www/html下
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    # 重定向 HTTP 到 HTTPS
    return 301 https://$host$request_uri;
}

# HTTPS 配置 - 监听 443 端口并启用 SSL
server {
    listen 443 ssl;
    server_name test.example.com;

    ssl_certificate /etc/nginx/ssl/test_fullchain.crt;
    ssl_certificate_key /etc/nginx/ssl/test.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1h;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```
其中，把`server_name`修改为自己的对应的域名，然后需要注意`ssl_certificate`和`ssl_certificate_key`后面的这两个路径，在之后的acme.sh的install命令中会要用到。一般也就顺便放在`/etc/nginx`中

### 进程类网页
```nginx
server {
    listen 80;
    server_name test.example.com;

    # HTTP 到 HTTPS 的重定向
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name test.example.com;

    # SSL 证书配置
    ssl_certificate /etc/nginx/ssl/test_fullchain.crt;
    ssl_certificate_key /etc/nginx/ssl/test.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1h;

    # 反向代理 Galene Web 服务器
    location / {
        proxy_pass http://127.0.0.1:8443;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket 代理配置（用于实时视频和音频）
    location /ws {
        proxy_pass http://127.0.0.1:8443;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
    }
}
```

## 安装证书到指定位置
需要记住之前使用到的`ssl_certificate`和`ssl_certificate_key`中的路径，接下来就是要把cert和key安装到这个目录下。
```shell
acme.sh --install-cert -d test.example.com --key-file /etc/nginx/tls/test.key --fullchain-file /etc/nginx/tls/test_fullchain.crt --cert-file /etc/nginx/tls/test.crt --reloadcmd "service nginx reload"
```
其中，`cert-file`其实是只有Apache才需要的，nginx并不需要，所以也可以选择不生成cert-file，只要确保有fullchain-file和key-file就可以了。

## 测试
重载nginx服务一遍，同时确保ping自己的域名的时候能够解析出服务器的IP。然后就可以通过浏览器登录了。


# 源码编译安装nginx with h3 compatible
源码编译nginx，包括stream模块，这样nginx就可以直接转发tcp流量。

同时支持h3(quic) 为此,不使用系统自带的openssl(因为版本过老 3.0.13,不完整支持quic),转而使用boringssl

## 安装boringssl
### 克隆boringssl
```sh
git clone https://boringssl.googlesource.com/boringssl
```
### 编译boringssl
```sh
cmake -G Ninja -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

## 编译nginx
### 配置nginx编译参数
```sh
./configure \
    --prefix=/usr/local/nginx \
    --conf-path=/etc/nginx/nginx.conf \
    --sbin-path=/usr/sbin/nginx \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --pid-path=/var/run/nginx.pid \
    --lock-path=/var/lock/nginx.lock \
    --modules-path=/usr/lib/nginx/modules \
    --with-http_ssl_module \
    --with-stream \
    --with-stream_ssl_module \
    --with-stream_ssl_preread_module \
    --with-threads \
    --with-file-aio \
    --with-http_v2_module \
    --with-http_gzip_static_module \
    --with-http_realip_module \
    --with-http_stub_status_module \
    --with-http_v3_module \
    --with-cc-opt="-O3 -march=native -pipe -I ../boringssl/include" \
    --with-ld-opt="-L ../boringssl/build -lssl -lcrypto -lstdc++"
```
### 编译并安装nginx
```sh
make -j$(nproc)
make install
```

### 测试nginx
```sh
nginx -t
nginx -V
```