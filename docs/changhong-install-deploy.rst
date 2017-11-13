.. _deploy_ha:

===========
安装部署环境
===========

1 台 Haproxy; 3 台 EMQX:

.. code-block:: properties

    Haproxy 192.168.1.100 centos7

    EMQX_1 192.168.1.101 centos7
    EMQX_2 192.168.1.102 centos7
    EMQX_3 192.168.1.103 centos7


安装 Haproxy 服务
----------------

编译安装HAProxy:

.. code-block:: properties

    wget http://www.haproxy.org/download/1.7/src/haproxy-1.7.5.tar.gz
    tar xf haproxy-1.7.5.tar.gz
    cd haproxy-1.7.5
    make TARGET=linux2628 USE_LINUX_TPROXY=1 USE_LIBCRYPT=1 USE_CRYPT_H=1 USE_ZLIB=1 USE_OPENSSL=1 USE_PCRE=1
    make install PREFIX=/usr/local/haproxy

安装成功后，查看版本详情:

.. code-block:: properties
    
    /usr/local/haproxy/sbin/haproxy -vv

----------------
部署 Haproxy 服务
----------------

复制haproxy文件到/usr/sbin下:

.. code-block:: properties

    cp /usr/local/haproxy/sbin/haproxy /usr/sbin/

复制haproxy脚本，到 `/etc/init.d` 下:

.. code-block:: properties

    cp ./examples/haproxy.init /etc/init.d/haproxy
    chmod 755 /etc/init.d/haproxy


创建系统账号:

.. code-block:: properties

    useradd -r haproxy

创建配置文件:
    
.. code-block:: properties

    mkdir /etc/haproxy

生成emqx.pem证书:

.. code-block:: properties

    cat /etc/emqx/certs/cert.pem /etc/emqx/certs/key.pem > /etc/emqx/certs/emqx.pem

修改配置文件:

.. code-block:: properties

    vi /etc/haproxy/haproxy.cfg

    global
        log 127.0.0.1 local3 info
        chroot /opt/apps/haproxy
        #user haproxy
        #group haproxy
        daemon
        maxconn 1024000

    defaults
        log global
        mode tcp
        option tcplog
        #option dontlognull
        timeout connect 5000

        # timeout > mqtt's keepalive * 1.2
        timeout client 120s
        timeout server 120s
        # 'full transparent proxy' for all
        #source 0.0.0.0 usesrc clientip

    frontend emqx_tcp
        bind *:1883
        option tcplog
        mode tcp
        default_backend emqx_tcp_back

    frontend emqx_ws
        bind *:8083
        option tcplog
        mode tcp
        default_backend emqx_ws_back

    frontend emqx_dashboard
        bind *:18083
        option tcplog
        mode tcp
        default_backend emqx_dashboard_back

    frontend emqx_api
        bind *:8080
        option tcplog
        mode tcp
        default_backend emqx_api_back

    frontend emqx_ssl
        bind *:8883 ssl crt /etc/emqx/certs/emqx.pem no-sslv3
        option tcplog
        mode tcp
        default_backend emqx_ssl_back

    frontend emqx_wss
        bind *:8084 ssl crt /etc/emqx/certs/emqx.pem no-sslv3
        option tcplog
        mode tcp
        default_backend emqx_wss_back

    backend emqx_tcp_back
        balance roundrobin
        server emqx_node_1 192.168.1.101:1883 check
        server emqx_node_2 192.168.1.102:1883 check
        server emqx_node_3 192.168.1.103:1883 check

    backend emqx_ws_back
        balance roundrobin
        server emqx_node_1 192.168.1.101:8083 check
        server emqx_node_2 192.168.1.102:8083 check
        server emqx_node_3 192.168.1.103:8083 check

    backend emqx_dashboard_back
        balance roundrobin
        server emqx_node_1 192.168.1.101:18083 check
        server emqx_node_2 192.168.1.102:18083 check
        server emqx_node_3 192.168.1.103:18083 check

    backend emqx_api_back
        balance roundrobin
        server emqx_node_1 192.168.1.101:8080 check
        server emqx_node_2 192.168.1.102:8080 check
        server emqx_node_3 192.168.1.103:8080 check

    backend emqx_ssl_back
        mode tcp
        balance source
        server emqx_node_1 192.168.1.101:1883 check inter 10000 fall 2 rise 5 weight 1
        server emqx_node_2 192.168.1.102:1883 check inter 10000 fall 2 rise 5 weight 1
        server emqx_node_3 192.168.1.103:1883 check inter 10000 fall 2 rise 5 weight 1
        #source 0.0.0.0 usesrc clientip

    backend emqx_wss_back
        mode tcp
        balance source
        server emqx_node_1 192.168.1.101:8083 check inter 10000 fall 2 rise 5 weight 1
        server emqx_node_2 192.168.1.102:8083 check inter 10000 fall 2 rise 5 weight 1
        server emqx_node_3 192.168.1.103:8083 check inter 10000 fall 2 rise 5 weight 1
        #source 0.0.0.0 usesrc clientip

