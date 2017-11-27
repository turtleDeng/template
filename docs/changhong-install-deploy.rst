
.. _deploy_redis:

============================
安装部署2台Redis+Sentinel集群
============================

----------
安装部署环境
----------

2台物理机器:

.. code-block:: properties
  
  节点1 192.168.1.101 centos7.3
  节点2 192.168.1.102 centos7.3

每个节点都安装Redis服务:

.. code-block:: properties

  cd /opt
  wget http://download.redis.io/releases/redis-4.0.2.tar.gz
  tar xzf redis-4.0.2.tar.gz
  cd redis-4.0.2
  yum install gcc
  make MALLOC=libc
  make install


------------
部署Redis服务
------------

192.168.1.101节点创建主redis:

.. code-block:: properties

  mkdir redis-master
  cp /opt/redis-4.0.2/redis.conf ./redis-master

修改master节点配置文件:

.. code-block:: properties
  
  vim /opt/redis-master/redis.conf
  bind 192.168.1.101
  port 6680


192.168.1.102节点创建从redis:

.. code-block:: properties
  
  mkdir redis-slave
  cp /opt/redis-4.0.2/redis.conf ./redis-slave


修改slave节点配置文件:

.. code-block:: properties

  vim redis-slave/redis.conf
  bind 192.168.1.102
  port 6680
  slaveof 192.168.1.101 6680


192.168.1.101创建sentinel节点:

.. code-block:: properties 

  mkdir redis-sentinel
  cp /opt/redis-4.0.2/sentinel.conf redis-sentinel


修改sentinel配置文件:

.. code-block:: properties

  vim redis-sentinel/sentinel.conf
  bind 192.168.1.101
  port 26680
  sentinel monitor mymaster 192.168.1.101 6680 1

启动redis主/从节点:

.. code-block:: properties

  redis-server ./redis-master/redis.conf &

  redis-server ./redis-slave/redis.conf &

启动sentinel:

.. code-block:: properties

  redis-sentinel ./redis-sentinel/sentinel.conf &


使用sentinel的命令查看sentinel管理的redis master，我们需要用redis-cli命令进入到sentinel:

.. code-block:: properties

  redis-cli -h 192.168.1.101 -p 26680
  sentinel get-master-addr-by-name mymaster


.. _deploy_kafka:

============================
安装部署3台Zookeeper+Kafka集群
============================


----------
安装部署环境
----------

3台物理机器:

.. code-block:: properties

  节点1 192.168.1.101 centos7.3
  节点2 192.168.1.102 centos7.3
  节点3 192.168.1.103 centos7.3
  

每个节点都安装java环境:

.. code-block:: properties

  cd /opt/
  yum install java-1.8.0


每个节点都下载 kafka:

.. code-block:: properties

  wget http://mirrors.hust.edu.cn/apache/kafka/1.0.0/kafka_2.12-1.0.0.tgz 
  tar -xzf kafka_2.12-1.0.0.tgz


----------------
部署Zookeeper服务
----------------

每个节点都创建zookeeper数据目录:

.. code-block:: properties

  mkdir -p zookeeper/zkdata zookeeper/zkdatalog

优化JVM虚拟机参数:

.. code-block:: properties

  vim kafka_2.12-1.0.0/bin/kafka-server-start.sh

  export KAFKA_HEAP_OPTS="-Xmx2G -Xms2G"

每个节点都修改zookeeper配置:

.. code-block:: properties

  vim kafka_2.12-1.0.0/config/zookeeper.properties

  maxClientCnxns=0
  tickTime=2000
  initLimit=10
  syncLimit=5
  dataDir=/opt/zookeeper/zkdata
  dataLogDir=/opt/zookeeper/zkdatalog
  clientPort=2181
  server.1=192.168.1.101:2888:3888
  server.2=192.168.1.103:2888:3888
  server.3=192.168.1.103:2888:3888


192.168.1.101节点执行:

.. code-block:: properties

  echo "1" > /opt/zookeeper/zkdata/myid

192.168.1.102节点执行:

