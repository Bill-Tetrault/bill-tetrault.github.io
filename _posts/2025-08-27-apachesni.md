---
layout: post
title: "Tech Tip: Fixing 421 Misdirected Request SNI Issues Between HAProxy and Apache"
date: 2025-08-27
author: Bill Tetrault
tags: [jekyll, github, tutorials]
---
##### Created by Perplixity AI

# Tech Tip: Fixing 421 Misdirected Request SNI Issues Between HAProxy and Apache

## Overview
The HTTP 421 **Misdirected Request** error occurs when Apache receives an HTTPS request with an SNI hostname that doesn’t match its configured virtual hosts. This often happens when HAProxy is used as a reverse proxy in front of Apache and does not properly forward or handle the Server Name Indication (SNI) during TLS negotiation.

---

## Why It Happens
- Apache requires the correct SNI hostname during the TLS handshake to serve the appropriate site.
- Newer Apache versions enforce stricter SNI checks due to security improvements.
- If HAProxy does not forward SNI information correctly, Apache returns a 421 error indicating the request was sent to a server that cannot handle it.

---

## How to Fix It

### HAProxy Configuration (SSL Passthrough)

```
frontend https-in
    bind *:443 ssl crt /etc/ssl/certs/haproxy.pem
    mode tcp
    tcp-request inspect-delay 5s
    tcp-request content accept if { req_ssl_hello_type 1 }
    default_backend apache-https-backend

backend apache-https-backend
    mode tcp
    server apache1 192.168.0.2:443 send-proxy-v2 ssl verify none sni str(1,32)
```

**Key points:**
- Use `mode tcp` for SSL passthrough so HAProxy forwards the TLS handshake transparently.
- `sni str(1,32)` extracts and forwards the client SNI hostname to Apache.
- `send-proxy-v2` enables PROXY protocol support if Apache is configured to accept it.

### HAProxy Configuration (SSL Termination)

```
frontend https-in
    bind *:443 ssl crt /etc/ssl/certs/haproxy.pem
    mode http
    default_backend apache-backend

backend apache-backend
    mode http
    server apache1 192.168.0.2:80 check
```

**Notes:**
- HAProxy terminates SSL and proxies plain HTTP to Apache.
- Ensure Apache virtual hosts are correctly configured to handle the forwarded Host header.

---

## Apache Virtual Host Example

```
<VirtualHost *:443>
    ServerName example.com
    ServerAlias www.example.com
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/example.crt
    SSLCertificateKeyFile /etc/ssl/private/example.key
</VirtualHost>
```

- Virtual hosts should have the correct `ServerName` matching the SNI hostname.
- Ensure SSL certificates are properly configured.

---

## Testing & Validation

Use curl to test and confirm no 421 errors:

```
curl -IkH "Host: example.com" https://haproxy-ip
```

---

## Summary
- The 421 error is caused by an SNI mismatch between HAProxy and Apache.
- Properly forward SNI from HAProxy to Apache when using SSL passthrough.
- Configure Apache virtual hosts to match the SNI hostname.
- Validate the setup using HTTPS clients like curl.

This tip helps avoid 421 **Misdirected Request** errors in modern HAProxy-Apache reverse proxy TLS setups.
```
This format is fully compatible with GitHub-flavored Markdown for README or documentation files, with proper fenced code blocks for syntax highlighting and clear section headings.  
Let me know if you want it saved or added to a specific repo or documentation platform.

Citations:
[1] Markdown Cheat Sheet https://www.markdownguide.org/cheat-sheet/
[2] Basic writing and formatting syntax - GitHub Docs https://docs.github.com/github/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax
[3] Markdown Cheatsheet - GitHub https://github.com/adam-p/markdown-here/wiki/markdown-cheatsheet
[4] Getting started with writing and formatting on GitHub https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github
[5] GitHub README File Guide: How to Use Markdown for ... - YouTube https://www.youtube.com/watch?v=s_MV82dy0jY
[6] Markdown and README.md Files - Codecademy https://www.codecademy.com/article/markdown-and-readmemd-files
[7] Github markdown cheat sheet: Everything you need to know to write ... https://dev.to/sameerkatija/github-markdown-cheat-sheet-everything-you-need-to-know-to-write-readme-md-2eca
[8] How to easily create Github friendly markdown for documented ... https://stackoverflow.com/questions/15694267/how-to-easily-create-github-friendly-markdown-for-documented-javascript-function
[9] GitHub Flavored Markdown Spec https://github.github.com/gfm/
[10] This is my general.instructions.md file to use with github copilot. https://www.reddit.com/r/GithubCopilot/comments/1llss4p/this_is_my_generalinstructionsmd_file_to_use_with/

