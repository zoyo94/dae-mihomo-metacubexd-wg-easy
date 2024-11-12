[TOC]

> 主要为了解决wireguard设备连接服务器走mihomo代理网络。
> 无需设备二次代理，设备先wireguard，然后再试用浏览器SwitchOmega代理socks5。
> mac、windows电脑还好可以上述连接，手机端只能VPN一次没办法二次代理。

## 一、 安装docker

## 二、部署mihomo、metacubexd、wg-easy
|服务|网络模式|ip|服务端口|映射到主机端口|
|---|---|---|---|---|
|mihomo|bridge|172.19.0.2|7895、9090|7895、9090|
|metacubexd|bridge|172.19.0.4|80|28002|
|wg-easy|host|无|51821、51820||

### 1、部署mihomo、metacubexd

#### 1.1、编辑config.yaml 
`vi config.yaml`
```yml
#port: 7890
log-level: error
#socks-port: 7891
#redir-port: 7892
#tproxy-port: 7893
mixed-port: 7895
ipv6: true
allow-lan: true
mode: rule
external-controller: 0.0.0.0:9090
secret: ZOYOZOYO
bind-address: "*"
find-process-mode: strict
unified-delay: true
tcp-concurrent: true
dns:
  enable: true
  ipv6: true
  respect-rules: false
  listen: 0.0.0.0:7874
  use-hosts: false
  use-system-hosts: false
  enhanced-mode: redir-host
  fake-ip-range: 198.18.0.1/16
  fake-ip-filter:
    - '*'
    - '+.lan'
    - '+.local'
  nameserver:
  - quic://dns.alidns.com
  - https://doh.pub/dns-query
  fallback:
  - 8.8.8.8
  - 8.8.4.4
  - tls://1.0.0.1:853
  - tls://dns.google:853
tun:
  enable: true
  stack: mixed
  device: utun
  strict-route: true
  auto-route: true
  auto-detect-interface: true
  dns-hijack:
  - tcp://any:53
  后面省略掉自己补充
```

#### 1.2、上网搜嘎所需的文件ASN.mmdb、geoip.dat、geoip.metadb、geosite.dat

#### 1.3、创建指定文件以及文件夹
` mkdir proxy_provider rule_provider`
` touch cache.db`

#### 2.1、编辑docker-compose.yml
`vi docker-compose.yml`
```yml
services:
  clash:
    container_name: clash-meta
    image: metacubex/mihomo:latest
    restart: always
    cap_add:
      - ALL
    security_opt:
      - apparmor=unconfined
    ports:
      - '9090:9090'
      - '7895:7895'
    volumes:
      - ./ASN.mmdb:/root/.config/mihomo/ASN.mmdb
      - ./cache.db:/root/.config/mihomo/cache.db
      - ./geoip.dat:/root/.config/mihomo/geoip.dat
      - ./geoip.metadb:/root/.config/mihomo/geoip.metadb
      - ./geosite.dat:/root/.config/mihomo/geosite.dat
      - ./proxy_provider:/root/.config/mihomo/proxy_provider
      - ./rule_provider:/root/.config/mihomo/rule_provider
      - ./config.yaml:/root/.config/mihomo/config.yaml
      - /dev/net/tun:/dev/net/tun
    networks:
      custom-net:
        ipv4_address: 172.19.0.2   # 静态 IP 地址
        aliases:
          - clash-gateway
    sysctls:
      - net.ipv4.ip_forward=1  # 打开 IP 转发功能

  metacubexd:
    container_name: metacubexd
    image: ghcr.io/metacubex/metacubexd
    restart: always
    networks:
      custom-net:
        ipv4_address: 172.19.0.4   # 静态 IP 地址
        aliases:
          - metacubexd-service
    ports:
      - '28002:80'
    depends_on:
      - clash

networks:
  custom-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.19.0.0/16  
```

#### 2.2、启动容器
` docker compose up -d `
或者
`docker-compose up -d `

#### 2.3、关闭移除容器
`docker compose down `
或者
`docker-compose down `

### 2、部署wg-easy容器

#### 1.1、生成管理密码
`docker run -it ghcr.io/wg-easy/wg-easy wgpw '[密码]'`

