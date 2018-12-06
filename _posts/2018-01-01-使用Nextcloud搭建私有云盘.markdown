---
layout: post
title: "使用Nextcloud搭建私有网盘"
date: 2018-01-01 01:00:00
autor: "姚国涛"
tags:
    - nextcloud
    - 私有网盘
---


Nextcloud是开源的一套个人及企业私有网盘方案，是Owncloud的升级版。关于Owncloud、Nextcloud的更多介绍，请自行google。  
下面简单记录一下Nextcloud的搭建。

* 安装基础依赖包

    ```shell
    yum install -y epel-release yum-utils unzip curl wget bash-completion policycoreutils-python mlocate bzip2
    ```

* 安装配置httpd

    ```shell
    yum install httpd

    # 创建配置文件
    touch /etc/httpd/conf.d/nextcloud.cnf

    # 输入以下内容：
    <VirtualHost *:80>
      DocumentRoot /var/www/html/
      ServerName  nextcloud.ygt.com

    <Directory "/var/www/html/">
      Require all granted
      AllowOverride All
      Options FollowSymLinks MultiViews
    </Directory>
    </VirtualHost>

    # 打开httpd自启动
    systemctl enable httpd

    # 启动httpd服务
    systemctl start httpd
    ```

* 安装PHP相关包

    ```shell
    rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm

    yum install -y php72w php72w-cli php72w-common php72w-curl php72w-gd php72w-mbstring php72w-mysqlnd php72w-process php72w-xml php72w-zip php72w-opcache php72w-pecl-apcu php72w-intl php72w-pecl-redis
    ```

* 安装数据库

    ```shell
    yum install -y mariadb mariadb-server

    # 配置mysql开机自启动
    systemctl enable mariadb

    # 启动mysql服务
    systemctl start mariadb
    ```

* 下载安装nextcloud包
可以去nextcloud download页面找相应的包： https://nextcloud.com/install  
提示： 因网络环境不同，下载速度不一，建议提前下载，准备好该包。

    ```shell
    wget https://download.nextcloud.com/server/releases/nextcloud-14.0.4.zip

    # 解压
    unzip nextcloud-14.0.4.zip

    # 将解压后的nextcloud目录复制到/var/www/html/目录下
    cp -R nextcloud/ /var/www/html/

    # 修改nextcloud目录属主属组为httpd的用户
    chown apache:apache /var/www/html/nextcloud/ -R
    ```

这个时候，就可以通过浏览器访问：http://<http server 地址>/nextcloud  
接下来，可以通过界面上的配置向导来配置，也可以直接修改nextcloud的配置文件来配置。  
我们这里直接在nextcloud server机器上来通过命令行配置。  

* 安装redis
redis主要用来做nextcloud的缓存，也可以选择其他的来缓存，比如memcache等。

    ```shell
    yum install redis

    # 配置redis开启自启动
    systemctl enable redis

    # 启动redis服务
    systemctl start redis
    ```

* 配置nextcloud使用redis

    ```shell
    vim /var/www/html/nextcloud/config/config.php

    # 输入如下内容（在$CONFIG = array（）的大数组里）
      'memcache.distributed' => '\OC\Memcache\Redis',
      'memcache.locking' => '\OC\Memcache\Redis',
      'memcache.local' => '\OC\Memcache\APCu',
      'redis' => array(
        'host' => 'localhost',
        'port' => 6379,
        ),
    ```

* 配置mysql数据库

    ```shell
    # 切换到nextcloud目录
    cd /var/www/html/nextcloud/

    # 使用nextcloud的命令行occ，occ更多使用可以查看：https://docs.nextcloud.com/server/14/admin_manual/configuration_server/occ_command.html
    sudo -u apache php occ maintenance:install --database "mysql" --database-name "nextcloud"  --database-user "root" --database-pass "password" --admin-user "admin" --a
    dmin-pass "password" 
    但是，我在这里失败了，因此使用了单独配置mysql的方式:
    首先进入mysql终端：
    mysql --user=root
    // 创建admin user，密码也是admin
    MariaDB [mysql]> create user 'admin'@'localhost' IDENTIFIED BY 'admin';
    // 配置admin的权限
    MariaDB [mysql]> GRANT ALL PRIVILEGES ON *.* TO 'admin'@'localhost' WITH GRANT OPTION;
    MariaDB [mysql]> FLUSH PRIVILEGES;
    ```

