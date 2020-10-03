# Khôi phục MariaDB Galera Cluster khi bị Partial hoặc Full Crash 
- Galera là một multi-master cluster cho MariaDB sao chép dữ liệu bằng cách sử dụng syschronous. Glera cho phép bất kì node nào trong cluster hoạt động như master và write vào bất kì node nào trong cùng một thời điểm. Cấu hình active-active của Galera cluster cung cấp nhiều hơn cơ chế load balancing và khả năng chịu lỗi nhiều hơn vì không có failover 
- Mặc dù sau khi được cấu hình, một Galera cluster có thể hoạt động mà không gặp nhiều vấn đề, nhiều lỗi node có thể gây ra cluster không hoạt động được. Bất cứ điều gì cũng có thể gây ra một node ngừng giao tiếp với cụm. Có thể là do shutdown, stop dịch vụ không đúng cách, mất điện. Ở đây thì nên có một node đứng vững để duy trì cụm .
- ở dưới dây là các trường hợp gặp phải 

## CLuster Crash Scenarios ## 
- Tùy thuộc vào số lượng node bị lỗi, ta có thể phân ra một số trường hợp như sau: 
  - Single node failure 
  - Multi-node failure 
  - Full cluster failure 
  
- Trong các trường hợp trên thì FUll cluster failure là nguyên nhân gây ra mất kết nối đến database.TRường hợp bình thường thì các node sẽ cần khởi động lại do cập nhật và bảo trì thường xuyên. Khi một node được khởi dộng lại một cách tốt đẹp thì nó chỉ cần tham join lại vào cụm và tự đồng bộ hóa với các node còn lại của cluster. 

## Verify helthy CLuster ## 
- Điểu quan trọng là cần kiểm tra một cluster Galera có thực sự là ổn định hay không một khi lỗi xảy ra và điều nhanh nhất đó là kiểm tra trạng thái của cụm. Một cụm thực sự coi là khỏe mạnh thì sẽ xuất hiện 3 node như sau trên shell database 
```
MariaDB [(none)]> show status like 'wsrep_incoming_addresses';
+--------------------------+----------------------------------------------+
| Variable_name | Value |
+--------------------------+----------------------------------------------+
| wsrep_incoming_addresses | 10.0.0.51:3306,10.0.0.52:3306,10.0.0.53:3306 |
+--------------------------+----------------------------------------------+
1 row in set (0.01 sec)
```

- Để kiểm tra tổng số node hiện tại là member của cluster 
```
MariaDB [(none)]> show status like 'wsrep_cluster_size';
+--------------------+-------+
| Variable_name | Value |
+--------------------+-------+
| wsrep_cluster_size | 3 |
+--------------------+-------+
1 row in set (0.01 sec)
```
- Để truy xuất UUID của cụm 
```
MariaDB [(none)]> show status like 'wsrep_cluster_state_uuid';
+--------------------------+-----------------------------------------+
| Variable_name | Value |
+--------------------------+-----------------------------------------+
| wsrep_cluster_state_uuid | 345098abd2-291a-9893-acbd3-30923abcdef9 |
+--------------------------+-----------------------------------------+
1 row in set (0.01 sec)
```

- Kiểm tra xem node member có dược đồng bộ với cụm hay không 
```
MariaDB [(none)]> show status like 'wsrep_local_state_comment';
+---------------------------+--------+
| Variable_name | Value |
+---------------------------+--------+
| wsrep_local_state_comment | Synced |
+---------------------------+--------+
1 row in set (0.01 sec)
```

## Single Node Failure Recovery 
- Trong trường hợp này, chỉ một node trong cluster bị lỗi. Không có dữ liệu nào bị lỗi cũng như database không kết nối được.Một node bị lỗi trong 3 node thì trạng thái sẽ xuất hiện như dưới đây 
```
 MariaDB [(none)]> show status like 'wsrep_incoming_addresses';
+--------------------------+-------------------------------+
| Variable_name | Value |
+--------------------------+-------------------------------+
| wsrep_incoming_addresses | 10.0.0.51:3306,10.0.0.52:3306 |
+--------------------------+-------------------------------+
1 row in set (0.01 sec)
```

```
MariaDB [(none)]> show status like 'wsrep_cluster_size';
+--------------------+-------+
| Variable_name | Value |
+--------------------+-------+
| wsrep_cluster_size | 2 |
+--------------------+-------+
1 row in set (0.01 sec)
```

- Sau khi khởi động lai node bị lỗi, chúng ta cần kiểm tra lại trạng thái để đảm bảo rằng nó đã join vào cluster. Nếu như kiểm tra không có, chỉ cần restart lại MariaDB
```
systemctl restart mariadb
```

