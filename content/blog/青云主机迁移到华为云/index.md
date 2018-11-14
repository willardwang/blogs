---
title : "青云主机迁移到华为云"
date : 2018-11-13T16:10:15+08:00
tags : []
categories : ["运维"]
banner : "img/banners/青云主机迁移到华为云.jpg"
---

> 客户的业务系统原来部署在青云上，根据客户公司战略要求需要将其迁移到华为云上。工作主要划分为两类：海量数据的迁移以及服务器环境、软件环境的复制，在软件环境中尤其以MySQL数据库的数据恢复最需慎重。本文为前一类：海量数据的迁移。服务器环境、软件环境的复制请见文章：[基于CentOS6.5对MySQL5.6进行整体迁移](/blog/2018/11/14/基于CentOS6.5对MySQL5.6进行整体迁移/)

<br>

# 前言

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;云上大数据量的迁移有两个特点：一是数据远在云端，无法用硬盘拷贝。二是数据量大，需要有标准协议支持断点续传。经过调查之后，选定青云和华为云都支持的“对象存储”服务来作为迁移工具。迁移过程分三步走：

1. 青云服务器的数据传输到青云对象存储的桶内
2. 青云对象存储的桶内的数据传输到华为云对象存储的桶内（这一步是跨云传输的关键）
3. 华为云对象存储的桶内的数据传输到华为云服务器上

## 环境

```
操作系统：CentOS release 6.5 (Final)

确保青云服务器和华为云服务器操作系统及其大小版本号一致，防止出现不可测的问题。
```

<br>

# 青云服务器 -> 青云对象存储

## 青云控制台的操作

1. 在服务器所在区域创建对象存储的桶（Bucket）

    ```
    大数据量传输的时候最好建立多个桶，这样在跨云传输的时候可以并发传输，加快效率。我的环境创建3个桶：aws、mysql、bpa
    ```
{{< imgproc 青云控制台-对象存储 Resize "807x" />}}   

2. 提交工单，将从青云到华为云的速度限制调整为10M

    ```
    这一步是为“青云对象存储 -> 华为云对象存储”步骤做准备，青云端默认的限制速度不详，总之比较慢。华为云端没有速度限制。
    ```


## 青云服务器的操作


1. 下载qscamel（利用青云提供的工具qscamel将青云服务器内的数据传输到桶内）

    ```
    wget https://github.com/yunify/qscamel/releases/download/v2.0.5/qscamel_v2.0.5_linux_amd64.tar.gz
    ```

2. 解压文件，并重命名为qscamel，该文件可以作为命令直接使用

    ```
    tar -xzf qscamel_v2.0.5_linux_amd64.tar.gz
    mv qscamel_v2.0.5_linux_amd64.tar.gz qscamel
    ```

3. 停止可能产生数据的服务，防止迁移以后产生新文件导致数据不全。

    ```
    我的环境停止aws、mysql、bpa
    service mysql stop
    ```

4. 查看服务器可用的硬盘容量是否足够，防止下面的打包步骤失败。

5. 将需要传输的文件打包为压缩文件

    ```
    打包后有两个优点，一个是比零散文件传输的快，第二个是方便计算文件的签名。计算签名的作用是确保文件在传输过程中没有丢数据。
    tar -czf 文件名.tar.gz 压缩的文件夹名
    ```

6. 创建yaml格式的任务文件

    ```
    传输的任务文件为yaml格式的，需要填入原路径、目的路径、桶名称、公钥、私钥等信息，详见：https://docs.qingcloud.com/qingstor/developer_tools/qscamel.html#%E5%BF%AB%E9%80%9F%E5%85%A5%E9%97%A8
    ```

7. 临时将服务器的带宽加大，传输完毕后恢复回去

    ```
    青云客服说同一区域内的服务器和桶进行传输是内网速度，能达到2g。但是我使用的时候不是这种情况，所以临时将服务器的带宽加大到配额内的最大值，待传输完毕后恢复回去。如果配额内的最大值还是不够大，可以提交工单申请修改配额。
    ```

8. 开始传输任务

    ```
    qscamel run task-name -t example-task.yaml
    1、task-name是本次传输任务的名字，也是断点续传的识别名称。
    2、如果想同时传输多个任务，需要用 -c 指定配置文件，因为单个任务传输能占满带宽，所以我没有使用多任务传输，配置文件如何编写详见：https://docs.qingcloud.com/qingstor/developer_tools/qscamel.html#%E9%85%8D%E7%BD%AE
    ```

9. 传输完毕后，控制台会提示，然后去青云的控制台去看桶内是否已经有相同大小的数据。

<br>

# 青云对象存储 -> 华为云对象存储