#### 1.2、创建.env文件
` vi .env`
**注意：**hash密码前面有两个\$
```bash
PASSWORD_HASH=$'生成的密码串'
```

#### 1.3、编辑docker-compose.yml 
`vi docker-compose.yml`
```yml
volumes:
  etc_wireguard:

services:
  wg-easy:
    environment:
      # 更改语言:
      # （支持: en, ua, ru, tr, no, pl, fr, de, ca, es, ko, vi, nl, is, pt, chs, cht, it, th, hi, ja, si）
      - LANG=en
      # ⚠️ 必填:
      # 更改为您的主机公网地址
      - WG_HOST=10.10.10.10

      # 可选:
      - PASSWORD_HASH=${PASSWORD_HASH}
      - PORT=51821
      - WG_PORT=51820
      - WG_DEFAULT_DNS=1.1.1.1
      # - WG_CONFIG_PORT=92820
      # - WG_DEFAULT_ADDRESS=10.8.0.x
      # - WG_MTU=1420
      # - WG_ALLOWED_IPS=192.168.15.0/24, 10.0.1.0/24
      # - WG_PERSISTENT_KEEPALIVE=25
      # - WG_PRE_UP=echo "Pre Up" > /etc/wireguard/pre-up.txt
      # - WG_POST_UP=echo "Post Up" > /etc/wireguard/post-up.txt
      # - WG_PRE_DOWN=echo "Pre Down" > /etc/wireguard/pre-down.txt
      # - WG_POST_DOWN=echo "Post Down" > /etc/wireguard/post-down.txt
      # - UI_TRAFFIC_STATS=true
      # - UI_CHART_TYPE=0 # （0 禁用图表, 1 折线图, 2 区域图, 3 条形图）
      # - WG_ENABLE_ONE_TIME_LINKS=true
      # - UI_ENABLE_SORT_CLIENTS=true
      # - WG_ENABLE_EXPIRES_TIME=true
      # - ENABLE_PROMETHEUS_METRICS=false
      # - PROMETHEUS_METRICS_PASSWORD=$$2a$$12$$vkvKpeEAHD78gasyawIod.1leBMKg8sBwKW.pQyNsq78bXV3INf2G # （需要双 $$，'prometheus_password' 的哈希值；参见 "How_to_generate_an_bcrypt_hash.md" 生成哈希）
    
    image: ghcr.io/wg-easy/wg-easy
    container_name: wg-easy
    volumes:
      - etc_wireguard:/etc/wireguard
    network_mode: host  # 使用 host 网络模式
    restart: unless-stopped
    cap_add:
      - NET_ADMIN 
      - SYS_MODULE
      - NET_RAW # ⚠️ 如果使用 Podman，请取消注释 #服务器上提示 iptable 没权限，需要额外执行 shell【modprobe iptable_raw】
    privileged: true # 启用特权模式
    #sysctls:
    # - net.ipv4.ip_forward=1
    # - net.ipv4.conf.all.src_valid_mark=1
```

#### 1.4、创建etc_wireguard文件夹
`mkdir etc_wireguard`

#### 1.5、启动容器
` docker compose up -d `
或者
`docker-compose up -d `


#### 1.6、关闭移除容器
`docker compose down `
或者
`docker-compose down `

## 三、安装dae服务
**首先升级服务器内核小于5.15不行**

### 1、升级内核

#### 1.1、查询内核版本
`uname -r`

#### 1.2、载入公钥
`rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org`

#### 1.3、安装 ELRepo
`dnf -y install https://www.elrepo.org/elrepo-release-9.el9.elrepo.noarch.rpm`

#### 1.4、设置国内源

##### 注释掉原生的镜像列表
`sed -i 's/mirrorlist=/#mirrorlist=/g' /etc/yum.repos.d/elrepo.repo`

##### 并将 elrepo.org/linux 地址替换为清华镜像源对应地址 mirrors.tuna.tsinghua.edu.cn/elrepo
`sed -i 's#elrepo.org/linux#mirrors.tuna.tsinghua.edu.cn/elrepo#g' /etc/yum.repos.d/elrepo.repo`
`dnf makecache`