启动haproxy:
    
.. code-block:: properties

    service haproxy start

----------
其他（可选）
----------

开启日志记录:

    参考: http://www.zhengdazhi.com/archives/1360

编辑系统日志配置：

.. code-block:: properties

    vim  /etc/rsyslog.d/haproxy.conf

    $ModLoad imudp
    $UDPServerRun 514
    local3.*     /var/log/haproxy.log

    # 如果不加下面的的配置则除了在 /var/log/haproxy.log 中写入日志外，也会写入message文件


配置rsyslog的主配置文件, 开启远程日志:

.. code-block:: properties

    vim /etc/sysconfig/rsyslog
    SYSLOGD_OPTIONS=”-c 2 -r -m 0″
    #-c 2 使用兼容模式，默认是 -c 5
    #-r 开启远程日志
    #-m 0 标记时间戳。单位是分钟，为0时，表示禁用该功能

配置完成后重启 haproxy 和 rsyslog 服务:
    
.. code-block:: properties

    systemctl restart rsyslog
    systemctl restart haproxy

开启全透明代理:

    参考: http://www.cnblogs.com/Bonker/p/6814183.html

--------------------
配置 Haproxy 代理服务
--------------------

配置 Haproxy, 开启全透明代理:

.. code-block:: properties

    vim /etc/haproxy/haproxy.cfg

在想要开启的全透明的 backend/default 下面加上配置:
    
.. code-block:: properties
    
    source 0.0.0.0 usesrc clientip

    配置 iptables

    iptables -F
    iptables -t mangle -N DIVERT
    iptables -t mangle -A PREROUTING -p tcp -m socket -j DIVERT
    iptables -t mangle -A DIVERT -j MARK --set-mark 222
    iptables -t mangle -A DIVERT -j ACCEPT
    ip rule add fwmark 222 lookup 100
    ip route add local 0.0.0.0/0 dev lo table 100


添加转发和重定向:
    
.. code-block:: properties

    vim /etc/sysctl.conf

    ##==================================================
    ##  Haproxy 透明代理参数
    ##==================================================
    # 允许 IP 转发
    net.ipv4.ip_forward = 1
    # 设置松散逆向路径过滤
    net.ipv4.conf.default.rp_filter = 2
    net.ipv4.conf.all.rp_filter = 2
    net.ipv4.conf.eth0.rp_filter = 0
    # 允许 ICMP 重定向
    net.ipv4.conf.all.send_redirects = 1
    net.ipv4.conf.default.send_redirects = 1

重启 Haproxy:

.. code-block:: properties
    
    systemctl restart haproxy

---------------
配置 EMQX 服务器
---------------

每台 EMQX 服务器应该将 Haproxy 代理服务器设置为它的网关，否则无法实现全透明代理:
    
.. code-block:: properties

    vim /etc/sysconfig/network-scripts/ifcfg-eth0

    # 修改 GATEWAY; DNS
    GATEWAY=192.168.1.100
    DNS1=192.168.1.100


重启网络服务:
    
.. code-block:: properties

    systemctl restart network