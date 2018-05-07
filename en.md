# MySQL 5.7 Performance Tuning Immediately After Installation

This blog updates [Stephane Combaudon’s blog on MySQL performance tuning](https://www.percona.com/blog/2014/01/28/10-mysql-performance-tuning-settings-after-installation/), and covers MySQL 5.7 performance tuning immediately after installation.

A few years ago, Stephane Combaudon wrote a blog post on [Ten MySQL performance tuning settings after installation](https://www.percona.com/blog/2014/01/28/10-mysql-performance-tuning-settings-after-installation/) that covers the (now) older versions of MySQL: 5.1, 5.5 and 5.6. In this post, I will look into what to tune in MySQL 5.7 (with a focus on InnoDB).

The good news is that MySQL 5.7 has significantly better default values. Morgan Tocker created a [page with a complete list of features in MySQL 5.7](http://www.thecompletelistoffeatures.com/), and is a great reference point. For example, the following variables are set _by default_:

*   [innodb\_file\_per_table](http://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_file_per_table)=ON
*   [innodb\_stats\_on_metadata](https://www.percona.com/blog/2013/12/03/innodb_stats_on_metadata-slow-queries-information_schema/) = OFF
*   [innodb\_buffer\_pool_instances](http://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_instances) = 8 (or 1 if innodb\_buffer\_pool_size < 1GB)
*   [query\_cache\_type](https://www.percona.com/blog/2015/01/02/the-mysql-query-cache-how-it-works-and-workload-impacts-both-good-and-bad/) = 0; query\_cache\_size = 0; (disabling mutex)

In MySQL 5.7, there are only four really important variables that need to be changed. However, there are other InnoDB and global MySQL variables that might need to be tuned for a specific workload and hardware.

To start, add the following settings to my.cnf under the \[mysqld\] section. You will need to restart MySQL:
```
[mysqld]
# other variables here
innodb_buffer_pool_size  =  1G  # (adjust value here, 50%-70% of total RAM)
innodb_log_file_size  =  256M
innodb_flush_log_at_trx_commit  =  1  # may change to 2 or 0
innodb_flush_method  =  O_DIRECT
```

Description:

|**Variable** | **Value**|
|---|---|
| innodb\_buffer\_pool_size | Start with 50% 70% of total RAM. Does not need to be larger than the database size |
|innodb\_flush\_log\_at\_trx_commit | *   1   (Default) *   0/2 (more performance, less reliability)|
| innodb\_log\_file_size | 128M – 2G (does not need to be larger than buffer pool) |
| innodb\_flush\_method | O_DIRECT (avoid double buffering)|


_**What is next?**_

Those are a good starting point for any new installation. There are a number of other variables that can increase MySQL performance for some workloads. Usually, I would setup a MySQL monitoring/graphing tool (for example, the [Percona Monitoring and Management platform](http://pmmdemo.percona.com)) and then check the MySQL dashboard to perform further tuning.

_**What can we tune further based on the graphs?**_

_InnoDB buffer pool size_. Look at the graphs:

[![MySQL 5.7 Performance Tuning](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.49.22-PM.png)](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.49.22-PM.png)

[![MySQL 5.7 Performance Tuning](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.48.13-PM.png)](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.48.13-PM.png)

As we can see, we can probably benefit from increasing the InnoDB buffer pool size a bit to ~10G, as we have RAM available and the number of free pages is small compared to the total buffer pool.

_InnoDB log file size._ Look at the graph:

[![MySQL 5.7 Performance Tuning](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.43.52-PM.png)](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.43.52-PM.png)

As we can see here, InnoDB usually writes 2.26 GB of data per hour, which exceeds the total size of the log files (2G). We can now increase the `innodb_log_file_size` variable and restart MySQL. Alternatively, use “show engine InnoDB status” to [calculate a good InnoDB log file size](https://www.percona.com/blog/2008/11/21/how-to-calculate-a-good-innodb-log-file-size/).

_**Other variables**_

There are a number of other InnoDB variables that can be further tuned:

_innodb\_autoinc\_lock_mode_

Setting [innodb\_autoinc\_lock_mode](http://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html) =2 (interleaved mode) can remove the need for table-level AUTO-INC lock (and can increase performance when multi-row insert statements are used to insert values into tables with auto_increment primary key). This requires `binlog_format=ROW`  or `MIXED`  (and ROW is the default in MySQL 5.7).

_innodb\_io\_capacity_ and_ innodb\_io\_capacity_max_

This is a more advanced tuning, and only make sense when you are performing a lot of writes all the time (it does not apply to reads, i.e. SELECTs). If you really need to tune it, the best method is knowing how many IOPS the system can do. For example, if the server has one SSD drive, we can set `innodb_io_capacity_max=6000` and `innodb_io_capacity=3000` (50% of the max). It is a good idea to run the sysbench or any other benchmark tool to benchmark the disk throughput.

But do we need to worry about this setting? Look at the graph of buffer pool’s “[dirty pages](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_dirty_page)“:

[![screen-shot-2016-10-03-at-7-19-47-pm](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-7.19.47-PM.png)](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-7.19.47-PM.png)

In this case, the total amount of dirty pages is high, and it looks like InnoDB can’t keep up with flushing them. If we have a fast disk subsystem (i.e., SSD), we might benefit from increasing `innodb_io_capacity` and `innodb_io_capacity_max`.

_**Conclusion or TL;DR version**_

The new MySQL 5.7 defaults are much better for general purpose workloads. At the same time, we still need to configure InnoDB variables to take advantages of the amount of RAM on the box. After installation, follow these steps:

1.  Add InnoDB variables to my.cnf (as described above) and restart MySQL
2.  Install a monitoring system, (e.g., Percona Monitoring and Management platform)
3.  Look at the graphs and determine if MySQL needs to be tuned further
