# Nguyên nhân dẫn đến lỗi cơ sở dữ liệu trên WordPress

Một web site wordpress đang chạy bình thường bỗng nhiên bị lỗi "`Lỗi kết nối tới cơ sở dữ liệu`" hoặc "`Error establishing a database connection`". Tức là wordpress đã không còn có thể kết nối tới cơ sở dữ liệu.

![Imgur](https://i.imgur.com/BGYGLCJ.png) 

Trong trường hợp này trước tiên, kiểm tra trạng thái của mysql hoặc mariadb : 

![Imgur](https://i.imgur.com/jApAKkp.png)

ta thấy rằng cơ sở dữ liệu mysql đã bị tắt. Trong trường hợp này bạn có thể khởi động lại cơ sở dữ liệu sẽ chạy lại bình thường. 

Nhưng còn nguyên nhân gây ra lỗi ? 

Hôm nay mình sẽ mô tả lại cách gây ra lỗi và truy tìm nguyên nhân gây ra lỗi cơ sở dữ liệu với mục đích mô phỏng lại cuộc tấn công vào cơ sở dữ liệu trang wordpress. 

>Lưu ý: Không nên thử đối với các trang web thuộc các tổ chức, cá nhân khác mà không được sự cho phép của họ. Chỉ nên thử đối với trang web của bạn hoặc do bạn quản lý. 


## 1. Môi trường : 

Đối với máy chủ sử dụng để chạy wordpress và maridb hoặc mysql, mình sử dụng CentOS 7. 

Tạo máy ảo với cấu hình nhỏ nhất để dễ dàng bench sập cơ sở dữ liệu trang web hơn. 
Mình sử dụng hệ điều hành CentOS 7 với cấu hình 1 core, 512 MB Ram, 10 G Disk. 

Với tool sử dụng để bench trang web, mình có để 1 số tools python [tại đây](https://github.com/hungviet99/thuc_tap/tree/master/Ghi_chep_python/Tools/Tool_bench_website). Sử dụng 1 trong các tool để bench trang web của bạn. 

Sử dụng trang web thông thường, không qua reverse proxy. 

## 2. Bắt đầu : 

Để bắt đầu, mình sẽ sử dụng tool cho web http, và thực hiện bench trang web. 

Cho đến khi trang web xuất hiện như sau :

![Imgur](https://i.imgur.com/heLVoJj.png)

sau đó, kiểm tra trạng thái cơ sở dữ liệu : 

![Imgur](https://i.imgur.com/jApAKkp.png)

Ta thấy răng cơ sở dữ liệu mysql đã bị tắt. 

Tiếp theo ta kiểm tra nguyên nhân dẫn đến cơ sở dữ liệu bị dừng : 

Sử dụng lệnh `dmesg` 

![Imgur](https://i.imgur.com/zKS8Uyx.png) 

Ta thấy rằng có 1 cảnh báo là có quá nhiều cờ SYN được gửi đến trên cổng 80. Đây có thể là 1 cuộc tấn công DoS. Dẫn đến hết ram và tiến trình của mariadb đã bị kill. 

Tiến hành vào file `/var/log/messages` để xác định được thời gian tiến trình mysql bị kill. 

```
tailf -n 500 /var/log/messages
```

![Imgur](https://i.imgur.com/rMiDet6.png)

Ta thấy rằng thời gian mà mysql bị kill là `9h58` và trong khoảng thời gian đó cũng có rất nhiều các tiến trình http được thiết lập 

Từ đây ta sẽ vào file access log của dịch vụ http để check các kết nối gần đây. 

Sử dụng lệnh `cat /var/log/httpd/access_log | grep -B 500 "9:58"` để xem các truy cập vào cơ sở dữ liệu trong khoảng thời gian `9h58`. 

![Imgur](https://i.imgur.com/Kl8UnID.png)

Ta thấy rằng gần đây có rất nhiều kết nối từ địa chỉ `10.10.34.196` thiết lập kết nối vào trang web. 

Nhìn kỹ 1 chút thì ta sẽ thấy rằng tuy rằng sẽ có thể có những địa chỉ khác nhưng điểm chung là chúng đều sử dụng chung 1 kiểu `User-Agent` (các user-agent hiển thị theo hình thức lặp đi lặp lại trong 1 list khoảng 10 user-agent khác nhau). Các `header-referers` cũng hiển thị theo hình thức tương tự như vậy. 
Và 1 đặc điểm nữa ta thấy rằng các IP trên khi truy cập không hề load CSS.

Để chắc chắn 1 lần nữa rằng đây là IP đã gây ra sự cố, ta xem những địa chỉ đã truy cập gần đây và hiển thị địa chỉ IP có lượt truy cập nhiều nhất. 

```
awk '{print $1}' /var/log/httpd/access_log | sort | uniq -c | sort -nr
```

![Imgur](https://i.imgur.com/F6podqS.png)

Từ đây ta cũng chắc chắn rằng truy cập từ địa chỉ `10.10.34.196` và địa chỉ `10.10.35.1` cũng có thể chính là nguyên nhân gây ra lỗi kết nối cơ sở dữ liệu. 

Từ đây ta có thể tìm thấy nguyên nhân gây ra lỗi kết nối cơ sở dữ liệu rồi. Hãy thực hiện chặn lại những địa chỉ bất thường hoặc đưa ra phương pháp phù hợp nhất với hệ thống của mình nhé. Sau đó khởi động lại cơ sở dữ liệu để trang web tiếp tục hoạt động bình thường nhé. 