## Multi-node Crash Recovery 
- Trong trường hợp này , tất cả các node ngoại trừ một node còn sống, nó sẽ gây ra hiện tượng lỗi của cơ chế quorum(Bởi số node sống lớn hơn 1) . Trong trường hợp này thì Galera cluster không thể xử lí được những request của SQL. Vì một node vẫn đang ở trạng thái hoạt động nên không có dữ liệu nào bị mất. Nếu như các node khác không up trở lại thì không thể join vào được cluster, bởi vì cluster không tồn tại. Chúng ta có thể sử dụng cluster size và cluster status command từ node còn sống. 
```
MariaDB [(none)]> show status like 'wsrep_cluster_size';
+--------------------+-------+
| Variable_name | Value |
+--------------------+-------+
| wsrep_cluster_size | 1 |
+--------------------+-------+
1 row in set (0.01 sec)
```

```
MariaDB [(none)]> show status like 'wsrep_cluster_status';
+----------------------+---------+
| Variable_name | Value |
+----------------------+---------+
| wsrep_cluster_status | Primary |
+----------------------+---------+
1 row in set (0.01 sec)
```

- Trong trường hợp đặc biệt, giá trị của cluster status có thể hiển thị là **non-Primary**. Nếu như có thể hiển thị là lỗi đó thì không chỉ là lỗi về quorum mà là lỗi về network . Đảm bảo rằng cluster status trả về được giá trị **Primary** trước khi đảm bảo cơ chế quorum
- Trước khi có thể tiếp tục với quorum, chúng ta phải đảm bảo rằng node còn sống thực sự có latest commits mới nhất. Chúng ta có thể kiểm tra bằng cách xem nội dung.Có hai cách đặt lại quorum để các node khác có thể tham gia vào cluster. 

### Automatic Bootstrap ### 
- Cách đon giản nhất có thể reset quorum đó là bootstrap tự động. Chúng ta có thể sử dụng command trong shell database để tự dộng bootstrap node
```
MariaDB [(none)]> set global wsrep_provider_options='pc.bootstrap=YES';
```
- Điều này sẽ bootstrap node còn sống trở thành node primary để các node khác lỗi có thể join vào cluster

### Manual Bootstrap ### 
- Chạy các command sau để bootstrap các node theo cách thủ công 
```
$ systemctl stop mariadb
$ galera_new_cluster
$ systemctl restart mariadb
```
- Sau khi node primary up và chạy, restart lại MariaDB service trên tất cả các node còn lại tại cùng một thời điểm. 


## Full Cluster Recovery ## 
- Trong trường hợp này , tất cả các node bị down một cách không tốt đẹp,lúc này thì cluster sẽ không nhận bất kì một request nào. Sau khi đó dữ liệu có thể bị crash,và nếu tất cả các node có thể up trở lại thì service MariaDB cũng không thể hoạt động. Điều này có thể là do mất nguồn điện hoặc shutdown một cách không tốt đẹp và không có node nào có thể thực hiện được last commit. Một cụm Galera có thể gặp sự cố theo nhiều cách khác nhau đến đễn các phương pháp khác nhau để khôi phục . 

### Recovery Based On Highest sepno Value ### 
- Phương pháp này hữu ích khi có ít nhất một node shutdowwn một cách tốt đẹp .Node có dữ liệu mới nhất sẽ có giá trị sepno cao nhất trong tất cả các node bị lỗi. CHúng ta có thể tìm trong file `/var/lib/mysql/grastate.dat` sẽ hiển thị giá trị sepno. Tùy thuộc vào tính chất của sự cố gặp phải mà tất cả các node sẽ có giá trị sepno âm (-1)giống nhau hoặc một trong các node sẽ có giá trị sepno dương cao nhất. 
- Dưới đây là hiển thị của file grastate.dat trong node 3. node này có sepno âm và không có ID nhóm. Trường hợp này xảy ra khi node crash trong quá trình xử lí Data Definition Language(DDL)
```
$ cat /var/lib/mysql/grastate.dat
# GALERA saved state
version: 2.1
uuid: 00000000-0000-0000-0000-000000000000
seqno: -1
safe_to_bootstrap: 0
```
- Dưới là hiển thị node2. Node này crash trong quá trình xử lí dẫn đến sepno âm và có ID nhóm
```
$ cat /var/lib/mysql/grastate.dat
# GALERA saved state
version: 2.1
uuid: 886dd8da-3d07-11e8-a109-8a3c80cebab4
seqno: -1
safe_to_bootstrap: 0
```
- Cuối cùng là node 1 với giá trị sepno cao nhất 
```
$ cat /var/lib/mysql/grastate.dat
# GALERA saved state
version: 2.1
uuid: 886dd8da-3d07-11e8-a109-8a3c80cebab4
seqno: 31929
safe_to_bootstrap: 1
```