* 配置nextcloud使用mysql

    ```shell
    # 在/var/www/html/nextcloud/config/config.php中增加如下配置：
    'dbtype' => 'mysql',
      'version' => '14.0.4.2',
      'overwrite.cli.url' => 'http://localhost',
      'dbname' => 'nextcloud',
      'dbhost' => 'localhost',
      'dbport' => '',
      'dbtableprefix' => 'oc_',
      'dbuser' => 'oc_admin',
      'dbpassword' => 'root',
      'installed' => true,
    ```

* 配置nextcloud数据目录

    ```shell
    # 在nextcloud配置文件中增加：
    'datadirectory' => '/var/www/html/nextcloud/data',

    # 创建nextcloud数据目录
    mkdir /var/www/html/nextcloud/data
    ```

* 配置nextcloud数据存储使用S3接口+RGW
数据存储使用S3不用删除掉nextcloud使用本地存储的配置，用户进入nextcloud账户后，默认进去的是data目录，但是上传文件的时候，还是会将文件保存在RGW中。  
**前提：** 先部署好Ceph和RGW，保证可以使用S3 client来上传文件到RGW中。

    ```shell
    在nextcloud配置文件中增加：
      'objectstore' => array(
            'class' => 'OC\\Files\\ObjectStore\\S3',
            'arguments' => array(
                    'bucket' => 'nextcloud',
                    'autocreate' => true,
                    'key'    => '7QG3884EQ9G9X4JCC5W6',
                    'secret' => 'QAb5u9xDEPYSF21etgg8O3vtzyuAVjix2mRxpjQn',
                    'hostname' => '10.10.8.7',
                    'port' => 7480,
                    'use_ssl' => false,
                    'region' => 'optional',
                    // required for some non Amazon S3 implementations
                    'use_path_style'=>true
            ),
      ),
    ```

我们来对上面需要改动的参数做一下解释：
`bucket`对应RGW中的bucket，
`autocreate`: true表示如果没有该bucket，就创建；false表示如果没有该bucket，也不创建。
`key`: S3 client的key
`secret`: S3 client的secret
`hostnmae`： RGW地址
`port`: RGW端口，默认就是7480
`use_ssl`: RGW访问是否使用ssl
`region`：*这个还不清楚对应的RGW的real？zonegroup？zone*，那就用默认的optional


* 重启httpd服务

    ```shell
    systemctl restart httpd
    ```

到这里，就可以通过web访问了，体验一下私有网盘，和常用的网盘有什么不同。

**TODO：** 贴几张搭建的网盘截图。


### 附：
nextcloud配置文件内容：
```php
<?php
$CONFIG = array (
  'instanceid' => 'oc33jbvw405p',
  "log_type" => "owncloud",
  "logfile" => "nextcloud.log",
  "loglevel" => "3",
  "logdateformat" => "F d, Y H:i:s",
  'memcache.distributed' => '\\OC\\Memcache\\Redis',
  'memcache.locking' => '\\OC\\Memcache\\Redis',
  'memcache.local' => '\\OC\\Memcache\\APCu',
  'redis' => 
  array (
    'host' => 'localhost',
    'port' => 6379,
  ),
  'passwordsalt' => '7SdDEWVjYLWeC05rYDcV8tK1yshwo9',
  'secret' => 'mHEmK9GTvQNXHi2J4o55cV9xtb2VT1Xdwo6PrNRzkuxATWO1',
  'trusted_domains' => 
  array (
    0 => '192.168.32.207',
  ),
  'datadirectory' => '/var/www/html/hualala/data',
  'dbtype' => 'mysql',
  'version' => '14.0.4.2',
  'overwrite.cli.url' => 'http://localhost',
  'dbname' => 'nextcloud',
  'dbhost' => 'localhost',
  'dbport' => '',
  'dbtableprefix' => 'oc_',
  'dbuser' => 'oc_admin',
  'dbpassword' => 'c2nZn+kvya6UeXcjo+jtYuOe3gIqeO',
  'installed' => true,
  'objectstore' => array(
        'class' => 'OC\\Files\\ObjectStore\\S3',
        'arguments' => array(
                'bucket' => 'nextcloud',
                'autocreate' => true,
                'key'    => '7QG3884EQ9G9X4JCC5W6',
                'secret' => 'QAb5u9xDEPYSF21etgg8O3vtzyuAVjix2mRxpjQn',
                'hostname' => '10.10.8.7',
                'port' => 7480,
                //'use_ssl' => true,
                'use_ssl' => false,
                //'region' => 'default',
                'region' => 'optional',
                // required for some non Amazon S3 implementations
                'use_path_style'=>true
        ),
  ),
);
```
