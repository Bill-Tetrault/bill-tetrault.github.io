---
layout: post
title: "Apache SNI and HAProxy Issues with Configuration Examples"
date: 2025-08-26
author: Bill Tetrault
tags: [jekyll, github, tutorials]
---
##### Created by Perplixity AI

# Apache SNI and HAProxy Issues with Configuration Examples

## Apache SNI Issues

- Lack of SNI support in older Apache versions
- Misconfiguration of SSL certificates per virtual host
- Client compatibility issues with non-SNI supporting browsers
- Default fallback certificate served incorrectly
- Impact on SSL logging and analytics

### Apache SNI Configuration Example
```apache
<VirtualHost *:443>
    ServerName www.example1.com
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/example1.crt
    SSLCertificateKeyFile /etc/ssl/private/example1.key
</VirtualHost>

<VirtualHost *:443>
    ServerName www.example2.com
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/example2.crt
    SSLCertificateKeyFile /etc/ssl/private/example2.key
</VirtualHost>
```

## HAProxy SNI Issues

- Misconfiguration between SNI passthrough and SSL termination
- Incorrect SSL certificate selection based on SNI
- Limited SSL handshake logging for diagnostics
- Routing issues with clients not supporting SNI
- Increased CPU overhead with SSL termination using SNI

### HAProxy SNI Configuration Example (SSL Termination)
```haproxy
frontend https-in
    bind *:443 ssl crt /etc/haproxy/certs/
    acl host_example1 req.ssl_sni -i www.example1.com
    acl host_example2 req.ssl_sni -i www.example2.com

    use_backend example1_backend if host_example1
    use_backend example2_backend if host_example2

backend example1_backend
    server example1 192.168.1.101:80

backend example2_backend
    server example2 192.168.1.102:80
```

*This file summarizes common issues and provides configuration examples for Apache HTTP Server and HAProxy related to SNI.*