- Lưu ý rằng một node có giá trị sepno cao nhất khi node đó được shutdown một cách tốt đẹp.Node này cần được khôi phục đầu tiên. 
- Nếu tất cả các node chứa giá trị -1 cho sepno và 0 cho `safe_to_bootstrap`,đó là dấu hiệu dữ liệu đã bị cash. Taị thời điểm này thì chúng ta có thể sử dụng bẳng cách `galera_new_cluster`.Nhưng cách này không được khuyến khích cho lắm vì mỗi node có thể sẽ có một bản copy dữ liệu database. 
- Trước khi khởi động lại 1 node chúng ta cần thực hiện thay đổi trong tệp /etc/my.cnf.d/servẻ.cnf và loại bỏ đi các ip trong cụm. Dưới là phần cấu hình ban đầu 
```
[galera]
# Mandatory settings
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
wsrep_cluster_address="gcomm://10.0.0.51,10.0.0.52,10.0.0.53"
wsrep_cluster_name='galeraCluster01'
wsrep_node_address='10.0.0.51'
wsrep_node_name='galera-01'
wsrep_sst_method=rsync
binlog_format=row
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
```
- Lưu ý rằng `wrep_cluster_address` hiển thị ip của tất cả member. Chúng ta cần remove đi địa chỉ ip đó 
```
wsrep_cluster_address="gcomm://"
```
- Restart lại MariaDB trên các node 
```
$ systemctl restart mariadb
```
- Sau khi xác đinh rằng các service khởi động thành công, chúng ta có thể tiền hành khởi động lại lần lượt các servie trên các node khác. Chỉ sau khi các node up thành công,cần chỉnh sửa lại cấu hình file trên node 1 để thêm địa chỉ IP của các node thành viên và khởi động lại cluster 
```
wsrep_cluster_address = "gcomm: //10.8.8.53,10.8.8.54,10.8.8.55"
```
- Cụm Galera sẽ hoạt dộng trở lại tại thời điểm này và tất cả các node sẽ được bồng bộ hóa với node còn sống. 


### Recovery Based On Last Committed 
- Đây là trường hợp xấu nhất xảy ra khi cluster Galera trong đó tất cả các node bị crash dữ liệu hoàn toàn và dẫn đến giá trị seqno là -1. Như đề cập trước đó, việc sử dụng command `galera_new_cluster` là không khuyến khích trên 1 node sau đó join vào cluster trên các node còn lại, trước khi kiểm tra xem node nào là latest commit. Khi command `galera_new_cluster` được sử dungjm nó có thể tạo ra một cụm mới với ID mới sau đó các node khác sẽ join vào cluster và bắt đầu đồng bộ hóa dữ liệu mới khác với dữ liệu cluster cũ. 
- Để kiểm tra xem node nào là latest commit, chúng ta có thể kiểm tra giá trị `wsrep_last_commit` trên từng node. Node nào có giá trị cao nhất là node có latest commit mới nhất. Chúng ta có thể khởi động node đó đẻ bắt đuầ cụm sau đó join các node khác. Quá trình này tương tự như khởi động node có giá trị sepno cao nhất như chúng ta thấy ở phần trước. 
- Stop dịch vụ MariaDB
```
$ systemctl dừng mariadb
```
- CHỉnh sửa wsrep_cluster_address trong file /etc/my.cnf.d/server.cnf
```
wsrep_cluster_address = "gcomm: //"
```
- Restart lại service MariaDB
```
$ systemctl start mariadb
```
- Từ database shell, kiểm tra giá trị last commit
```
MariaDB [(none)]> show status like 'wsrep_last_committed';
+----------------------+---------+
| Variable_name | Value |
+----------------------+---------+
| wsrep_last_committed | 319589 |
+----------------------+---------+
1 row in set (0.01 sec)
```
- Lặp lại quá trình trên tất cả các node để lấy giá trị last commit cuối cùng. Node có dữ liệu mới nhất sẽ có giá trị cao nhất.Tạo một cluster mới trên node có giá trị cao nhất 
```
$ galera_new_cluster
```
- Thay đổi giá trị `wsrep_cluster_address` thêm các địa chỉ ip của các node còn lại , sau đó khởi động lại service MariaDB. 
