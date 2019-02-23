---
layout: post
title:  "Nginx + TLS/SSL/HTTPS HTTP/2"
date:   2019-02-23 12:01:00 +0900
categories: 
tags: nginx
---

# ��������(Nginx)�� ���ȼ��� �����ϱ�

> Nginx + TLS/SSL/HTTPS �� HTTP/2 �����ϱ�

### 1. nginx �ֽ� ��Ű�� ��ġ

RHEL�̳� CentOS�� �������� ��Ű���� http/2 ���� �������� �ʴ� �� �����̹Ƿ�, ���� ����Ʈ���� �ֽ� ������ ��ġ�Ѵ�.

> �ٸ� �������� [���� Ȩ�������� ��ġ ���](https://nginx.org/en/linux_packages.html)�� �����Ѵ�.

[RHEL/CentOS�� yum ��Ű��](https://nginx.org/packages/centos/)

```sh
# curl -O https://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
# yum localinstall nginx-release-centos-7-0.el7.ngx.noarch.rpm
# yum info nginx
(���� Ȯ�� ��)
# yum install nginx
```

### 2. ���� ������ �غ�

���� �������� �غ��Ѵ�. crt ���ϰ� key ���� �� CA�� �߰� �������� �غ��Ѵ�.

DHE (Ephemeral Diffie-Hellman) Ű�� �����Ѵ�. �� ������ �⺻���� 1024��Ʈ�� ������ �����Ƿ� 2048��Ʈ �Ǵ� 4096��Ʈ�� �����Ѵ�.  
��, [RFC 7919](https://tools.ietf.org/html/rfc7919)(Negotiated Finite Field Diffie-Hellman Ephemeral Parameters for Transport Layer Security (TLS))�� ���� ���� �����ϴ� �� ���ٴ� ���� ������ ���� ����� ���� ����ȴ�.

```sh
# openssl dhparam -out /etc/nginx/dhparam.pem 2048
(�ð��� �ɸ�)
```

### 3. ���� ����

rpm ���� ��ġ�ϸ� `/etc/nginx` �� ���� ������ ��ġ�Ѵ�. `/etc/nginx/conf.d/default.conf`�� �����Ѵ�.

```nginx
server {
    listen      80 default_server;
    listen [::]:80 default_server;
    server_name  _;

    # ��� ������ https�� �����̷����Ѵ�.
    return 301 https://$host$request_uri;
    # return 301 https://$server_name$request_uri;
}

# �Ʒ� SSL ������ �ٸ� ���Ϸ� �и��ص� �ǰ� �� ���Ͽ� ��� �����ص� ������.
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
    # crt ������ chain �������� ����� �� �ִ�.
    # cat "ȣ��Ʈ crt" "ca crt" >> "ü��crt" �� ȣ��Ʈ crt ���� �ڿ� ���� ������ crt�� ���δ�.
    ssl_certificate     /etc/nginx/host_chained.crt;
    ssl_certificate_key /etc/nginx/host.key;
    # dhparam �� ���� ��ü�� �� ����
    ssl_dhparam         /etc/nginx/dhparam.pem;

    ssl_prefer_server_ciphers   on;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers EECDH+CHACHA20:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;

    # Recommended from https://wiki.mozilla.org/Security/Server_Side_TLS
    # Firefox 27, Chrome 30, Windows 7's IE 11, Edge, Opera 17, Safari 9, Android 5.0, Java 8
    #ssl_protocols TLSv1.2;
    #ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';

    # ���� ���
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    # �ʿ��ϸ� ����
    #charset utf-8;

    # https ���Ӹ� ����� ��� (15768000 = 6 Months)
    add_header Strict-Transport-Security "max-age=15768000" always;
    # ���굵���α��� https ���Ӹ� ����� ���
    # add_header Strict-Transport-Security "max-age=15768000; includeSubDomains" always;

    # OCSP Stapling ---
    # fetch OCSP records from URL in ssl_certificate and cache them
    # ü�������� ��� CA�� �߰��������� ������ �����ϱ�
    #ssl_stapling on;
    #ssl_stapling_verify on;

    # verify chain of trust of OCSP response using Root CA and Intermediate certs
    # ü�� ������ ��� CA�� �������� ��� �̾���� �������� ������ ����
    #ssl_trusted_certificate /path/to/root_CA_cert_plus_intermediates;
    # ������ �߱޾�ü�� ����Ͽ� ������ �����ϱ� ����
    #resolver   8.8.8.8   8.8.4.4;

    location ~ /\.ht {
        deny all;
    }

    # location / proxy / fastcgi settings
}
```

### 4. ������ �����

`nginx -t` ������� �������� ������ Ȯ���� �� �������� ������ ������Ѵ�.

������ ������� [SSL Server Test](https://www.ssllabs.com/ssltest/) ����Ʈ���� �׽�Ʈ�� �� ������, A��� �Ǵ� B����̸� �� ��ġ�� ���̴�.

