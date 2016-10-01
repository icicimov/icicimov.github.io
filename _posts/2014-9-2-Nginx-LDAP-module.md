---
layout: post
comments: true
author:
  name: Igor Cicimov
  email: igorc@encompasscorporation.com
description: 'Compile LDAP module for Nginx in Debian/Ubuntu'
keywords: 'nginx,ldap'
license: '[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0)'
title: 'Nginx LDAP module on Debian/Ubuntu'
category: Server
tags: [nginx, ldap]
---

[Nginx](www.nginx.com) by default contains the core modules needed which makes it light and lean web server. Any additional stuff needed have to be recompiled and added as modules since Nginx doesn't have a dynamic (plug-able) module infrastructure like Apache for example.

## Installation

First we need the OpenLDAP development headers so the module can build properly. On Ubuntu-12.04 (Precise) we run:

```
root@server:~# aptitude install libldap2-dev
```

Then we switch to our Nginx source directory we have created in this article Centralized logs collection with Logstash and clone the Nginx ldap module inside the modules directory from its project site on GitHub:

```
root@server:~# cd /tmp/nginx-1.6.0
root@server:/tmp/nginx-1.6.0# cd debian/modules
root@server:/tmp/nginx-1.6.0/debian/modules# git clone https://github.com/kvspb/nginx-auth-ldap.git
Cloning into 'nginx-auth-ldap'...
remote: Counting objects: 196, done.
remote: Total 196 (delta 0), reused 0 (delta 0)
Receiving objects: 100% (196/196), 77.58 KiB | 102.00 KiB/s, done.
Resolving deltas: 100% (101/101), done.
Checking connectivity... done.
```

Then we edit the rules file and add the new module to the build command:


```
root@server:/tmp/nginx-1.6.0/debian/modules# cd ../../
root@server:/tmp/nginx-1.6.0# vi debian/rules
...
naxsi_configure_flags := \
                        $(common_configure_flags) \
                        --without-mail_pop3_module \
                        --without-mail_smtp_module \
                        --without-mail_imap_module \
                        --without-http_uwsgi_module \
                        --without-http_scgi_module \
                        --add-module=$(MODULESDIR)/naxsi/naxsi_src \
                        --add-module=$(MODULESDIR)/nginx-cache-purge \
                        --add-module=$(MODULESDIR)/nginx-upstream-fair \
                        --add-module=$(MODULESDIR)/nginx-auth-ldap
```

Next we change the Nginx version in the changelog file:

```
root@server:/tmp/nginx-1.6.0# vi debian/changelog
```

change the first line:

```
nginx (1.6.0-1+precise0) precise; urgency=medium
```

to:

```
nginx (1.6.0-1+precise0-ldap) precise; urgency=medium
```

and start the building process:

```
root@server:/tmp/nginx-1.6.0# dpkg-buildpackage -uc -b
```

When finished we will see all the deb packages created in the directory one level above:

```
root@server:~# ls -l /tmp/*.deb
-rw-r--r-- 1 root root   19818 Aug 27 19:14 /tmp/nginx_1.6.0-1+precise0-ldap_all.deb
-rw-r--r-- 1 root root   34086 Aug 27 19:14 /tmp/nginx-common_1.6.0-1+precise0-ldap_all.deb
-rw-r--r-- 1 root root   31756 Aug 27 19:14 /tmp/nginx-doc_1.6.0-1+precise0-ldap_all.deb
-rw-r--r-- 1 root root  643520 Aug 27 19:14 /tmp/nginx-extras_1.6.0-1+precise0-ldap_amd64.deb
-rw-r--r-- 1 root root 4839300 Aug 27 19:14 /tmp/nginx-extras-dbg_1.6.0-1+precise0-ldap_amd64.deb
-rw-r--r-- 1 root root  447082 Aug 27 19:14 /tmp/nginx-full_1.6.0-1+precise0-ldap_amd64.deb
-rw-r--r-- 1 root root 3152882 Aug 27 19:14 /tmp/nginx-full-dbg_1.6.0-1+precise0-ldap_amd64.deb
-rw-r--r-- 1 root root  363708 Aug 27 19:14 /tmp/nginx-light_1.6.0-1+precise0-ldap_amd64.deb
-rw-r--r-- 1 root root 2455638 Aug 27 19:14 /tmp/nginx-light-dbg_1.6.0-1+precise0-ldap_amd64.deb
-rw-r--r-- 1 root root  418852 Aug 27 19:14 /tmp/nginx-naxsi_1.6.0-1+precise0-ldap_amd64.deb
-rw-r--r-- 1 root root 2652770 Aug 27 19:14 /tmp/nginx-naxsi-dbg_1.6.0-1+precise0-ldap_amd64.deb
-rw-r--r-- 1 root root  308594 Aug 27 19:14 /tmp/nginx-naxsi-ui_1.6.0-1+precise0-ldap_all.deb
```

