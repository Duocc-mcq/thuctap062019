# Split Brain # 
- Tình trạng mất liên lạc giữa các cluster node gây ra bởi các kết nối mạng được gọi là network partiton, có thể dẫn đến split brain (phân chia brain). Split brain là một thuật ngữ trong máy tính, nó thực sự đau đầu đố với mỗi một quản trị viên mỗi khi giải quyết vấn đề liên quan đến split brain
- Split brain có thể xảy ra nếu các node đang hoạt động mất kết nối đồng bộ và heartbeat kết nối giữa chúng cùng một lúc và không thể giao tiếp nữa để xác định trạng thái đồng bộ hóa của các node đối tác. Split brain chỉ ra sự mâu thuẫn về dữ liệu hoặc tính khả dụng 
- Split brain có thể dẫn đến lỗi nghiêm trọng không thể sửa chữa.Trong trường hợp mỗi node được coi là primary, thì bạn không thể đồng bộ hóa lại các node vì không có cách nào để "mearge" thông tin trên mỗi node để tạo một storage single HA duy nhất có thông tin chính xác. Trong thực tế transactions commmitted trong bộ nhớ "wrong primary" trong tính huống primary dual bị mất. 
- Vì thế khi xư lí network partiton, ưu tiên đầu tiên đó là loại bỏ split brain và data corruption 
- Có một số cách tiếp cận giải quyết vấn đề sau khai network partiton xảy ra. 

  - Cách đầu tiên đó là "optimistic". Quản trị viên cần khôi phục lại channel giao tiếp giữa các node và cho phép các node được partiton hoạt động bình thường sau một thời gian,với hi vọng chúng sẽ tự đồng bộ hóa sau một thời gian. Nhưng đây là cách tiếp cận không tối ưu. Bởi dữ liệu có thể bị hỏng do Split brain. 
  - Cách thứ hai đó là "pessimistic". Người quản trị viên phải bỏ đi tính availability để dành cho sự nhất quán dữ liệu (data consistency). Khi một network partiton được detected, thì việc truy cập và các sub-partiton bị giới hạn để duy trì tính nhất quán. Trường hợp này thì chỉ một số thành phần mới có thể thực hiện những request read/write đến storage , tránh trường hợp history phân kì. 
  
  - Ở đây thì các cụm HA mục đích thương mại điện tử thường đưuọc sử dụng kết hợp các kết nối mạng heartbeat giữa cluster host và quorum witness storage. 
  
  
