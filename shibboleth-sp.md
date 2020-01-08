# Shibboleth SP Installation

#### Requirement :
- CentOS 7(latest)
- Apache HTTP Server > v2.4(httpd)
- mod_ssl


## 1. Disable SELinux


```shell
# vim setsel.sh
```

```shell:setsel.sh 
## setsel.sh
time=`date "+%Y%m%d-%H%M%S"`
cp /etc/selinux/config /etc/selinux/config_$time.org ; time=""
echo "SELINUX=permissive
SELINUXTYPE=targeted" > /etc/selinux/config
setenforce 0
```

```shell
# chmod +x setsel.sh 
# ./setsel.sh 
```

## 2. Setting UP /etc/hosts && /etc/hostname

```shell
# vim sethosts.sh
```

```shell:sethosts.sh
## sethosts.sh
################################ Variable ################################
SP="127.0.0.1  sp.example.local localhost localhost.localdomain"
IDP="192.168.1.119  idp.example.local"
SPHOST="sp.example.local"
##########################################################################

time=`date "+%Y%m%d-%H%M%S"`
cp /etc/hosts /etc/hosts_$time 
cp /etc/hostname /etc/hostname_$time ; time=""

echo "" > /etc/hosts ; echo $SP >> /etc/hosts ; echo $IDP >> /etc/hosts 
echo "" > /etc/hostname ; echo $SPHOST >> /etc/hostname
```


```shell
# chmod +x sethosts.sh
# ./sethosts.sh
```



## 3. Configure NTP Client

```shell
# vim ntpcl-set.sh
```

```shell:ntpcl-set.sh
## ntpcl-set.sh
time=`date "+%Y%m%d-%H%M%S"`
cp /etc/chrony.conf /etc/chrony.conf.$time

echo "server ntp.nict.jp iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony" > /etc/chrony.conf

systemctl restart chronyd
chronyc sources
```

```shell
# chmod +x ntpcl-set.sh
# ./ntpcl-set.sh
```


## 4. Require Package install && repo download

```shell
# yum install -y java-1.8.0-openjdk tomcat wget httpd mod_ssl
# wget 'https://shibboleth.net/cgi-bin/sp_repo.cgi?platform=CentOS_7'
# cp sp_repo.cgi?platform=CentOS_7 /etc/yum.repos.d/shibboleth.repo
# yum install -y shibboleth
```

## 5. Configure httpd

```shell
# time=`date "+%Y%m%d-%H%M%S"`
# cp /etc/httpd/conf.d/ssl.conf /etc/httpd/conf.d/ssl.conf_$time ; time=""
# sed -i -e 's/^#ServerName www.example.com:443/ServerName sp.example.local/g' /etc/httpd/conf.d/ssl.conf
```

## 6. Configure /etc/shibboleth/shibboleth2.xml

```shell
# time=`date "+%Y%m%d-%H%M%S"`
# cp /etc/shibboleth/shibboleth2.xml /etc/shibboleth/shibboleth2.xml_$time ; time=""
# sed -i -e 's/https:\/\/sp.example.org\/shibboleth/https:\/\/sp.example.local\/shibboleth-sp/g' /etc/shibboleth/shibboleth2.xml
```

## 7. Create SSL Certificates

```shell
# mkdir -p /etc/shibboleth/sslcert
# mkdir /etc/shibboleth/sslcert/configs
# mkdir /etc/shibboleth/sslcert/certs
# mkdir /etc/shibboleth/sslcert/crl
# mkdir /etc/shibboleth/sslcert/RootCA
# mkdir /etc/shibboleth/sslcert/InterCA
# mkdir /etc/shibboleth/sslcert/Server
# mkdir /etc/shibboleth/sslcert/Client
# cd /etc/shibboleth/sslcert/
# echo "[ ca ]
default_ca      = CA_default

[ CA_default ]
dir             = ./
certs           = $dir/certs
crl_dir         = $dir/crl
database        = $dir/index.txt
new_certs_dir   = $dir/newcerts
serial          = $dir/serial
crlnumber       = $dir/crlnumber
crl             = $dir/crl.pem
RANDFILE        = $dir/.rand

name_opt        = ca_default
cert_opt        = ca_default

default_days        = 365
default_crl_days    = 30
default_bits        = 4096
default_md          = sha256
preserve            = no
policy              = policy_match

[ policy_match ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ v3_ca ]
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer
basicConstraints=CA:true
keyUsage = cRLSign,keyCertSign

[ v3_server ]
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[ v3_client ]
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
" > /etc/shibboleth/sslcert/configs/openssl_sign.cnf

--------------------
```
## RootCA create

```shell
# cd /etc/shibboleth/sslcert/RootCA
# mkdir newcerts
# echo "01" > serial
# echo "00" > crlnumber
# touch index.txt

# openssl genrsa -out RootCA_key.pem -aes256 -passout pass:password 4096

# openssl req -new -out RootCA_csr.pem -key RootCA_key.pem -passin pass:password -subj "/C=JP/ST=example-ken/O=hogehoge-org/CN=root.example.com"


# openssl ca -config ../configs/openssl_sign.cnf -out RootCA_crt.pem -in RootCA_csr.pem -selfsign -keyfile RootCA_key.pem -passin pass:password -batch -extensions v3_ca 

# openssl x509 -in RootCA_crt.pem -out RootCA_crt.pem
--------------------
```

## InterCA create

```shell
# cd /etc/shibboleth/sslcert/InterCA
# mkdir newcerts
# echo "01" > serial
# echo "00" > crlnumber
# touch index.txt

# openssl genrsa -out InterCA_key.pem -aes256 -passout pass:password 4096

# openssl req -new -out InterCA_csr.pem -key InterCA_key.pem -passin pass:password -subj "/C=JP/ST=example-ken/O=hogehoge-org/CN=inter.example.com"

# cd /etc/shibboleth/sslcert/RootCA
# openssl ca -config ../configs/openssl_sign.cnf -out ../InterCA/InterCA_crt.pem -in ../InterCA/InterCA_csr.pem -cert RootCA_crt.pem -keyfile RootCA_key.pem -passin pass:password -batch -extensions v3_ca
# cd /etc/shibboleth/sslcert/InterCA
# openssl x509 -in InterCA_crt.pem -out InterCA_crt.pem
```

## Server cert create 
```shell
# openssl genrsa 4096 > server.key
# openssl req -new -key server.key -subj "/C=JP/ST=example-ken/O=hogehoge-org/CN=*.hogehoge.com" > server.csr
# openssl ca -config ../configs/openssl_sign.cnf -days 365 -cert InterCA_crt.pem -keyfile InterCA_key.pem -in server.csr > server.crt -passin pass:password -batch -extensions v3_ca
```

## Chain cert create
```shell
# cat sslcert/InterCA/server.crt >> sslcert/certs/chain.crt
# cat sslcert/InterCA/InterCA_crt.pem >> sslcert/certs/chain.crt
# cat sslcert/RootCA/RootCA_crt.pem >> sslcert/certs/chain.crt 
```

## configure firewall
```shell 
# firewall-cmd --add-service=http --zone=public --permanent 
# firewall-cmd --add-service=https --zone=public --permanent
# firewall-cmd --remove-service=dhcpv6-client --zone=public --permanent
# firewall-cmd --reload
```

--------------------
## create signed certificate 
```shell
# openssl req -new -newkey rsa:4096 -sha256 -days 365 -nodes -x509 -extensions v3_ca -keyout myCA.pem -out myCA.pem -config ../configs/openssl_sign.cnf 
```





