#Nginx 以及健康检测模块的安装 


## 下载
upstream check模块
https://github.com/yaoweibin/nginx_upstream_check_module

nginx，选择版本下载
https://nginx.org/en/download.html

本例中，我选择nginx最新版本(1.13.9),patch选择check_1.12.1+.patch

## 安装
* 安装gcc
yum -y install make gcc gcc-c++ ncurses-devel
* 安装pcre
yum install -y pcre-deve
* 安装zlib
yum install -y zlib-devel
* 安装openssl
yum -y install openssl openssl-devel
* 打patch
patch -p0 < /xxx/nginx_upstream_check_module-master/check_1.13.9+.patch （绝对地址）
* 编译

```
    ./configure --user=xx --group=xx --prefix=/data/dev/nginx-1.13.9 --conf-path=/etc/nginx/nginx.conf --with-http_stub_status_module --with-http_ssl_module --with-http_v2_module --with-http_gzip_static_module --with-ipv6 --with-http_sub_module --add-module=../nginx_upstream_check_module/
    
    make
    
    sudo make install
    
    cp ../sbin/nginx /usr/sbin
```

* bak
在实际打patch的过程中，发现check_1.12.1+.patch的脚本是有一些问题的，nginx的模块位置在脚本中都写的是1.12.1的路径，需要根据你的nginx的实际路径调整

## 配置nginx
根据自己的服务来配置nginx.conf

## 启动
sudo nginx -t 检查无误
sudo nginx 启动
sudo nginx -s reload 平滑启动
sudo nginx -s stop 停止


## 正向代理的配置

http正向代理，可以如下直接配置

```
    server{
    	access_log    xx/proxy_access.log main;
    	error_log     xx/proxy_error.log;
    	resolver  100.100.2.136 100.100.2.138 8.8.8.8; 
        resolver_timeout 30s;   
        listen 82;  
        location / {  
                proxy_pass http://$http_host$request_uri;  
                proxy_set_header Host $http_host;
                proxy_http_version 1.1;  
                proxy_buffers 256 4k;  
                proxy_max_temp_file_size 0;  
                proxy_connect_timeout 30;  
                proxy_cache_valid 200 302 10m;  
                proxy_cache_valid 301 1h;  
                proxy_cache_valid any 1m;  
        }  
    }  
```

https的正向代理，如下配置，将proxy_pass后面配置为https（1、代码中需要配置proxy，2、在代码中https的请求，要在url中把https改为http）

```
    server{
        access_log    xx/proxy_access.log main;
        error_log     xx/proxy_error.log;
        resolver 100.100.2.136 100.100.2.138;
        resolver_timeout 30s;
        listen 83;
        location / {
                proxy_pass https://$http_host$request_uri;
                proxy_set_header Host $http_host;
                proxy_buffers 256 4k;
                proxy_max_temp_file_size 0;
                proxy_connect_timeout 30;
                proxy_cache_valid 200 302 10m;
                proxy_cache_valid 301 1h;
                proxy_cache_valid any 1m;
        }
    }
```

## nginx的日志处理（在root的crontab中配置）

shell脚本如下


```
    BACKUP_PATH="/data1/logs/nginx"
    
    mv  $BACKUP_PATH/access.log  $BACKUP_PATH/access_$(date -d "yesterday" "+%Y%m%d").log
    
    mv  $BACKUP_PATH/proxy_access.log  $BACKUP_PATH/proxy_access_$(date -d "yesterday" "+%Y%m%d").log
    
    # 通知nginx将文件写到新的文件中
    kill -USR1  `cat /data1/dev/nginx/nginx-1.9.9/logs/nginx.pid`
```
