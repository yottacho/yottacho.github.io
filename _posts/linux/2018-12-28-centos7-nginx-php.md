---
layout: post
title:  "CentOS 7 + Nginx + php7"
date:   2018-12-28 21:00:00 +0900
categories: linux
tags: linux
---

### CentOS7 + Nginx + php7 설정하기

구글 검색 정리한 것


#### 1. CentOS 설치하기

[CentOS 배포 페이지](https://centos.org)에서 Minimal ISO를 다운로드해서 설치한다.

* SELinux 끄기
    * SELinux를 잘 다룰 수 없으면 끄는게 낫다.

`sudo vi /etc/selinux/config` 파일을 편집한다.
```
SELINUX=disabled
```

저장 후 재부팅한다.

* 텔넷 설치하기
    * 프로그램 테스트할 때 텔넷이 없으면 불편하다.

`sudo yum install telnet`

* ifconfig 등 네트워크 유틸리티 설치하기
    * ip가 제대로 설정되어 있는지 확인

`sudo yum install net-tools`

#### 2. Nginx 설치하기

[nginx.org CentOs packages](https://nginx.org/packages/centos/) 사이트에서 rpm 을 다운로드하고 설치한다.

* /packages/centos/7/x86_64/nginx-1.14.2-1.el7_4.ngx.x86_64.rpm
* `sudo rpm -Uvh nginx-1.14.2-1.el7_4.ngx.x86_64.rpm`

`/etc/nginx/conf.d/default.conf`에 php 설정을 추가한다.

```
server {
    listen       80;
    server_name  localhost;

    # 홈 디렉터리 지정
    root   /usr/share/nginx/html;
    # 문서 찾는 순서 지정
    index  index.php index.html index.htm;

    location / {
        # 기존것은 주석처리
        #root   /usr/share/nginx/html;
        #index  index.html index.htm index.php;
        try_files  $uri          $uri/ =404;
    }

    error_page  404              /404.html;

    # php-fpm 설정 (기본 포트 9000 을 사용)
    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    location ~ \.php$ {
    #    root           html;
        try_files      $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        include        fastcgi_params;

    # fastcgi 버퍼 사이즈 조절~
    # 502 에러를 없애기 위한 proxy 버퍼 관련 설정입니다.
        proxy_buffer_size               128k;
        proxy_buffers                 4 256k;
        proxy_busy_buffers_size         256k; 
    # 502 에러를 없애기 위한 fastcgi 버퍼 관련 설정입니다.
        fastcgi_buffering               on;
        fastcgi_buffer_size             16k;
        fastcgi_buffers              16 16k;
 
    # 최대 timeout 설정입니다.
        fastcgi_connect_timeout         600s;
        fastcgi_send_timeout            600s;
        fastcgi_read_timeout            600s;
 
   # 아래 설정은 PHP 성능 향상을 위한 옵션입니다. 추가해 주시면 좋답니다.
        sendfile                        on;
        tcp_nopush                      off;
        keepalive_requests              0;
    }
}
```

#### 3. php-fpm 설치하기

php는 `libargon2`의존성으로 설치가 안 되므로 epel 릴리즈에서 libargon2을 받는다.

[pkgs.org](https://pkgs.org/download/libargon2(x86-64)) libargon2-20161029-2.el7.x86_64.rpm

php는 [Remi's RPM repository](https://rpms.remirepo.net/enterprise/7/)에서 다운로드할 수 있다.

* php-common-7.3.0-1.el7.remi.x86_64.rpm
* php-json-7.3.0-1.el7.remi.x86_64.rpm
* php-fpm-7.3.0-1.el7.remi.x86_64.rpm
* php-cli-7.3.0-1.el7.remi.x86_64.rpm

db 등 다른 모듈은 필요하면 다운로드한다.

```sh
sudo rpm -Uvh libargon2-20161029-2.el7.x86_64.rpm

# 의존성이 있어서 common과 json은 함께 설치한다.
sudo rpm -Uvh php-common-7.3.0-1.el7.remi.x86_64.rpm php-json-7.3.0-1.el7.remi.x86_64.rpm 
sudo rpm -Uvh php-fpm-7.3.0-1.el7.remi.x86_64.rpm 
sudo rpm -Uvh php-cli-7.3.0-1.el7.remi.x86_64.rpm 

sudo chgrp nginx /var/lib/php/*
```

`/etc/php.ini`를 편집한다.

```
cgi.fix_pathinfo = 0 ; 주석해제
allow_url_fopen = Off
expose_php = Off
display_errors = Off
short_open_tag = On
```

`/etc/php–fpm.d/www.conf`를 편집한다.

```
user = nginx 
group = nginx
listen.owner = nobody  # 주석해제
```


#### 4. 방화벽 해제 및 서비스 등록

```sh
sudo firewall-cmd --permanent --zone=public --add-service=http
sudo firewall-cmd --permanent --zone=public --add-service=https
sudo firewall-cmd --reload

sudo systemctl start php–fpm
sudo systemctl enable php–fpm
sudo systemctl start nginx
sudo systemctl enable nginx

sudo systemctl reload nginx php–fpm
```

끝
