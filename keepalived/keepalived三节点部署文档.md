#### 部署keepalived负载均衡
下载并解压keepalived(所有节点操作):
```
wget -P /opt/tempData  http://www.keepalived.org/software/keepalived-2.2.4.tar.gz
tar -zxf /opt/tempData/keepalived-2.2.4.tar.gz -C /opt/tempData
```

编译(所有节点操作):
```
cd /opt/tempData/keepalived-2.2.4 && ./configure --prefix=/opt/keepalived && make && make install
cp /opt/keepalived/sbin/keepalived /usr/local/bin/
cp /opt/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
```

systemd管理keepalived(所有节点操作):
```
cat > /usr/lib/systemd/system/keepalived.service << EOF
[Unit]
Description=LVS and VRRP High Availability Monitor
After=syslog.target network-online.target

[Service]
Type=forking
PIDFile=/var/run/keepalived.pid
KillMode=process
EnvironmentFile=-/etc/sysconfig/keepalived
ExecStart=/usr/local/bin/keepalived \$KEEPALIVED_OPTIONS
ExecReload=/bin/kill -HUP \$MAINPID

[Install]
WantedBy=multi-user.target
EOF
```

配置keepalived主节点(k8s-master1节点操作)：
```
mkdir /etc/keepalived
cat > /etc/keepalived/keepalived.conf << EOF
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id SERVICE
   script_user root
   enable_script_security
}
 
vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
}
 
vrrp_instance VI_1 {
    state MASTER  # 主
    interface eth0  # 修改为实际网卡名
    virtual_router_id 20 # VRRP 路由 ID实例，每个实例是唯一的
    priority 100    # 优先级，备服务器设置比100小，且均不相同
    advert_int 1    # 指定VRRP 心跳包通告间隔时间，默认1秒
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    # 虚拟IP
    virtual_ipaddress {
        192.168.0.7/16 # vip及子网
    }
    track_script {
        check_nginx
    }
}
EOF

cat > /etc/keepalived/check_nginx.sh << EOF
#!/bin/bash
count=\$(ps -ef |grep nginx | grep sbin | egrep -cv "grep|\$\$")
if [ "\$count" -eq 0 ];then
    systemctl stop keepalived
fi
EOF
chmod a+x /etc/keepalived/check_nginx.sh
```

配置keepalived从节点(k8s-master2节点操作)：
```
mkdir /etc/keepalived
cat > /etc/keepalived/keepalived.conf << EOF
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id SERVICE
   script_user root
   enable_script_security
}
 
vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
}
 
vrrp_instance VI_1 {
    state BACKUP  # 备
    interface eth0  # 修改为实际网卡名
    virtual_router_id 20 # VRRP 路由 ID实例，每个实例是唯一的
    priority 80    # 优先级
    advert_int 1    # 指定VRRP 心跳包通告间隔时间，默认1秒
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    # 虚拟IP
    virtual_ipaddress {
        192.168.0.7/16 # vip及子网
    }
    track_script {
        check_nginx
    }
}
EOF

cat > /etc/keepalived/check_nginx.sh << EOF
#!/bin/bash
count=\$(ps -ef |grep nginx | grep sbin | egrep -cv "grep|\$\$")
if [ "\$count" -eq 0 ];then
    systemctl stop keepalived
fi
EOF
chmod a+x /etc/keepalived/check_nginx.sh
```

配置keepalived从节点(k8s-master3节点操作)：
```
mkdir /etc/keepalived
cat > /etc/keepalived/keepalived.conf << EOF
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id SERVICE
   script_user root
   enable_script_security
}
 
vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
}
 
vrrp_instance VI_1 {
    state BACKUP  # 备
    interface eth0  # 修改为实际网卡名
    virtual_router_id 20 # VRRP 路由 ID实例，每个实例是唯一的
    priority 70    # 优先级
    advert_int 1    # 指定VRRP 心跳包通告间隔时间，默认1秒
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    # 虚拟IP
    virtual_ipaddress {
        192.168.0.7/16 # vip及子网
    }
    track_script {
        check_nginx
    }
}
EOF

cat > /etc/keepalived/check_nginx.sh << EOF
#!/bin/bash
count=\$(ps -ef |grep nginx | grep sbin | egrep -cv "grep|\$\$")
if [ "\$count" -eq 0 ];then
    systemctl stop keepalived
fi
EOF
chmod a+x /etc/keepalived/check_nginx.sh
```

启动keepalived主节点(k8s-master1)、keepalived从节点(k8s-master2，k8s-master3)：
```
systemctl daemon-reload
systemctl enable keepalived
systemctl restart keepalived
```
