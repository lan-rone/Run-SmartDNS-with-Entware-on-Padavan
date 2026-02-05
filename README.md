# Run-SmartDNS-on-Padavan
在Padavan固件的路由器上，使用内置功能安装SmartDNS，用来优化网络访问质量并过滤广告。  

## 安装方法：  
1. 确保路由器可以正常连接到互联网。  

2. 将“在路由器启动后执行”里面的脚本复制并粘贴到Padavan-参数设置-脚本-在路由器启动后执行（文本框里）。  
```sh
#!/bin/sh

### --- Config SmartDNS Start ---
logger "脚本启动: 开始部署 SmartDNS"

# =========================================================
# 第一步：生成公共函数库 (存在内存 /tmp 中，所有脚本共用)
# =========================================================

cat > /tmp/smartdns_lib.sh <<'LIBEOF' # 【重要！！！】使用时请将此行取消注释
#!/bin/sh

# 定义变量
SMARTDNS_BIN="/opt/sbin/smartdns"
SMARTDNS_CONF="/opt/etc/smartdns/smartdns.conf"
AD_RULE_FILE="/opt/etc/smartdns/anti-ad-smartdns.conf"
DNSMASQ_CONF_SRC="/etc/dnsmasq.conf"
DNSMASQ_CONF_OPT="/opt/etc/dnsmasq.conf"

# --- 函数：检查网络 ---
check_network() {
  # 尝试 Ping 3次，任意一次通则返回 0 (成功)
  ping -c 1 -W 1 223.5.5.5 >/dev/null 2>&1 && return 0
  ping -c 1 -W 1 223.6.6.6 >/dev/null 2>&1 && return 0
  ping -c 1 -W 1 1.12.12.12 >/dev/null 2>&1 && return 0
  return 1
}

# --- 函数：安装环境 (内存模式) ---
install_env() {
  logger "SmartDNS Lib: 正在安装 Entware 和 SmartDNS..."
  # 挂载 tmpfs (48M)
  mount -t tmpfs tmpfs /opt -o size=48M

  # 创建目录结构
  for folder in bin etc lib/opkg tmp var/lock var/run etc/smartdns; do
    mkdir -p /opt/$folder
  done

  # 复制系统文件修复用户环境
  chmod 777 /opt/tmp
  for file in passwd group shells shadow TZ; do
    [ -f /etc/$file ] && cp /etc/$file /opt/etc/$file
  done

  # 使用镜像安装 opkg 和 smartdns
  wget http://mirror.nju.edu.cn/entware/mipselsf-k3.4/installer/opkg -O /opt/bin/opkg
  chmod 755 /opt/bin/opkg
  wget http://mirror.nju.edu.cn/entware/mipselsf-k3.4/installer/opkg.conf -O /opt/etc/opkg.conf
  sed -i 's|bin.entware.net|mirror.nju.edu.cn/entware|g' /opt/etc/opkg.conf
  logger "SmartDNS Lib: opkg 安装完成"

  /opt/bin/opkg update
  logger "SmartDNS Lib: Entware 环境安装完成"

  # 安装 SmartDNS 及其依赖
  /opt/bin/opkg install smartdns wget-ssl ca-certificates
  logger "SmartDNS Lib: SmartDNS 安装完成"

  # 写入 SmartDNS 主配置
  cat >"$SMARTDNS_CONF" <<EOF
user nobody
server-name smartdns
restart-on-crash yes
# log-level notice
# log-size 128K
# audit-enable yes
bind [::]:60053
force-qtype-SOA 65
# 如果有IPv6, 将下一行注释
force-AAAA-SOA yes
prefetch-domain yes
speed-check-mode ping,tcp:80

# 上游服务器
# 本地联通 DNS
server 202.106.195.68
server 202.106.46.151
# 阿里 DNS
server 223.5.5.5
# 腾讯 DNS
server 119.29.29.29
# 360 DNS
server 101.226.4.6

# DoH/DoT
# 阿里 DoH
server-https https://223.6.6.6/dns-query
# 腾讯 DoH
server-https https://1.12.12.12/dns-query
# 360 DoH
server-https https://101.226.4.6/dns-query
# Cloudflare (国内通常较慢，作为最后的保底)
server-https https://1.0.0.1/dns-query

# 引用广告规则文件
conf-file $AD_RULE_FILE
EOF
}

# --- 函数：下载/更新广告规则 (带回滚机制) ---
update_ad_rule() {
  logger "SmartDNS Lib: 开始更新广告规则..."

  # 确保 SmartDNS 已经安装
  if [ ! -f "$SMARTDNS_BIN" ]; then
    logger "SmartDNS Lib: SmartDNS 未安装，跳过更新。"
    return 1
  fi

  # 检查网络
  if ! check_network; then
    logger "SmartDNS Lib: 网络不通，取消更新。"
    return 1
  fi

  TMP_RULE="/opt/etc/smartdns/anti-ad-new.conf"

  # 下载新规则
  if /opt/libexec/wget-ssl https://v6.gh-proxy.org/https://raw.githubusercontent.com/privacy-protection-tools/anti-AD/master/anti-ad-smartdns.conf \
    -O "$TMP_RULE" --no-check-certificate; then

    # 验证文件大小是否异常小(例如小于10KB可能是错误页)
    filesize=$(wc -c <"$TMP_RULE")
    if [ "$filesize" -lt 10000 ]; then
      logger "SmartDNS Lib: 下载的规则文件过小，判定为失败。"
      rm -f "$TMP_RULE"
      return 1
    fi

    logger "SmartDNS Lib: 规则下载成功，应用并重启服务..."
    mv "$TMP_RULE" "$AD_RULE_FILE"

    # 重启 SmartDNS
    killall smartdns 2>/dev/null
    $SMARTDNS_BIN -c "$SMARTDNS_CONF" -p /opt/var/run/smartdns.pid &
    logger "SmartDNS Lib: SmartDNS 重启完成 (新规则)"
  else
    logger "SmartDNS Lib: 规则下载失败，保持旧规则。"
    rm -f "$TMP_RULE"
  fi
}

# --- 函数：接管 DNSMasq ---
link_dnsmasq() {
  # 只有当 SmartDNS 进程存在时才执行
  if ! pidof smartdns >/dev/null; then
    logger "SmartDNS Lib: SmartDNS 未运行，不执行 DNSMasq 接管。"
    return 1
  fi

  logger "SmartDNS Lib: 正在配置 DNSMasq 转发 -> SmartDNS..."

  # 复制原配置
  cp "$DNSMASQ_CONF_SRC" "$DNSMASQ_CONF_OPT"

  # 清理旧参数 (暴力清理，防止残留)
  sed -i '/cache-size=/d' "$DNSMASQ_CONF_OPT"
  sed -i '/no-resolv/d' "$DNSMASQ_CONF_OPT"
  sed -i '/no-hosts/d' "$DNSMASQ_CONF_OPT"
  sed -i '/server=/d' "$DNSMASQ_CONF_OPT"
  # sed -i '/query-port=/d' "$DNSMASQ_CONF_OPT"

  # 写入新参数
  cat >>"$DNSMASQ_CONF_OPT" <<EOF
no-resolv
no-hosts
cache-size=0
# query-port=65353
server=127.0.0.1#60053
EOF

  # 重启 DNSMasq (必须用 -C 指定新配置文件)
  killall dnsmasq
  dnsmasq -C "$DNSMASQ_CONF_OPT" &
  logger "SmartDNS Lib: DNSMasq 已重启并接管。"
}

LIBEOF
# =========================================================
# 函数库生成结束
# =========================================================

# 1. 赋予执行权限并加载库
chmod +x /tmp/smartdns_lib.sh
source /tmp/smartdns_lib.sh

# 2. 等待网络连接
until check_network; do sleep 2; done
logger "SmartDNS Startup: Internet 已连接"

# 3. 执行安装
install_env

# 4. 下载规则 (首次)
update_ad_rule

# 5. 启动 SmartDNS (update_ad_rule 里其实已经启动了一次，这里为了双保险确保进程在)
if ! pidof smartdns >/dev/null; then
  $SMARTDNS_BIN -c "$SMARTDNS_CONF" -p /opt/var/run/smartdns.pid &
fi

# 6. 接管 DNSMasq
link_dnsmasq

logger "SmartDNS Startup: SmartDNS脚本执行完毕"
### --- Config SmartDNS End ---
```
3. 将“在 WAN 上行下行启动后执行”里面的脚本复制并粘贴到Padavan-参数设置-脚本-在 WAN 上行下行启动后执行（文本框里）。  
```sh
#!/bin/sh

### Custom user script
### Called after internal WAN up/down action
### $1 - WAN action (up/down)
### $2 - WAN interface name (e.g. eth3 or ppp0)
### $3 - WAN IPv4 address

### --- Config SmartDNS Start ---
# 获取脚本的执行程序路径和参数
script_name=$0
action=$1
interface=$2
script_params="$@"
logger "SmartDNS WAN 脚本启动: $script_name，参数: $script_params"

# 加载函数库
if [ -f /tmp/smartdns_lib.sh ]; then
    source /tmp/smartdns_lib.sh
else
    logger "SmartDNS WAN: 未找到函数库，可能启动脚本尚未执行完毕"
    exit 1
fi

# -----------------------------------
# 场景 A: WAN 口启动 (up)
# -----------------------------------
if [ "$action" = "up" ] && [ "$interface" = "ppp0" ]; then
    logger "SmartDNS WAN: WAN ($interface) Up 发现"

    # 1. 检查网络连通性
    count=0
    # 设置最大尝试次数 5 次，每次 5 s
    max_retries=5

    while [ $count -lt $max_retries ]; do
        if check_network; then
            logger "SmartDNS WAN: 网络连接校验通过"
            break
        fi

        count=$((count + 1))
        # logger "WAN: 等待网络连通... ($count/$max_retries)" # 可选：不想刷屏日志就注释掉
        sleep 5
    done

    if [ $count -ge $max_retries ]; then
        logger "SmartDNS WAN: 警告 - 超过 $max_retries 次尝试网络仍不通，放弃配置 SmartDNS。"
        exit 1
    fi

    # 2. 检查 SmartDNS 是否存活
    if ! pidof smartdns >/dev/null; then
        logger "SmartDNS WAN: SmartDNS 未运行，尝试重新启动..."
        /opt/sbin/smartdns -c /opt/etc/smartdns/smartdns.conf &
        sleep 2
    fi

    # 3. 强制刷新 DNSMasq 配置 (防止 WAN 重连后 dnsmasq 被系统重置)
    link_dnsmasq
fi

# -----------------------------------
# 场景 B: 手动/定时更新规则 (updateadrule)
# -----------------------------------
if [ "$action" = "updateadrule" ]; then
    update_ad_rule
fi
### --- Config SmartDNS End ---
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
