nginx必须为443端口，保证443端口不被占用

1.购买vps
  hostwinds挑选vps
  
2.vps配置：
  Linux系统选择CentOS7
  更新系统：
  yum remove -y epel-release
  yum install -y epel-release
  yum update -y
  yum install -y vim git zip unzip
  
  
3.域名申请：
  在godaddy进行域名申请，购买域名1年的使用权，同样是选最便宜且不需要实名认证的域名  https://dcc.godaddy.com
  首年域名，明年可能会进行替换
  
  配置域名解析：
  CloudFlare CDN
  CloudFlare 是一家全球知名的 CDN 服务商，并且提供了免费的 CDN 套餐，还不限流量，所以我们完全不需要花一分钱就能使用它的 CDN 服务
  
  进行域名解析，type选择A，Name填写为：@和www，地址填写vps地址，status状态打开
  
  修改nameserves，cdn会提供两个域名服务器地址，登录godaddy进行域名服务器地址替换
  路径：DNS——域名服务器——修改，删除所有非Clound提供的域名服务器
  
  tips： cloud flare的SSL/TLS设置，一定要将设置改成FULL
  完成后访问自己的域名进行测试
  
  vps端新建目录
  mkdir -p /www/root
  
  增加index.html
  vim /www/root/index.html
  
  将腾讯404页面添加其中
  <!DOCTYPE html>
  <html>
  <head>
    <title>404</title>
    <meta http-equiv="Content-Type" content="text/html" charset="UTF-8">
    <script type="text/javascript" src="//qzonestyle.gtimg.cn/qzone/hybrid/app/404/search_children.js" charset="utf-8" homePageUrl="https://www.example.cc/" homePageName="回到我的主页"></script>
  </head>
  </html>
  
 4.安装配置Nginx
   安装Nginx
   yum install -y nginx
   配置Nginx
   vim /etc/nginx/nginx.conf
   配置文件内容：
   
    user nginx;
    worker_processes auto;
    error_log /dev/null;
    pid /run/nginx.pid;
    include /usr/share/nginx/modules/*.conf;

    events {
        worker_connections 1024;
    }

    http {
        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';
        access_log          off;
        sendfile            on;
        tcp_nopush          on;
        tcp_nodelay         on;
        keepalive_timeout   65;
        types_hash_max_size 2048;

        include             /etc/nginx/mime.types;
        default_type        application/octet-stream;

        include /etc/nginx/conf.d/*.conf;

        server {
            listen       80 default_server;
            listen       [::]:80 default_server;
            server_name  二级域名 www.二级域名;
            root         /www/root;
            index        index.html index.htm;

            location / {
            }
        }
    }
    
    
    
   启动Nginx服务
      systemctl enable nginx
      systemctl start nginx
      查看运行状态
      systemctl status nginx
  
  5.安装V2Ray
    bash <(curl -L -s https://install.direct/go.sh)
    安装完成后自动启动，这里先把它给停了
    systemctl stop v2ray
    选用acme.sh申请证书
    安装acme.sh
    curl  https://get.acme.sh | sh
    安装完成后执行下
    source /root/.bashrc
    申请证书
    acme.sh --issue -d example.cc -d www.example.cc --webroot /www/root/ -k ec-256
    将证书安装到目录
    这里将证书放到/etx/v2ray目录下
    acme.sh --installcert -d example.cc -d www.example.cc --key-file /etc/v2ray/v2ray.key --fullchain-file /etc/v2ray/v2ray.crt --ecc --reloadcmd  "service nginx force-reload && systemctl restart v2ray"
    这行命令除了将证书放到指定目录下外，还会自动创建crontab定时任务，后面引号里的命令是定时任务更新证书后执行的命令
    配置Nginx支持Https访问
    
    user nginx;
    worker_processes auto;
    error_log /dev/null;
    pid /run/nginx.pid;

    include /usr/share/nginx/modules/*.conf;

    events {
        worker_connections 1024;
    }

    http {
        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  off;

    server_tokens       off;
    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;

    # Http Server，强制跳转Https
    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  example.cc www.example.cc;
        rewrite      ^(.*)$ https://www.example.cc$1 permanent;
    }

    # Https Server
    server {
        listen       443 ssl http2 default_server;
        listen       [::]:443 ssl http2 default_server;
        server_name  www.example.cc;
        root         /www/root;
        index        index.html index.htm;

        ssl_certificate "/etc/v2ray/v2ray.crt";
        ssl_certificate_key "/etc/v2ray/v2ray.key";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;

        include /etc/nginx/default.d/*.conf;

        location / {
        }

        # 反向代理V2Ray
        location /wss {
            proxy_redirect off;
            proxy_pass http://127.0.0.1:10443;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $http_host;
        }

        error_page 404 /404.html;
        location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
    }
    
  ——————此处进行HTTPS的强制跳转，页面会报错：该网站进行了过多的重定向
  猜测是强制跳转的原因，总之将强制跳转Https部分注释掉了
  
  配置完之后重新启动Nginx
  systemctl restart nginx
  
  访问你的二级域名，如果正确显示公益404则已经配置正确。
  
  配置V2Ray并启动
  
  为V2Ray生成一个UUID：https://www.uuidgenerator.net/ 自动生成
  为mtproto生成一个密钥
  使用linux系统创建伪随机数作为密钥
  $ head -c 16 /dev/urandom | xxd -ps
  
  修改/etc/v2ray/config.json
  
  完整配置文件如下：
  
    {
      "log": {
          "loglevel": "none",
          "access": "/var/log/v2ray/access.log",
          "error": "/var/log/v2ray/error.log"
      },
      "inbounds": [
          {
              "port": 10443,
              "listen": "127.0.0.1",
              "protocol": "vmess",
              "settings": {
                  "clients": [
                      {
                          "id": "27e0efcc-8e13-fef1-9e82-febebc469b2b",
                          "alterId": 64
                      }
                  ]
              },
              "streamSettings": {
                  "network": "ws",
                  "wsSettings": {
                      "path": "/wss"
                  }
              }
          },
          {
              "tag": "tg-in",
              "port": 8080,
              "protocol": "mtproto",
              "settings": {
                  "users": [
                      {
                          "secret": "80e2e037610bac1444ac02979364f666"
                      }
                  ]
              }
          }
      ],
      "outbounds": [
          {
              "protocol": "freedom",
              "settings": {}
          },
          {
              "protocol": "blackhole",
              "settings": {
                  "response": {
                      "type": "none"
                  }
              },
              "tag": "blocked"
          },
          {
              "tag": "tg-out",
              "protocol": "mtproto",
              "settings": {}
          }
      ],
      "routing": {
          "domainStrategy": "IPOnDemand",
          "settings": {
              "rules": [
                  {
                      "type": "field",
                      "ip": [
                          "geoip:private"
                      ],
                      "outboundTag": "blocked"
                  },
                  {
                      "type": "field",
                      "inboundTag": [
                          "tg-in"
                      ],
                      "outboundTag": "tg-out"
                  }
              ]
          }
      }
  }
  ——————outbounds中将
        {
            "protocol": "freedom",
            "settings": {}
        }
        以外的东西全部删掉才可正常进行访问，搜了很久才查到，至于原因，尚未可知
        
  开启开机启动并启动

  systemctl enable v2ray
  systemctl start v2ray
  验证是否运行
  systemctl status v2ray
  6.客户端配置
    PC用的支持国内外分流
    
    {
    "inbounds": [
        {
            "port": 1087,
            "listen": "127.0.0.1",
            "protocol": "http",
            "settings": {
                "allowTransparent": true
            }
        },
        {
            "port": 本地端口,
            "listen": "127.0.0.1",
            "protocol": "socks",
            "domainOverride": [
                "tls",
                "http"
            ],
            "settings": {
                "auth": "noauth",
                "udp": true
            }
        }
    ],
    "outbounds": [
        {
            "protocol": "vmess",
            "settings": {
                "vnext": [
                    {
                        "address": "域名",
                        "port": 443,
                        "users": [
                            {
                                "id": "uuid",
                                "alterId": 64,
                                "security": "auto"
                            }
                        ]
                    }
                ]
            },
            "streamSettings": {
                "network": "ws",
                "wsSettings": {
                    "path": "/wss"
                },
                "security": "tls"
            },
            "mux": {
                "enabled": false,
                "concurrency": 8
            },
            "tag": "proxy"
        },
        {
            "protocol": "freedom",
            "settings": {},
            "tag": "direct"
        },
        {
            "protocol": "blackhole",
            "settings": {},
            "tag": "block"
        }
    ],
    "log": {
        "loglevel": "none",
        "access": "D:/v2ray_access.log",
        "error": "D:/v2ray_error.log"
    },
    "dns": {
        "hosts": {
            "example.com": "127.0.0.1"
        },
        "servers": [
            "223.5.5.5",
            "8.8.8.8",
            "localhost"
        ]
    },
    "routing": {
        "strategy": "rules",
        "settings": {
            "domainStrategy": "IPIfNonMatch",
            "rules": [
                {
                    "type": "field",
                    "domain": [
                        "dropbox",
                        "github",
                        "google",
                        "instagram",
                        "netflix",
                        "pinterest",
                        "pixiv",
                        "tumblr",
                        "twitter",
                        "domain:facebook.com",
                        "domain:fbcdn.net",
                        "domain:fivecdm.com",
                        "domain:ggpht.com",
                        "domain:gstatic.com",
                        "domain:line-scdn.net",
                        "domain:line.me",
                        "domain:medium.com",
                        "domain:naver.jp",
                        "domain:pximg.net",
                        "domain:t.co",
                        "domain:twimg.com",
                        "domain:youtube.com",
                        "domain:ytimg.com"
                    ],
                    "outboundTag": "proxy"
                },
                {
                    "type": "field",
                    "ip": [
                        "125.209.222.0/24",
                        "149.154.167.0/24",
                        "149.154.175.0/24",
                        "91.108.56.0/24"
                    ],
                    "outboundTag": "proxy"
                },
                {
                    "type": "field",
                    "domain": [
                        "cctv",
                        "geosite:cn",
                        "umeng",
                        "domain:apple.com",
                        "domain:crashlytics.com",
                        "domain:icloud.com",
                        "domain:ixigua.com",
                        "domain:pstatp.com",
                        "domain:snssdk.com",
                        "domain:toutiao.com"
                    ],
                    "outboundTag": "direct"
                },
                {
                    "type": "field",
                    "ip": [
                        "geoip:cn",
                        "geoip:private"
                    ],
                    "outboundTag": "direct"
                },
                {
                    "type": "field",
                    "domain": [
                        "domain:doubleclick.net"
                    ],
                    "outboundTag": "block"
                }
            ]
        }
    }
    }

// chorme浏览器
  chorme配置：
  使用SwitchyOmega进行代理管理
  SwitchyOmega情景模式只保留需要的一个，端口和v2client端口保持一致即可
  
  另：v2rayN也可以进行配置，实测可用
  
  v2客户端版本：4.20.0
  v2服务器端版本：4.23.3
  
  
  参考文献：
  https://guide.v2fly.org/basics/vmess.html（需翻）
  https://alecthw.github.io/2018/11/18/v2ray-tutorial/
  https://blog.sprov.xyz/2019/03/11/cdn-v2ray-safe-proxy/
  小工具汇总：
  v2配置生成器：
  https://tools.sprov.xyz/v2ray/
  uuid生成器：
  https://www.uuidgenerator.net/
  测试vps是否被ban：
  http://ping.chinaz.com/
