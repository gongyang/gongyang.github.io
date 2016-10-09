---
title : ngxtop---nginx日志实时监控工具
date: 2016-9-17
---
## ngxtop---nginx日志实时监控工具
### 前言
在工作当中，我们会经常通过tail -f去实时查看nginx的日志，但是这样很难提取到有用的信息，今天我们就来介绍一个好用的工具，[ngxtop](https://github.com/lebinh/ngxtop)。
ngxtop是一个实时查看nginx日志，并能对其进行分析的工具，它的输出有点类似于我们常用的top命令

### 安装
由于ngxtop是用python编写的，首先得安装依赖库[pip](http://ask.xmodulo.com/install-pip-linux.html)
然后我们使用如下命令安装ngxtop
```
[root@gy-vm02 ~]# pip install ngxtop
```

### 使用
我们使用ngxtop --help查看使用方法
```
[root@gy-vm02 ~]# ngxtop --help
ngxtop - ad-hoc query for nginx access log.

Usage:
    ngxtop [options]
    ngxtop [options] (print|top|avg|sum) <var> ...
    ngxtop info
    ngxtop [options] query <query> ...

Options:
    -l <file>, --access-log <file>  access log file to parse.
    -f <format>, --log-format <format>  log format as specify in log_format directive. [default: combined]
    --no-follow  ngxtop default behavior is to ignore current lines in log
                     and only watch for new lines as they are written to the access log.
                     Use this flag to tell ngxtop to process the current content of the access log instead.
    -t <seconds>, --interval <seconds>  report interval when running in follow mode [default: 2.0]

    -g <var>, --group-by <var>  group by variable [default: request_path]
    -w <var>, --having <expr>  having clause [default: 1]
    -o <var>, --order-by <var>  order of output for default query [default: count]
    -n <number>, --limit <number>  limit the number of records included in report for top command [default: 10]
    -a <exp> ..., --a <exp> ...  add exp (must be aggregation exp: sum, avg, min, max, etc.) into output

    -v, --verbose  more verbose output
    -d, --debug  print every line and parsed record
    -h, --help  print this help message.
    --version  print version information.

    Advanced / experimental options:
    -c <file>, --config <file>  allow ngxtop to parse nginx config file for log format and location.
    -i <filter-expression>, --filter <filter-expression>  filter in, records satisfied given expression are processed.
    -p <filter-expression>, --pre-filter <filter-expression> in-filter expression to check in pre-parsing phase.
```
**实时监控日志**
ngxtop -l /usr/local/nginx/logs/access.log
这样，我们所看到的就类似于top命令一样的输出，当然我们可以加上-n参数，显示最多的几行信息。
```
running for 8 seconds, 154 records processed: 19.01 req/sec

Summary:
|   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|---------+------------------+-------+-------+-------+-------|
|     154 |          133.260 |    31 |   123 |     0 |     0 |

Detailed:
| request_path              |   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|---------------------------+---------+------------------+-------+-------+-------+-------|
| /api/v1/patch             |	   89 |            0.607 |    12 |    77 |     0 |     0 |
| /api/v1/tabbar_settings   |      35 |           12.229 |     4 |    31 |     0 |     0 |
| /api/v1/carts             |      16 |          141.000 |    11 |     5 |     0 |     0 |
| /api/v1/carts.json        |       6 |            0.000 |     0 |     6 |     0 |     0 |
| /api/v1/orders/stats.json |       4 |            0.000 |     0 |     4 |     0 |     0 |
```
如果要分析以前的日志，我们可以加上--no-follow参数，默认是分析实时的日志，而不会分析以前的。
```
[root@bhqa logs]# ngxtop -l passport-access.log --no-follow
running for 4 seconds, 46051 records processed: 13119.42 req/sec

Summary:
|   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|---------+------------------+-------+-------+-------+-------|
|   46051 |          628.178 | 38283 |  4024 |  3642 |   102 |

Detailed:
| request_path                              |   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|-------------------------------------------+---------+------------------+-------+-------+-------+-------|
| /api/v1/users/(null)                      |   13094 |          307.985 | 12910 |     2 |   178 |     4 |
| /api/v1/users/login                       |    7395 |          320.068 |  6211 |     0 |  1181 |     3 |
| /login                                    |    6153 |         1989.940 |  6083 |    33 |    27 |    10 |
| /admin/sessions                           |    3355 |          462.457 |   489 |  2791 |    75 |     0 |
| /api/v1/user_connections.json             |    2011 |           94.427 |  1864 |   124 |    14 |     9 |
| /api/v1/users/3819                        |    1378 |          344.372 |  1377 |     0 |     1 |     0 |
| /api/v1/user_connections/authenticate_sns |    1219 |          827.696 |  1132 |     0 |    48 |    39 |
| /api/v1/user_connections                  |     999 |          108.591 |   513 |   435 |    49 |     2 |
| /api/v1/users/14                          |     798 |          348.147 |   798 |     0 |     0 |     0 |
| /admin/users                              |     716 |         1285.564 |   444 |   249 |    23 |     0 |
```
我们还可以使用-g -o参数对结果进行group和order
**order-by**
```
[root@bhqa logs]# ngxtop -l passport-access.log --no-follow -o "avg_bytes_sent"
running for 4 seconds, 46051 records processed: 13151.85 req/sec

Summary:
|   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|---------+------------------+-------+-------+-------+-------|
|   46051 |          628.178 | 38283 |  4024 |  3642 |   102 |

Detailed:
| request_path                  |   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|-------------------------------+---------+------------------+-------+-------+-------+-------|
| /admin/administrators/64/edit |       1 |         3409.000 |     1 |     0 |     0 |     0 |
| /admin/administrators/20/edit |       1 |         3402.000 |     1 |     0 |     0 |     0 |
| /admin/administrators/46/edit |       2 |         3398.000 |     2 |     0 |     0 |     0 |
| /admin/administrators/71/edit |       3 |         3387.667 |     3 |     0 |     0 |     0 |
| /admin/administrators/69/edit |       1 |         3384.000 |     1 |     0 |     0 |     0 |
| /admin/administrators/56/edit |       3 |         3374.333 |     3 |     0 |     0 |     0 |
| /admin/administrators/63/edit |       4 |         3308.500 |     4 |     0 |     0 |     0 |
| /admin/administrators/66/edit |       3 |         3058.333 |     3 |     0 |     0 |     0 |
| /admin/administrators/36/edit |       1 |         3053.000 |     1 |     0 |     0 |     0 |
| /admin/administrators/54/edit |       7 |         2954.714 |     7 |     0 |     0 |     0 |
```
**group-by**
```
[root@bhqa logs]# ngxtop -l passport-access.log --no-follow -g remote_addr
running for 3 seconds, 46051 records processed: 13252.85 req/sec

Summary:
|   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|---------+------------------+-------+-------+-------+-------|
|   46051 |          628.178 | 38283 |  4024 |  3642 |   102 |

Detailed:
| remote_addr    |   count |   avg_bytes_sent |   2xx |   3xx |   4xx |   5xx |
|----------------+---------+------------------+-------+-------+-------+-------|
| 58.246.136.4   |   29045 |          563.002 | 23989 |  3199 |  1795 |    62 |
| 58.246.136.2   |   11409 |          530.525 |  9520 |   641 |  1218 |    30 |
| 127.0.0.1      |     224 |          139.362 |   185 |     0 |    39 |     0 |
| 211.22.145.234 |     215 |          307.242 |   214 |     1 |     0 |     0 |
| 101.80.45.210  |     170 |          335.000 |   170 |     0 |     0 |     0 |
| 222.65.137.160 |     149 |          259.685 |   144 |     0 |     5 |     0 |
| 116.230.205.93 |     116 |          338.828 |   116 |     0 |     0 |     0 |
| 101.229.195.1  |     108 |          314.370 |   100 |     0 |     8 |     0 |
| 180.164.242.31 |     104 |          474.654 |    53 |     0 |    51 |     0 |
| 114.88.231.181 |      90 |          264.633 |    89 |     0 |     1 |     0 |
```

**显示400或更高返回状态码**
```
[root@bhqa logs]# ngxtop -l passport-access.log --no-follow -n 5 -i 'status >=400' print remote_addr status 
running for 4 seconds, 3744 records processed: 972.38 req/sec

remote_addr, status:
| remote_addr     |   status |
|-----------------+----------|
| 1.188.172.188   |      401 |
| 101.226.227.95  |      404 |
| 101.226.33.201  |      404 |
| 101.226.33.216  |      404 |
| 101.226.33.219  |      404 |
| 101.226.51.226  |      404 |
| 101.226.64.174  |      404 |
| 101.226.65.104  |      499 |
| 101.226.66.191  |      404 |
| 101.229.17.2    |      404 |
| 101.229.195.1   |      401 |
| 101.229.30.8    |      404 |
| 103.240.124.89  |      401 |
| 107.191.53.74   |      401 |
| 107.191.53.74   |      404 |
| 108.61.200.226  |      404 |
| 111.206.36.13   |      401 |
| 112.224.67.54   |      400 |
| 112.64.189.74   |      401 |
| 112.64.235.247  |      500 |
| 112.64.235.91   |      404 |
```
另外一些变量可以在分析时用到，名字含义同日志格式里的设置：
```
remote_addr、remote_user、time_local、request、request_path、status、body_bytes_sent、http_referer、http_user_agent
```
