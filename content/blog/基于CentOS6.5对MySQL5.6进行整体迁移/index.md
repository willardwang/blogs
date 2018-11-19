---
title : "基于CentOS6.5对MySQL5.6进行整体迁移"
date : 2018-11-14T10:34:06+08:00
tags : ["迁移"]
categories : ["运维"]
banner : "img/banners/迁移MySQL.jpg"
---

> MySQL整体迁移（所有数据库同时迁移）是客户业务系统迁移的一部分，前提是MySQL数据（由MySQL variable datadir指定的目录）已经迁移到新系统了（迁移过程详见：[青云主机迁移到华为云](/blog/2018/11/13/青云主机迁移到华为云/)），本文继续新服务器上的MySQL安装以及配置，包括最核心的地方——数据如何导入到新数据库里。

# 环境

```
操作系统：CentOS release 6.5 (Final)
MySQL：5.6.21
```

<br>

# 安装

1. 检测是否安装了MySQL

    ```
    yum list installed | grep mysql
    ```

2. 如果有MySQL，则卸载

    ```
    yum -y remove mysql-libs.x86_64
    ```

3. 下载MySQL

    ```
    wget http://dev.mysql.com/Downloads/MySQL-5.6/MySQL-client-5.6.21-1.rhel5.x86_64.rpm
    wget http://dev.mysql.com/Downloads/MySQL-5.6/MySQL-server-5.6.21-1.rhel5.x86_64.rpm
    ```

4. 安装MySQL

    ```
    rpm -ivh MySQL-client-5.6.21-1.rhel5.x86_64.rpm
    rpm -ivh MySQL-server-5.6.21-1.rhel5.x86_64.rpm
    注意：
        1、安装server后，会在/root/.mysql_secret里存放一个随机的默认密码
        2、默认开机启动，使用chkconfig查看
    ```

5. 启动MySQL

    ```
    service mysql start
    ```

6. 进行MySQL的初始化

    ```
    /usr/bin/mysql_secure_installation --user=root // 注意：这里使用的是MySQL并不建议使用的root用户
    按照向导进行一些配置：
       1、修改密码
       2、Remove anonymous users? [Y/n] y  // 删除匿名用户
       3、Disallow root login remotely? [Y/n] n // 允许远程登录
       4、Remove test database and access to it? [Y/n] y // 删除测试数据库
       5、Reload privilege tables now? [Y/n] Y // 立即重新加载权限表
    ```

    <br>

# 配置

1. 登录MySQL

    ```
    使用上面重置的密码登录MySQL
    mysql -uroot -p
    ```

2. 在进行配置之前先比较配置的差异，做到心中有数

    ```
    分别在两个系统中利用show variables命令导出配置，然后利用beyond compare对比差异。
    根据下图可见，左侧红色差异标记为24项
    ```

    {{< imgproc variables对比图-前 Resize "807x" />}}

3. 停止MySQL，防止复制数据文件和配置文件的过程中出错。

    ```
    service mysql stop
    ```

4. 将MySQL数据文件放到指定位置

    ```
    因为原服务器指定的MySQL数据的存放位置为/data/mysql，所以将新服务器的数据也存放到该位置。
    经过对象存储传输后的文件权限信息也自动和原服务器保持了一致。
    ```

    {{< imgproc "mysql文件列表" Resize "807x" />}}

5. 复利原服务器下的/etc/my.cnf到新服务器的相同位置

    ```
      MySQL会默认使用该位置下的my.cnf配置文件。
    ```

6. 重启MySQL，再次重复1中的步骤，比较配置差异

    ```
    根据下图可见，左侧红色差异减少为8项，其中5项为服务器名称修改后的正常差异，比如：
    hostname         从 i-bxf89yzg       变为 ecs-5fa6.novalocal
    general_log_file 从 general_log_file 变为 /data/mysql/ecs-5fa6.log
    另外3项差异为open_files_limit、pseudo_thread_id和timestamp，下面专门探讨
    ```

    {{< imgproc variables对比图-后 Resize "807x" />}}

7. variable差异：open_files_limit，从 65536 变为 5010

    ```
    The number of files that the operating system permits mysqld to open. 
    1) 10 + max_connections + (table_open_cache * 2)
    2) max_connections * 5
    3) operating system limit if positive
    The server attempts to obtain the number of file descriptors using the maximum of those three values.
    ```

    ```
    MySQL文档显示，这是操作系统允许mysqld进程打开的文件数量，MySQL会取以上3个值的最大值。根据此规则，分别查询max_connections和table_open_cache值，如下表：
    ```

    |          | max_connections | table_open_cache | 按照公式计算后 |
    | -------- | --------------- | ---------------- | -------------- |
    | 原服务器 | 1000            | 2000             | 5010           |
    | 新服务器 | 1000            | 2000             | 5010           |

    ```
    根据上表可知，两服务器根据max_connections和table_open_cache所计算的值均为5010，但是实际原服务器为65536，新服务器为5010，这就说明是第3)条规则起了作用（取最大值），即操作系统限制的值，所以调整新服务器的操作系统限制的值即可。(其实根据my.cnf配置文件一模一样也可分析出是操作系统层面的原因导致的)，请见如下两种方式：
    ```

    ```
    方式1：使用开机启动执行命令的方式
    vim /etc/rc.local
    添加
    ulimit -HSn 65536
    service mysql restart
    ```

    ```
    方式2：修改配置文件的方式
    vim /etc/security/limits.conf  
    添加
    ;* soft nofile 65535
    ;* hard nofile 65535
    *代表用户，全部用户或用户组(由于Markdown语法原因，无法书写出独立的星号，请去除上方两行的;)
    
    注意：网上许多文档都是这种方式，但是我的场景下不生效，未知原因。如果你的场景下此方法生效的话，请使用此方法，修改配置文件的方式优于使用开机启动执行命令的方式。
    ```

8. variable差异：pseudo_thread_id

    ```
    This variable is for internal server use.
    ```

    ```
    MySQL文档显示，这是一个系统变量，用于标记当前会话的连接ID，所以差异为正常差异。
    ```

9. variable差异：timestamp

    ```
    Set the time for this client. 
    ```

    ```
    MySQL文档显示，这是客户端的时间戳，由于导出variable时的时间不同，所以此差异为正常差异。
    ```

10. 重启后弱报数据库是lock状态的话删除lock文件重启MySQL即可，如果不行就重启服务器

    ```
    该问题是复制数据和配置文件的过程中未停止MySQL造成的
    rm -rf /var/lock/subsys/mysql
    ```

11. 如果Java局域网连接数据库报错：java.sql.SQLException: null, message from server: “Host ‘xxx’ is not allowed to connect

     ```
     GRANT ALL PRIVILEGES ON *.* TO 'root'@'Java服务器的内网IP' IDENTIFIED BY 'root的密码' WITH GRANT OPTION;
     ```


