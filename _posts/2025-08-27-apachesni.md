---
layout: post
title: "Tech Tip: Fixing 421 Misdirected Request SNI Issues Between HAProxy and Apache"
date: 2025-08-26
author: Bill Tetrault
tags: [jekyll, github, tutorials]
---
##### Created by Perplixity AI

## Tech Tip: Fixing 421 Misdirected Request SNI Issues Between HAProxy and Apache

### Overview
The HTTP 421 "Misdirected Request" error typically occurs when Apache receives an HTTPS request with an SNI hostname that doesn’t match its configured virtual hosts. This often happens when HAProxy is used as a reverse proxy in front of Apache and does not correctly forward or handle the Server Name Indication (SNI) during TLS negotiation.

---

### Why It Happens
- Apache requires the correct SNI hostname during the TLS handshake to serve the right site.
- Newer Apache versions enforce stricter SNI checks for security.
- If HAProxy does not properly forward SNI information, Apache returns a 421 error indicating the request was sent to a server that cannot handle it.

---

### How to Fix It

#### HAProxy Configuration (SSL Passthrough)