## 华为云控制台的操作

1. 在服务器所在区域内打开“对象存储服务”页面，创建对象存储的桶（Bucket）

    ```
    分别创建和青云端一一对应的桶。
    建议选择服务器所在区，从华为云对象存储传输到服务器上，在同一区内实测是内网传输，速度非常快。
    桶的存储类别选择：低频访问存储（价格比标准存储便宜，适合传输完毕就删除的场景），其余选项默认即可。
    ```

    {{< imgproc 华为云控制台-对象存储列表 Resize "807x" />}}

2. 打开“对象存储迁移服务”页面，注意此服务和“对象存储服务”不同

3. 分别创建和桶一一对应的传输任务，并开始传输

    ```
    在创建任务的向导页面，分别填入青云和华为云的公钥、私钥、桶名称即可。
    ```

    任务列表

    {{< imgproc 华为云控制台-对象存储迁移列表 Resize "807x" />}}

    任务详情
    {{< imgproc 华为云控制台-对象存储迁移任务详情 Resize "807x" />}}

4. 传输完毕后检查华为云对象存储的桶内是否有相同大小的数据。上图可见传输速度还是很可观的。

<br>

# 华为云对象存储 -> 华为云服务器

## 华为云服务器的操作

```
华为云对象存储和服务器之间的传输工具名为obscmd，其安装要比青云复杂不少，华为云传输软件obscmd所依赖的软件环境跟服务器现有环境的差别。obscmd需要的python最低版本为2.7.9，但是centos6.5自带的python版本为2.6.6。如果你的环境符合此该条件，请忽略安装python的步骤。
```

1. 安装openssl（python需要），详细参见：https://support.huaweicloud.com/clientogl-obs/obs_02_0422.html

    ```
    wget https://www.openssl.org/source/openssl-1.0.2n.tar.gz
    tar -xvf  openssl-1.0.2n.tar.gz    // 解压到当前目录
    cd openssl-1.0.2n    // 进入解压后的目录
    ./config -fPIC  // 进行配置
    make && make install // 编译并安装openssl
    rm /usr/bin/openssl  // 删除之前的openssl PATH变量
    ln -s /usr/local/ssl/bin/openssl /usr/bin/openssl  //创建新的openssl版本路径
    openssl version // 验证安装
    ```

2. 安装openssl-devel

    ```
    yum -y install openssl-devel
    ```

3. 升级python2.6.6到2.7.14（obscmd需要的版本最低为2.7.9，centos6.5自带的为2.6.6），详细参见：https://www.cnblogs.com/cuijianxin/p/6408757.html，请注意该文章内的版本为2.7.6，将其替换为2.7.14，否则不符合最低版本要求

    ```
    wget https://www.python.org/ftp/python/2.7.14/Python-2.7.14.tgz //下载Python-2.7.14
    tar  vxf Python-2.7.6.tgz // 解压
    cd Python-2.7.14 // 更改工作目录
    ./configure --prefix=/usr/local // 安装
    make && make install // 安装
    /usr/local/bin/python2.7 -V // 查看版本信息
    mv /usr/bin/python /usr/bin/python2.6.6 // 建立软连接,使系统默认的 python指向 python2.7
    ln -s /usr/local/bin/python2.7 /usr/bin/python 
    python -V // 重新检验Python 版本
    ```

4. 下载obscmd，并解压

    ```
    wget http://static.huaweicloud.com/upload/files/obs/cmd.zip
    unzip cmd.zip
    ```

5. 配置传输所需的配置文件config.dat，详情参见：https://support.huaweicloud.com/clientogl-obs/obs_02_0270.html

    | 配置项          | 名称                        | 备注                         |
    | --------------- | --------------------------- | ---------------------------- |
    | DownloadTarget  | 待下载的OBS桶内目标         | 留空，全部下载               |
    | SavePath        | 本地保存路径                | 和青云服务器的保存路径一致   |
    | Operation       | 指定对文件的操作类型        | 必填，可选Upload、Download。 |
    | AK              | Access Key ID接入键标识     | 必填                         |
    | SK              | Secret Access Key安全接入键 | 必填                         |
    | BucketNameFixed | 桶名                        | 必填                         |

    ```
    每次任务的传输配置不同的地方仅仅在于桶名(BucketNameFixed)不一致，其余参数相同
    ```

6. 开始传输

    ```
    进入到obscmd目录，输入命令开始传输：
    cd obscmd
    python run.py
    ```

7. 检查传输的文件是否一致

    ```
    分别在青云服务器和华为云服务器使用摘要命令计算传输的文件是否一致，如果一致才说明传输的文件是完整的。
    sha512sum 文件名
    ```
