---
layout: post
title:  "Nginx + TLS/SSL/HTTPS HTTP/2"
date:   2019-02-23 12:01:00 +0900
tags: nginx
---

# 엔진엑스(Nginx)로 보안서버 구축하기

> Nginx + TLS/SSL/HTTPS 및 HTTP/2 구축하기

### 1. nginx 최신 패키지 설치

RHEL이나 CentOS의 엔진엑스 패키지는 http/2 등을 지원하지 않는 구 버전이므로, 공식 사이트에서 최신 버전을 설치한다.

> 다른 배포판은 [공식 홈페이지의 설치 방법](https://nginx.org/en/linux_packages.html)을 참고한다.

[RHEL/CentOS용 yum 패키지](https://nginx.org/packages/centos/)

```sh
# curl -O https://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
# yum localinstall nginx-release-centos-7-0.el7.ngx.noarch.rpm
# yum info nginx
(버전 확인 등)
# yum install nginx
```

### 2. 서버 인증서 준비

서버 인증서를 준비한다. crt 파일과 key 파일 및 CA의 중간 인증서를 준비한다.

DHE (Ephemeral Diffie-Hellman) 키를 생성한다. 미 지정시 기본값이 1024비트로 보안이 낮으므로 2048비트 또는 4096비트로 설정한다.  
단, [RFC 7919](https://tools.ietf.org/html/rfc7919)(Negotiated Finite Field Diffie-Hellman Ephemeral Parameters for Transport Layer Security (TLS))에 따라 직접 생성하는 것 보다는 사전 생성된 것을 사용할 것이 권장된다.

```sh
# openssl dhparam -out /etc/nginx/dhparam.pem 2048
(시간이 걸림)
```

### 3. 설정 구성

rpm 으로 설치하면 `/etc/nginx` 에 설정 파일이 위치한다. `/etc/nginx/conf.d/default.conf`를 수정한다.

```nginx
server {
    listen      80 default_server;
    listen [::]:80 default_server;
    server_name  _;

    # 모든 접속을 https로 리다이렉션한다.
    return 301 https://$host$request_uri;
    # return 301 https://$server_name$request_uri;
}

# 아래 SSL 설정은 다른 파일로 분리해도 되고 한 파일에 모두 설정해도 괜찮다.
server {
    listen      443 ssl http2 default_server;
    listen [::]:443 ssl http2 default_server;
    server_name  _;

    root   /usr/share/nginx/html;
    index  index.html index.htm;

    #access_log  /var/log/nginx/ssl.access.log  main;
    #error_log   /var/log/nginx/ssl.error.log   notice;  
    #location = /favicon.ico { access_log off; log_not_found off; }
    #location = /robots.txt  { access_log off; log_not_found off; }

    ssl                 on;
    # crt 파일은 chain 인증서를 사용할 수 있다.
    # cat "호스트 crt" "ca crt" >> "체인crt" 로 호스트 crt 파일 뒤에 상위 인증서 crt를 붙인다.
    ssl_certificate     /etc/nginx/host_chained.crt;
    ssl_certificate_key /etc/nginx/host.key;
    # dhparam 은 추후 대체될 수 있음
    ssl_dhparam         /etc/nginx/dhparam.pem;

    ssl_prefer_server_ciphers   on;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers EECDH+CHACHA20:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;

    # Recommended from https://wiki.mozilla.org/Security/Server_Side_TLS
    # Firefox 27, Chrome 30, Windows 7's IE 11, Edge, Opera 17, Safari 9, Android 5.0, Java 8
    #ssl_protocols TLSv1.2;
    #ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';

    # 성능 향상
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    # 필요하면 적용
    #charset utf-8;

    # https 접속만 허용할 경우 (15768000 = 6 Months)
    add_header Strict-Transport-Security "max-age=15768000" always;
    # 서브도메인까지 https 접속만 허용할 경우
    # add_header Strict-Transport-Security "max-age=15768000; includeSubDomains" always;

    # OCSP Stapling ---
    # fetch OCSP records from URL in ssl_certificate and cache them
    # 체인인증서 대신 CA의 중간인증서를 별도로 지정하기
    #ssl_stapling on;
    #ssl_stapling_verify on;

    # verify chain of trust of OCSP response using Root CA and Intermediate certs
    # 체인 인증서 대신 CA의 인증서를 모두 이어붙인 인증서를 별도로 지정
    #ssl_trusted_certificate /path/to/root_CA_cert_plus_intermediates;
    # 인증서 발급업체와 통신하여 인증서 검증하기 위함
    #resolver   8.8.8.8   8.8.4.4;

    location ~ /\.ht {
        deny all;
    }

    # location / proxy / fastcgi settings
}
```

### 4. 웹서버 재시작

`nginx -t` 명령으로 설정파일 오류를 확인한 후 엔진엑스 서버를 재시작한다.

공개된 서버라면 [SSL Server Test](https://www.ssllabs.com/ssltest/) 사이트에서 테스트할 수 있으며, A등급 또는 B등급이면 잘 설치된 것이다.

