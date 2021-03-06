---
layout: post
title:  "CentOS 7 + Nginx + php7"
date:   2018-12-28 21:21:00 +0900
categories: linux
tags: linux
---

## CentOS 7 + Nginx + php7 설정하기

구글 검색 정리한 것


### 1. CentOS 설치하기

[CentOS 배포 페이지](https://centos.org)에서 Minimal ISO를 다운로드해서 설치한다.

1. SELinux 끄기  
    권장하지는 않지만, SELinux를 잘 다룰 수 없으면 끄는게 낫다.  
    `/etc/selinux/config` 파일을 편집한다.  
    적용하려면 재부팅이 필요하다.
    > SELINUX=disabled

    * [SELinux란? - lesstif.com](https://www.lesstif.com/pages/viewpage.action?pageId=18219472)

2. 텔넷 설치하기
    > yum install telnet  

    Minimal 설치에도 `curl`은 설치되어 있으니 http(s) 테스트는 `curl`로 대체가 가능하다.

3. ifconfig 등 네트워크 유틸리티 설치하기
    * ip가 제대로 설정되어 있는지 확인
    > sudo yum install net-tools

4. cdrom 마운트하기  
    automounter가 설치되지 않았으므로 수동으로 마운트한다.
    > mount /dev/cdrom /mnt

    경고가 뜨지만 이상없이 동작한다.

5. 가상머신의 경우 콘솔 해상도 바꾸기
    1. 부팅할 때 grub에서 `tab`키로 인터럽트를 건 후 `e`키로 편집모드에 진입한다.
    2. `linux16` 라인의 끝에 `vga=ask`를 추가하고 `Ctrl-x`키로 종료하면 `Enter`키를 누르라고 나온다.
    3. 지시에 따라 `Enter`키를 누르면 사용 가능한 해상도가 나오며 원하는 해상도의 번호를 입력한다.
    > 예: 334 1152x864x16 -> 해당도 왼쪽의 334 입력하고 엔터

    4. `/etc/default/grub`를 편집한다.
    > GRUB_CMDLINE_LINUX 설정의 끝에 vga=(0x해상도번호, 예: 0x334)

    5. grub 설정파일을 재생성한다.
    > grub2-mkconfig -o /boot/grub2/grub.cfg

6. 커널 파라마터 튜닝  
> [리눅스 서버의 TCP 네트워크 성능을 결정짓는 커널 파라미터 이야기](https://meetup.toast.com/posts/55)
> * TCP 대역폭을 증가시키려면 receive window size를 늘려야 한다.
> ```sh
> $ sysctl -w net.ipv4.tcp_window_scaling="1"
> $ sysctl -w net.core.rmem_default="253952"
> $ sysctl -w net.core.wmem_default="253952"
> $ sysctl -w net.core.rmem_max="16777216"
> $ sysctl -w net.core.wmem_max="16777216"
> $ sysctl -w net.ipv4.tcp_rmem="253952 253952 16777216"
> $ sysctl -w net.ipv4.tcp_wmem="253952 253952 16777216"
> ```
> * 애플리케이션에서 알지 못하는 네트워크 패킷 유실을 방지하기 위해서는 in-bound queue 크기를 늘려야 한다.
> ```sh
> $ sysctl -w net.core.netdev_max_backlog="30000"
> $ sysctl -w net.core.somaxconn="1024"
> $ sysctl -w net.ipv4.tcp_max_syn_backlog="1024"
> $ ulimit -SHn 65535
> ```
> * TIME_WAIT 상태의 소켓은 일반적으로 서버 소켓의 경우 신경쓸 필요 없지만, 다른 서버로 다시 질의하는 경우(예를 들어 프록시 서버) 성능을 제약할 수 있으며 이 상황에서 TW_REUSE 옵션은 고려할 만 하다. (TW_RECYCLE 옵션은 NAT 환경에서 문제가 있고, socket linger 옵션은 데이터 유실이 있을 수 있으니 사용하지 말자.)
> ```sh
> $ sysctl -w net.ipv4.tcp_max_tw_buckets="1800000"
> $ sysctl -w ipv4.tcp_timestamps="1"
> $ sysctl -w net.ipv4.tcp_tw_reuse="1"
> ```

> [TIME\_WAIT 상태란 무엇인가](http://docs.likejazz.com/time-wait/)
> ```sh
> $ sysctl -w net.ipv4.tcp_fin_timeout="90"
> ```

<https://klaver.it/linux/sysctl.conf> 를 `/etc/sysctl.conf`에 적용한다.  

7. 번외: `curl`로 접속 테스트 또는 파일 받아오기
    * -X http method(GET/POST)
    * -k https 인증서 검증 생략
    * -d POST data
    * -v 상세 출력
    * -O 다운로드시 파일명을 자동으로 작성하여 저장(-o 보다 편리)

### 2. Nginx 설치하기

1. [nginx.org Centos Packages](https://nginx.org/packages/centos/) 사이트에서 rpm들을 다운로드하고 설치한다.
    ```sh
    $ curl -O https://nginx.org/packages/centos/7/x86_64/RPMS/nginx-1.14.2-1.el7_4.ngx.x86_64.rpm
    $ sudo yum localinstall nginx-1.14.2-1.el7_4.ngx.x86_64.rpm
    ```

2. `/etc/nginx/conf.d/default.conf`에 php를 실행하기 위한 설정을 추가한다.  
   아래는 http 설정이지만, http는 https로 중계하는 역할만 하고 `ssl.conf`를 작성해서 https에서 동작하도록 한다.  
   (nginx의 https 설정은 다른 포스팅에서 할 계획)

```nginx
server {
    listen       80;
    server_name  localhost;

    # 홈 디렉터리 지정 (location 밖으로 꺼낸다)
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

    # REST API 구현용 /api/resource/path 형식의 주소를 /api/index.php/resource/path 로 변환하는 샘플
    # 여기에 regex를 사용하면 안 됨
    location /api/ {
        rewrite  /api/(.*)$   /api/index.php/$1;
    }

    # php-fpm 설정 (기본 포트 9000 을 사용)
    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    location ~ [^/]\.php(/|$) {
    #    root           html;
        try_files      $uri =404;

        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        if (!-f $document_root$fastcgi_script_name) {
            return 404;
        }

        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
    #   SCRIPT_FILENAME 부분은 요구 케이스별로 조정합니다
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
    #   REST API 구현하면서 php 파일명 뒤에 추가한 리소스 경로를 $_SERVER['PATH_INFO']로 전달
        fastcgi_param  PATH_INFO        $fastcgi_path_info;
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
        client_max_body_size            10M;
        tcp_nopush                      off;
        keepalive_requests              0;
    }
}
```

### 3. php-fpm 설치하기

1. php의 CentOS용 rpm 패키지는 Remi에서 다운로드할 수 있다.
    * [Remi's RPM repository](https://rpms.remirepo.net/enterprise/7/)
    * remi 패키지는 `libargon2`에 의존성이 있어 EPEL 릴리즈에서 다운로드한다.
    > [pkgs.org download](https://centos.pkgs.org/7/epel-x86_64/libargon2-20161029-2.el7.x86_64.rpm.html)  
    ```sh
    $ curl -O http://dl.fedoraproject.org/pub/epel/7/x86_64/Packages/l/libargon2-20161029-2.el7.x86_64.rpm
    $ sudo yum localinstall libargon2-20161029-2.el7.x86_64.rpm
    ```

2. php를 설치한다.
    ```sh
    # common과 json은 의존성이 있고 fpm과 cli는 common에 의존성이 있다.
    $ curl -O https://rpms.remirepo.net/enterprise/7/php73/x86_64/php-common-7.3.0-1.el7.remi.x86_64.rpm
    $ curl -O https://rpms.remirepo.net/enterprise/7/php73/x86_64/php-json-7.3.0-1.el7.remi.x86_64.rpm
    $ curl -O https://rpms.remirepo.net/enterprise/7/php73/x86_64/php-fpm-7.3.0-1.el7.remi.x86_64.rpm
    $ curl -O https://rpms.remirepo.net/enterprise/7/php73/x86_64/php-cli-7.3.0-1.el7.remi.x86_64.rpm

    # 다른 모듈은 필요하면 다운로드한다. (DB 등)

    $ sudo yum localinstall libargon2-20161029-2.el7.x86_64.rpm
    $ sudo yum localinstall php-common-7.3.0-1.el7.remi.x86_64.rpm \
        php-json-7.3.0-1.el7.remi.x86_64.rpm \
        php-fpm-7.3.0-1.el7.remi.x86_64.rpm \
        php-cli-7.3.0-1.el7.remi.x86_64.rpm

    # apache 기준으로 gid가 셋팅되어 있어서 nginx로 바꿔준다.
    $ sudo chgrp nginx /var/lib/php/*
    ```


3. `/etc/php.ini`를 편집한다.
    ```
    ;cgi.fix_pathinfo = 0 ; 기본값은 1이며 php-fpm에서 못 찾을 경우 0으로 셋팅
    allow_url_fopen = Off
    expose_php = Off
    display_errors = Off
    short_open_tag = On
    ```

4. `/etc/php–fpm.d/www.conf`를 편집한다.
    ```
    user = nginx 
    group = nginx
    listen.owner = nobody  # 주석해제
    ```

5. php 세션 디렉터리의 owner/group 을 변경한다.  
   `find /var -group apache` 여기서 보이는 디렉터리에 가서 `chown nginx:nginx 디렉터리명`으로 오너/그룹을 변경한다.  
   보통 `/var/lib/php/session`에 있으나 간혹 `/var/opt/remi/php73/lib/php`에 위치하기도 하니 확인하여 변경한다.


### 4. 방화벽 해제 및 서비스 등록

```sh
$ sudo firewall-cmd --permanent --zone=public --add-service=http
$ sudo firewall-cmd --permanent --zone=public --add-service=https
$ sudo firewall-cmd --reload

$ sudo systemctl start php–fpm
$ sudo systemctl enable php–fpm
$ sudo systemctl start nginx
$ sudo systemctl enable nginx

$ sudo systemctl reload nginx php–fpm
```

