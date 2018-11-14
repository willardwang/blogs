---
title : "简单几步将log4j2产生的日志采集到阿里云"
date : 2018-10-29T17:44:18+08:00
tags : ["log4j"]
categories : ["运维"]
banner : "img/banners/aliyun-log.jpg"
---

```
    你有没有遇到这样的场景：线上程序出bug了，需要去日志里寻找线索。但是打开日志目录后，面对几十上百M的日志量十分头疼。按照时间线挨个打开日志，查询(Ctrl+F、Command+F)关键字。。。面对WARN、INFO、ERROR各种级别的日志焦头烂额。如果你每天都需要登录系统查看ERROR情况呢~如果你管理的是数个系统呢~是不是更加崩溃！
    
    现在，让阿里云日志服务来拯救你吧，采集、查找、分析一条龙服务~
    
    而且，对小型系统来说来说: 几！乎！免！费！
    
    阿里云支持的日志采集方式众多，本文以log4j2产生的日志为例，简单几步就能将日志采集到阿里云进行查询和分析。
```



</br>

# 创建阿里云Logstore

1. 准备一个阿里云账号(https://www.aliyun.com/) ， 登录 -> `控制台` -> `日志服务`
2. `创建Project`(*所属地域*  选择离服务器最近的) -> `创建Logstore`，关闭`数据接入向导`对话框。
3. 注：`数据接入向导`中显示只 *仅支持 Log4j1.x* ，应该是文档更新不及时的原因，实际支持log4j2。

</br>

# 创建AccessKey

1. 鼠标over到头像 -> `accesskeys` ，建议使用子账户AccessKey，这样权限控制的比较细、比较安全。
2. 填写子账户名 -> 选择 `AliyunLogFullAccess`（管理日志服务(Log)的权限）即可，无需给 AdministratorAccess（管理所有阿里云资源的权限）权限

</br>

# Log4j2 appender

- maven 工程中引入依赖

     ```
     <dependency>
         <groupId>com.google.protobuf</groupId>
         <artifactId>protobuf-java</artifactId>
         <version>2.5.0</version>
     </dependency>
     <dependency>
         <groupId>com.aliyun.openservices</groupId>
         <artifactId>aliyun-log-log4j2-appender</artifactId>
         <version>0.1.10</version>
     </dependency>
     ```

- 修改配置文件


     ```
     <Appenders>
           <Loghub name="Loghub"
                   projectName="your project"
                   logstore="your logstore"
                   endpoint="your project endpoint"
                   accessKeyId="your accesskey id"
                   accessKey="your accesskey"
                   packageTimeoutInMS="3000"
                   logsCountPerPackage="4096"
                   logsBytesPerPackage="3145728"
                   memPoolSizeInByte="104857600"
                   retryTimes="3"
                   maxIOThreadSizeInPool="8"
                   topic="your topic"
                   source="your source"
                   timeFormat="yyyy-MM-dd'T'HH:mmZ"
                   timeZone="UTC"
                   ignoreExceptions="true">
               <PatternLayout pattern="%d %-5level [%thread] %logger{0}: %msg"/>
           </Loghub>
     </Appenders>
     <Loggers>
           <Root level="warn">
               <AppenderRef ref="Loghub"/>
           </Root>
     </Loggers>
     ```

   	必填参数：

      1. projectName、logstore比较好理解，分别填入创建时的名称即可。
      2. endpoint要根据project所在地域选择，比如华北2（北京）的endpoint为`cn-beijing.log.aliyuncs.com`，其他地址详见：https://help.aliyun.com/document_detail/29008.html?spm=5176.10695662.1996646101.searchclickresult.407b560amiw5u5
      3. accessKeyId和accessKey在创建完之后会下载到Excel文件中，分别对应填入即可。
      4. 启动应用程序后，等待几分钟就被采集到阿里云了。

</br>

# 日志查询、分析

1. 日志服务 -> `Project列表` -> 点击某个Project的`名称` -> `Logstore列表` -> 点击某个Logstore的`查询`

2. `查询分析属性` -> `设置` -> 指定字段查询，比如添加`level`字段的查询，`类型`选择`text`，`分词符`为空即可。

3. 左侧`快速分析`Tab会出现`level`，一般情况下最关注ERROR类型的日志。

4. 此外在页面上很容易找到设置`内容列`和`时间`的位置

5. 如果经常需要查询某类场景，可以使用`另存为快速查询`功能将查询条件存储。比如将ERROR类型的查询存储起来，以便每天检查系统运行是否出错。在左侧的快速查询功能里点击即可。附图：

   {{< imgproc aliyun Resize "841x" />}}

</br>

# 费用

1. 计费规则分为主要计费项和次要计费项，主要计费项包括：读写流量、存储空间、索引流量，次要计费项包括：读写次数、活跃 Shard租用、外网读取流量。
2. 主要计费项每月有500M的免费流量（压缩后，文本文件的压缩比还是很可观的，能达到数十比一）。
3. 以我使用的小型系统为例，每月的主要计费项都免费，只收取少量次要计费项，大概每月15元，可以忽略不计了。在此感谢一下马云大佬~
4. 计费方式详见：https://help.aliyun.com/document_detail/48220.html?spm=5176.215333.1147926.6.702b4ae7M4VvBB

</br></br>

<center>
```
结语：阿里云的日志功能非常强大：支持众多采集方式、查询方式、分析方式，更多功能等你来挖掘使用。
```
</center>

