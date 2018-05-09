# MySQL 5.7 Performance Tuning Immediately After Installation
# MySQL 5.7 Điều chỉnh hiệu năng ngay sau khi cài đặt

Blog này cập nhật [blog của Stephane Combaudon về điều chỉnh hiệu năng MySQL](https://www.percona.com/blog/2014/01/28/10-mysql-performance-tuning-settings-after-installation/) và bao gồm điều chỉnh hiệu năng MySQL 5.7 ngay sau khi cài đặt.

Một vài năm trước, Stephane Combaudon đã viết một bài đăng trên blog về [những cài đặt để điều chỉnh hiệu suất của Ten MySQL sau khi cài đặt](https://www.percona.com/blog/2014/01/28/10-mysql-performance-tuning-settings-after-installation/) bao gồm các phiên bản hiện tại và cũ hơn của MySQL: 5.1, 5.5 và 5.6. Trong bài này, tôi sẽ xem xét những điều chỉnh trong MySQL 5.7 (tập trung vào InnoDB).

Tin tốt là MySQL 5.7 có giá trị mặc định tốt hơn đáng kể. Morgan Tocker đã tạo một [trang với danh sách các tính năng trong MySQL 5.7](http://www.thecompletelistoffeatures.com/), và là một điểm tham chiếu tuyệt vời. Ví dụ: các biến sau được đặt _bởi mặc định_:

*   [innodb\_file\_per_table](http://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_file_per_table)=ON
*   [innodb\_stats\_on_metadata](https://www.percona.com/blog/2013/12/03/innodb_stats_on_metadata-slow-queries-information_schema/) = OFF
*   [innodb\_buffer\_pool_instances](http://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_instances) = 8 (or 1 if innodb\_buffer\_pool_size < 1GB)
*   [query\_cache\_type](https://www.percona.com/blog/2015/01/02/the-mysql-query-cache-how-it-works-and-workload-impacts-both-good-and-bad/) = 0; query\_cache\_size = 0; (disabling mutex)

Trong MySQL 5.7, chỉ có bốn biến thực sự quan trọng cần được thay đổi. Tuy nhiên, có các biến InnoDB và biến MySQL toàn cục khác cần được điều chỉnh cho một khối lượng công việc và phần cứng cụ thể.

Để bắt đầu, hãy thêm các cài đặt sau vào my.cnf trong phần \[mysqld\]. Bạn sẽ cần phải khởi động lại MySQL:

```
[mysqld]
# other variables here
innodb_buffer_pool_size  =  1G  # (điều chỉnh giá trị tại đây, 50%-70% RAM )
innodb_log_file_size  =  256M
innodb_flush_log_at_trx_commit  =  1  # có thể thay thành 2 hoặc 0
innodb_flush_method  =  O_DIRECT
```

Mô tả:

|**Variable** | **Value**|
|---|---|
| innodb\_buffer\_pool_size | Bắt đầu với 50% 70% tổng số RAM. Không cần phải lớn hơn kích thước cơ sở dữ liệu |
|innodb\_flush\_log\_at\_trx_commit | *   1   (Mặc định) *   0/2 (hiệu năng hơn, ít chắc chắn hơn)|
| innodb\_log\_file_size | 128M – 2G (không cần phải lớn hơn buffer pool) |
| innodb\_flush\_method | O_DIRECT (tránh đệm đôi)|


_**Tiếp theo là gì?**_

Đó là một điểm khởi đầu tốt cho bất kỳ cài đặt mới nào. Có một số biến khác có thể tăng hiệu năng MySQL cho một số khối lượng công việc. Thông thường, tôi sẽ thiết lập một công cụ giám sát / vẽ đồ thị MySQL (ví dụ, [nền tảng giám sát và quản lý Percona](http://pmmdemo.percona.com)) và sau đó kiểm tra bảng điều khiển MySQL để thực hiện điều chỉnh thêm.

_**Những gì chúng ta có thể điều chỉnh thêm dựa trên các đồ thị?**_

_InnoDB buffer pool size_. nhìn vào biểu đồ:

[![MySQL 5.7 Performance Tuning](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.49.22-PM.png)](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.49.22-PM.png)

[![MySQL 5.7 Performance Tuning](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.48.13-PM.png)](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.48.13-PM.png)

Như chúng ta có thể thấy, chúng ta có thể có lợi từ việc tăng InnoDB buffer pool size một chút lên ~ 10G, vì chúng ta có RAM và số trang khả dụng nhỏ so với tổng số vùng đệm.

_InnoDB log file size._ nhìn vào biểu đồ:

[![MySQL 5.7 Performance Tuning](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.43.52-PM.png)](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.43.52-PM.png)

Như chúng ta có thể thấy ở đây, InnoDB thường ghi 2,26 GB dữ liệu mỗi giờ, vượt quá tổng kích thước của các file log (2G). Bây giờ chúng ta có thể tăng biến `innodb_log_file_size` và khởi động lại MySQL. Ngoài ra, hãy sử dụng "hiển thị trạng thái InnoDB" để [tính toán kích thước tệp nhật ký InnoDB tốt](https://www.percona.com/blog/2008/11/21/how-to-calculate-a-good-innodb-log-file-size/).


_**Các biến khác**_

Có một số biến InnoDB khác có thể được điều chỉnh thêm:

_innodb\_autoinc\_lock_mode_


Thiết lập [innodb\_autoinc\_lock_mode](http://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html) = 2 (chế độ xen kẽ) có thể loại bỏ sự phụ thuộc vào khóa AUTO-INC ở mức độ bảng (và có thể tăng hiệu suất khi các câu lệnh chèn nhiều hàng được sử dụng để chèn các giá trị vào các bảng có khóa chính auto_increment). Điều này đòi hỏi `binlog_format = ROW` hoặc`MIXED` (và ROW là mặc định trong MySQL 5.7).

_innodb\_io\_capacity_ và _innodb\_io\_capacity_max_

Đây là một điều chỉnh nâng cao hơn, và chỉ có ý nghĩa khi bạn đang thực hiện rất nhiều việc ghi quá nhiều (nó không áp dụng cho việc đọc, tức là SELECT). Nếu bạn thực sự cần phải điều chỉnh nó, phương pháp tốt nhất là biết bao nhiêu IOPS hệ thống có thể làm. Ví dụ: nếu máy chủ có một ổ SSD, chúng tôi có thể đặt `innodb_io_capacity_max = 6000` và `innodb_io_capacity = 3000` (50% giá trị tối đa). Đó là một ý tưởng tốt để chạy sysbench hoặc bất kỳ công cụ benchmark khác nào để chuẩn hóa thông lượng đĩa.

Nhưng chúng ta có cần phải lo lắng về cài đặt này không? Nhìn vào biểu đồ [những trang rác](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_dirty_page) của vùng đệm:

[![screen-shot-2016-10-03-at-7-19-47-pm](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-7.19.47-PM.png)](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-7.19.47-PM.png)

Trong trường hợp này, tổng số trang rác cao, và có vẻ như InnoDB không thể theo kịp với việc xóa chúng. Nếu chúng tôi có hệ thống ổ đĩa SSD, chúng tôi có thể hưởng lợi từ việc tăng `innodb_io_capacity` và `innodb_io_capacity_max`.

_**Kết luận hoặc phiên bản TL;DR**_

Các mặc định mới của MySQL 5.7 tốt hơn nhiều cho các khối lượng công việc có mục đích chung. Đồng thời, chúng ta vẫn cần cấu hình các biến InnoDB để tận dụng số lượng RAM trên hộp. Sau khi cài đặt, hãy làm theo các bước sau:

1.  Thêm biến InnoDB vào my.cnf (như mô tả ở trên) và khởi động lại MySQL
2.  Cài đặt hệ thống giám sát, (ví dụ: nền tảng giám sát và quản lý Percona)
3.  Nhìn vào biểu đồ và xác định xem liệu MySQL có cần được điều chỉnh thêm hay không