## Heartbeat strategy hoặc hand on the pulse
- Heartbeat là một cơ chế nâng cao được sử dụng để tránh hỏng dữ liệu trong trương hợp channel đồng bộ hóa bị lỗi. Nếu dữ liệu không thể truyền qua kênh đồng bộ hóa. StarWind VSAN sẽ cố gắng ping các kết nối đối tác bằng các liên kết heartbeat được cung cấp. Nếu các node đối tác không phản hồi. StarWind VSAN giả sử rằng chúng đang offile. Trong trường hợp này, StarWind VSAN đánh dấu các node khác là không được đồng bộ hóa và tất cả thiết bị HA trên node hiện tại sẽ chuyển dữ liệ từ bộ nhớ cache vào đãi và tiếp tục hoạt động ở chế độ ghi vào bộ đệm. Điều này được thực hiện để bảo toàn tính toàn vẹn của dữ liệu trong trương hợp node ngừng hoạt động đột ngột. Nếu ping heartbeat thành công.StarWind sẽ chặn các node có mưc độ ưu tiên thấp hơn cho đến khi các kênh đồng bộ hóa được thiết lập lại. 
- Nói chung thì chức năng này dựa trên việc kiểm tra liên tục của những node tham gia vào cụm. 
- Trong trường hợp có hai node HA. Khi thực hiện request đến node đầu tiên, dữ liệu được gửi đến node đối tác thông qua kênh đồng bộ sync. Trong trường hợp Sync bị lỗi, dữ liệu không thể được gửi đến node đối tác nào, do đó các thiết bị không thể đồng bộ hóa. Node Primary bắt đầu kiểm tra trạng thái hệ thống. Cả hai node đều được đồng bộ hóa, hai node đều trong điều kiện bình đẳng. Do đó, node có mức độ ưu tiên chính vẫn hoạt động , node mức độ ưu tiên thứ cấp trên nên "không được đồng bộ hóa" và ngừng nhận và phản hồi các yêu cầu của  client. Node Secondary kiểm tra định kì tính đồng bộ . Ngay khi có sẵn, quá trình đồng bộ hóa bắt đầu, node Secondary sẽ hoạt động và cho phép client kết nối đến. 
![Atom](https://i.imgur.com/XbpECSN.png)

- Lợi ích: Thiết bị sẽ vẫn hoạt động ngay  cả khi chỉ có một node còn sống 
- Bất lợi: Trong trường hợp tất cả các kênh kết nối giữa các node bị lost, nhưng kết nối của các node tới client vẫn được duy trì, thì tình trạng split brain sẽ được phát sinh. Vấn đề này sẽ được cấu hình với heartbeat kết nối thông qua các kênh kết nối tương tự để sử dụng kết nối đến client. 

## Node majority strategy hoặc heartbeat life support
- Một cách khác đối với vấn đề split brain đó là thêm một node "witness" (nhân chứng). 
- Node witness được gọi là "router" giữa các node chính và nó được sủ dụng để lấy thông tin về trạng thái node đối tác trong trường hợp mất kết nối trực tiếp giữa các node 1 và node 2. Nó có thể được triển khai như một phiên bản StarWind riêng biệt trên cloud hoặc trên máy chủ vật lý, được kết nối với các node lưu trữ. Node witness không tham gia vào việc trao đổi dữ liệu và không có sẵn cho thiết bị client. Với sao chép 3 node, node witness không yêu cầu cùng một lượng bộ nhớ đẻ dữ liệu sao  chép và chỉ giữ cài đặt cụm lưu trữ. Node này sẽ có một cuộc votes, vì thế nó sẽ tham gia votes quorum khi xem xét network partiton nào là primary
![Atom](https://i.imgur.com/FtlA86D.png)

- ở đây chúng ta có hai node HA lưu trữ. Ví dụ các yêu cầu của client được chuyển từ node1 sang node2 qua kênh đồng bộ hóa. Trong trường hợp kênh Sync bị lỗi, các thiết bị không thể đồng bộ hóa . Trong trường hợp này, witness node được kết nối với cả hai node lưu trữ, sẽ chiếm phần lớn với node1 vì nó có dữ liệu phù hợp nhất. Do đó, node2 trở nên :không đồng bộ hóa được và ngừng nhận và phản hồi các yêu cầu của khách hàng. Bây giờ, có một nhân chứng, bạn có thể mất một node và giữ nguyên số quorum. Ngay cả khi một node không hoạt động, cụm vẫn hoạt động. Ngay sau khi kênh đồng bộ khả dụng trở lại, quá trình đồng bộ hóa sẽ bắt đầu. Sau khi đồng bộ hóa thành công. Node2 sẽ hoạt động trở lại và cho phép các kết nối đến client. 
![Atom](https://i.imgur.com/P55CqEj.png)
- Vì thế, khi có lượng số node bằng chẵn, thì việc triển khai quorum witness là bắt buộc. Nhưng đẻ giữ đa số vote majority , khi bạn có một số node lẻ, bạn không nên triển khai quorum witness. 
- Kết luận: Vì thế , chúng ta có thể chia ra. Các thành viên trong cụm không thể xác định các nào sẽ hoạt động trong trường hợp tất cả các network interfaces bị lỗi. Node witness là một cách được chấp thuận để giữ cho cụm lưu trữ hoạt động bình thường trong những tình huống như vậy và vô hiệu hóa cơ hội split brain. Do đó chúng ta có thể chỉ ra những thuận và bất lợi của việc triển khai cấu hình node majority.
  - thuận tiện: 
     - Khả năng split brain là hoàn toàn bị loại trừ. 
	 - Không yêu cầu một kênh liên lạc bổ sung cho heartbeat
  - Bất lợi 
     - Trong cấu hình có hai node lưu trữ, cần có node thứ 3
	 - Trong trường hợp nếu có một thiết bị HA bao gồm node lưu trữ hoặc hai node lưu trữ và một node witness thì chỉ có thể vô hiệu hóa một node. Trong trường hợp hỏng hai node, node thứ 3 cũng sẽ ngừng hoạt động. 
