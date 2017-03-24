---
title: linux装机(tengine、jdk8安装教程)
date: 2017-03-24 13:21:46
categories: 
- 编程
tags: 
- tengine
- jdk8
---
# jdk8安装
* oracle官方未提供jdk镜像地址，默认apt-get是无法安装jdk8的，需要额外设置jdk镜像源。
```shell
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install -y oracle-java8-installer
sudo apt-get install -y oracle-java8-set-default
sudo update-alternatives --config java
#安装目录/usr/lib/jvm/java-8-oracle
```
    细节参考：
    * [添加oracle jdk镜像](http://askubuntu.com/questions/593433/error-sudo-add-apt-repository-command-not-found)
    * [安装](https://www.digitalocean.com/community/tutorials/how-to-install-java-on-ubuntu-with-apt-get)
    * [卸载](http://www.2daygeek.com/uninstall-oracle-java-openjdk-on-ubuntu-centos-debian-fedora-mint-rhel-opensuse)
<!-- more -->

* jce补丁安装
[jdk1.8 JCE安装](http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html)
安装目录 /usr/lib/jvm/java-8-oracle/jre/lib/security

# tengine安装教程

* [tengine安装教程](http://www.nginx.cn/install http://tengine.taobao.org/document_cn/install_cn.html)

* pcre安装 [地址一](ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/) 或者 [地址二](https://sourceforge.net/projects/pcre/files/pcre/8.39/pcre-8.39.tar.gz/download)
./configure -prefix=/yit/srv/pcre
make&&make install

* zlib安装 [下载地址](http://zlib.net/)
./configure -prefix=/yit/srv/zlib
make&&make install

* [openssl安装下载地址](https://www.openssl.org/source/ https://wiki.openssl.org/)index.php/Compilation_and_Installation
./config -prefix=/yit/srv/openssl
make depend
make install

* tengine编译安装

```shell
./configure --prefix=/yit/srv/tengine/ --with-pcre=/yit/srv/tengine-source/pcre-8.39 --with-zlib=/yit/srv/tengine-source/zlib-1.2.8 --with-openssl=/yit/srv/tengine-source/openssl-1.0.2h --user=yitsrv --group=yitsrv 
#--with-pcre=/usr/src/pcre-8.34 指的是pcre-8.34 的源码路径。
#--with-zlib=/usr/src/zlib-1.2.7 指的是zlib-1.2.7 的源码路径.
```


# memcache
memcached启动 http://www.cnblogs.com/lhj588/p/3268694.html
/opt/memcached/bin/memcached -d -u root -m 512 127.0.0.1 -p 11211 