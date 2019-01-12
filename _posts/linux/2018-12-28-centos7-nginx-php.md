---
layout: post
title:  "CentOS 7 + Nginx + php7"
date:   2018-12-28 21:01:00 +0900
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
[sysctl.conf 번역](https://m.blog.naver.com/PostView.nhn?blogId=koromoon&logNo=220740270851)

```
# Kernel sysctl configuration file for Linux
#
# Version 1.13 - 2018-08-15
# Michiel Klaver - IT Professional
# http://klaver.it/linux/ for the latest version - http://klaver.it/bsd/ for a BSD variant
#
# This file should be saved as /etc/sysctl.conf and can be activated using the command:
# sysctl -e -p /etc/sysctl.conf
#
# For binary values, 0 is disabled, 1 is enabled.  See sysctl(8) and sysctl.conf(5) for more details.
#
# Tested with: Ubuntu 14.04 LTS kernel version 3.13
#              Debian 7 kernel version 3.2
#              CentOS 7 kernel version 3.10

#
# Intended use for dedicated server systems at high-speed networks with loads of RAM and bandwidth available
# Optimised and tuned for high-performance web/ftp/mail/dns servers with high connection-rates
# DO NOT USE at busy networks or xDSL/Cable connections where packetloss can be expected
# ----------

# Credits:
# http://www.enigma.id.au/linux_tuning.txt
# http://www.securityfocus.com/infocus/1729
# http://fasterdata.es.net/TCP-tuning/linux.html
# http://fedorahosted.org/ktune/browser/sysctl.ktune
# http://www.cymru.com/Documents/ip-stack-tuning.html
# http://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt
# http://www.frozentux.net/ipsysctl-tutorial/chunkyhtml/index.html
# http://knol.google.com/k/linux-performance-tuning-and-measurement
# http://www.cyberciti.biz/faq/linux-kernel-tuning-virtual-memory-subsystem/
# http://www.redbooks.ibm.com/abstracts/REDP4285.html
# http://www.speedguide.net/read_articles.php?id=121
# http://lartc.org/howto/lartc.kernel.obscure.html
# http://en.wikipedia.org/wiki/Sysctl



###
### GENERAL SYSTEM SECURITY OPTIONS ###
###

# Controls the System Request debugging functionality of the kernel
kernel.sysrq = 0

# Controls whether core dumps will append the PID to the core filename.
# Useful for debugging multi-threaded applications.
kernel.core_uses_pid = 1

#Allow for more PIDs
kernel.pid_max = 65535

# The contents of /proc/<pid>/maps and smaps files are only visible to
# readers that are allowed to ptrace() the process
kernel.maps_protect = 1

#Enable ExecShield protection
kernel.exec-shield = 1
kernel.randomize_va_space = 2

# Controls the maximum size of a message, in bytes
kernel.msgmnb = 65535

# Controls the default maxmimum size of a mesage queue
kernel.msgmax = 65535

# Restrict core dumps
fs.suid_dumpable = 0

# Hide exposed kernel pointers
kernel.kptr_restrict = 1



###
### IMPROVE SYSTEM MEMORY MANAGEMENT ###
###

# Increase size of file handles and inode cache
fs.file-max = 209708

# Do less swapping
vm.swappiness = 30
vm.dirty_ratio = 30
vm.dirty_background_ratio = 5

# specifies the minimum virtual address that a process is allowed to mmap
vm.mmap_min_addr = 4096

# 50% overcommitment of available memory
vm.overcommit_ratio = 50
vm.overcommit_memory = 0

# Set maximum amount of memory allocated to shm to 256MB
kernel.shmmax = 268435456
kernel.shmall = 268435456

# Keep at least 64MB of free RAM space available
vm.min_free_kbytes = 65535



###
### GENERAL NETWORK SECURITY OPTIONS ###
###

#Prevent SYN attack, enable SYNcookies (they will kick-in when the max_syn_backlog reached)
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_syn_retries = 2
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_max_syn_backlog = 4096

# Disables packet forwarding
net.ipv4.ip_forward = 0
net.ipv4.conf.all.forwarding = 0
net.ipv4.conf.default.forwarding = 0
net.ipv6.conf.all.forwarding = 0
net.ipv6.conf.default.forwarding = 0

# Disables IP source routing
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0

# Enable IP spoofing protection, turn on source route verification
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Disable ICMP Redirect Acceptance
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# Enable Log Spoofed Packets, Source Routed Packets, Redirect Packets
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# Decrease the time default value for tcp_fin_timeout connection
net.ipv4.tcp_fin_timeout = 7

# Decrease the time default value for connections to keep alive
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15

# Don't relay bootp
net.ipv4.conf.all.bootp_relay = 0

# Don't proxy arp for anyone
net.ipv4.conf.all.proxy_arp = 0

# Turn on the tcp_timestamps, accurate timestamp make TCP congestion control algorithms work better
net.ipv4.tcp_timestamps = 1

# Don't ignore directed pings
net.ipv4.icmp_echo_ignore_all = 0

# Enable ignoring broadcasts request
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Enable bad error message Protection
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Allowed local port range
net.ipv4.ip_local_port_range = 16384 65535

# Enable a fix for RFC1337 - time-wait assassination hazards in TCP
net.ipv4.tcp_rfc1337 = 1

# Do not auto-configure IPv6
net.ipv6.conf.all.autoconf=0
net.ipv6.conf.all.accept_ra=0
net.ipv6.conf.default.autoconf=0
net.ipv6.conf.default.accept_ra=0
net.ipv6.conf.eth0.autoconf=0
net.ipv6.conf.eth0.accept_ra=0



###
### TUNING NETWORK PERFORMANCE ###
###

# For high-bandwidth low-latency networks, use 'htcp' congestion control
# Do a 'modprobe tcp_htcp' first
net.ipv4.tcp_congestion_control = htcp

# For servers with tcp-heavy workloads, enable 'fq' queue management scheduler (kernel > 3.12)
net.core.default_qdisc = fq

# Turn on the tcp_window_scaling
net.ipv4.tcp_window_scaling = 1

# Increase the read-buffer space allocatable
net.ipv4.tcp_rmem = 8192 87380 16777216
net.ipv4.udp_rmem_min = 16384
net.core.rmem_default = 262144
net.core.rmem_max = 16777216

# Increase the write-buffer-space allocatable
net.ipv4.tcp_wmem = 8192 65536 16777216
net.ipv4.udp_wmem_min = 16384
net.core.wmem_default = 262144
net.core.wmem_max = 16777216

# Increase number of incoming connections
net.core.somaxconn = 32768

# Increase number of incoming connections backlog
net.core.netdev_max_backlog = 16384
net.core.dev_weight = 64

# Increase the maximum amount of option memory buffers
net.core.optmem_max = 65535

# Increase the tcp-time-wait buckets pool size to prevent simple DOS attacks
net.ipv4.tcp_max_tw_buckets = 1440000

# try to reuse time-wait connections, but don't recycle them (recycle can break clients behind NAT)
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_tw_reuse = 1

# Limit number of orphans, each orphan can eat up to 16M (max wmem) of unswappable memory
net.ipv4.tcp_max_orphans = 16384
net.ipv4.tcp_orphan_retries = 0

# Limit the maximum memory used to reassemble IP fragments (CVE-2018-5391)
net.ipv4.ipfrag_low_thresh = 196608
net.ipv6.ip6frag_low_thresh = 196608
net.ipv4.ipfrag_high_thresh = 262144
net.ipv6.ip6frag_high_thresh = 262144


# don't cache ssthresh from previous connection
net.ipv4.tcp_no_metrics_save = 1
net.ipv4.tcp_moderate_rcvbuf = 1

# Increase size of RPC datagram queue length
net.unix.max_dgram_qlen = 50

# Don't allow the arp table to become bigger than this
net.ipv4.neigh.default.gc_thresh3 = 2048

# Tell the gc when to become aggressive with arp table cleaning.
# Adjust this based on size of the LAN. 1024 is suitable for most /24 networks
net.ipv4.neigh.default.gc_thresh2 = 1024

# Adjust where the gc will leave arp table alone - set to 32.
net.ipv4.neigh.default.gc_thresh1 = 32

# Adjust to arp table gc to clean-up more often
net.ipv4.neigh.default.gc_interval = 30

# Increase TCP queue length
net.ipv4.neigh.default.proxy_qlen = 96
net.ipv4.neigh.default.unres_qlen = 6

# Enable Explicit Congestion Notification (RFC 3168), disable it if it doesn't work for you
net.ipv4.tcp_ecn = 1
net.ipv4.tcp_reordering = 3

# How many times to retry killing an alive TCP connection
net.ipv4.tcp_retries2 = 15
net.ipv4.tcp_retries1 = 3

# Avoid falling back to slow start after a connection goes idle
# keeps our cwnd large with the keep alive connections (kernel > 3.6)
net.ipv4.tcp_slow_start_after_idle = 0

# Allow the TCP fastopen flag to be used, beware some firewalls do not like TFO! (kernel > 3.7)
net.ipv4.tcp_fastopen = 3

# This will enusre that immediatly subsequent connections use the new values
net.ipv4.route.flush = 1
net.ipv6.route.flush = 1



###
### Comments/suggestions/additions are welcome!
###
```

* 번외: `curl`로 테스트하기
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
    cgi.fix_pathinfo = 0 ; 주석해제
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