.. code-block:: properties

  echo "2" > /opt/zookeeper/zkdata/myid

192.168.1.103节点执行:

.. code-block:: properties

  echo "3" > /opt/zookeeper/zkdata/myid

每个节点都启动Zookeeper服务器:

.. code-block:: properties

  ./kafka_2.12-1.0.0/bin/zookeeper-server-start.sh kafka_2.12-1.0.0/config/zookeeper.properties &


-------------
部署Kafka服务
-------------

每个节点都创建Kafka存储目录:

.. code-block:: properties

  mkdir -p kafka/kafkalogs

每个节点都修改kafka配置文件:

.. code-block:: properties

  vim kafka_2.12-1.0.0/config/server.properties

  192.168.1.101节点配置

  broker.id=1  #这里的数字和zookeeper配置的数字最好一致
  host.name=192.168.1.101
  num.network.threads=9
  num.io.threads=16
  log.dirs=/opt/kafka/kafkalogs/
  message.max.byte=5242880
  default.replication.factor=1
  replica.fetch.max.bytes=5242880
  zookeeper.connect=192.168.1.101:2181,192.168.1.102:2181,192.168.1.103:2181

192.168.1.102节点配置:

.. code-block:: properties

  vim kafka_2.12-1.0.0/config/server.properties

  broker.id=2  #这里的数字和zookeeper配置的数字最好一致
  host.name=192.168.1.102
  num.network.threads=9
  num.io.threads=16
  log.dirs=/opt/kafka/kafkalogs/
  message.max.byte=5242880
  default.replication.factor=1
  replica.fetch.max.bytes=5242880
  zookeeper.connect=192.168.1.101:2181,192.168.1.102:2181,192.168.1.103:2181

192.168.1.103节点配置:

.. code-block:: properties

  vim kafka_2.12-1.0.0/config/server.properties

  broker.id=3  #这里的数字和zookeeper配置的数字最好一致
  host.name=192.168.1.103
  num.network.threads=9
  num.io.threads=16
  log.dirs=/opt/kafka/kafkalogs/
  message.max.byte=5242880
  default.replication.factor=1
  replica.fetch.max.bytes=5242880
  zookeeper.connect=192.168.1.101:2181,192.168.1.102:2181,192.168.1.103:2181

每个节点都启动Kafka服务:

.. code-block:: properties

  ./kafka_2.12-1.0.0/bin/kafka-server-start.sh kafka_2.12-1.0.0/config/server.properties &

-------------------
测试Zookeeper+Kafka
-------------------

测试集群是否成功:

.. code-block:: properties

  ./kafka_2.12-1.0.0/bin/kafka-topics.sh --zookeeper 192.168.1.101:2181 --replication-factor 2 --partitions 1 --topic testTopic --create
  ./kafka_2.12-1.0.0/bin/kafka-console-consumer.sh --zookeeper 192.168.1.102:2181 --topic testTopic --from-beginning
  ./kafka_2.12-1.0.0/bin/kafka-console-producer.sh --broker-list 192.168.1.103:9092 --topic testTopic


.. _deploy_emqx:

=================
安装部署3台EMQX集群
=================

----------
安装部署环境
----------

3台物理机器:

.. code-block:: properties

  节点1 192.168.1.101 centos7.3
  节点2 192.168.1.102 centos7.3
  节点3 192.168.1.103 centos7.3

每个节点都安装emqx:

.. code-block:: properties

  rpm -ivh --force emqx-changhong-centos7-v2.2-1.el7.centos.x86_64.rpm

192.168.1.101节点修改配置文件:

.. code-block:: properties

  vim /etc/emqx/emqx.conf
  node.name = emqx@192.168.1.101

192.168.1.102节点修改配置文件:

.. code-block:: properties

  vim /etc/emqx/emqx.conf
  node.name = emqx@192.168.1.102

192.168.1.103节点修改配置文件:

.. code-block:: properties

  vim /etc/emqx/emqx.conf
  node.name = emqx@192.168.1.103

