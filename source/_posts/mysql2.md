---
title : rails项目mysql编码由utf8改成utf8mb4
date: 2016-7-17
---
## rails项目mysql编码由utf8改成utf8mb4
### 为什么要改编码
因为utf8编码的mysql不能保存含有4个字节的emoji表情，为了能让客户端支持输入保存emoji表情，数据库必须升级并把编码改成utf8mb4。mysql从5.5.3版本开始支持utf8mb4 字符集。

首先确保mysql server端支持utf8mb4 字符集。
```
mysql> SHOW CHAR SET WHERE Charset LIKE "%utf8%";
+---------+---------------+--------------------+--------+
| Charset | Description   | Default collation  | Maxlen |
+---------+---------------+--------------------+--------+
| utf8    | UTF-8 Unicode | utf8_general_ci    |      3 |
| utf8mb4 | UTF-8 Unicode | utf8mb4_general_ci |      4 |
+---------+---------------+--------------------+--------+
2 rows in set (0.00 sec)
```

然后升级部署应用的服务器上的mysql client版本也得大于5.5.3。我这边是用rpm包单独升级。先卸载掉低版本的rpm包，然后再进行安装。

- MySQL-client：包含最少的访问mysql服务器所需要的客户端命令。里面包含的是像mysql，mysqladmin这样的工具。
- MySQL-devel：包含开发mysql客户端所需要的库。里面没有包含工具，都是包含.a这样的库链接文件
- MySQL-server：包含安装mysql所需要的所有工具。里面包含的像是mysqld_safe，mysqld_multi这样的服务器启动安装工具。里面并不包含mysql这样的客户端工具。
- Mysql-shared：包含开发mysql客户端所需要的动态链接库。里面并没有工具，都是像libmysqlclient.so*这样的动态库文件。
- Mysql-shared-compact：上面动态链接库的以前各个版本的文件。就是进行这个安装会在lib64中放上之前所有release版本的libmysqlclient.so

###  mysql2支持utf8mb4
当mysql server端和client端都支持utf8mb4之后，我们需要对mysql2
进行重新安装。这里需要注意的是，mysql2的版本必须大于0.3.11，不然是不支持utf8mb4字符集的。

进入到应用部署目录，使用bundle exec gem uninstall mysql2，选择当前应用使用的mysql2版本，然后再bundler重新安装mysql2
```
[bhuser@bhqa current]$ bundle show | grep mysql2
  * mysql2 (0.4.3)
[bhuser@bhqa current]$ bundle exec gem uninstall mysql2
Successfully uninstalled mysql2-0.4.3
[bhuser@bhqa current]$ bundle
```

###  检验
使用下面代码在rails 下进行检验
```
Mysql2::Client.new(
 :host => 'host',
 :username => 'user',
 :password => 'passwd',
 :encoding => 'utf8mb4'
)

```

```
2.3.0 :007 > Mysql2::Client.new(
2.3.0 :008 >      :host => 'x.x.x.x',
2.3.0 :009 >      :username => 'user',
2.3.0 :010 >      :password => 'passwd',
2.3.0 :011 >      :encoding => 'utf8mb4'
2.3.0 :012?>   )
 => #<Mysql2::Client:0x000000040c17b0 @read_timeout=nil, @query_options={:as=>:hash, :async=>false, :cast_booleans=>false, :symbolize_keys=>false, :database_timezone=>:local, :application_timezone=>nil, :cache_rows=>true, :connect_flags=>2147525125, :cast=>true, :default_file=>nil, :default_group=>nil, :host=>"host", :username=>"user", :password=>"passwd", :encoding=>"utf8mb4"}> 
2.3.0 :013 >
```
上面结果表示成功升级
下面的提示是没有升级mysql2的错误信息。
```
  Character set 'utf8mb4' is not a compiled character set and is not specified in the '/path/mysql/charsets/Index.xml' file
  rake aborted!
  Mysql2::Error: Can't initialize character set utf8mb4 (path: /path/mysql/charsets/)
```

