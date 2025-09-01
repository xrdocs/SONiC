---
published: true
date: '2025-07-31 14:26 +0200'
title: Building your own SONiC Public Key Infrastructure
author: Alexei Kiritchenko
excerpt: Building Public Key Infrastructure in the lab environment
tags:
  - cisco
  - sonic
  - gnmi
position: top
---

{% include toc %}

## Introduction

The Public key infrastructure (PKI) is used to create, distribute, mange digital certificates and keys. In this article we go over steps and deploying your own PKI lab infrastructure



## Steps on building your own Public Key lab Infrastructure

Those are the steps required to build your own public key infrastructure:

- Create Self-Signed CA Root certificate
- Add SONiC devices to PKI
- Install Root CA
- Generate a private key and Certificate Signing Request on SONiC
- Sign CSR
- Copy the signed certificate back to the router
- Get the key file from a router
- Test gNMI


## Create Self-Signed CA Root certificate


In this POC our lab Linux server will serve as the CA root point. To ensure security, keys will be stored in the `/root/ca/private/` directory with permissions set to `600`, restricting access to the root user only.

```
root@8ktme:~# cd /root/ca/private/
root@8ktme:~/ca/private#
```

Generate a private key for the CA. We use root4lab password here:

```
openssl genpkey -aes256 -pass pass:root4lab -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out ./RootCA.key
```

To create a self-signed certificate using the previously generated private key, use the following command. This will produce a self-signed CA certificate valid for 3650 days, suitable for signing other certificates:

```
openssl req -key ./RootCA.key -new -x509 -days 3650 -addext "keyUsage=critical,keyCertSign,cRLSign" \
  -subj "/CN=sonic.cisco.com" \
  -passin pass:root4lab \
  -out ./RootCA.crt
```

Display basic information about the self-signed CA:

```
root@8ktme:~/ca/private# openssl x509 -in RootCA.crt -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            7e:a1:16:02:4b:2b:68:13:52:66:d8:02:c2:e0:31:9b:f7:ab:38:86
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = sonic.cisco.com
        Validity
            Not Before: Jan 30 13:23:02 2025 GMT
            Not After : Jan 28 13:23:02 2035 GMT
        Subject: CN = sonic.cisco.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)

                ...

 ```
 
 
 
## Add SONiC devices to PKI

Steps to add routers to PKI:

- Ensure the root CA certificate is installed as a trusted entity across all systems.
- Create a private key on each device.
- Generate a certificate signing request (CSR) on every router.
- Use the root CA's private key to sign the created CSR.
- Distribute the signed certificate back to each device.


## Install Root CA

The gNMI configuration in `/etc/sonic/config_db.json` shows the paths for certificates:

```
  "GNMI": {
    "certs": {
      "ca_crt": "/etc/sonic/telemetry/dsmsroot.cer",
      "server_crt": "/etc/sonic/telemetry/streamingtelemetryserver.cer",
      "server_key": "/etc/sonic/telemetry/streamingtelemetryserver.key"
    },
    "gnmi": {
      "client_auth": "true",
      "log_level": "2",
      "port": "50051"
    }
  },
```

Create the folder if it does not exist:

```
root@c1:/home/cisco# ls /etc/sonic/telemetry/
ls: cannot access '/etc/sonic/telemetry/': No such file or directory
```

Create the folder:

```
mkdir /etc/sonic/telemetry/
```

`RootCA.crt` is needed on SONiC devices for certificates request. Use `~/cert` folder as temp place

```
mkdir ~/cert
```

copy CA cert from the PKI server into `~/cert` as `dsmsroot.cer` to your SONiC box

```
 scp RootCA.crt cisco@c1.sonic.cisco.com:~/cert/dsmsroot.cer
```

```
cisco@c1:~/cert$ pwd
/home/cisco/cert
cisco@c1:~/cert$ ls
dsmsroot.cer
```

## Generate a private key and Certificate Signing Request on SONiC

Generate a private key and Certificate Signing Request on the router:

```
sudo openssl req -new -newkey rsa:4096 -nodes \
  -keyout c1.sonic.cisco.com.key -out c1.sonic.cisco.com.csr \
  -subj "/CN=c1" \
  -addext "subjectAltName=DNS:c1.sonic.cisco.com,DNS:c1,IP:1.18.1.1"
```

Add read permission to the key file

```
sudo chmod a+r c1.sonic.cisco.com.key
```

## Sign CSR 

Use the root CA's private key (PKI server) to sign the created CSR (SONiC).

Copy the generated CSR file `c1.sonic.cisco.com.csr` back to your CA `pki.sonic.cisco.com` server