#### 1.5、载入 elrepo-kernel 元数据
`dnf --disablerepo=\* --enablerepo=elrepo-kernel repolist`

#### 1.6、查看可用内核包
`dnf --disablerepo=\* --enablerepo=elrepo-kernel list kernel*`

#### 1.7、安装最新版本的kernel
lt long term，长期支持版本，更稳定
ml main line，主线版本，特性更新
`dnf --disablerepo=\* --enablerepo=elrepo-kernel install -y kernel-ml.x86_64`

#### 1.8、删除旧版本工具包
`dnf -y remove kernel-tools-libs.x86_64 kernel-tools.x86_64`

#### 1.9、安装新版本内核工具包
`dnf -y install pciutils-libs`
`dnf --disablerepo=\* --enablerepo=elrepo-kernel install -y kernel-ml-tools.x86_64`

#### 1.10、查看所有内核启动项
`grubby --info=ALL `

#### 1.11、设置内核启动项
` grubby --set-default 0`

#### 1.12、 重启服务器
`reboot`

### 2、删除旧的内核

#### 2.1、查看已安装的内核
`rpm -qa | grep kernel`

#### 2.1、对应的limit值，可以修改配置文件
`grep limit /etc/dnf/dnf.conf`

#### 2.2、删除多余内核，只保留最后两个
目前就是两个内核，执行没用
`dnf remove --oldinstallonly --setopt installonly_limit=2 kernel`

#### 2.3、如果您只想保留当前活动内核，只能手动删除对应的内核包名，保留kernel-headers
`dnf remove kernel-core-5.14.0-70.13.1.el9_0.x86_64 kernel-modules-5.14.0-70.13.1.el9_0.x86_64 kernel-5.14.0-70.13.1.el9_0.x86_64 4 `

#### 2.4、删除不需要的内核启动项
`grubby --remove-kernel=/boot/vmlinuz-5.14.0-70.13.1.el9_0.x86_64`

### 3、安装dae
`mkdir /etc/dae && cd /etc/dae`

#### 3.1、下载安装包
`wget https://github.com/daeuniverse/dae/releases/download/v0.8.0/dae-linux-x86_64.zip`

#### 3.2、解压包
`unzip dae-linux-x86_64.zip `

#### 3.3、编辑运行配置文件
WAN 是为本机绑定的网卡，填入 VPS 自身的网卡。
LAN 是为局域网转发的网卡，填写wg0，使得为容器wg-easy的连接端提供网络
`vi config.dae`
```yml
global{
        log_level: info
        wan_interface: auto
        lan_interface: wg0
        auto_config_kernel_parameter: true
}
group {
        my_group{
                policy: fixed(0)
        }
}

dns {
  upstream {
    googledns: 'tcp+udp://dns.google:53'
    alidns: 'udp://dns.alidns.com:53'
  }
  routing {
    # According to the request of dns query, decide to use which DNS upstream.
    # Match rules from top to bottom.
    request {
      # Lookup China mainland domains using alidns, otherwise googledns.
      qname(geosite:cn) -> alidns
      # fallback is also called default.
      fallback: googledns
    }
  }
}

routing{
        pname(systemd-resolved) -> must_direct
        domain(geosite:cn) -> direct
        ip(geoip:private) -> direct
        ip(geoip:cn) -> direct
        fallback: my_group
}
node{
        local:'socks5://127.0.0.1:7895'
}
```

#### 3.4、使用screen启动服务
`screen -S dae-server`
`cd /etc/dae`
`./dae-linux-x86_64 run -c config.dae`
按键ctrl+a+d后台运行会话

#### 3.5、添加自启动
`vi /etc/rc.local`
```bash
screen -dmS dae-server  bash -c "cd /etc/dae && ./dae-linux-x86_64 run -c config.dae"
```

##### 参考：
[rockylinux内核升级](https://www.rockylinux.cn/notes/rocky-linux-9-nei-he-sheng-ji-zhi-6.html)
[dae](https://blog.skyju.cc/post/dae-docker-full-proxy/)
[dae-github](https://github.com/daeuniverse/dae/issues/244)
