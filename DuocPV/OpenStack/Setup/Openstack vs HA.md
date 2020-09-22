# Cài đặt Openstack với mô hình Haproxy # 
## Sơ đồ , quy hoạch tài nguyên ## 
### Sơ đồ  ### 
![Atom](https://i.imgur.com/hEiCVKe.png)

### Quy hoạch tài nguyên ### 
![Atom](https://i.imgur.com/r8LnazB.png)


### Phần 1: chuẩn bị cho các node ### 
- Các phần : 
  - Cấu hình hostname, static interfaces 
  - Cấu hình host file 
  - Tắt Selinux 
  - Cài đặt các gói mặc định 
  - test ping các node 

### Bước 1: Cấu hình hostname, static interfaces ### 
- **Trên tất cả các node: controller+compute**
```
hostnamectl set-hostname name-controller 
#name từng node
```
  - Cấu hình file host: `/etc/hosts`
```
192.168.30.121 controller1
192.168.30.122 controller2
192.168.30.123 controller3
192.168.30.124 compute1
192.168.30.125 compute2
192.168.30.120 controller   #Ip vip  
```
  - Tắt selinux 
```
setenforce 0
```
     - Hoặc chỉnh sửa trong file /etc/selinux/config sau đó reboot server 
  - Cài đặt Openstack và các package 
```
yum install -y centos-release-openstack-rocky net-tools
yum update 
yum install -y python-openstackclient  openstack-selinux 
yum install  -y chrony 
```
 

- **Tại node controller** 
  - Cài đặt và cấu hình memcached 
```
yum install -y memcached python-memcached 
```
  - Cho phép truy cập qua địa chỉ IP Management ( địa chỉ  của các node ) 
```
sed -i "s/-l 127.0.0.1,::1/-l 127.0.0.1,::1,`ip address`/g" /etc/sysconfig/memcached
```
  - cấu hình firewalld 
```
firewall-cmd --add-port=11211/tcp --permanent 
firewall-cmd --reload 
```
  - Khởi động dịch vụ 
```
systemctl enable memcached.service 
systemctl start memcached.service 
```
   
  - Khởi tạo Bond interfaces cho Bond0: 
  - Khởi tạo bonding 
```
modprobe bonding 
```
- Cấu hình trong file /etc/sysconfig/network-scripts/ifcfg-bond0  
```
DEVICE=bond0 
NAME=bond0 
TYPE=Bond 
BONDING_MASTER=yes 
ONBOOT=yes 
NM_CONTROLLED="no" 
BOOTPROTO=none 
DNS1=8.8.8.8 
DNS2=8.8.4.4 
BONDING_OPTS="mode=1 miimon=100" 
```
- Cấu hình file  /etc/sysconfig/network-scripts/ifcfg-eth0 
```
TYPE=Ethernet 
PROXY_METHOD=none 
BROWSER_ONLY=no 
BOOTPROTO=none 
DEFROUTE=yes 
IPV4_FAILURE_FATAL=no 
IPV6INIT=yes 
IPV6_AUTOCONF=yes 
IPV6_DEFROUTE=yes 
IPV6_FAILURE_FATAL=no 
NAME=eth0 
DEVICE=eth0 
ONBOOT=yes 
NM_CONTROLLED="no" 
MASTER=bond0 
SLAVE=yes
```
- Cấu hình file  /etc/sysconfig/network-scripts/ifcfg-eth2
```
TYPE=Ethernet 
PROXY_METHOD=none 
BROWSER_ONLY=no 
BOOTPROTO=none 
DEFROUTE=yes 
IPV4_FAILURE_FATAL=no 
IPV6INIT=yes 
IPV6_AUTOCONF=yes 
IPV6_DEFROUTE=yes 
IPV6_FAILURE_FATAL=no 
NAME=eth2 
DEVICE=eth2 
ONBOOT=yes 
NM_CONTROLLED="no" 
MASTER=bond0 
SLAVE=yes 
```

  - Cấu hình interfaces cho Bond1 
     - file  /etc/sysconfig/network-scripts/ifcfg-bond1 
```
DEVICE=bond1 
NAME=bond1 
TYPE=Bond 
BONDING_MASTER=yes 
IPADDR={IP_ADDR} 
PREFIX=24 
GATEWAY=192.168.30.1 
ONBOOT=yes 
NM_CONTROLLED="no" 
BOOTPROTO=none 
DNS1=8.8.8.8 
DNS2=8.8.4.4 
BONDING_OPTS="mode=1 miimon=100"
```
 
- Cấu hình file /etc/sysconfig/network-scripts/ifcfg-eth1 
```
TYPE=Ethernet 
PROXY_METHOD=none 
BROWSER_ONLY=no 
BOOTPROTO=none 
DEFROUTE=yes 
IPV4_FAILURE_FATAL=no 
IPV6INIT=yes 
IPV6_AUTOCONF=yes 
IPV6_DEFROUTE=yes 
IPV6_FAILURE_FATAL=no 
NAME=eth1 
DEVICE=eth1 
ONBOOT=yes 
NM_CONTROLLED="no" 
MASTER=bond1 
SLAVE=yes
```

- cấu hình file /etc/sysconfig/network-scripts/ifcfg-eth3 
```
TYPE=Ethernet 
PROXY_METHOD=none 
BROWSER_ONLY=no 
BOOTPROTO=none 
DEFROUTE=yes 
IPV4_FAILURE_FATAL=no 
IPV6INIT=yes 
IPV6_AUTOCONF=yes 
IPV6_DEFROUTE=yes 
IPV6_FAILURE_FATAL=no 
NAME=eth3 
DEVICE=eth3 
ONBOOT=yes 
NM_CONTROLLED="no" 
MASTER=bond1
SLAVE=yes 
```
  - Cấu hình Firewalld 
```
firewall-cmd --zone=public --add-interface=bond1 --permanent 
firewall-cmd --reload 
```
  - Khởi động lại dịch vụ network 
```
systemctl restart network 
```

- Cấu hình trên compute
  - Khởi tạo Bond interfaces cho Bond0 
     - Khởi tạo Bonding 
```
modprobe bonding 
```
- Cấu hình file /etc/sysconfig/network-scripts/ifcfg-bond0 
```
DEVICE=bond0 
NAME=bond0 
TYPE=Bond 
BONDING_MASTER=yes 
ONBOOT=yes 
NM_CONTROLLED="no" 
BOOTPROTO=none 
DNS1=8.8.8.8 
DNS2=8.8.4.4 
BONDING_OPTS="mode=1 miimon=100" 
```
- cấu hình file  /etc/sysconfig/network-scripts/ifcfg-eth0 
```
TYPE=Ethernet 
PROXY_METHOD=none 
BROWSER_ONLY=no 
BOOTPROTO=none 
DEFROUTE=yes 
IPV4_FAILURE_FATAL=no 
IPV6INIT=yes 
IPV6_AUTOCONF=yes 
IPV6_DEFROUTE=yes 
IPV6_FAILURE_FATAL=no 
NAME=eth0 
DEVICE=eth0 
ONBOOT=yes 
NM_CONTROLLED="no" 
MASTER=bond0 
SLAVE=yes 
```
- cấu hình file  /etc/sysconfig/network-scripts/ifcfg-eth2 
```
TYPE=Ethernet 
PROXY_METHOD=none 
BROWSER_ONLY=no 
BOOTPROTO=none 
DEFROUTE=yes 
IPV4_FAILURE_FATAL=no 
IPV6INIT=yes 
IPV6_AUTOCONF=yes 
IPV6_DEFROUTE=yes 
IPV6_FAILURE_FATAL=no 
NAME=eth2 
DEVICE=eth2 
ONBOOT=yes 
NM_CONTROLLED="no" 
MASTER=bond0 
SLAVE=yes
```
  - Cấu hình interface cho Bond1 
     - Cấu hình file  /etc/sysconfig/network-scripts/ifcfg-bond1 
```
DEVICE=bond1 NAME=bond1 
TYPE=Bond 
BONDING_MASTER=yes 
IPADDR={IP_ADDR} 
PREFIX=24 
GATEWAY=192.168.30.1
ONBOOT=yes 
NM_CONTROLLED="no" 
BOOTPROTO=none 
DNS1=8.8.8.8 
DNS2=8.8.4.4 
BONDING_OPTS="mode=1 miimon=100"
```
- Cấu hình file  /etc/sysconfig/network-scripts/ifcfg-eth1 
```
TYPE=Ethernet PROXY_METHOD=none 
BROWSER_ONLY=no 
BOOTPROTO=none 
DEFROUTE=yes 
IPV4_FAILURE_FATAL=no 
IPV6INIT=yes 
IPV6_AUTOCONF=yes 
IPV6_DEFROUTE=yes 
IPV6_FAILURE_FATAL=no 
NAME=eth1 
DEVICE=eth1 
ONBOOT=yes 
NM_CONTROLLED="no" 
MASTER=bond1 
SLAVE=yes
```

- Cấu hình file  /etc/sysconfig/network-scripts/ifcfg-eth3 
```
TYPE=Ethernet 
PROXY_METHOD=none 
BROWSER_ONLY=no 
BOOTPROTO=none
DEFROUTE=yes 
IPV4_FAILURE_FATAL=no 
IPV6INIT=yes 
IPV6_AUTOCONF=yes 
IPV6_DEFROUTE=yes 
IPV6_FAILURE_FATAL=no 
NAME=eth3 
DEVICE=eth3 
ONBOOT=yes NM_CONTROLLED="no" 
MASTER=bond1 
SLAVE=yes 
```

- Cấu hình FIrewalld 
```
firewall-cmd --zone=public --add-interface=bond1 –permanent 
firewall-cmd --reload 
```
- Khởi động lại dịch vụ network 
```
systemctl restart network 
```

## Haproxy  
- Cấu hình trên các node controller
  - Cài đặt Haproxy
```
yum install -y haproxy 
```
  - cấu hình file /etc/haproxy/haproxy.cfg
```
global     
  chroot  /var/lib/haproxy     
  daemon     
  group  haproxy     
  maxconn  4000     
  pidfile  /var/run/haproxy.pid     
  user  haproxy

defaults     
  log  global     
  maxconn  4096     
  option  redispatch 
  retries  3     
  mode    tcp     
  timeout  http-request 60s     
  timeout  queue 1m     
  timeout  connect 60s     
  timeout  client 1m     
  timeout  server 1m     
  timeout  check 60s

listen stats      
   bind 192.168.30.120:9000     
   mode http     
   stats enable     
   stats uri /stats     
   stats show-legends     
   stats realm HAProxy\ Statistics     
   stats auth admin:123@123Aa     
   stats admin if TRUE

listen dashboard_cluster     
   bind 192.168.30.120:80     
   balance roundrobin     
   mode http     
   option  httpchk     
   option  tcplog     
   cookie  SERVERID insert indirect nocache     
   server controller1 192.168.30.121:80 cookie controller1 check inter 2000 rise 2 fall 5      
   server controller2 192.168.30.122:80 cookie controller2 check inter 2000 rise 2 fall 5      
   server controller3 192.168.30.123:80 cookie controller3 check inter 2000 rise 2 fall 5 
   
listen mariadb_cluster      
   bind 192.168.30.120:3306     
   mode tcp     
   balance roundrobin     
   option httpchk GET /     
   option tcpka     
   option httpchk     
   timeout client  28800s     
   timeout server  28800s       
   stick-table type ip size 1000     
   stick on dst     
   server controller1 192.168.30.121:3306 check port 9200 inter 5s fastinter 2s rise 3 fall 3     
   server controller2 192.168.30.122:3306 check port 9200 inter 5s fastinter 2s rise 3 fall 3      
   server controller3 192.168.30.123:3306 check port 9200 inter 5s fastinter 2s rise 3 fall 3   

listen keystone_public_internal_cluster     
   bind 192.168.30.120:5000     
   balance  roundrobin     
   option  tcpka     
   option  httpchk 
   option  tcplog     
   server controller1 192.168.30.121:5000 check inter 2000 rise 2 fall 5     
   server controller2 192.168.30.122:5000 check inter 2000 rise 2 fall 5      
   server controller3 192.168.30.123:5000 check inter 2000 rise 2 fall 5     
   
listen glance_api_cluster     
   bind 192.168.30.120:9292     
   balance  roundrobin     
   option  tcpka     
   option  httpchk     
   option  tcplog     
   server controller1 192.168.30.121:9292 check inter 2000 rise 2 fall 5     
   server controller2 192.168.30.122:9292 check inter 2000 rise 2 fall 5      
   server controller3 192.168.30.123:9292 check inter 2000 rise 2 fall 5   

listen glance_registry_cluster     
   bind 192.168.30.120:9191     
   balance  roundrobin     
   option  tcpka     
   option  tcplog     
   server controller1 192.168.30.121:9191 check inter 2000 rise 2 fall 5     
   server controller2 192.168.30.122:9191 check inter 2000 rise 2 fall 5      
   server controller3 192.168.30.123:9191 check inter 2000 rise 2 fall 5   

listen nova_compute_api_cluster     
   bind 192.168.30.120:8774     
   balance  roundrobin     
   option  tcpka     
   option  httpchk     
   option  tcplog     
   server controller1 192.168.30.121:8774 check inter 2000 rise 2 fall 5     
   server controller2 192.168.30.122:8774 check inter 2000 rise 2 fall 5      
   server controller3 192.168.30.123:8774 check inter 2000 rise 2 fall 5  
   
listen nova_vncproxy_cluster     
   bind 192.168.30.120:6080     
   balance  roundrobin     
   option  tcpka     
   option  tcplog     
   capture request header X-Auth-Project-Id len 50     
   capture request header User-Agent len 50     
   server controller1 192.168.30.121:6080 check inter 2000 rise 2 fall 5     
   server controller2 192.168.30.122:6080 check inter 2000 rise 2 fall 5     
   server controller3 192.168.30.123:6080 check inter 2000 rise 2 fall 5 

listen nova_metadata_api_cluster   
   bind  192.168.30.120:8775   
   balance  roundrobin   
   option  tcpka   
   option  tcplog
   server controller1 192.168.30.121:8775 check inter 2000 rise 2 fall 5   
   server controller2 192.168.30.122:8775 check inter 2000 rise 2 fall 5   
   server controller3 192.168.30.123:8775 check inter 2000 rise 2 fall 5 

listen nova_placement_api     
   bind 192.168.30.120:8778     
   balance roundrobin     
   option tcpka     
   option tcplog     
   http-request del-header X-Forwarded-Proto     
   server controller1 192.168.30.121:8778 check inter 2000 rise 2 fall 5     
   server controller2 192.168.30.122:8778 check inter 2000 rise 2 fall 5     
   server controller3 192.168.30.123:8778 check inter 2000 rise 2 fall 5     

listen neutron_api_cluster     
   bind 192.168.30.120:9696     
   balance  roundrobin     
   option  tcpka     
   option  httpchk     
   option  tcplog     
   server controller1 192.168.30.121:9696 check inter 2000 rise 2 fall 5     
   server controller2 192.168.30.122:9696 check inter 2000 rise 2 fall 5      
   server controller3 192.168.30.123:9696 check inter 2000 rise 2 fall 5 
```

- Cấu hình firewalld 
```
firewall-cmd --add-port=9000/tcp --permanent 
firewall-cmd --reload
```

- Pakemaker 
  - Cài đặt cấu hình trên controller node 
     - Cài đặt Pakemaker
```
yum install -y pacemaker pcs resource-agents 
```
- Khởi động lại dịch vụ 
```
systemctl start pcsd.service 
systemctl enable pcsd.service 
```
- Cấu hình Firewalld
```
firewall-cmd --add-service=high-availability --permanent 
firewall-cmd --reload 
```
- Cấu hình mật khẩu cho tài khoản hacluster. Trên các node có thể sử dụng mật khẩu khác nhau 
```
echo "hacluster:123" | chpasswd
```
 
- Khởi động cluster trên controller1 
     - gửi resquest đến các node và đăng nhập 
```
pcs cluster auth controller1 controller2 controller3 -u hacluster -p 123@123Aa 
```
- bosstrap cluster 
```
pcs cluster setup --force --name ops_ctl_cluster controller1 controller2 controller3 
```
- Khởi động cluster 
```
pcs cluster start --all 
pcs cluster enable --all
``` 
- Khởi tạo virtual IP 
```
pcs resource create VirtualIP ocf:heartbeat:IPaddr2 ip=192.168.30.120 cidr_netmask=32 op monitor interval=2s
```
- Khởi tạo Resource HAproxy 
```
pcs resource create HAproxy systemd:haproxy op monitor interval=2s 
```
- Cấu hình HAProxy và VIP hoạt động trên 1 node 
```
pcs constraint order VirtualIP then 
HAproxy pcs constraint colocation add VirtualIP with HAproxy INFINITY 	
```
- Chuyển Resource về Node Controlller 1 ( để node 1 làm master cho các cấu hình Service) 
```
pcs resource enable VirtualIP 
pcs resource enable HAproxy 
pcs resource move VirtualIP controller1 
```
- Disable chức năng stonith 
```
pcs property set stonith-enabled=false --force 
```
- Disable mode quorum 
```
pcs property set no-quorum-policy=ignore 
``` 
  
##  Galera MariaDB Cluster 
  - Cài đặt cấu hình trên các controller node 
     - Cài đặt MariaDB Repository 10.3 : Chỉnh trong file  /etc/yum.repos.d/MariaDB.repo 
```
[mariadb]  
name = MariaDB  
baseurl = http://yum.mariadb.org/10.3/centos7-amd64  
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB 
gpgcheck=1 
```
- Cài đặt mariadb và galera 
```
yum remove –y mariadb-* 
yum install -y MariaDB-server MariaDB-client python2-PyMySQL galera 
```
- Cấu hình MarriaDB Server cho OPS và Galera : Chỉnh file  /etc/my.cnf.d/server.cnf 
```
cat <<EOF >   /etc/my.cnf.d/server.cnf  
[server] 
[mysqld] 
log_error   = /var/log/mariadb/error.log 
general_log_file   = /var/log/mariadb/mysql.log  
[galera] 
wsrep_on=ON 
binlog_format=ROW 
default-storage-engine=innodb 
innodb_autoinc_lock_mode=2 
innodb_locks_unsafe_for_binlog=1 
query_cache_size=0 
query_cache_type=0 
bind-address="ip add"
datadir=/var/lib/mysql 
innodb_log_file_size=100M 
innodb_file_per_table 
innodb_flush_log_at_trx_commit=2 
wsrep_provider=/usr/lib64/galera/libgalera_smm.so 
wsrep_cluster_address="gcomm://192.168.30.121,192.168.30.122,192.168.30.123" 
wsrep_cluster_name=galera_cluster	
wsrep_node_address="ip address"  
wsrep_node_name="hostname" 
wsrep_sst_method=rsync max_connections = 10240 
#max_allowed_packet = 16M 
skip-name-resolve 
#key_buffer = 16M 
#thread_stack = 192K 
#thread_cache_size = 8 
#innodb_buffer_pool_size = 64M 
[embedded] 
[mariadb] 
[mariadb-10.3] 
```

- Cấu hình firewalld trên các node 
```
firewall-cmd --add-port={3306/tcp,4567/tcp,4568/tcp,4444/tcp} --permanent 
firewall-cmd --reload 
``` 
- Khởi tạo thư mục chứa log 
```
mkdir /var/log/mariadb 
chown -R mysql:mysql /var/log/mariadb 
```


- Khởi động galera cluster trên các node controller 
  - Trên controller1 
     - Bootstrap cluster , khởi động dịch vụ 
```
galera_new_cluster 
systemctl start mariadb 
systemctl enable mariadb 
```
  - Trên các node controller2,3
```
systemctl start mariadb 
systemctl enable mariadb 
```

  - Chạy script và chọn password phù hợp cho user root của database trên controller1
```
/usr/bin/mysqladmin -u root -h localhost password 123@123Aa
```

- Cài đặt và cấu hình Cluster Check trên các controller 
  - Cài đặt package 
```
yum install -y xinetd wget 
```
  - Cấu hình Script check 
```
rm -f clustercheck ; rm -f  /usr/bin/clustercheck 
wget  https://raw.githubusercontent.com/nguyenhungsync/perconaclustercheck/master/clustercheck 
mv  clustercheck /usr/bin && chmod 555 /usr/bin/clustercheck 
mysql -u root -p123@123Aa -e "GRANT PROCESS ON *.* TO 'clustercheckuser'@'localhost' IDENTIFIED BY '123@123Aa'" 
```
  
  - Khởi tạo service 
     - Cấu hình file  /etc/xinetd.d/mysqlchk 
```
# default: on 
# description: mysqlchk 
service mysqlchk 
{ 
        disable = no 
        flags = REUSE 
        socket_type = stream 
        port = 9200
        wait = no 
        user = nobody 
        server = /usr/bin/clustercheck 
        log_on_failure += USERID 
        only_from = 0.0.0.0/0 
        per_source = UNLIMITED 
}
```
  - khai báo service 
```
echo 'mysqlchk 9200/tcp # MySQL check' >> /etc/services 
```
  - Khởi động lại serive 
```
systemctl start xinetd 
systemctl enable xinetd 
```
 - Cấu hình firewalld
```
firewall-cmd --add-port={9200/tcp,4444/tcp} --permanent 
firewall-cmd --reload 
```

- RabbitMQ 
  - Cài đặt RabbitMQ 
     - Cài đặt các pakage 
```
yum -y install rabbitmq-server  
 ```
- Cấu hình firewalld
```
firewall-cmd --add-port=15672/tcp --permanent ## Web Interface 
firewall-cmd --add-port=5672/tcp --permanent ## RabitMQ Server 
firewall-cmd --add-port={4369/tcp,25672/tcp} --permanent ## RabitMQ Cluster Port 
firewall-cmd --reload 
```

- Khởi động cluster check
  - Khởi động dịch vụ trên controller1 
```
systemctl  enable rabbitmq-server.service 
systemctl start rabbitmq-server.service 
```
  - Trên Controller 1 thực hiện copy Erlang cookie sang Controller 2 và Controller 3  
```
scp /var/lib/rabbitmq/.erlang.cookie root@controller2:/var/lib/rabbitmq/ 
scp /var/lib/rabbitmq/.erlang.cookie root@controller3:/var/lib/rabbitmq/ 
rabbitmqctl start_app 
```
  - Phân quyền và khởi động dịch vụ trên các node Controller 2 và Controller 3 
```
chown -R rabbitmq:rabbitmq /var/lib/rabbitmq 
systemctl  enable rabbitmq-server.service 
systemctl start rabbitmq-server.service 
```
  - Trên các node Controller 2 và Controller 3 thực hiện khởi động  và tham vào cluster 
```
rabbitmqctl stop_app 
rabbitmqctl join_cluster rabbit@controller1 ## yêu cầu sử dụng hostname  
rabbitmqctl start_app 
```
  - Kiểm tra trạng thái cluster check 
```
rabbitmqctl cluster_status 
```
  - Cấu hình HA policy trên 3 node controler 
```
rabbitmqctl set_policy ha-all '^(?!amq\.).*' '{"ha-mode": "all"}' 
```
  - Xóa tài khoản guest, khởi động tài khoản mới ttrên Controllerr 1 
```
rabbitmqctl delete_user guest 
rabbitmqctl add_user duocpham 123@123Aa 
rabbitmqctl add_user openstack rabbitmq_123 
rabbitmqctl set_permissions openstack ".*" ".*" ".*"   
```
  
  
## Cài đặt Openstack IDENTIFIED service 
  - Khởi tạo database trên controller1 
     - Khởi tạo keystone database: 
```
mysql -u root -p123@123Aa 
CREATE DATABASE  keystone; 
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'keystone_123'; 
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystone_123'
```
  - Cài đặt và cấu hình các thành phần: thực hiện trên tất cả các controller 
     - cài đặt pakage
```
yum install -y openstack-keystone httpd mod_wsgi 
```
- khởi tạo file cấu hình :  /etc/keystone/keystone.conf 
```
[database] 
connection = mysql+pymysql://keystone:keystone_123@controller/keystone 
[token] 
provider = fernet 
```
- Cấu hình apache 
```
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/ cp -np /etc/httpd/conf.d/wsgi-keystone.conf /etc/httpd/conf.d/wsgi-keystone.conf.origin 
cp -np /etc/httpd/conf/httpd.conf  /etc/httpd/conf/httpd.conf.origin 
echo "ServerName `hostname`" >> /etc/httpd/conf/httpd.conf 
sed -i -e "s/VirtualHost \*/VirtualHost `hostname -i`/g" /etc/httpd/conf.d/wsgi-keystone.conf  
sed -i -e "s/Listen 5000/Listen `hostname -i`:5000/g" /etc/httpd/conf.d/wsgi-keystone.conf  
sed -i -e "s/^Listen.*/Listen `hostname -i`:80/g" /etc/httpd/conf/httpd.conf  
systemctl enable httpd.service 
systemctl start httpd.service 
```
- Cấu hình firewalld 
```
firewall-cmd --add-port=5000/tcp --permanent  
firewall-cmd --reload 
```
     
  - Bootstrap trên controller1 
     - Đồng bộ database
```
su -s /bin/sh -c "keystone-manage db_sync" keystone 
```
-  Khởi tạo Fermet Repository 
```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone 
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone 
```
-  Sao chép fernet repository sang Controller 2 và Controller3
```
for node in controller2 controller3 
do 
scp -rp /etc/keystone/credential-keys root@$node:/etc/keystone/ 
scp -rp /etc/keystone/fernet-keys root@$node:/etc/keystone/ 
ssh root@$node "chown -R keystone:keystone /etc/keystone/fernet-keys/" 
ssh root@$node "chown -R keystone:keystone /etc/keystone/credential-keys/" 
done 
```
-  Boostrap Service 
```
keystone-manage bootstrap --bootstrap-password keystone_123 \ 
  --bootstrap-admin-url http://controller:5000/v3/ \ 
  --bootstrap-internal-url http://controller:5000/v3/ \ 
  --bootstrap-public-url http://controller:5000/v3/ \   --bootstrap-region-id RegionOne
```
- Khởi tạo thông tin đăng nhập 
```
cat <<EOF> /root/admin-login 
export OS_USERNAME=admin 
export OS_PASSWORD=keystone_123 
export OS_PROJECT_NAME=admin 
export OS_USER_DOMAIN_NAME=Default 
export OS_PROJECT_DOMAIN_NAME=Default 
export OS_AUTH_URL=http://controller:5000/v3 
export OS_IDENTITY_API_VERSION=3 

EOF
```

  -  Kiểm tra authencation 
```
openstack token issue 
```
  - Khởi tạo Service Project 
```
source /root/admin-login  
openstack project create --domain default --description "Service Project" service   
```


## Cài đặt Openstack IMAGE SERVICE 
 - Cấu hình trên Controller1 
     -  Khởi tạo Glance Database 
```
mysql -u root -p123@123Aa
CREATE DATABASE   glance; 
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'glance_123'; 
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'glance_123'; 
```

-  Khởi tạo User và Glance Service 
```
openstack user create --domain default --password=glance_123 glance 
openstack role add --project service --user glance admin 
openstack service create --name glance --description "OpenStack Image" image 
```
-  Khởi tạo các Endpoint 
```
openstack endpoint create --region RegionOne image public http://controller:9292 
openstack endpoint create --region RegionOne image internal http://controller:9292 
openstack endpoint create --region RegionOne image admin http://controller:9292 
```

  - Cấu hình trên tất cả các controller 
     - Cài đặt package
```
yum install -y openstack-glance 
```
- cấu hình file /etc/glance/glance-api.conf 
```
[DEFAULT] 
bind_host = `hostname -i` #ip management từng node
show_image_direct_url = True 
[database] 
connection = mysql+pymysql://glance:glance_123@controller/glance 
[keystone_authtoken] 
www_authenticate_uri = http://controller:5000 
auth_url = http://controller:5000 
memcached_servers = controller1:11211,controller2:11211,controller3:11211 
auth_type = password 
project_domain_name = Default 
user_domain_name = Default 
project_name = service 
username = glance 
password = glance_123 
[paste_deploy] 
flavor = keystone 
[glance_store] 
stores = file,http 
default_store = file 
filesystem_store_datadir = /var/lib/glance/images/ 
```

- Cấu hình file  /etc/glance/glance-registry.conf 
```
[DEFAULT] 
bind_host = `hostname -i` 
[database] 
connection = mysql+pymysql://glance:glance_123@controller/glance 
[keystone_authtoken] 
www_authenticate_uri = http://controller:5000 
auth_url = http://controller:5000 
memcached_servers = controller1:11211,controller2:11211,controller3:11211 
auth_type = password 
project_domain_name = Default 
user_domain_name = Default 
project_name = service 
username = glance 
password = glance_123 
[paste_deploy] 
flavor = keystone 
```
-  Đồng bộ database 
```
su -s /bin/sh -c "glance-manage db_sync" glance repository 
```
- Khởi động dịch vụ 
```
systemctl enable openstack-glance-api.service openstack-glance-registry.service 
systemctl restart openstack-glance-api.service openstack-glance-registry.service 
systemctl status openstack-glance-api.service openstack-glance-registry.service 
```
- Cấu hình FirewallD 
```
firewall-cmd --add-port={9191/tcp,9292/tcp} --permanent 
firewall-cmd --reload 
```



##  CÀI ĐẶT OPENSTACK COMPUTE SERVICE 	 
 - Cấu hình trên các controller node 
   -  Cấu hình trên Controlller 1 
     - Khởi tạo Nova Database 
```
mysql -u root -p123@123Aa 
CREATE DATABASE  nova_api; 
CREATE DATABASE  nova; 
CREATE DATABASE  nova_cell0; 
CREATE DATABASE  placement; 
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'nova_123'; 
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'nova_123'; 
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'nova_123'; 
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'nova_123'; 
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'nova_123'; 
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'nova_123'; 
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'placement_123'; 
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'placement_123'; 
```
-  Khởi tạo Nova User và Service 
```
openstack user create --domain default --password=nova_123 nova 
openstack role add --project service --user nova admin 
openstack service create --name nova --description "OpenStack Compute" compute 
```
- Khởi tạo Nova Endpoint 
```
openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1 
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1 
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1 
```
-  Khởi tạo Placement User và Service 
```
openstack user create --domain default --password=placement_123 placement 
openstack role add --project service --user placement admin 
openstack service create --name placement --description "Placement API" placement 
```
-  Khởi tạo Placement Endpoint 
```
openstack endpoint create --region RegionOne placement public http://:controller:8778 
openstack endpoint create --region RegionOne placement internal http://controller:8778 
openstack endpoint create --region RegionOne placement admin http://controller:8778 
```

   - cấu hình trên tất cả các controler 
     -  Cài đặt package 
```
yum install -y openstack-nova-api openstack-nova-conductor \ 
openstack-nova-console openstack-nova-novncproxy \ 
openstack-nova-scheduler openstack-nova-placement-api 
```
-  Khởi tạo tập tin cấu hình Nova : file /etc/nova/nova.conf
```
[DEFAULT] 
my_ip = `hostname -i` 
osapi_compute_listen = \$my_ip 
rpc_backend=rabbit 
enabled_apis = osapi_compute,metadata 
use_neutron = true 
metadata_host = \$my_ip 
metadata_listen = \$my_ip 
firewall_driver = nova.virt.firewall.NoopFirewallDriver 
transport_url=rabbit://openstack:rabbitmq_123@controller1,openstack:rabbitmq_123@contr oller2,openstack:rabbitmq_123@controller3 
rpc_response_timeout = 180 
[api_database] 
connection = mysql+pymysql://nova:nova_123@controller/nova_api 
[database] 
connection = mysql+pymysql://nova:nova_123@controller/nova 
[placement_database] 
connection = mysql+pymysql://placement:placement_123@controller/placement 
[api] 
auth_strategy = keystone 
[keystone_authtoken] 
auth_url = http://controller:5000/v3 
memcached_servers = controller1:11211,controller2:11211,controller3:11211 
auth_type = password 
project_domain_name = default 
user_domain_name = default 
project_name = service 
username = nova 
password = nova_123 
[vnc] 
enabled = true 
server_listen = 0.0.0.0 
server_proxyclient_address = \$my_ip 
novncproxy_host = \$my_ip 
[glance] 
api_servers = http://controller:9292 
[oslo_concurrency] 
lock_path = /var/lib/nova/tmp 
[placement] 
region_name = RegionOne 
project_domain_name = Default 
project_name = service 
auth_type = password 
user_domain_name = Default 
auth_url = http://controller:5000/v3 
username = placement 
password = placement_123 
[oslo_messaging_rabbit] 
rabbit_retry_interval=1 
rabbit_retry_backoff=2 
rabbit_max_retries=0 
rabbit_ha_queues= true 
```

   - Cấu hình Placement API : File /etc/httpd/conf.d/00-nova-placement-api.conf 
```
Listen `hostname -i`:8778 
<VirtualHost `hostname -i`:8778> 
WSGIProcessGroup nova-placement-api 
WSGIApplicationGroup %{GLOBAL} 
WSGIPassAuthorization On 
WSGIDaemonProcess nova-placement-api processes=3 threads=1 user=nova group=nova 
WSGIScriptAlias / /usr/bin/nova-placement-api 
<IfVersion >= 2.4> 
ErrorLogFormat "%M" 
</IfVersion> 
ErrorLog /var/log/nova/nova-placement-api.log 
#SSLEngine On 
#SSLCertificateFile ... 
#SSLCertificateKeyFile ... 
</VirtualHost> 
Alias /nova-placement-api /usr/bin/nova-placement-api 
<Location /nova-placement-api> 
SetHandler wsgi-script 
Options +ExecCGI 
WSGIProcessGroup nova-placement-api 
WSGIApplicationGroup %{GLOBAL} 
WSGIPassAuthorization On 
</Location> 
<Directory /usr/bin> 
<IfVersion >= 2.4> 
Require all granted 
</IfVersion> 
<IfVersion < 2.4> 
Order allow,deny 
Allow from all 
</IfVersion> 
</Directory> repository 
```
- Đồng bộ database trên node Controller 1 
```
su -s /bin/sh -c "nova-manage api_db sync" nova 
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova 
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova  
su -s /bin/sh -c "nova-manage db sync" nova 
nova-manage cell_v2 list_cells 
```
- Cấu hình FirewalD 
```
firewall-cmd --add-port={6080/tcp,6081/tcp,6082/tcp,8774/tcp,8775/tcp,8778/tcp}--permanent 
firewall-cmd --reload
```
 - khởi động dịch vụ 
```
systemctl restart httpd 
# systemctl enable openstack-nova-api.service \
  openstack-nova-consoleauth openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service
# systemctl start openstack-nova-api.service \
  openstack-nova-consoleauth openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service
```
  - Cấu hình trên compute node 
      - Cài đặt Openstack Nova Compute 
```
yum install -y openstack-nova-compute 
```
- Khởi tạo cấu hình Nova: Cấu hình trong file /etc/nova/nova.conf 
```
[DEFAULT] 
my_ip = `hostname -i` 
enabled_apis = osapi_compute,metadata 
transport_url=rabbit://openstack:rabbitmq_123@controller1,openstack:rabbitmq_123@contr oller2,openstack:rabbitmq_123@controller3 
use_neutron = true 
firewall_driver = nova.virt.firewall.NoopFirewallDriver 
rpc_response_timeout = 180 
[api] 
auth_strategy = keystone 
[keystone_authtoken] 
auth_url = http://controller:5000/v3 
memcached_servers = controller1:11211,controller2:11211,controller3:11211 
auth_type = password 
project_domain_name = default 
user_domain_name = default 
project_name = service 
username = nova 
password = nova_123 
[vnc] 
enabled = true 
server_listen = 0.0.0.0 
server_proxyclient_address = \$my_ip 
novncproxy_base_url = http://192.168.50.120:6080/vnc_auto.html 
[glance] 
api_servers = http://controller:9292 
[oslo_concurrency] 
lock_path = /var/lib/nova/tmp 
[placement] 
region_name = RegionOne 
project_domain_name = Default 
project_name = service 
auth_type = password 
user_domain_name = Default 
auth_url = http://controller:5000/v3 
username = placement 
password = placement_123 
[oslo_messaging_rabbit] 
rabbit_retry_interval=1 
rabbit_retry_backoff=2 
rabbit_max_retries=0 
rabbit_ha_queues= true 
```

- khởi động dịch vụ 
```
systemctl enable libvirtd.service openstack-nova-compute.service 
systemctl restart libvirtd.service openstack-nova-compute.service 
systemctl status libvirtd.service openstack-nova-compute.service 
```
- Cấu hinh Firewalld 
```
firewall-cmd --add-port=5900-5999/tcp --permanent 
firewall-cmd --reload 
```
- kiểm tra ảo hóa 
```
egrep -c '(vmx|svm)' /proc/cpuinfo 
```
- Thêm host vào Cell trên các controller 
```
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova 
```
- Kiểm tra các dịch vụ trên controller 
```
openstack compute service list 
```


## CÀI ĐẶT OPENSTACK NETWORKING SERVICE 
  - Cấu hình trên controller node 
    - Cấu hình trên controller1 
	 - Khởi tạo database
```
mysql -u root -p123@123Aa 
CREATE DATABASE  neutron; 
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'neutron_123'; 
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'neutron_123'; 
```
- Khởi tạo Neutron user và các Service 
```
openstack user create --domain default --password=neutron_123 neutron 
openstack role add --project service --user neutron admin 
openstack service create --name neutron --description "OpenStack Networking" network 
```
- Khởi tạo Endpoint 
```
openstack endpoint create --region RegionOne network public http://controller:9696 
openstack endpoint create --region RegionOne network internal http://controller:9696 
openstack endpoint create --region RegionOne network admin http://controller:9696 
```
  - Cấu hình trên các controller 
     - Cài đặt các pakage 
```
yum install -y openstack-neutron openstack-neutron-ml2 
```
- KHởi tạo cấu hình Neutron : Cấu hình trong file  /etc/neutron/neutron.conf 
```
[DEFAULT] 
transport_url=rabbit://openstack:rabbitmq_123@controller1,openstack:rabbitmq_123@contr oller2,openstack:rabbitmq_123@controller3 
auth_strategy = keystone 
core_plugin = ml2 
dhcp_agents_per_network = 2 
notify_nova_on_port_status_changes = true 
notify_nova_on_port_data_changes = true 
bind_host = `hostname -i` 
rpc_response_timeout = 180 
[database] 
connection = mysql+pymysql://neutron:neutron_123@controller/neutron 
[keystone_authtoken] 
www_authenticate_uri = http://controller:5000 
auth_url = http://controller:5000 
memcached_servers = controller1:11211,controller2:11211,controller3:11211 
auth_type = password 
project_domain_name = default 
user_domain_name = default 
project_name = service 
username = neutron 
password = neutron_123 
[nova] 
auth_url = http://controller:5000 
auth_type = password 
project_domain_name = default 
user_domain_name = default 
region_name = RegionOne 
project_name = service 
username = nova 
password = nova_123 
[oslo_messaging_rabbit] 
rabbit_retry_interval=1 
rabbit_retry_backoff=2 
rabbit_max_retries=0 
rabbit_ha_queues= true 
[oslo_concurrency] 
lock_path = /var/lib/neutron/tmp 
```
 
- khởi tạo cấu hình  Layer 2 Plugin: Cấu hình trong file  /etc/neutron/plugins/ml2/ml2_conf.ini 	 
```
[ml2] 
type_drivers = flat,vlan 
tenant_network_types = 
mechanism_drivers = openvswitch 
extension_drivers = port_security 
[ml2_type_flat] 
flat_networks = provider 
[ml2_type_vlan] 
network_vlan_ranges = provider 
```

- Cấu hình No chova phép sử dụng Networking Service; Cấu hình file  /etc/nova/nova.conf 
```
[neutron] 
url = http://controller:9696 
auth_url = http://controller:5000 
auth_type = password 
project_domain_name = default 
user_domain_name = default 
region_name = RegionOne 
project_name = service 
username = neutron 
password = neutron_123 
service_metadata_proxy = true 
metadata_proxy_shared_secret = metadata_123 
```
- Đồng bộ database trên các node controller 
```
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini 
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```
- KHởi động dịch vụ trên các controller node 
```
systemctl restart openstack-nova-api.service 
systemctl enable neutron-server.service 
systemctl restart neutron-server.service 
systemctl status neutron-server.service openstack-nova-api.service 
```
- Cấu hình Firewalld 
```
firewall-cmd --add-port={9696/tcp,8775/tcp} --permanent 
firewall-cmd --reload 
```

  - Cấu hình trên compute node 
     - Cài đặt các pakage 
```
yum install -y openstack-neutron openstack-neutron-openvswitch  
```
- Khởi tạo cấu hình Neutron: /etc/neutron/neutron.conf  
```
[DEFAULT] 
transport_url=rabbit://openstack:rabbitmq_123@controller1,openstack:rabbitmq_123@contr oller2, openstack:rabbitmq_123@controller3 
auth_strategy = keystone 
rpc_response_timeout = 180 
[keystone_authtoken] 
www_authenticate_uri = http://controller:5000 
auth_url = http://controller:5000 
memcached_servers = controller1:11211,controller2:11211,controller3:11211 
auth_type = password 
project_domain_nameingfh = default 
user_domain_name = default 
project_name = service 
username = neutron 
password = neutron_123 
[oslo_concurrency] 
lock_path = /var/lib/neutron/tmp 
[oslo_messaging_rabbit] 
rabbit_retry_interval=1 
rabbit_retry_backoff=2 
rabbit_max_retries=0 
rabbit_ha_queues= true 
```
  
-  Khởi tạo cấu hình Metadata agent : Cấu hình file /etc/neutron/metadata_agent.ini 
```
[DEFAULT] 
nova_metadata_host = controller 
metadata_proxy_shared_secret = metadata_123 
memcached_servers = controller1:11211,controller2:11211,controller3:11211 
```
- Khởi tạo cấu hình DHCP agent: file  /etc/neutron/dhcp_agent.ini 	 
```
[DEFAULT] 
interface_driver = openvswitch 
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq 
enable_isolated_metadata = true 
enable_metadata_network = True 
```
- Khởi tạo Openvswitch Privider 
```
systemctl start openvswitch 
ovs-vsctl add-br br-provider 
ovs-vsctl add-port br-provider bond0 
```
- KHởi tạo cấu hình openvswitch agent: cấu hình file /etc/neutron/plugins/ml2/openvswitch_agent.ini 
```
[ovs] 
bridge_mappings = provider:br-provider 
[securitygroup] 
firewall_driver = openvswitch 
enable_security_group = true 
enable_ipset = true 
```
- Cấu hình Nova cho phép sử dụng Networking: cấu hình file /etc/nova/nova.conf 
```
[neutron] 
url = http://controller:9696 
auth_url = http://controller:5000 
auth_type = password 
project_domain_name = default 
user_domain_name = default 
region_name = RegionOne 
project_name = service 
username = neutron 
password = neutron_123 
```
- Khởi động dịch vụ 
```
systemctl restart openstack-nova-compute.service 
for service in dhcp-agent openvswitch-agent metadata-agent 
do 
systemctl enable neutron-$service 
systemctl restart neutron-$service 
systemctl status neutron-$service 
done 
```
- Kiểm tra các agent 
```
openstack network agent list  
```
## Cấu hình  DASHBOARD – HORIZON 
  - Cài đặt và cấu hình 
```
yum install -y openstack-dashboard 
```
  - Khởi tạo cấu hình file * /etc/openstack-dashboard/local_settings 
```
cp /etc/openstack-dashboard/local_settings /etc/openstack-dashboard/local_settings.origin 
cp horizon.conf /etc/openstack-dashboard/local_settings 
```

```
# -*- coding: utf-8 -*-

import os

from django.utils.translation import ugettext_lazy as _

from horizon.utils import secret_key

from openstack_dashboard.settings import HORIZON_CONFIG

DEBUG = True

ALLOWED_HOSTS = ['*']

WEBROOT = '/dashboard/'


OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "volume": 3,
    "compute": 2,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"

CONSOLE_TYPE = "AUTO"

HORIZON_CONFIG["bug_url"] = "http://bug-report.example.com"

HORIZON_CONFIG["modal_backdrop"] = "static"

HORIZON_CONFIG["password_autocomplete"] = "off"


LOCAL_PATH = '/tmp'

SECRET_KEY='8b30d01c6d9bcb8f1be9'

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': [ 'controller1:11211', 'controller2:11211', 'controller2:11211', ],
    }
}

#AVAILABLE_REGIONS = [
#    ('http://cluster1.example.com:5000/v3', 'cluster1'),
#    ('http://controllerhcm:5000/v3', 'HCM'),
#]


EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'

OPENSTACK_HOST = "controller"
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "member"

OPENSTACK_SSL_NO_VERIFY = True

OPENSTACK_SSL_CACERT = '/path/to/cacert.pem'

OPENSTACK_KEYSTONE_BACKEND = {
    'name': 'native',
    'can_edit_user': True,
    'can_edit_group': True,
    'can_edit_project': True,
    'can_edit_domain': True,
    'can_edit_role': True,
}

OPENSTACK_ENABLE_PASSWORD_RETRIEVE = True

# The Xen Hypervisor has the ability to set the mount point for volumes
# attached to instances (other Hypervisors currently do not). Setting
# can_set_mount_point to True will add the option to set the mount point
# from the UI.
OPENSTACK_HYPERVISOR_FEATURES = {
    'can_set_mount_point': False,
    'can_set_password': True,
    'requires_keypair': False,
    'enable_quotas': True
}

# This settings controls whether IP addresses of servers are retrieved from
# neutron in the project instance table. Setting this to False may mitigate
# a performance issue in the project instance table in large deployments.
#OPENSTACK_INSTANCE_RETRIEVE_IP_ADDRESSES = True

# The OPENSTACK_CINDER_FEATURES settings can be used to enable optional
# services provided by cinder that is not exposed by its extension API.
OPENSTACK_CINDER_FEATURES = {
    'enable_backup': True,
}

# The OPENSTACK_NEUTRON_NETWORK settings can be used to enable optional
# services provided by neutron. Options currently available are load
# balancer service, security groups, quotas, VPN service.
OPENSTACK_NEUTRON_NETWORK = {
    'enable_router': False,
    'enable_quotas': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    'enable_fip_topology_check': False,
    'default_dns_nameservers': ["8.8.8.8", "8.8.4.4", "208.67.222.222"],
    'supported_provider_types': ['*'],
    'supported_vnic_types': ['*'],
    'physical_networks': [],

}

OPENSTACK_HEAT_STACK = {
    'enable_user_pass': True,
}

OPENSTACK_IMAGE_BACKEND = {
    'image_formats': [
        ('', _('Select format')),
        ('aki', _('AKI - Amazon Kernel Image')),
        ('ami', _('AMI - Amazon Machine Image')),
        ('ari', _('ARI - Amazon Ramdisk Image')),
        ('docker', _('Docker')),
        ('iso', _('ISO - Optical Disk Image')),
        ('ova', _('OVA - Open Virtual Appliance')),
        ('qcow2', _('QCOW2 - QEMU Emulator')),
        ('raw', _('Raw')),
        ('vdi', _('VDI - Virtual Disk Image')),
        ('vhd', _('VHD - Virtual Hard Disk')),
        ('vhdx', _('VHDX - Large Virtual Hard Disk')),
        ('vmdk', _('VMDK - Virtual Machine Disk')),
    ],
}

IMAGE_CUSTOM_PROPERTY_TITLES = {
    "architecture": _("Architecture"),
    "kernel_id": _("Kernel ID"),
    "ramdisk_id": _("Ramdisk ID"),
    "image_state": _("Euca2ools state"),
    "project_id": _("Project ID"),
    "image_type": _("Image Type"),
}


IMAGE_RESERVED_CUSTOM_PROPERTIES = []

CREATE_IMAGE_DEFAULTS = {
    'image_visibility': "public",
}


#OPENSTACK_ENDPOINT_TYPE = "publicURL"


API_RESULT_LIMIT = 1000
API_RESULT_PAGE_SIZE = 20

SWIFT_FILE_TRANSFER_CHUNK_SIZE = 512 * 1024

INSTANCE_LOG_LENGTH = 35

DROPDOWN_MAX_ITEMS = 30

TIME_ZONE = "UTC"

POLICY_FILES_PATH = '/etc/openstack-dashboard'

#ENFORCE_PASSWORD_CHECK = False


LOGGING = {
    'version': 1,
    # When set to True this will disable all logging except
    # for loggers specified in this configuration dictionary. Note that
    # if nothing is specified here and disable_existing_loggers is True,
    # django.db.backends will still log unless it is disabled explicitly.
    'disable_existing_loggers': False,
    # If apache2 mod_wsgi is used to deploy OpenStack dashboard
    # timestamp is output by mod_wsgi. If WSGI framework you use does not
    # output timestamp for logging, add %(asctime)s in the following
    # format definitions.
    'formatters': {
        'console': {
            'format': '%(levelname)s %(name)s %(message)s'
        },
        'operation': {
            # The format of "%(message)s" is defined by
            # OPERATION_LOG_OPTIONS['format']
            'format': '%(message)s'
        },
    },
    'handlers': {
        'null': {
            'level': 'DEBUG',
            'class': 'logging.NullHandler',
        },
        'console': {
            # Set the level to "DEBUG" for verbose output logging.
            'level': 'INFO',
            'class': 'logging.StreamHandler',
            'formatter': 'console',
        },
        'operation': {
            'level': 'INFO',
            'class': 'logging.StreamHandler',
            'formatter': 'operation',
        },
    },
    'loggers': {
        'horizon': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'horizon.operation_log': {
            'handlers': ['operation'],
            'level': 'INFO',
            'propagate': False,
        },
        'openstack_dashboard': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'novaclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'cinderclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'keystoneauth': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'keystoneclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'glanceclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'neutronclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'swiftclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'oslo_policy': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'openstack_auth': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'django': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        # Logging from django.db.backends is VERY verbose, send to null
        # by default.
        'django.db.backends': {
            'handlers': ['null'],
            'propagate': False,
        },
        'requests': {
            'handlers': ['null'],
            'propagate': False,
        },
        'urllib3': {
            'handlers': ['null'],
            'propagate': False,
        },
        'chardet.charsetprober': {
            'handlers': ['null'],
            'propagate': False,
        },
        'iso8601': {
            'handlers': ['null'],
            'propagate': False,
        },
        'scss': {
            'handlers': ['null'],
            'propagate': False,
        },
    },
}

# 'direction' should not be specified for all_tcp/udp/icmp.
# It is specified in the form.
SECURITY_GROUP_RULES = {
    'all_tcp': {
        'name': _('All TCP'),
        'ip_protocol': 'tcp',
        'from_port': '1',
        'to_port': '65535',
    },
    'all_udp': {
        'name': _('All UDP'),
        'ip_protocol': 'udp',
        'from_port': '1',
        'to_port': '65535',
    },
    'all_icmp': {
        'name': _('All ICMP'),
        'ip_protocol': 'icmp',
        'from_port': '-1',
        'to_port': '-1',
    },
    'ssh': {
        'name': 'SSH',
        'ip_protocol': 'tcp',
        'from_port': '22',
        'to_port': '22',
    },
    'smtp': {
        'name': 'SMTP',
        'ip_protocol': 'tcp',
        'from_port': '25',
        'to_port': '25',
    },
    'dns': {
        'name': 'DNS',
        'ip_protocol': 'tcp',
        'from_port': '53',
        'to_port': '53',
    },
    'http': {
        'name': 'HTTP',
        'ip_protocol': 'tcp',
        'from_port': '80',
        'to_port': '80',
    },
    'pop3': {
        'name': 'POP3',
        'ip_protocol': 'tcp',
        'from_port': '110',
        'to_port': '110',
    },
    'imap': {
        'name': 'IMAP',
        'ip_protocol': 'tcp',
        'from_port': '143',
        'to_port': '143',
    },
    'ldap': {
        'name': 'LDAP',
        'ip_protocol': 'tcp',
        'from_port': '389',
        'to_port': '389',
    },
    'https': {
        'name': 'HTTPS',
        'ip_protocol': 'tcp',
        'from_port': '443',
        'to_port': '443',
    },
    'smtps': {
        'name': 'SMTPS',
        'ip_protocol': 'tcp',
        'from_port': '465',
        'to_port': '465',
    },
    'imaps': {
        'name': 'IMAPS',
        'ip_protocol': 'tcp',
        'from_port': '993',
        'to_port': '993',
    },
    'pop3s': {
        'name': 'POP3S',
        'ip_protocol': 'tcp',
        'from_port': '995',
        'to_port': '995',
    },
    'ms_sql': {
        'name': 'MS SQL',
        'ip_protocol': 'tcp',
        'from_port': '1433',
        'to_port': '1433',
    },
    'mysql': {
        'name': 'MYSQL',
        'ip_protocol': 'tcp',
        'from_port': '3306',
        'to_port': '3306',
    },
    'rdp': {
        'name': 'RDP',
        'ip_protocol': 'tcp',
        'from_port': '3389',
        'to_port': '3389',
    },
}

#OPENSTACK_TOKEN_HASH_ALGORITHM = 'md5'


REST_API_REQUIRED_SETTINGS = ['OPENSTACK_HYPERVISOR_FEATURES',
                              'LAUNCH_INSTANCE_DEFAULTS',
                              'OPENSTACK_IMAGE_FORMATS',
                              'OPENSTACK_KEYSTONE_BACKEND',
                              'OPENSTACK_KEYSTONE_DEFAULT_DOMAIN',
                              'CREATE_IMAGE_DEFAULTS',
                              'ENFORCE_PASSWORD_CHECK']


OPERATION_LOG_ENABLED = True
OPERATION_LOG_OPTIONS = {
    'mask_fields': ['password'],
    'target_methods': ['POST'],
    'ignored_urls': ['/js/', '/static/', '^/api/'],
    'format': ("[%(client_ip)s] [%(domain_name)s]"
        " [%(domain_id)s] [%(project_name)s]"
        " [%(project_id)s] [%(user_name)s] [%(user_id)s] [%(request_scheme)s]"
        " [%(referer_url)s] [%(request_url)s] [%(message)s] [%(method)s]"
        " [%(http_status)s] [%(param)s]"),
}


ALLOWED_PRIVATE_SUBNET_CIDR = {'ipv4': [], 'ipv6': []}
```
```
chown root:apache /etc/openstack-dashboard/local_settings 
```
- Cấu hình firewalld 
```
firewall-cmd --add-service={http,https} --permanent 
firewall-cmd --reload 
 
```
-  Compress 
```
cd /usr/share/openstack-dashboard 
python manage.py compress 
```
-  Khởi động dịch vụ 
```
systemctl restart httpd memcached 
```

## Tham khảo: Tài liệu triển khai Openstack mô hình  HIGH AVAIBABILITY  
