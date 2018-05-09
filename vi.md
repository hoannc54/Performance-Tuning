
[Source](https://www.percona.com/blog/2016/10/12/mysql-5-7-performance-tuning-immediately-after-installation/ "Permalink to MySQL 5.7 Performance Tuning Immediately After Installation")

# Điều chỉnh hiệu năng sau khi cài đặt MySQL 5.7 

Bài viết này cập nhật thêm về [Blog của Stephane Combaudon về điều chỉnh hiệu năng của MySQL][1], và bao gồm cả điều chỉnh hiệu năng của MySQL 5.7 sau khi cài đặt.

Một vài năm trước, Stephane Combaudon đã viết 1 bài blog đăng trên [10 cài đặt điều chỉnh hiệu năng MySQL sau khi cài đặt ][1] cho các phiên bản cũ hơn của MySQL như: 5.1, 5.5 và 5.6. Trong bài viết này, tôi sẽ xem qua những gì cần điều chỉnh trong phiên bản MySQL 5.7 (tập trung vào InnoDB).

Tin vui là phiên bản MySQL 5.7 có những giá trị mặc định tốt hơn đáng kể. Morgan Tocker đã tạo một [trang có danh sách đầy đủ các tính năng trong MySQL 5.7][2], và nó là một điểm để tham khảo tuyệt vời. Ví dụ, các biến sau được cài đặt _mặc định_:
* innodb_file_per_table=ON
* innodb_stats_on_metadata = OFF
* innodb_buffer_pool_instances = 8 (or 1 if innodb_buffer_pool_size < 1GB)
* query_cache_type = 0; query_cache_size = 0; (disabling mutex)

Trong phiên bản MySQL 5.7, chỉ có 4 biến thực sự quan trọng mà bạn cần phải thay đổi. Tuy nhiên, có những InnoDB khác và các biến toàn cục của MySQL có thể cần được điều chỉnh cho khối lượng công việc và phần cứng cụ thể.

Để bắt đầu, thêm những cài đặt sau vào file my.cnf, bên dưới của phần group [mysqld]. Bạn cần phải khởi động lại MySQL:

[mysqld] 


innodb_buffer_pool_size = 1G # (điều chỉnh giá trị tại đây, 50%-70% tổng lượng RAM)

innodb_log_file_size = 256M

innodb_flush_log_at_trx_commit = 1 # có thể thay đổi thành 2 hoặc 0

innodb_flush_method = O_DIRECT

 | 

Mô tả:

| ----- |
| **Variable** |  **Value** |  
| innodb_buffer_pool_size |  Bắt đầu với khoảng 50% đến 70% trên tổng lượng RAM. Không cần phải lớn hơn kích thước của dữ liệu  |  
| innodb_flush_log_at_trx_commit | 

* 1   (Mặc định)
* 0/2 (hiệu suất cao hơn, độ tin cậy thấp hơn)
 |  
| innodb_log_file_size |  128M – 2G (Không cần phải lớn hơn bộ nhớ đệm) |  
| innodb_flush_method |  O_DIRECT (tránh đệm 2 lần) | 

 

_**Tiếp theo là gì?**_

Đây là những điểm tốt nhất cho bất kỳ một cài đặt mới nào.Đây là những biến khác mà có thể tăng hiệu năng cho MySQL cho một vài việc. Thông thường, Tôi sẽ thiết lập công cụ theo dõi/công cụ ghi đồ thị (ví dụ, [Nền tảng quản lý và giám sát Percona][3]) và sau đó kiểm tra bảng điều khiển của MySQL để thực hiện thêm các hiệu chỉnh.

_**Chúng ta có thể điều chỉnh thêm những gì dựa trên các đồ thị?**_

_Kích thước của bộ nhớ đệm InnoDB_. Nhìn vào các biểu đồ sau:

![MySQL 5.7 Performance Tuning][4]

![MySQL 5.7 Performance Tuning][5]

Như chúng ta có thể thấy, chúng ta có thể có lợi từ việc tăng kích thước bộ nhớ đệm của InnoDB lên một chút ~10G, như những gì chúng ta có lượng RAM khả dụng và số trang trống nhỏ hơn so hới tổng bộ nhớ đệm.+

_Kích thước file log của InnoDB._ Nhìn vào đồ thị sau:

![MySQL 5.7 Performance Tuning][6]

Như chúng ta có thể thấy, InnoDB thường ghi khoảng 2.26 GB dữ liệu mỗi giờ, vượt quá tổng kích thước của các file log (2G). chúng ta có thể tăng biến innodb_log_file_size và khởi động lại MySQL. Một cách khác, sử dụng "hiển thị trạng thái của InnoDB" để [tính toán kích thước file log tốt nhất cho InnoDB][7].

_**Các biến khác**_

Có một số biến khác của InnoDB có thể điều chỉnh thêm:

_innodb_autoinc_lock_mode_

Cài đặt [innodb_autoinc_lock_mode][8] =2 (chế độ xen kẽ) có thể loại bỏ nhu cầu khoá AUTO-INC ở mức bảng (và có thể tăng hiệu suất khi các câu lệnh chèn nhiều hàng được sử dụng để chèn giá trị vào bảng với khoá chính tự động tăng). điều này yêu cầu binlog_format=ROW  hoặc MIXED  (và ROW là giá trị mặc định trong phiên bản MySQL 5.7).

_innodb_io_capacity _ và _ innodb_io_capacity_max_

Đây là một điều chỉnh nầng cao hơn, và chỉ có ý nghĩa khi bạn luôn thực hiện việc ghi rất nhiều(nó không áp dụng cho việc đọc, ví dụ như SELECTs). Nếu bạn thực sự cần phải điều chỉnh nó,phương pháp tốt nhất là phải biết có bao nhiêu hệ thống IOPS có thể làm việc này. Ví dụ, nếu server có một ổ SSD, chúng ta có thể đặt innodb_io_capacity_max=6000 và innodb_io_capacity=3000 (50% của giá trị tối đa). Đây là 1 ý tưởng tốt để chạy sysbench hay bất cứ công cụ benchmark nào để chuẩn hoá thông qua ổ đĩa.

Nhưng chúng ta có cần phải quan tâm về cài đặt này không? xem biểu đồ về bộ nhớ đệm của những "[dirty pages][9]":

![screen-shot-2016-10-03-at-7-19-47-pm][10]

Trong trường hợp này, tổng số những trang bẩn khá cao, và có vẻ như InnoDB không thể theo kịp chúng. Nếu chúng ta có một hệ thống chứa ổ đĩa tốc độ cao (ví dụ, SSD), chúng ta có thể hưởng lợi từ innodb_io_capacity và innodb_io_capacity_max.

_**Kết luận hoặc TL (total loss: Tổng kết); Phiên bản DR**_

Phiên bản mặc định MySQL 5.7 mới tốt hơn nhiều đối với những công việc có mục đích chung lớn.Cùng một lúc, chúng ta vẫn có thể cấu hình các biến của InnoDB để tận dụng lượng Ram trong máy.Sau khi cài đặt, thực hiện theo các bước:

1. Thêm các biến vào file my.cnf (như mô tả ở trên) và khởi động lại MySQL
2. Cài đặt các hệ thống giám sát, (ví dụ, Percona Monitoring và Management platform)
3. Nhìn vào các biểu đồ và xác định xem MySQL cần phải điều chỉnh thêm không.

### More resources:

#### Posts

#### Webinars

#### Presentations

#### Free eBooks

#### Tools

### _Related_

![][11]

[Alexander Rubin][12]

Alexander joined Percona in 2013. Alexander worked with MySQL since 2000 as DBA and Application Developer. Before joining Percona he was doing MySQL consulting as a principal consultant for over 7 years (started with MySQL AB in 2006, then Sun Microsystems and then Oracle). He helped many customers design large, scalable and highly available MySQL systems and optimize MySQL performance. Alexander also helped customers design Big Data stores with Apache Hadoop and related technologies.

[1]: https://www.percona.com/blog/2014/01/28/10-mysql-performance-tuning-settings-after-installation/
[2]: http://www.thecompletelistoffeatures.com/
[3]: http://pmmdemo.percona.com
[4]: https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.49.22-PM.png
[5]: https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.48.13-PM.png
[6]: https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.43.52-PM.png
[7]: https://www.percona.com/blog/2008/11/21/how-to-calculate-a-good-innodb-log-file-size/
[8]: http://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html
[9]: http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_dirty_page
[10]: https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-7.19.47-PM.png
[11]: https://secure.gravatar.com/avatar/79877aeedbd68531a30468cd771d5d07?s=84&d=mm&r=g
[12]: https://www.percona.com/blog/author/alexanderrubin/

  