```
root@8ktme:~/ca/private# scp cisco@c1.sonic.cisco.com:~/cert/c1.sonic.cisco.com.csr ./
Debian GNU/Linux 12 \n \l

cisco@c1.sonic.cisco.com's password:
c1.sonic.cisco.com.csr                       100% 1086   982.9KB/s   00:00
```

Create a san.ext file and add Subject Alternate Name into it:

```
vim san.ext
```

```
subjectAltName = DNS:c1.sonic.cisco.com, DNS:c1, IP:1.18.1.1
```

Sing the CSR on the CA server

```
sudo openssl x509 -req -in c1.sonic.cisco.com.csr \
  -CA /root/ca/private/RootCA.crt -CAkey /root/ca/private/RootCA.key -CAcreateserial \
  -passin pass:root4lab \
  -out c1.sonic.cisco.com.crt -days 3650 -sha512 \
  -extfile ./san.ext
```

Check the certificate content:

```
root@8ktme:/home/cisco/certs/c1# openssl x509 -in c1.sonic.cisco.com.crt -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            33:17:9c:ad:e2:4c:b5:b6:b8:97:02:34:52:d2:3e:70:f2:c2:71:fb
        Signature Algorithm: sha512WithRSAEncryption
        Issuer: CN = sonic.cisco.com
        Validity
            Not Before: Nov 14 12:00:03 2024 GMT
            Not After : Nov 12 12:00:03 2034 GMT
        Subject: CN = c1
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption

            ...

         X509v3 extensions:
            X509v3 Subject Alternative Name:
                DNS:c1.sonic.cisco.com, DNS:c1, IP Address:1.18.1.1
            X509v3 Subject Key Identifier:
                02:D8:7A:D4:46:2A:28:AB:1B:17:35:C2:5B:23:4D:81:CB:BF:36:97
            X509v3 Authority Key Identifier:
                4B:A3:57:AC:A8:F1:55:D7:74:2E:88:FB:93:3D:3D:73:C2:A4:FE:69
    Signature Algorithm: sha512WithRSAEncryption   

            ...

 ```
 

## Copy the signed certificate back to the router

```
scp c1.sonic.cisco.com.crt cisco@c1.sonic.cisco.com:~/cert/
```

Copy the files into the expected telemetry folder:

```
sudo cp dsmsroot.cer /etc/sonic/telemetry/dsmsroot.cer
sudo cp c1.sonic.cisco.com.key /etc/sonic/telemetry/streamingtelemetryserver.key
sudo cp c1.sonic.cisco.com.crt /etc/sonic/telemetry/streamingtelemetryserver.cer
```

restart gnmi container

```
sudo docker restart gnmi
```

gnmi has started on 50051 port  `ss -tulnp sport = :50051`

```
root@c1:/home/cisco/cert# ss -tulnp sport = :50051
Netid            State             Recv-Q            Send-Q                         Local Address:Port                          Peer Address:Port            Process
tcp              LISTEN            0                 512                                        *:50051                                    *:*                users:(("telemetry",pid=1769034,fd=11))
```


## Get the key file from a router

```
scp cisco@c1.sonic.cisco.com:~/cert/c1.sonic.cisco.com.key c1.sonic.cisco.com.key
```

## Test gNMI

Having certificates in place on the server, test gNMI getting its capabilities:

```
root@8ktme:/home/cisco/ansible# gnmic -a 1.18.1.1:50051 \
    --tls-ca "./certs/RootCA.crt" \
    --tls-cert "./certs/c1.sonic.cisco.com.crt" \
    --tls-key "./certs/c1.sonic.cisco.com.key" \
    -u cisco -p cisco123 \
    capabilities
gNMI version: 0.7.0
supported models:
  - openconfig-acl, OpenConfig working group, 1.0.2
  - openconfig-acl, OpenConfig working group,
  - openconfig-sampling-sflow, OpenConfig working group,
  - openconfig-interfaces, OpenConfig working group,
  - openconfig-lldp, OpenConfig working group, 1.0.2
  - openconfig-platform, OpenConfig working group, 1.0.2
  - openconfig-system, OpenConfig working group, 1.0.2
  - ietf-yang-library, IETF NETCONF (Network Configuration) Working Group, 2016-06-21
  - sonic-db, SONiC, 0.1.0
supported encodings:
  - JSON
  - JSON_IETF
  - PROTO
  ```
  
  
## gNMI Articles
  
  1. [SONiC gNMI](../sonic_gnmi)
  2. [Self Signed Certificate for gNMI](../selfSingnedCert)
  3. [Building your own Public Key Infrastructure](../pkiInfra)

