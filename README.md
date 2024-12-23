# Run-SmartDNS-on-Padavan
在Padavan固件的路由器上，使用内置功能安装SmartDNS，用来优化网络访问质量并过滤广告。  

## 安装方法：  
1. 确保路由器可以正常连接到互联网。  
2. 将“在路由器启动后执行”里面的脚本复制并粘贴到Padavan-参数设置-脚本-在路由器启动后执行（文本框里）。  
3. 将“在 WAN 上行下行启动后执行”里面的脚本复制并粘贴到Padavan-参数设置-脚本-在 WAN 上行下行启动后执行（文本框里）。  
4. 点击Padavan-参数设置-脚本-应用设置。  
5. 将Padavan-系统管理-服务-调度任务 (Crontab)文本框里，加入0 3 * * * source "/etc/storage/post_wan_script.sh" updateadrule 并本页面的应用设置。
点击Padavan首页的重启按钮。

## 删除方法：  
1. 到上述文本框里手动清除相关shell脚本，而后重启Padavan。  