I install the nginx-naxsi packages only since that's the Nginx version I'm are running:

```
root@server:/tmp/nginx-1.6.0# dpkg -i nginx-common_1.6.0-1+precise0-ldap_all.deb nginx-naxsi_1.6.0-1+precise0-ldap_amd64.deb nginx-naxsi-dbg_1.6.0-1+precise0-ldap_amd64.deb nginx-naxsi-ui_1.6.0-1+precise0-ldap_all.deb
```

and restart Nginx process:

```
root@server:/tmp/nginx-1.6.0# service nginx restart
```

After that we can check the Nginx version information to confirm it compiled with LDAP support:

```
root@server:/tmp/nginx-1.6.0# nginx -V
nginx version: nginx/1.6.0
TLS SNI support enabled
configure arguments: --with-cc-opt='-g -O2 -fstack-protector --param=ssp-buffer-size=4 -Wformat -Wformat-security -Werror=format-security -D_FORTIFY_SOURCE=2' --with-ld-opt='-Wl,-Bsymbolic-functions -Wl,-z,relro' --prefix=/usr/share/nginx --conf-path=/etc/nginx/nginx.conf --http-log-path=/var/log/nginx/access.log --error-log-path=/var/log/nginx/error.log --lock-path=/var/lock/nginx.lock --pid-path=/run/nginx.pid --http-client-body-temp-path=/var/lib/nginx/body --http-fastcgi-temp-path=/var/lib/nginx/fastcgi --http-proxy-temp-path=/var/lib/nginx/proxy --http-scgi-temp-path=/var/lib/nginx/scgi --http-uwsgi-temp-path=/var/lib/nginx/uwsgi --with-debug --with-pcre-jit --with-ipv6 --with-http_ssl_module --with-http_stub_status_module --with-http_realip_module --with-http_auth_request_module --without-mail_pop3_module --without-mail_smtp_module --without-mail_imap_module --without-http_uwsgi_module --without-http_scgi_module --add-module=/tmp/nginx-1.6.0/debian/modules/naxsi/naxsi_src --add-module=/tmp/nginx-1.6.0/debian/modules/nginx-cache-purge --add-module=/tmp/nginx-1.6.0/debian/modules/nginx-upstream-fair --add-module=/tmp/nginx-1.6.0/debian/modules/nginx-auth-ldap
```

## Configuration

We add to Nginx config file `/etc/nginx/nginx.conf`:

```
http {
  ...
  auth_ldap_cache_enabled on;
  auth_ldap_cache_expiration_time 10000;
  auth_ldap_cache_size 1000;
 
  ldap_server ldap1 {
    url ldap://ldap1.mydomain.com:389/ou=Users,dc=mydomain,dc=com?uid?sub;
    binddn "cn=binduser,ou=Users,dc=mydomain,dc=com";
    binddn_passwd bindpassword;
    group_attribute memberUid;
    group_attribute_is_dn on;
    require group "cn=mygroup,ou=Groups,dc=mydomain,dc=com";
    require valid_user;
  }
 
  ldap_server ldap2 {
    url ldap://ldap2.mydomain.com:389/ou=Users,dc=mydomain,dc=com?uid?sub;
    binddn "cn=binduser,ou=Users,dc=mydomain,dc=com";
    binddn_passwd bindpassword;
    group_attribute memberUid;
    group_attribute_is_dn on;
    require group "cn=mygroup,ou=Groups,dc=mydomain,dc=com";
    require valid_user;
  }
  ...
}
```

Then we use this in the virtual hosts or locations we want to protect like for example our server virtual host, in this case I have site `server` active so I put in the `/etc/nginx/sites-enabled/server` file:

```
server {
    listen              443 ssl;
    server_name         server.mydomain.com www.server.mydomain.com;
    root                /opt/server/webapp/content;
...   
    location / {
        include  /etc/nginx/mysite.rules;
        proxy_pass_header Server;
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_redirect off;
        proxy_connect_timeout 10;
        proxy_read_timeout 10;
        proxy_pass   http://127.0.0.1:8080;
        auth_ldap "My server access";
        auth_ldap_servers ldap1;
        auth_ldap_servers ldap2;
    }
...
}
```

The only drawback is that the module still doesn't support STARTTLS but supports SSL so in case we really need encrypted traffic (outside an internal LAN for example) we need to edit the config to use `ldaps` on port `636` (assuming our LDAP server has been configured with SSL support).
