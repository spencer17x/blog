---
title: 利用rsshub+rsstt实现订阅推特用户并推送消息到telegram
date: 2024-03-11 10:23:22
tags:
  - Others
---

我们要实现一个订阅推特用户并将消息推送到 telegram 上，这里我们用 rss 的方式来实现，所以决定采用 [rsshub](https://github.com/DIYgod/RSSHub) 来生成订阅源，再用 [rsstt](https://github.com/Rongronggg9/RSS-to-Telegram-Bot) 来实现订阅对应用户的 rss 源。这样就可以将订阅的推特用户消息推送到 telegram 频道或者用户上。

>  由于我买的是国内的服务器，但是访问 推特/telegram 需要科学上网。所以我们需要先在服务器上配置科学上网。

>  当然，如果你的服务器可以直接访问国外的网站，那就不需要配置科学上网这一步了。同时后续的 docker-compose.yml 文件里的 proxy 代理相关配置也不需要了。

## 配置科学上网

相关链接: https://github.com/spencer17x/clash-for-linux

1. 克隆 clash 项目

```shell
$ git clone https://github.com/spencer17x/clash-for-linux
```

2. 进入到项目目录，编辑 `.env` 文件，修改变量 `CLASH_URL` 的值。

> **注意：** `.env` 文件中的变量 `CLASH_SECRET` 为自定义 Clash Secret，值为空时，脚本将自动生成随机字符串。

3. 启动程序

```shell
$ sudo bash start.sh

正在检测订阅地址...
Clash订阅地址可访问！                                      [  OK  ]

正在下载Clash配置文件...
配置文件config.yaml下载成功！                              [  OK  ]

正在启动Clash服务...
服务启动成功！                                             [  OK  ]

Clash Dashboard 访问地址：http://<ip>:9090/ui
Secret：xxxxxxxxxxxxx

请执行以下命令加载环境变量: source /etc/profile.d/clash.sh

请执行以下命令开启系统代理: proxy_on

若要临时关闭系统代理，请执行: proxy_off
```

```shell
$ source /etc/profile.d/clash.sh
$ proxy_on
```

4. 检查服务端口

```shell
$ netstat -tln | grep -E '9090|789.'
tcp        0      0 127.0.0.1:9090          0.0.0.0:*               LISTEN     
tcp6       0      0 :::7890                 :::*                    LISTEN     
tcp6       0      0 :::7891                 :::*                    LISTEN     
tcp6       0      0 :::7892                 :::*                    LISTEN
```

5. 检查环境变量

```shell
$ env | grep -E 'http_proxy|https_proxy'
http_proxy=http://127.0.0.1:7890
https_proxy=http://127.0.0.1:7890
```

以上步鄹如果正常，说明服务clash程序启动成功，现在就可以体验高速下载 github 资源了。

从 https://github.com/spencer17x/clash-for-linux/blob/master/start.sh#L163-L178 可以看出，clash 采用了 nohup 实现了系统后台运行程序。

6. 由于上述只完成了 clash 配置以及后台运行，但是每次登陆到服务器还要实现启动代理，所以需要配置启动代理。

```shell
# 在 ~/.bashrc 文件末尾添加一段 proxy_on 即可
$ vi ~/.bashrc
```

通过 ```curl https://www.google.com``` 即可测试科学上网是否配置成功，如果成功了即返回正常的数据，失败了则需要继续排查问题。

## rsshub+rsstt

相关链接: https://docs.rsshub.app/zh/install#docker-compose-%E9%83%A8%E7%BD%B2

这里使用是 docker-compose 来部署。

1. 新建个项目，命名为 rsshub-rsstt

```shell
$ mkdir rsshub-rsstt && cd rsshub-rsstt
```

2. 配置 rsshub

参考: https://docs.rsshub.app/zh/install#docker-compose-%E9%83%A8%E7%BD%B2

```shell
# 下载 docker-compose.yml
$ wget https://raw.githubusercontent.com/DIYgod/RSSHub/master/docker-compose.yml

# 检查有无需要修改的配置
$ vi docker-compose.yml  # 也可以是你喜欢的编辑器

# 创建 volume 持久化 Redis 缓存
$ docker volume create redis-data

# 启动
$ docker-compose up -d
```

docker-compose.yml: 参考后面最终的 docker-compose.yml

注: 172.17.0.1 是 docker0 虚拟网卡的主机 ip，在 Windows 和 mac 下机制不同所以不适用。由于我的服务器是 linux 的，所以 PROXY_URI 的 ip 为 172.17.0.1。mac 的需要改为 host.docker.internal。其他的根据自己的需求进行调整。

浏览器地址栏输入 http://ip:1200 可以查看 rsshub 服务是否部署成功。

输入 http://ip:1200/twitter/user/elonmusk 可以查看订阅源是否可以正常生成。

3. 配置 rsstt

```shell
$ vi docker-compose.yml
```

docker-compose.yml:

```yaml
version: '3.9'

services:
    rsshub:
        # two ways to enable puppeteer:
        # * comment out marked lines, then use this image instead: diygod/rsshub:chromium-bundled
        # * (consumes more disk space and memory) leave everything unchanged
        image: diygod/rsshub
        restart: always
        ports:
            - '1200:1200'
        environment:
            NODE_ENV: production
            CACHE_TYPE: redis
            REDIS_URL: 'redis://redis:6379/'
            PUPPETEER_WS_ENDPOINT: 'ws://browserless:3000'  # marked
            PROXY_URI: 'http://172.17.0.1:7890'
            TWITTER_USERNAME: xxx
            TWITTER_PASSWORD: xxx
        depends_on:
            - redis
            - browserless  # marked

    browserless:  # marked
        image: browserless/chrome  # marked
        restart: always  # marked
        ulimits:  # marked
          core:  # marked
            hard: 0  # marked
            soft: 0  # marked

    redis:
        image: redis:alpine
        restart: always
        volumes:
            - redis-data:/data

    warp-socks:
        image: monius/docker-warp-socks:latest
        privileged: true
        restart: always
        volumes:
            - /lib/modules:/lib/modules
        cap_add:
            - NET_ADMIN
            - SYS_MODULE
        sysctls:
            net.ipv6.conf.all.disable_ipv6: 0
            net.ipv4.conf.all.src_valid_mark: 1
        healthcheck:
            test: ["CMD", "curl", "-f", "https://www.cloudflare.com/cdn-cgi/trace"]
            interval: 30s
            timeout: 10s
            retries: 5

    rsstt:
        image: rongronggg9/rss-to-telegram:dev # stable image: rongronggg9/rss-to-telegram
        container_name: rsstt # need to be unique
        restart: unless-stopped
        volumes:
            - ./config:/app/config
        environment:
            - TOKEN=xxx # get it from @BotFather
            - MANAGER=xxx # get it from @userinfobot, can be a list (e.g., 1234567890;987654321)

            # ↓------ To disable sending via Telegraph, comment out this area ------↓ #
            # Get Telegraph API access tokens: https://api.telegra.ph/createAccount?short_name=RSStT&author_name=Generated%20by%20RSStT&author_url=https%3A%2F%2Fgithub.com%2FRongronggg9%2FRSS-to-Telegram-Bot
            # Refresh the page every time you get a new token.
            # If you have a lot of subscriptions, make sure to get at least 5 tokens.
            #                            ↓ Replace with your access tokens ↓
            - TELEGRAPH_TOKEN= 
              xxx 
              xxx 
              xxx 
              xxx 
              xxx
            # ↑------ To disable sending via Telegraph, comment out this area ------↑ #

            # Please read https://github.com/Rongronggg9/RSS-to-Telegram-Bot/blob/master/docs/advanced-settings.md for more details.
            # ↓------ Advanced settings ------↓ #
            #- ERROR_LOGGING_CHAT=-1001234567890  # default: the first MANAGER
            #- MULTIUSER=0  # default: 1
            #- CRON_SECOND=30  # 0-59, default: 0
            #- DATABASE_URL=postgres://username:password@host:port/db_name  # default: sqlite://path/to/config/db.sqlite3
            #- API_ID=1025907  # get it from https://core.telegram.org/api/obtaining_api_id
            #- API_HASH=452b0359b988148995f22ff0f4229750  # get it from https://core.telegram.org/api/obtaining_api_id
            #- IMG_RELAY_SERVER=https://wsrv.nl/?url=  # default: https://rsstt-img-relay.rongrong.workers.dev/
            #- IMAGES_WESERV_NL=https://t0.nl/  # default: https://wsrv.nl/
            #- USER_AGENT=Mozilla/5.0 (Android 12; Mobile; rv:68.0) Gecko/68.0 Firefox/96.0  # default: RSStT/2.2 RSS Reader
            #- IPV6_PRIOR=1  # default: 0
            - T_PROXY=http://172.17.0.1:7890  # Proxy used to connect to the Telegram API
            - R_PROXY=http://172.17.0.1:7890  # Proxy used to fetch feeds
            #- PROXY_BYPASS_PRIVATE=1  # default: 0
            #- PROXY_BYPASS_DOMAINS=example.com;example.net
            #- HTTP_TIMEOUT=30  # default: 12
            #- HTTP_CONCURRENCY=0  # default: 1024
            #- HTTP_CONCURRENCY_PER_HOST=0  # default: 16
            #- TABLE_TO_IMAGE=1  # default: 0
            #- TRAFFIC_SAVING=1  # default: 0
            #- LAZY_MEDIA_VALIDATION=1  # default: 0
            #- MANAGER_PRIVILEGED=1  # default: 0
            #- NO_UVLOOP=1  # default: 0
            #- MULTIPROCESSING=1  # default: 0
            #- EXECUTOR_NICENESS_INCREMENT=5  # default: 2
            #- DEBUG=1  # debug logging, default: 0
            # ↑------ Advanced settings ------↑ #


volumes:
    redis-data:
```

## 本地 mac 配置

如果你想在本地配置测试，可参考如下配置:

docker-compose.yml:

```yaml
version: '3.9'

services:
    rsshub:
        # two ways to enable puppeteer:
        # * comment out marked lines, then use this image instead: diygod/rsshub:chromium-bundled
        # * (consumes more disk space and memory) leave everything unchanged
        image: diygod/rsshub
        restart: always
        ports:
            - '1200:1200'
        environment:
            NODE_ENV: production
            CACHE_TYPE: redis
            REDIS_URL: 'redis://redis:6379/'
            PUPPETEER_WS_ENDPOINT: 'ws://browserless:3000'  # marked
            PROXY_URI: 'http://host.docker.internal:7890'
            TWITTER_USERNAME: xxx
            TWITTER_PASSWORD: xx
        depends_on:
            - redis
            - browserless  # marked

    browserless:  # marked
        image: browserless/chrome  # marked
        restart: always  # marked
        ulimits:  # marked
          core:  # marked
            hard: 0  # marked
            soft: 0  # marked

    redis:
        image: redis:alpine
        restart: always
        volumes:
            - redis-data:/data

    warp-socks:
        image: monius/docker-warp-socks:latest
        privileged: true
        restart: always
        volumes:
            - /lib/modules:/lib/modules
        cap_add:
            - NET_ADMIN
            - SYS_MODULE
        sysctls:
            net.ipv6.conf.all.disable_ipv6: 0
            net.ipv4.conf.all.src_valid_mark: 1
        healthcheck:
            test: ["CMD", "curl", "-f", "https://www.cloudflare.com/cdn-cgi/trace"]
            interval: 30s
            timeout: 10s
            retries: 5

    rsstt:
        image: rongronggg9/rss-to-telegram:dev # stable image: rongronggg9/rss-to-telegram
        container_name: rsstt # need to be unique
        restart: unless-stopped
        volumes:
            - ./config:/app/config
        environment:
            - TOKEN=xxx # get it from @BotFather
            - MANAGER=xxx # get it from @userinfobot, can be a list (e.g., 1234567890;987654321)

            # ↓------ To disable sending via Telegraph, comment out this area ------↓ #
            # Get Telegraph API access tokens: https://api.telegra.ph/createAccount?short_name=RSStT&author_name=Generated%20by%20RSStT&author_url=https%3A%2F%2Fgithub.com%2FRongronggg9%2FRSS-to-Telegram-Bot
            # Refresh the page every time you get a new token.
            # If you have a lot of subscriptions, make sure to get at least 5 tokens.
            #                            ↓ Replace with your access tokens ↓
            - TELEGRAPH_TOKEN= 
              xxx 
              xxx 
              xxx 
              xxx 
              xxx
            # ↑------ To disable sending via Telegraph, comment out this area ------↑ #

            # Please read https://github.com/Rongronggg9/RSS-to-Telegram-Bot/blob/master/docs/advanced-settings.md for more details.
            # ↓------ Advanced settings ------↓ #
            #- ERROR_LOGGING_CHAT=-1001234567890  # default: the first MANAGER
            #- MULTIUSER=0  # default: 1
            #- CRON_SECOND=30  # 0-59, default: 0
            #- DATABASE_URL=postgres://username:password@host:port/db_name  # default: sqlite://path/to/config/db.sqlite3
            #- API_ID=1025907  # get it from https://core.telegram.org/api/obtaining_api_id
            #- API_HASH=452b0359b988148995f22ff0f4229750  # get it from https://core.telegram.org/api/obtaining_api_id
            #- IMG_RELAY_SERVER=https://wsrv.nl/?url=  # default: https://rsstt-img-relay.rongrong.workers.dev/
            #- IMAGES_WESERV_NL=https://t0.nl/  # default: https://wsrv.nl/
            #- USER_AGENT=Mozilla/5.0 (Android 12; Mobile; rv:68.0) Gecko/68.0 Firefox/96.0  # default: RSStT/2.2 RSS Reader
            #- IPV6_PRIOR=1  # default: 0
            - T_PROXY=http://host.docker.internal:7890  # Proxy used to connect to the Telegram API
            - R_PROXY=http://host.docker.internal:7890  # Proxy used to fetch feeds
            #- PROXY_BYPASS_PRIVATE=1  # default: 0
            #- PROXY_BYPASS_DOMAINS=example.com;example.net
            #- HTTP_TIMEOUT=30  # default: 12
            #- HTTP_CONCURRENCY=0  # default: 1024
            #- HTTP_CONCURRENCY_PER_HOST=0  # default: 16
            #- TABLE_TO_IMAGE=1  # default: 0
            #- TRAFFIC_SAVING=1  # default: 0
            #- LAZY_MEDIA_VALIDATION=1  # default: 0
            #- MANAGER_PRIVILEGED=1  # default: 0
            #- NO_UVLOOP=1  # default: 0
            #- MULTIPROCESSING=1  # default: 0
            #- EXECUTOR_NICENESS_INCREMENT=5  # default: 2
            #- DEBUG=1  # debug logging, default: 0
            # ↑------ Advanced settings ------↑ #


volumes:
    redis-data:
```

经测试，rsshub可以正常生成 rss 源，rsstt 也可以正常订阅源。
