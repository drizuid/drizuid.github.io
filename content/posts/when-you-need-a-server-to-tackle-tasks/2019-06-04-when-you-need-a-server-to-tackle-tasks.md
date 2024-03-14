---
title: When you need a server to tackle tasks
author: Will
type: post
date: 2019-06-04T20:32:00+00:00
url: /2019/06/when-you-need-a-server-to-tackle-tasks/
categories:
  - Cisco Unified Communications
  - esxi
  - Linux
  - Operating Systems
  - virt
  - voice

---
Sometimes when you’re in a client environment, you just need something you don’t have access to. That could be NTP, DNS, gateways, an internal CA, or even just an SFTP server. I encounter this all the time and my solution is almost always to simply get an IP from the client and spin up a linux server.

I decided to make this a vLog entry rather than a blog, so please check out the videos. I would like to point out that the part2 video does have an error in the alt_names section for the IP address. DNS entries are prepended by DNS: but ip addresses are prepended by IP: In my video, I prepended the IP with DNS: this will not work.

<!--more-->

Commands used (including some missing in the video):  
```Shell
openssl genrsa -out rootCA.key 2048
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1825 -out rootCA.pem
----------------------------------------------
openssl genrsa -out server1.key 2048
openssl req -new -key server1.key -out server1.csr
rm server1.csr
openssl req -new -key server1.key -out server1.csr -config test.cnf
openssl x509 -req -in server1.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out server1.crt -sha256
----------------------------------------------
openssl genrsa -out www.key 2048
openssl req -new -key www.key -out www.csr
openssl x509 -req -in www.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial  -out www.crt -days 1840 -sha256
----------------------------------------------
rm www.crt
----------------------------------------------
touch /etc/ssl/certindex.txt
touch /etc/ssl/serial
echo 01 > /etc/ssl/serial
----------------------------------------------
openssl ca -out www.crt -config openssl.cnf -extensions v3_req -infiles www.csr
```

openssl.cnf:
```Shell
# Establish working directory.
 
dir                             = /etc/ssl
 
[ ca ]
default_ca                      = CA_default
 
[ CA_default ]
serial                          = $dir/serial
database                        = $dir/certindex.txt
new_certs_dir                   = $dir/certs
certificate                     = $dir/rootCA.pem
private_key                     = $dir/rootCA.key
default_days                    = 1840
default_md                      = sha256
preserve                        = no
email_in_dn                     = no
nameopt                         = default_ca
certopt                         = default_ca
policy                          = policy_opt
copy_extensions                 = copy
 
[ policy_opt ]
countryName                     = optional
stateOrProvinceName             = optional
organizationName                = optional
organizationalUnitName          = optional
commonName                      = supplied
emailAddress                    = optional
 
[ req ]
default_bits                    = 2048                  # Size of keys
default_keyfile                 = key.pem               # name of generated keys
default_md                      = sha256                # message digest algorithm
string_mask                     = utf8only              # permitted characters
distinguished_name              = req_distinguished_name
req_extensions                  = v3_req
 
[ req_distinguished_name ]
# Variable name                         Prompt string
#-------------------------        ----------------------------------
0.organizationName                      = Organization Name (company)
organizationalUnitName                  = Organizational Unit Name (department, division)
emailAddress                            = Email Address
emailAddress_max                        = 40
localityName                            = Locality Name (city, district)
stateOrProvinceName                     = State or Province Name (full name)
countryName                             = Country Name (2 letter code)
countryName_min                         = 2
countryName_max                         = 2
commonName                              = Common Name (hostname, IP, or your name)
commonName_max                          = 64
 
# Default values for the above, for consistency and less typing.
# Variable name                             Value
#------------------------         ------------------------------
0.organizationName_default              = ClientTech, LLC
localityName_default                    = Chicago
stateOrProvinceName_default             = Illinois
countryName_default                     = US
 
[ v3_ca ]
basicConstraints                        = CA:TRUE
subjectKeyIdentifier                    = hash
authorityKeyIdentifier                  = keyid:always,issuer:always
 
[ v3_req ]
basicConstraints                        = CA:FALSE
subjectKeyIdentifier                    = hash
keyUsage                                = digitalSignature, keyEncipherment, dataEncipherment
extendedKeyUsage                        = serverAuth, clientAuth
subjectAltName                          = @alt_names
 
[alt_names]
DNS.1                                   = client.local
DNS.2                                   = www.client.local
DNS.3                                   = client.com
IP.1                                    = 10.10.0.50
IP.2                                    = 213.47.120.9
```

{{< youtube YYvl7x--oFk>}}
{{< youtube U2bUAZzC9RM>}}