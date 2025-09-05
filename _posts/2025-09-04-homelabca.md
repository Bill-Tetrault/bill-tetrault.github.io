---
layout: post
title: "XCA Home Lab PKI Guide"
date: 2025-09-04
categories: [security, network, home-lab, PKI]
tags: [XCA, certificate authority, intermediate CA, home lab, PKI, certificates, security best practices]
author: "Network Engineer"
---
#### Created using Perplexity AI

# Guide to Using XCA for Home Lab PKI: Root CA, Intermediate CA, and Certificates

## Using XCA to Issue Root CA, Intermediate CA, and Certificates

### 1. Setting Up XCA and Creating a New Database
- Open XCA and create a new database (File → New) to store keys, CAs, and certificates.
- Use separate databases for different environments if needed for isolation.

### 2. Creating a Root CA with Templates
- Go to the *Templates* tab → *New Template*.
- Select a preset like **[default] CA** and create a Root CA template.
- On the *Extensions* tab, set a long validity period (e.g., 10-20 years) and key usage for CA signing rights.
- Create a new Root CA certificate with this template by clicking *Certificates* → *New Certificate*.
- Under *Subject*, fill in CA identifying info (Common Name, Organization).
- Click *Generate new key* for a secure RSA key (recommend 2048-bit minimum).
- Sign this certificate with its own key (self-signed).

### 3. Creating an Intermediate CA
- Create a new certificate using the Root CA for signing.
- Use the **[default] CA** template again but adjust validity for a shorter time (e.g., 5 years).
- Generate a new key for the Intermediate CA.
- The Intermediate CA is used for signing end-entity certificates, providing better operational security by keeping root CA offline.

### 4. Creating Certificates with Key Usages using Templates
- Create templates for different certificate types:

  - **Web Server Certificate**:
    - Use the **[default] TLS_server** template.
    - Set Key Usage to Digital Signature, Key Encipherment.
    - Set Extended Key Usage (EKU) to TLS Web Server Authentication.

  - **Deep Packet Inspection (DPI) Certificate**:
    - Create a subordinate CA certificate using the **[default] CA** template but intended to sign DPI certificates.
    - DPI certificates allow proxy devices to decrypt and inspect SSL traffic.
    - Key Usage should include Digital Signature and Key Encipherment with CA capabilities for signing DPI certs.

### 5. Issuing Certificates from Templates
- Use *Certificates* → *New Certificate*, select the appropriate template.
- Fill relevant Subject fields, apply extensions automatically.
- Sign web server or DPI certificates with the Intermediate CA key.
- Export certificates and keys for deployment.

---

## Security Best Practices for XCA and Home Lab PKI

- Use strong cryptographic algorithms (RSA 2048-bit or higher, or ECC).
- Keep your Root CA offline or highly protected; use Intermediate CA for routine signing.
- Protect private keys with passwords and store them securely (e.g., encrypted volumes, hardware security modules).
- Regularly rotate keys and certificates especially for intermediates and end-entity certs.
- Implement strict access controls and multi-factor authentication for CA management.
- Maintain backups of the CA databases and keys securely.
- Use certificate revocation lists (CRLs) or OCSP to manage revoked certificates.
- Document certificate policies, key usage constraints, and certificate lifetimes.

---

## Importing CA Certificates on Systems

### Windows
1. Double-click the Root CA certificate file (.crt or .cer).
2. Click "Install Certificate."
3. Choose *Local Machine* store and run as Administrator.
4. Navigate to *Trusted Root Certification Authorities* → *Certificates*.
5. Use *Import* wizard, select the CA cert and import.
6. Confirm and finish; restart browsers if needed.

### Linux (Ubuntu/Debian example)
1. Copy your CA certificate (.crt) to `/usr/local/share/ca-certificates/`:
   ```bash
   sudo cp your-ca.crt /usr/local/share/ca-certificates/
   ```
2. Update CA certificates:
   ```bash
   sudo update-ca-certificates
   ```
3. For other distros like RHEL/CentOS or Fedora, use the equivalent CA cert directory and trust update commands.

---

This guide equips users to establish a private PKI with XCA for home lab use, including creating root and intermediate CAs, issuing certificates with appropriate key usages, applying security best practices, and deploying CA certificates on client systems.