每个节点都需要修改的配置文件:

.. code-block:: properties

  vim /etc/emqx/plugins/emqx_changhong.conf

  ##redis sentinel服务器地址
  changhong.redis.server = xxx.xxx.xxx.xxx:26680

  ## sentinel监听redis master的名字
  changhong.redis.sentinel = mymaster


  vim /etc/emqx/plugins/emqx_auth_jwt.conf
  ## JWT的密钥
  auth.jwt.secret = xxxxxx

  vim /etc/emqx/plugins/emqx_bridge_kafka.conf
  bridge.kafka.pool1.server = xxx.xxx.xxx.xx1:9092 xxx.xxx.xxx.xx2:9092 xxx.xxx.xxx.xx3:9092

192.168.1.101节点启动emqx服务:

.. code-block:: properties

  service emqx start

192.168.1.102节点启动emqx服务并加入EMQX集群:

.. code-block:: properties

  service emqx start
  emqx_ctl cluster join emqx@192.168.1.101

192.168.1.103节点启动emqx服务并加入EMQX集群:

.. code-block:: properties

  service emqx start
  emqx_ctl cluster join emqx@192.168.1.101

查看emqx是否集群成功:

.. code-block:: properties
  
  emqx_ctl cluster status


.. _deploy_haproxy:

=================================
安装部署haproxy，实现3台emqx负载均衡
=================================

----------
安装部署环境
----------

2台物理机器:

.. code-block:: properties

  节点1 192.168.1.101 centos7.3
  节点2 192.168.1.102 centos7.3

--------------
安装HAProxy服务
--------------

编译安装HAProxy:

.. code-block:: properties

  wget http://www.haproxy.org/download/1.7/src/haproxy-1.7.5.tar.gz
  tar xf haproxy-1.7.5.tar.gz
  cd haproxy-1.7.5
  make TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1 USE_CRYPT_H=1 USE_LIBCRYPT=1 PREFIX=/usr/local/haproxy
  make install PREFIX=/usr/local/haproxy
  安装成功后，查看版本
  /usr/local/haproxy/sbin/haproxy -v

--------------
部署HAProxy服务
--------------

复制haproxy文件到/usr/sbin下:
  
.. code-block:: properties

  cp /usr/local/haproxy/sbin/haproxy /usr/sbin/

复制haproxy脚本，到/etc/init.d下:

.. code-block:: properties

  cp ./examples/haproxy.init /etc/init.d/haproxy
  chmod 755 /etc/init.d/haproxy

创建系统账号:

.. code-block:: properties

  useradd -r haproxy

创建配置文件:

.. code-block:: properties
  
  mkdir /etc/haproxy

修改配置文件:

.. code-block:: properties

  vi /etc/haproxy/haproxy.cfg

  global
      log 127.0.0.1 local3 info
      chroot /usr/local/haproxy
      user haproxy
      group haproxy
      daemon
      maxconn 1024000

  defaults
      log global
      mode tcp
      option httplog
      option dontlognull
      timeout connect 5000
      timeout client 50000
      timeout server 50000

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
      timeout server 50s
      timeout check 5000
      server emqx_node_1 192.168.1.101:1883 check inter 10000 fall 2 rise 5 weight 1
      server emqx_node_2 192.168.1.101:1883 check inter 10000 fall 2 rise 5 weight 1
      server emqx_node_3 192.168.1.101:1883 check inter 10000 fall 2 rise 5 weight 1
      source 0.0.0.0 usesrc clientip

  backend emqx_wss_back
      mode tcp
      balance source
      timeout server 50s
      timeout check 5000
      server emqx_node_1 192.168.1.101:8083 check inter 10000 fall 2 rise 5 weight 1
      server emqx_node_2 192.168.1.101:8083 check inter 10000 fall 2 rise 5 weight 1
      server emqx_node_3 192.168.1.101:8083 check inter 10000 fall 2 rise 5 weight 1
      source 0.0.0.0 usesrc clientip

启动haproxy:

.. code-block:: properties

  service haproxy start
