# Đóng image Centos7 # 
## 1. Chuẩn bị ## 
- KVM host đã được cài đặt sẵn 
- Chuẩn bị file iso của Centos7 
- Sử dụng `Standard` với định dạng `ext4` cho phân vùng OS không sử dụng LVM 
- Sử dụng công cụ virt-manager kết nối tới console máy ảo
- Phiên bản Openstack sử dụng là Train 
- Time zone VietNam 

## Bước 1. Trên KVM host tạo máy ảo Centos7 ## 
- Bật `virt-manager` 
```
virt-manager
```
  - Trên virt-manager thực hiện các step sau để cài OS 
- Click chuột phải vào `QEMU/KVM` chọn `Details` 
- ![Atom]() 
- click vào tab `Storage` create một image mới 
- ![Atom](https://i.imgur.com/4juY54o.png) 

- Set tên và dung lượng cho image này là với định dạng .qcow2
- ![Atom](https://i.imgur.com/qx73VU4.png) 

- Quay lại màn hình chính của virt-manager và chọn create một VM mới 
- ![Atom](https://i.imgur.com/aSIwtLg.png) 

- Chọn boot từ file ISO 
- ![Atom](https://i.imgur.com/hxvjMg4.png) 

- Trỏ đường dẫn đến file ISO 
- ![Atom](https://i.imgur.com/whcpkLe.png) 

- Cấu hình RAM và CPU cho VM 
- ![Atom](https://i.imgur.com/zPybEQK.png) 

- Ở đây chúng ta sẽ chọn image `Centos7.qcow2` đã được tạo 
- ![Atom](https://i.imgur.com/3CAUftE.png) 

- Đặt tên cho VM, click vào `Customize...` để bổ sung các cấu hình khác 
- ![Atom](https://i.imgur.com/Jw22gvd.png) 

- Cấu hình cho CPU 
- ![Atom](https://i.imgur.com/fZy0Iw1.png) 

- Cấu hình boot, chỉnh lại menu boot để boot ISO cài đặt OS 
- ![Atom](https://i.imgur.com/2tY1z5O.png) 

- Cấu hình lại file ISO 
- ![Atom](https://i.imgur.com/gDXVFd9.png) 
- ![Atom]([Imgur](https://i.imgur.com/ZTre48i.png) 

- Cấu hình NIC mode `virtio` và bắt đầu quá trình cài đặt 
- ![Atom](https://i.imgur.com/oPsKX8t.png) 

- Cấu hình ngôn ngữ bạn chọn `English(English)` 
- ![Atom](https://i.imgur.com/cRi03sB.png) 

- Cấu hình timezone về Ho_Chi_Minh 
- ![Atom](https://i.imgur.com/ddyY4Y0.png) 

- Cấu hình disk để cài đặt 
- ![Atom](https://i.imgur.com/oPrLcUy.png) 
- ![Atom](https://i.imgur.com/xPn9afQ.png) 

- Chọn `Standard Partition` cho ổ disk 
- ![Atom](https://i.imgur.com/xPn9afQ.png) 

- Cấu hình mount point `/` cho toàn bộ disk 
- ![Atom](https://i.imgur.com/SXwxtka.png) 
- Định dạng lại `ext4` cho phân vùng 
- ![Atom](https://i.imgur.com/TMIGAa6.png) 
- ![Atom](https://i.imgur.com/38inbd9.png) 

- Kết thúc quá trình và confirm lại quá trình chia lại partition cho disk 
- ![Atom](https://i.imgur.com/38inbd9.png) 

- Cấu hình network 
- ![Atom](https://i.imgur.com/0OyMsbp.png) 

- Kết thúc quá trình cấu hình và bắt đầu quá trình cài đặt OS 

## 2.Cài đặt trên máy ảo ## 
- update package 
```
yum update -y   
```
- Cài đặt QEMU-guest-agent 
```
yum install -y qemu-guest-agent
systemctl enable qemu-guest-agent.service
systemctl start qemu-guest-agent.service
```
- Tắt firewalld 
```
systemctl disable firewalld
systemctl stop firewalld
```
- Cấu hình Selinux Policy cho phép QEMU và cloud-init hoạt động 
```
yum -y install policycoreutils-python 
yum reinstall -y selinux-policy-targeted
semanage permissive -a virt_qemu_ga_t
semanage permissive -a cloud_init_t
```
- Cài đặt acpid 
```
yum install -y acpid
systemctl enable acpid
```
- Cài đặt cloud-init và cloud-utils-growpart 
```
yum install -y epel-release 
yum install -y cloud-init cloud-utils cloud-utils-growpart dracut-modules-growroot
```
- Sau khi cài đặt cloud-init, thực hiện chỉnh một số cấu hình 
```
sed -i 's/disable_root: 1/disable_root: 0/g' /etc/cloud/cloud.cfg  
sed -i 's/ssh_pwauth:   0/ssh_pwauth:   1/g' /etc/cloud/cloud.cfg
sed -i 's/name: centos/name: root/g' /etc/cloud/cloud.cfg
```
- Để nhận hostname từ cloud-init, xóa file hostname 
```
rm -f /etc/hostname
```
- Cho phép VM có thể route với link local address 
```
echo "NOZEROCONF=yes" >> /etc/sysconfig/network
```
- Tắt Ipv6 net-module 
```
echo "net.ipv6.conf.all.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6 = 1" >> /etc/sysctl.conf
sysctl -p
```
- xóa cấu hình net-card mặc định 
```
rm -rf /etc/sysconfig/network-scripts/ifcfg-eth0
```
- cho phép interface tự động cấu hình card mạng và tự động up sau khi atach 
```
systemctl start NetworkManager
systemctl enable NetworkManager
```
- Để nova console-log có thể get log. CHỉnh sửa tại file /etc/default/grub, trên section `GRUB_CMDLINE_LINUX` thay thế `rhgb quiet` bằng `console=tty0 console=ttyS0,115200n8` 
```
GRUB_CMDLINE_LINUX="crashkernel=auto console=tty0 console=ttyS0,115200n8"
```
- Rebuild GRUB 
```
grub2-mkconfig -o /boot/grub2/grub.cfg
```
- Xóa cloud init info 
```
rm -rf /var/lib/cloud/*
```
- xóa log
```
yum clean all
rm -f /var/log/wtmp /var/log/btmp
rm /root/.bash_history; history -c 
```
- shutdown máy ảo 
```
poweroff
```

## 3.Trên KVM host ## 
- clean MAC Addr 
```
yum install /usr/bin/virt-sysprep
virt-sysprep -d centos7.0
```
- Giảm kích thước image 
```
virt-sparsify --compress /var/lib/libvirt/images/centos7.qcow2 centos7.0.img
```
- copy image lên controller 
```
scp centos7.0.img root@controller:/root/
```
## 4.Trên controller ## 
```
- glance image-create --name centos7.0 \
--disk-format qcow2 \
--min-disk 5 \
--container-format bare \
--file /root/centos7.0.img \
--visibility=public \
--property hw_qemu_guest_agent=yes \
--property hw_disk_bus=scsi \
--property hw_scsi_model=virtio-scsi --progress
```
