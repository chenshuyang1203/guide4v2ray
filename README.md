# 安装vpn
#### 在雨雀上写好了，但是涉嫌违规无法分享，移动到github上
## 安装v2ray软件
官方文档是`https://www.v2ray.com/chapter_00/install.html`,不过按照上面链接，执行安装脚本。不过v2ray 的官方脚本在执行安装时会提示被**discard**，提示迁移到新的安装脚本`fhs-install-v2ray`。

因此可以直接从下面的链接文档开始。
- https://github.com/v2fly/fhs-install-v2ray

我用的是一台ubuntu云服务器，先下载安装shell脚本。

`wget https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh`

执行脚本会安装v2ray并创建服务。

`sudo bash ./install-release.sh`

先执行一次手动启动v2ray服务，并查看确保服务已经启动
```
sudo systemctl enable v2ray
sudo systemctl start v2ray
sudo systemctl status v2ray
```
从**status**的输出中可以看到v2ray安装的位置和配置文件默认保存的位置

`/usr/local/bin/v2ray -config /usr/local/etc/v2ray/config.json`

## 提前准备
云服务器需要有一个公网IP，同时最好有一个域名，需要解析到自己的公网IP。云服务提供的DNS解析也可以使用，毕竟https可以自签名，v2ray客户端指定允许TLS不安全就可以了。按照下面操作可以生成网站的https证书。
```
sudo openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout /tmp/private.key -out /tmp/server.crt
```
其中**private.key**是网站的私钥，配置到`keyFile(v2ray)`或`ssl_certificate_key(nginx)`，**server.crt**是网站的证书，配置到`certificateFile(v2ray)`或`ssl_certificate(nginx)`。

## 配置v2ray
#### 我尝试过两种模式，vmess和websocket(ws)，ws的方式会麻烦一点，不过它的伪装效果更好。
### Vmess模式
修改配置文件`/usr/local/etc/v2ray/config.json`后重启，修改时可能需要`sudo`特权，内容如下：

PS: **log**可以写到任意路径，当客户端访问失败或者不能连接，我们可以查看**access.log**或者**error.log**进行诊断, inbounds监听443端口是为了伪装成网站服务，直接影响是服务器没法在nginx下部署https网站，有必要的话可以考虑**WebSocket模式**，由nginx转发给它，这就是ws模型的伪装方式。配置中的**id**可以找一个uuid生成器随机做一个。
```
{
  "log": {
    "loglevel": "warning", 
    "access": "/var/log/v2ray/access.log",  
    "error": "/var/log/v2ray/error.log"
  },
  "inbounds": [
    {
      "port": 443, 
      "protocol": "vmess",    
      "settings": {
        "clients": [
          {
            "id": "304c998c-04ed-4788-a621-17675c1bc871",  
            "alterId": 64
          }
        ]
       },
      "streamSettings": {
        "network": "tcp",
        "security": "tls",
        "tlsSettings": {
          "certificates": [
            {
              "certificateFile": "/etc/v2ray/v2ray.crt", 
              "keyFile": "/etc/v2ray/v2ray.key" 
            }
          ]
          }
        }
      }
    ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
```

客户端采用**shadowsocket**，这里需要配置的有：
- Host => 自己的域名或者公网IP
- 端口 => 443
- UUID => 304c998c-04ed-4788-a621-17675c1bc871
- 额外ID => 64  
- 算法 => auto
- TLS => 是
- 允许不安全 => 是
- 输方式 => none即可（里面有vmess选项）

### WebSocket模式
和**vmess**一样，我们要先配置`config.json`文件, 和纯vmess模式的区别是ssl证书不用配置在v2ray这侧，具体配置如下：

```
{
  "log": {
    "loglevel": "warning",
    "access": "/var/log/v2ray/access.log",
    "error": "/var/log/v2ray/error.log"
  },
  "inbounds": [
    {
      "port": 10086,
      "listen":"127.0.0.1",
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "304c998c-04ed-4788-a621-17675c1bc871",
            "alterId": 0
          }
         ]
       },
      "streamSettings": {
        "network": "ws",
        "wsSettings": {
        "path": "/ray"
      }
     }
    }
   ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
```
这端口10086一看就是国人所为，再重启一下v2ray。

`sudo systemctl restart v2ray`

可以通过netstat 看一下10086端口已经在监听状态了
```
sudo netstat -anp | grep v2ray
tcp6       0      0 :::10086                :::*                    LISTEN      18601/v2ray 
```

v2ray工作起来之后，需要v2ray前面再配置一个nginx的代理服务, 如下：

站点证书是配置在nginx这里的，nginx收到用户数据后转发给后面的v2ray，需要注意**proxy_pass**转发后端v2ray的**10086**端口。修改成功后先通过`sudo nginx -T`检查一下配置文件是否有错误，没问题再`sudo service nginx restart`重启nginx服务。
```
server {
  listen 443 ssl;
  server_name 自己的域名或者公网IP;
  ssl_certificate       /etc/v2ray/v2ray.crt;
  ssl_certificate_key   /etc/v2ray/v2ray.key;
  ssl_protocols         TLSv1.2;
  ssl_ciphers           ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
  ssl_prefer_server_ciphers off;
  
  # 遇到 /ray 以后才转发v2ray，一般情况下访问网站就是一个普通站点。
  location /ray {
    proxy_redirect off;
    proxy_pass http://127.0.0.1:10086;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    # Show real IP in v2ray access.log
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
```

客户端的设置，在shadowsocket里面要指定ws模式和转发的目录/ray（这个目录可以自己定义，总之要保证nginx里面的location转发规则和客户端一致）
- Host => 自己的域名或者公网IP
- 端口 => 443
- UUID => 304c998c-04ed-4788-a621-17675c1bc871
- 额外ID => 64  
- 算法 => auto
- TLS => 是
- 允许不安全 => 是
- 输方式 => websocket
 - 名称 => websocket
 - Host => 自己的域名（在v2ray 的config.json里面配置过）
 - 路径 => /ray （ 在v2ray 的config.json里面配置过）
 
 ## 故障排查
 发生任何连接不到的问题都可以通过查看下面的日志来排查，例如shadowrocket访问不到nginx或者流量无法转发等等。
 ```
 # nginx
 /var/log/nginx/access.log
 /var/log/nginx/error.log
 # v2ray
 /var/log/v2ray/access.log
 /var/log/v2ray/error.log
 ```
