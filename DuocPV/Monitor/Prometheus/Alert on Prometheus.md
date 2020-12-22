# Alert trong Prometheus 
- Hoạt động cảnh báo trong hệ thống Prometheus chia làm 2 phần 
  - Phần 1: các rule cảnh báo được thiết lập ở Prometheus Server và gửi cảnh báo đó đến Alertmanager 
  - Phần 2: Alertmanager sẽ quản lý các cảnh báo (alert) ,xử lý nội dung alert nếu có tùy biến này và điều hướng đầu tiếp nhận thông tin cảnh báo như email, chat platform,call,...
- Các bước chính để cài đặt và thiết lập thông báo là 
  - Cài đặt và cấu hình Alertmanager 
  - Cấu hình Prometheus nói chuyện với Alertmanager 
  - Tạo alert rules trong Prometheus 

## Alertmanager 
- ![Atom](https://i.imgur.com/zrirhAY.png) 

- Alertmanager xử lý cảnh báo được gửi bới các ứng dụng khách như Prometheus server. 

## Grouping 
- Nó phân loại cảnh báo có tính chất tương tự nhau thành một thông báo đồng nhất. Điều này hữu ích khi nhiều hệ thống cùng lỗi một lúc và hàng trăm hàng nghìn cảnh báo có thể được kích hoạt cùng một lúc. 
- Ví dụ : một hệ thống với  nhiều server mất kết nối đến cơ sở dữ liệu, thay vì rất nhiều cảnh báo được gửi về Alertmanager thì Grouping giúp cho việc giảm số lượng cảnh báo trùng lặp, thay vào đó là một cảnh báo để chúng ta có thể biết được chuyện gì đang xảy ra đối với hệ thống của bạn. 

## Inhibition 
- Là một khái niệm về việc ngăn chặn thông báo cho một số cảnh báo nhất định nếu các cảnh báo khác đã được kích hoạt. 
- Ví dụ: một cảnh báo đang kích hoạt, thông báo là cluster không thể truy cập(not areachable). Alertmanager có thể được cấu hình là tắt các cảnh báo khác liên quan đến cluster này nếu cảnh báo đó đang kích hoạt. Điều này lọc bớt những cảnh báo không liên quan đến vấn đề hiện tại.

## Silence 
- Là một cách đơn giản để tắt cảnh báo trong một khoảng thời gian nhất định. Nó được cấu hình cơ bản dựa trên matcher, giống như routing tree. Nếu nó match với các điều kiện thì sẽ không có cảnh báo nào được gửi khi đó. 

## Hight avability 
- Alertmanager hỗ trợ cấu hình để tạo một cluster với độ khả dụng cao. 
_ Nhưng nó không cân bằng tải giữa Prometheus và Alertmanager,nhưng thay vào đó hãy trỏ Prometheus đến tất cả danh sách của Alertmanager 


### 2.Alert rules
- Alert rule cho phép bạn xác định các điều kiện đưa ra cảnh báo, dựa trên language expressions và gửi thông báo đi. 
```
ALERT <alert name>
  expr <expression>
  [ FOR <duration> ]
  [ LABELS <label set> ]
  [ ANNOTATIONS <label set> ]
```
  - expr: Biểu thức, điều kiện để đưa ra cảnh báo

  - FOR: Chờ đợi trong 1 khoảng thời gian để đưa ra cảnh báo

  - LABELS: Đặt nhãn cho cảnh báo

  - ANNOTATIONS: Chứa thêm các thông tin cho cảnh báo
- ví dụ 
```
# Alert for any instance that is unreachable for >5 minutes.
ALERT InstanceDown
  expr up == 0
  FOR 5m
  LABELS { severity = "page" }
  ANNOTATIONS {
    summary = "Instance {{ $labels.instance }} down",
    description = "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes.",
  }

# Alert for any instance that have a median request latency >1s.
ALERT APIHighRequestLatency
  expr api_http_request_latencies_second{quantile="0.5"} > 1
  FOR 1m
  ANNOTATIONS {
    summary = "High request latency on {{ $labels.instance }}",
    description = "{{ $labels.instance }} has a median request latency above 1s (current value: {{ $value }}s)",
  }
```

- Kiểm tra các cảnh báo đang hoạt động 
  - Web Prometheus có tab Alerts hiển thị thông tin các alert đang hoạt động.
- Alert 2 chế độ (Pending và Firing), chúng cũng được lưu dưới dạng time series:
```
ALERTS{alertname="<alert name>", alertstate="pending|firing", <additional alert labels>}
```
- Nếu alert hoạt động sẽ có 1 value là 1, không hoạt động thì value là 0 

- **Gửi thông báo** 
- Để gửi thông báo đi ta cần tới thành phần Alermanager 