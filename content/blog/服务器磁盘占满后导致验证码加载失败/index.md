---
title : "服务器磁盘使用率100%后导致验证码加载失败"
date : 2018-11-05T11:33:28+08:00
tags : ["异常"]
categories : ["运维"]
banner : "img/banners/磁盘使用率百分之百.jpg"
---

```
    公司某个平台登录页的验证码加载失败，导致无法登录。经过排查，原因为磁盘被Tomcat产生的日志占满导致。虽出身程序员，但身兼平台的开发和运维，需要掌握的除了开发能力之外，还有经常被忽略的运维技能，运维的过程也会给开发带来思考。
```

<br>

# 发现问题

<br>

## 环境

```
操作系统：CentOS release 6.5 (Final)
Tomcat：8.5.32
云提供商：青云
```

<br>

## 直接原因
通过外部日志系统查看错误日志：
```
thread: http-nio-8044-exec-2 throwable: java.io.IOException: No space left on device
    at java.io.RandomAccessFile.writeBytes(Native Method)
    at java.io.RandomAccessFile.write(RandomAccessFile.java:525)
    at javax.imageio.stream.FileCacheImageOutputStream.write(FileCacheImageOutputStream.java:141)
    at com.sun.imageio.plugins.jpeg.JPEGImageWriter.writeOutputData(JPEGImageWriter.java:1869)
    at com.sun.imageio.plugins.jpeg.JPEGImageWriter.writeImage(Native Method)
    at com.sun.imageio.plugins.jpeg.JPEGImageWriter.writeOnThread(JPEGImageWriter.java:1067)
    at com.sun.imageio.plugins.jpeg.JPEGImageWriter.write(JPEGImageWriter.java:363)
    at javax.imageio.ImageWriter.write(ImageWriter.java:615)
    at javax.imageio.ImageIO.doWrite(ImageIO.java:1612)
    at javax.imageio.ImageIO.write(ImageIO.java:1578)
    at com.awspaas.web.login.web.controller.CodeController.codeGenerator(CodeController.java:90)
```
`可见是由于磁盘使用100%导致了写图片验证码时出现Java IO异常`，此时外部日志系统的优点之一展现出来：由于磁盘使用率为100%，导致无法登录服务器查看异常日志，但是可以通过外部日志系统查看。搭建外部日志系统请参见：[简单几步将log4j2产生的日志采集到阿里云](/blog/2018/10/29/简单几步将log4j2产生的日志采集到阿里云/)

<br>

# 解决问题

<br>

## 恢复Tomcat

首先恢复Tomcat将影响降低，因为租用的青云服务器，所以登录到青云控制台执行以下操作：

1. 关机（开机状态下无法扩容硬盘）
2. 扩容硬盘，然后开机
3. 发现无法通过原有秘钥登录系统，报错：`Permission denied (publickey).`，原因未知。
4. 在青云控制台重置服务器密码
5. 使用新密码登录
6. 启动tomcat

<br>

## 恢复秘钥登录

1. 查看公钥文件/root/.ssh/authorized_keys，文件确实为空，印证了为什么无法使用秘钥登录。
2. 修改/etc/ssh/sshd_config文件，将PasswordAuthentication设为no，将RSAAuthentication、PubkeyAuthentication和PermitRootLogin设为yes
3. 重启ssh服务：service sshd restart
4. 在青云上重新向主机绑定秘钥，查看/root/.ssh/authorized_keys文件，发现确实有了一个公钥。(如果知道原公钥内容也可以直接追加至此文件)
5. 尝试用新秘钥登录，成功。

<br>

## 查找占用原因

1. 查看磁盘使用率：
    ```
    [root@i-01b5gxms logs]# df -h
    Filesystem            Size  Used Avail Use% Mounted on
    /dev/sda1              30G   20G  8.4G  71% /
    tmpfs                 1.9G     0  1.9G   0% /dev/shm
    ```

    原有大小20G，扩容到30G，使用率为：71%，符合预期。

2. 在根目录下执行:du -sh *，查看过大的目录，然后进入过大目录，继续执行该命令。最终，发现tomcat日志过大。
    ```
    [root@i-01b5gxms apache-tomcat-8.5.32]# du -sh *
    848K	bin
    244K	conf
    7.6M	lib
    56K	LICENSE
    18G	logs
    4.0K	NOTICE
    8.0K	RELEASE-NOTES
    16K	RUNNING.txt
    4.0K	temp
    209M	webapps
    220K	work
    ```
3. 按照大小排序，发现catalina.out过大导致，18G！

	```
    [root@i-01b5gxms logs]# ll -Sh
    total 18G
    -rw-r----- 1 root root  18G Nov  5 11:11 catalina.out
    -rw-r----- 1 root root 716K Sep 29 18:40 catalina.2018-09-29.log
    -rw-r----- 1 root root 373K Aug  1 11:29 catalina.2018-08-01.log
    -rw-r----- 1 root root 201K Aug  2 15:34 catalina.2018-08-02.log
    -rw-r----- 1 root root  74K Aug  1 11:29 error-debug.2018-08-01.log
    -rw-r----- 1 root root  62K Jul 30 21:33 catalina.2018-07-30.log
    -rw-r----- 1 root root  55K Jul 30 21:07 error-debug.2018-07-30.log
  ```
4. 删除catalina.out，执行df -h仍然为71%，磁盘使用率并未如期下降。原因是日志文件被删除之前被Tomcat占用，删除后依然占用空间，重启tomcat后再次查看，磁盘使用率已经降下来：

    ```
    [root@i-01b5gxms PaaS]# df -h
    Filesystem            Size  Used Avail Use% Mounted on
    /dev/sda1              30G  2.1G   26G   8% /
    tmpfs                 1.9G     0  1.9G   0% /dev/shm
    ```

<br>

# 补救措施

<br>

## 磁盘监控

 ```
   在青云控制台创建资源使用率告警，当资源达到设置的阈值时会发送短信、邮件、微信通知。方便及时处理潜在危险，好过危险出现以后再解决。
 ```

   {{< imgproc 青云资源监控告警 Resize "841x" />}}

<br>

## 开机启动Tomcat

将Tomcat设为开机启动，这样当无法登录时Tomcat也能提供服务，为解决登录问题赢取时间。设置时请注意Linux发新版本，我的是CentOS release 6.5 (Final)
​    

```
vi /etc/rc.d/rc.local
```

```
nohup sh /data/userconsole/apache-tomcat-8.5.31/bin/startup.sh > /dev/null 2>&1 &
```

