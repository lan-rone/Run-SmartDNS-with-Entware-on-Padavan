# Run-SmartDNS-on-Padavan
在Padavan固件的路由器上，使用内置功能安装SmartDNS，用来优化网络访问质量并过滤广告。  

## 安装方法：  
1. 确保路由器可以正常连接到互联网。  

2. 将“在路由器启动后执行”里面的脚本复制并粘贴到Padavan-参数设置-脚本-在路由器启动后执行（文本框里）。  
```sh
#!/bin/sh

###使用镜像加速安装entware环境
until ping -c 1 223.5.5.5 >/dev/null 2>&1; do sleep 1; done
logger "internet已接通"
if [ ! -f /opt/bin/opkg ]; then
  mount -t tmpfs tmpfs /opt -o size=30M
  for folder in bin etc lib/opkg tmp var/lock var/run; do
    [ ! -d "/opt/$folder" ] && mkdir -p /opt/$folder
  done
  wget http://mirror.nju.edu.cn/entware/mipselsf-k3.4/installer/opkg -O /opt/bin/opkg
  chmod 755 /opt/bin/opkg
  wget http://mirror.nju.edu.cn/entware/mipselsf-k3.4/installer/opkg.conf -O /opt/etc/opkg.conf
  sed -i 's|bin.entware.net|mirror.nju.edu.cn/entware|g' /opt/etc/opkg.conf
  /opt/bin/opkg update
  ###Fix for multiuser environment
  chmod 777 /opt/tmp
  for file in passwd group shells shadow; do
    cp /etc/$file /opt/etc/$file
  done
  cp /etc/TZ /opt/etc/TZ
  logger "entware运行环境创建完成"
fi

###安装dropbear
/opt/bin/opkg install dropbear
/opt/sbin/dropbear -p 22 &
pidof dropbear && logger "dropbear已启动"

###安装ttyd
enable=false
if $enable; then
/opt/bin/opkg install ttyd
/opt/bin/ttyd -p 8080 login &
pidof ttyd && logger "ttyd已启动,地址ip:8080"
else
  logger "没安装ttyd"
fi

###安装vsftpd
enable=false
if $enable; then
  /opt/bin/opkg install vsftpd
  cat >/opt/etc/vsftpd/vsftpd.conf <<"EOF"
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
use_localtime=YES
connect_from_port_20=YES
listen=YES
pam_service_name=vsftpd
EOF
  /opt/sbin/vsftpd &
  pidof vsftpd && logger "vsftpd已启动"
else
  logger "没安装vsftpd"
fi

###安装smartdns
/opt/bin/opkg install smartdns
cat >/opt/etc/smartdns/smartdns.conf <<"EOF"
user nobody
server-name smartdns
restart-on-crash yes
log-level notice
log-size 128K
log-num 2
bind [::]:60053
audit-enable yes
audit-size 128K
audit-num 2
force-qtype-SOA 65
#@@@@@@@@@@@
#@@@@@@@@@@@
server-https https://223.5.5.5/dns-query
server-https https://223.6.6.6/dns-query
server-https https://1.12.12.12/dns-query
server-https https://120.53.53.53/dns-query
server-https https://doh.360.cn/dns-query
server-https https://1.0.0.1/dns-query
server-https https://dns64.dns.google/dns-query
server-https https://doh-pure.onedns.net/dns-query
#@@@@@@@@@@@
#@@@@@@@@@@@
conf-file /opt/etc/smartdns/anti-ad-smartdns.conf
EOF
###安装wget-ssl,下载anti-ad规则
/opt/bin/opkg install wget-ssl ca-certificates
/opt/libexec/wget-ssl https://hub.gitmirror.com/https://raw.githubusercontent.com/privacy-protection-tools/anti-AD/master/anti-ad-smartdns.conf -O /opt/etc/smartdns/anti-ad-smartdns.conf --no-check-certificate
killall smartdns
/opt/sbin/smartdns -c /opt/etc/smartdns/smartdns.conf &
pidof smartdns && logger "smartdns已加载过滤广告规则"

###如果smartdns启动失败,dnsmasq继续以默认配置运行
if pidof smartdns; then
  cp /etc/dnsmasq.conf /opt/etc/dnsmasq.conf
  sed -i '/cache-size=/d' /opt/etc/dnsmasq.conf
  cat >>/opt/etc/dnsmasq.conf <<"EOF"
no-resolv
no-hosts
cache-size=0
query-port=65353
server=127.0.0.1#60053
EOF
  killall dnsmasq
  dnsmasq -C /opt/etc/dnsmasq.conf &
  logger "dnsmasq已重启并禁用缓存"
  logger "smartdns已启用,开始转发dnsmasq的dns解析请求"
else
  logger "smartdns启动异常,dnsmasq以默认配置继续运行"
fi
```
3. 将“在 WAN 上行下行启动后执行”里面的脚本复制并粘贴到Padavan-参数设置-脚本-在 WAN 上行下行启动后执行（文本框里）。  
```sh
#!/bin/sh

### Custom user script
### Called after internal WAN up/down action
### $1 - WAN action (up/down)
### $2 - WAN interface name (e.g. eth3 or ppp0)
### $3 - WAN IPv4 address

# 获取脚本的执行程序路径和参数
script_name=$0
script_params="$@"
logger "在 WAN 上行/下行启动后执行:"
logger "脚本启动: $script_name，参数: $script_params"

# 获取传入的两个参数
action=$1
interface=$2

# 判断传入的参数是否符合条件
if [ "$action" = "up" ] && [ "$interface" = "ppp0" ]; then
    # 使用ping命令检测网络连接状态
    if ! ping -c 10 223.5.5.5 >/dev/null 2>&1; then
        # 如果网络不通，执行重连命令
        logger "Network is down. Restarting WAN..."
        restart_wan
        exit
    else
        logger "Network is up. No need to restart WAN."
        
        # 使用 until 循环等待网络恢复
        until ping -c 1 223.5.5.5 >/dev/null 2>&1; do
            sleep 1
        done

        # 重启 DHCPD
        restart_dhcpd
        
        # 如果 SmartDNS 进程存在，更新 DNS 配置
        if pidof smartdns >/dev/null 2>&1; then
            cp /etc/dnsmasq.conf /opt/etc/dnsmasq.conf
            sed -i '/cache-size=/d' /opt/etc/dnsmasq.conf
            cat >>/opt/etc/dnsmasq.conf <<"EOF"
no-resolv
no-hosts
cache-size=0
query-port=65353
server=127.0.0.1#60053
EOF
            killall dnsmasq
            dnsmasq -C /opt/etc/dnsmasq.conf
        else
            logger "dnsmasq以默认参数运行中"
        fi
    fi
fi

# 如果是 "updateadrule" 操作，调用 updateadrule 函数
if [ "$action" = "updateadrule" ]; then
    # 定义函数 updateadrule
    updateadrule() {
        # 等待网络连接
        until ping -c 1 223.5.5.5 >/dev/null 2>&1; do
            sleep 1
        done

        # 生成 SmartDNS 配置文件
        cat >/opt/etc/smartdns/smartdns-new.conf <<"EOF"
user nobody
server-name smartdns
log-level notice
log-size 128K
log-num 2
bind [::]:60053
audit-enable yes
audit-size 128K
audit-num 2
force-qtype-SOA 65
server-https https://223.5.5.5/dns-query
server-https https://223.6.6.6/dns-query
server-https https://1.12.12.12/dns-query
server-https https://120.53.53.53/dns-query
server-https https://doh.360.cn/dns-query
server-https https://1.0.0.1/dns-query
server-https https://dns64.dns.google/dns-query
server-https https://doh-pure.onedns.net/dns-query
conf-file /opt/etc/smartdns/anti-ad-smartdns-new.conf
EOF

        # 检测 SmartDNS 进程并更新规则
        if pidof smartdns >/dev/null 2>&1; then
            logger "检测到 SmartDNS 进程，开始更新广告过滤规则"

            # 下载广告过滤规则
            if /opt/libexec/wget-ssl https://hub.gitmirror.com/https://raw.githubusercontent.com/privacy-protection-tools/anti-AD/master/anti-ad-smartdns.conf \
                -O /opt/etc/smartdns/anti-ad-smartdns-new.conf --no-check-certificate; then
                logger "广告过滤规则下载成功"

                # 暂时加载新规则验证其有效性
                killall smartdns
                /opt/sbin/smartdns -c "/opt/etc/smartdns/smartdns-new.conf"
                sleep 1

                # 验证加载是否成功
                if pidof smartdns >/dev/null 2>&1; then
                    logger "SmartDNS 已成功加载新广告过滤规则"
                    mv /opt/etc/smartdns/anti-ad-smartdns-new.conf /opt/etc/smartdns/anti-ad-smartdns.conf
                    # 重启 SmartDNS 服务以确保生效
                    killall smartdns
                    /opt/sbin/smartdns &
                    logger "SmartDNS 服务已重新启动（加载了更新的过滤广告规则）"
                else
                    logger "加载新规则失败，保留旧规则文件"
                    rm /opt/etc/smartdns/anti-ad-smartdns-new.conf
                    # 重启 SmartDNS 服务以确保生效
                    killall smartdns
                    /opt/sbin/smartdns &
                    logger "SmartDNS 服务已重新启动（加载了旧的过滤广告规则）"
                fi
            else
                logger "广告过滤规则下载失败，跳过更新流程"
                exit 1
            fi
        else
            logger "未检测到 SmartDNS 进程，跳过更新流程"
        fi
    }
    updateadrule
fi
```
4. 点击Padavan-参数设置-脚本-应用设置。  

5. 将Padavan-系统管理-服务-调度任务 (Crontab)文本框里，填入0 3 * * * source "/etc/storage/post_wan_script.sh" updateadrule 并应用设置。  
```sh
0 3 * * * source "/etc/storage/post_wan_script.sh" updateadrule
```

6. 保存设置（Padavan-系统管理-配置管理-保存内部存储到闪存:提交）。

7. 点击Padavan首页的重启按钮。

## 删除方法：  
1. 到上述文本框里手动清除相关shell脚本并应用设置。
2. 保存设置（Padavan-系统管理-配置管理-保存内部存储到闪存:提交）。
3. 点击Padavan首页的重启按钮。  
