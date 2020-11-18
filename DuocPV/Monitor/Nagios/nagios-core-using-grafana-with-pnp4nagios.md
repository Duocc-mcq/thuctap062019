# Nagios Core - Sử dụng Grafana với PNP4Nagios 
- PNP4Nagios là một tiện ích bổ sung cho Nagios phân tích dữ liệu cũng như hiệu suất được cung cấp bởi các Plugin và lưu trữ chúng vào database-RRD(Round Robin Database).Nó hiển thị dưới dạng đồ thị đối với người dùng dễ quản lý. 
## Cài đặt PNP4Nagios ## 
- Trước khi cài đặt PNP4Nagios chúng ta cần cài đặt môi trường và các plugin với Nagioscore tại [đây](https://github.com/Duocc-mcq/thuctap062019/blob/master/DuocPV/Monitor/Nagios/2.Install%20Nagios.md)
- Cài đặt các package cần thiết với sử dụng yum 
```
yum install rrdtool rrdtool-perl perl-Time-HiRes perl-GD -y
```
- Dowload phiên bản PNP4Nagios mới nhất từ  http://www.pnp4nagios.org/
- sau khi dowload ta tiến hành giải nén 
```
tar xvf pnp4nagios-0.6.25.tar.gz
```
- Đi tới thư mục PNP4Nagios đã giải nén, và tiến hành cài đặt nó. 
```
cd pnp4nagios-0.6.25
./configure
make all
make fullinstall
```
- Chỉnh sửa file `nagios.cfg` và command (#) tất cả các thông số hiệu suất 
```
vi /usr/local/nagios/etc/nagios.cfg
```
- Thêm các tham số sau vào file `nagios.cfg`
```
process_performance_data=1
# service performance data
service_perfdata_file=/usr/local/pnp4nagios/var/service-perfdata
service_perfdata_file_template=DATATYPE::SERVICEPERFDATA\tTIMET::$TIMET$\tHOSTNAME::$HOSTNAME$\tSERVICEDESC::$SERVICEDESC$\tSERVICEPERFDATA::$SERVICEPERFDATA$\tSERVICECHECKCOMMAND::$SERVICECHECKCOMMAND$\tHOSTSTATE::$HOSTSTATE$\tHOSTSTATETYPE::$HOSTSTATETYPE$\tSERVICESTATE::$SERVICESTATE$\tSERVICESTATETYPE::$SERVICESTATETYPE$
service_perfdata_file_mode=a
service_perfdata_file_processing_interval=15
service_perfdata_file_processing_command=process-service-perfdata-file

# host performance data
host_perfdata_file=/usr/local/pnp4nagios/var/host-perfdata
host_perfdata_file_template=DATATYPE::HOSTPERFDATA\tTIMET::$TIMET$\tHOSTNAME::$HOSTNAME$\tHOSTPERFDATA::$HOSTPERFDATA$\tHOSTCHECKCOMMAND::$HOSTCHECKCOMMAND$\tHOSTSTATE::$HOSTSTATE$\tHOSTSTATETYPE::$HOSTSTATETYPE$
host_perfdata_file_mode=a
host_perfdata_file_processing_interval=15
host_perfdata_file_processing_command=process-host-perfdata-file
```
- Sau đó chỉnh sửa `commands.cfg` 
```
vi /usr/local/nagios/etc/objects/commands.cfg
```
- Thêm command vào file `commands.cfg`
```
define command{
       command_name    process-service-perfdata-file
       command_line    /bin/mv /usr/local/pnp4nagios/var/service-perfdata /usr/local/pnp4nagios/var/spool/service-perfdata.$TIMET$
}

define command{
       command_name    process-host-perfdata-file
       command_line    /bin/mv /usr/local/pnp4nagios/var/host-perfdata /usr/local/pnp4nagios/var/spool/host-perfdata.$TIMET$
}
```
- Start và enable `npcd` service 
```
/usr/local/pnp4nagios/bin/npcd -d -f /usr/local/pnp4nagios/etc/npcd.cfg
systemctl enable npcd.service
```
- Restart `nagios` và `httpd` service 
```
systemctl restart nagios.service
systemctl restart httpd.service
```
- Tiếp đến remove file ` /usr/local/pnp4nagios/share/install.php`
```
rm -f /usr/local/pnp4nagios/share/install.php
```
- Sau khi hoàn thành ta kết nối đến địa chỉ http://192.168.100.50/pnp4nagios
- ![Atom](https://i.imgur.com/kd8GGE6.png)


## 2. Cài đặt Grafana trên Centos7 
- Khởi tạo Repo cho Grafana 
```
cat <<EOF> /etc/yum.repos.d/grafana.repo
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF
```
- Cài đặt Grafana 
```
sudo yum install grafana
yum install fontconfig
yum install freetype*
yum install urw-fonts
```
- Khởi động service 
```
systemctl start grafana-server
systemctl status  grafana-server
```
- Cấu hình firewalld 
```
firewall-cmd --zone=public --permanent --add-port=3000/tcp
firewall-cmd --reload
```
- Truy cập địa chỉ  http://IP:3000 với tài khoản và mật khẩu mặc định `admin-admin`. 

## 3.Cài đặt PNP4Nagios trong Grafana 
- Thực thi những câu lệnh dưới để cài đặt PNP4Nagios cho Grafana
```
grafana-cli plugins install sni-pnp-datasource
cd /usr/local/pnp4nagios/share/application/controllers/
wget -O api.php "https://github.com/lingej/pnp-metrics-api/raw/master/application/controller/api.php"
```
- Restart lại `grafana-server` service 
```
systemctl restart grafana-server.service
```
- Cấp quyền localhost cho PNP4Nagios 
- Lệnh sau sẽ thêm user grafana với mật khẩu `ANAFARG` tới file `htpasswd.users`
```
htpasswd -b /usr/local/nagios/etc/htpasswd.users grafana ANAFARG
```
- Thêm command vào file `/etc/httpd/conf.d/pnp4nagios.conf` 
```
sed -i '/Require valid-user/a\        Require ip 192.168.100.50 ::1' /etc/httpd/conf.d/pnp4nagios.conf
```
- Restart lại httpd service 
```
systemctl restart httpd.service
```
## 4. Grafana configure
- Grafana cần được cấu hình để sử dụng API PNP4Nagios. Mở trình duyệt web của bạn và nhập 
- http://nagios_server:3000

- Thay thế `nagios_server` với địa chỉ Ip của Nagios core server máy 
- Bạn cần login với page, và mặc định username và password là admin như trên 
- Sau đó ta cần **Add data source** trong **setting** và search `PNP` ta sẽ thấy như hình 
- ![Atom](https://i.imgur.com/um2Ev1A.png)

- Bạn sẽ điền các thông tin sau 
 - HTTP 
   - URL: http://localhost/pnp4nagios
   - Access: proxy
- ![Atom](https://i.imgur.com/7Qq5xvt.png)

## 5.Create Dashboard + Graph
- 