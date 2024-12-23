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
/opt/libexec/wget-ssl https://raw.githubusercontents.com/privacy-protection-tools/anti-AD/master/anti-ad-smartdns.conf -O /opt/etc/smartdns/anti-ad-smartdns.conf --no-check-certificate
killall smartdns
/opt/sbin/smartdns &
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
4. 点击Padavan-参数设置-脚本-应用设置。  
5. 将Padavan-系统管理-服务-调度任务 (Crontab)文本框里，加入0 3 * * * source "/etc/storage/post_wan_script.sh" updateadrule 并本页面的应用设置。
点击Padavan首页的重启按钮。

## 删除方法：  
1. 到上述文本框里手动清除相关shell脚本，而后重启Padavan。  
