## 安装所需环境

Nginx 是 C语言 开发，建议在 Linux 上运行，当然，也可以安装 Windows 版本，本篇则使用 [CentOS](http://www.linuxidc.com/topicnews.aspx?tid=14) 7 作为安装环境。

 

**一. gcc 安装**
安装 nginx 需要先将官网下载的源码进行编译，编译依赖 gcc 环境，如果没有 gcc 环境，则需要安装：

```
yum install gcc-c++
```

**二. PCRE pcre-devel 安装**
PCRE(Perl Compatible Regular Expressions) 是一个Perl库，包括 perl 兼容的正则表达式库。nginx 的 http 模块使用 pcre 来解析正则表达式，所以需要在 linux 上安装 pcre 库，pcre-devel 是使用 pcre 开发的一个二次开发库。nginx也需要此库。命令：

```
yum install -y pcre pcre-devel
```

**三. zlib 安装**
zlib 库提供了很多种压缩和解压缩的方式， nginx 使用 zlib 对 http 包的内容进行 gzip ，所以需要在 Centos 上安装 zlib 库。

```
yum install -y zlib zlib-devel
```

**四. OpenSSL 安装**
OpenSSL 是一个强大的安全套接字层密码库，囊括主要的密码算法、常用的密钥和证书封装管理功能及 SSL 协议，并提供丰富的应用程序供测试或其它目的使用。
nginx 不仅支持 http 协议，还支持 https（即在ssl协议上传输http），所以需要在 Centos 安装 OpenSSL 库。

```
yum install -y openssl openssl-devel
```

## 官网下载

1.直接下载`.tar.gz`安装包，地址：https://nginx.org/en/download.html

![img](https://images2015.cnblogs.com/blog/26794/201704/26794-20170415143759345-1159867247.png)

 

2.使用`wget`命令下载（推荐）。确保系统已经安装了wget，如果没有安装，执行 yum install wget 安装。

```
wget -c https://nginx.org/download/nginx-1.12.0.tar.gz
```

![img](https://images2015.cnblogs.com/blog/26794/201704/26794-20170415143847533-1700322865.png)

 

我下载的是1.12.0版本，这个是目前的稳定版。

## 解压

依然是直接命令：

```
tar -zxvf nginx-1.12.0.tar.gz
cd nginx-1.12.0
```

## 配置

其实在 nginx-1.12.0 版本中你就不需要去配置相关东西，默认就可以了。当然，如果你要自己配置目录也是可以的。
1.使用默认配置

```
./configure
```

2.自定义配置（不推荐）

```
./configure \
--prefix=/usr/local/nginx \
--conf-path=/usr/local/nginx/conf/nginx.conf \
--pid-path=/usr/local/nginx/conf/nginx.pid \
--lock-path=/var/lock/nginx.lock \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--with-http_gzip_static_module \
--http-client-body-temp-path=/var/temp/nginx/client \
--http-proxy-temp-path=/var/temp/nginx/proxy \
--http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
--http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
--http-scgi-temp-path=/var/temp/nginx/scgi
```

> 注：将临时文件目录指定为/var/temp/nginx，需要在/var下创建temp及nginx目录

## 编译安装

```
make
make install
```

查找安装路径：

```
whereis nginx
```

## 启动、停止nginx

```
cd /usr/local/nginx/sbin/
./nginx 
./nginx -s stop
./nginx -s quit
./nginx -s reload
启动时报80端口被占用:
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)

```

` 解决办法：1、安装net-tool 包：`yum install net-tools

```
 
```

> `./nginx -s quit`:此方式停止步骤是待nginx进程处理任务完毕进行停止。
> `./nginx -s stop`:此方式相当于先查出nginx进程id再使用kill命令强制杀掉进程。

查询nginx进程：

```
ps aux|grep nginx
```

## 重启 nginx

1.先停止再启动（推荐）：
对 nginx 进行重启相当于先停止再启动，即先执行停止命令再执行启动命令。如下：

```
./nginx -s quit
./nginx
```

2.重新加载配置文件：
当 ngin x的配置文件 nginx.conf 修改后，要想让配置生效需要重启 nginx，使用`-s reload`不用先停止 ngin x再启动 nginx 即可将配置信息在 nginx 中生效，如下：
./nginx -s reload

启动成功后，在浏览器可以看到这样的页面：

![nginx-welcome.png](http://www.linuxidc.com/upload/2016_09/160905180451093.png)

## 开机自启动

即在`rc.local`增加启动代码就可以了。

```
vi /etc/rc.local
```

增加一行 `/usr/local/nginx/sbin/nginx`
设置执行权限：

```
chmod 755 rc.local
```

![nginx-rclocal.png](http://www.linuxidc.com/upload/2016_09/160905180451095.png)

到这里，nginx就安装完毕了，启动、停止、重启操作也都完成了，当然，你也可以添加为系统服务，我这里就不在演示了。

## 配置axure

**nginx配置路径**

 vim /usr/local/nginx/conf/nginx.conf

```
server {
        listen       8080;
        server_name  localhost;
		
		location / {
            root   /usr/local/axurefile/demo1;
            index  start.html;
        }

        location /irm {
            alias  /usr/local/axure/irm;
            index  start.html;
        }
		
		location /gz {
            alias  /usr/local/axure/gz;
            index  start.html;
        }
        location /xiaoqi {
            alias  /usr/local/axure/xiaoqi;
            index  start.html;
        }

	}
```

 

重启nginx

cd /usr/local/nginx/sbin

 ./nginx -s reload

 

访问url即可以看到

[http://ip:8080/demo1](http://192.168.1.105:8080/demo1)

[http://ip:8080/demo](http://192.168.1.105:8080/demo1)2