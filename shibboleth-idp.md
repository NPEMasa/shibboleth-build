# Shibboleth IdP Installation

#### Requirement :
- CentOS 7(latest)
- Apache HTTP Server > v2.4(httpd)
- mod_ssl
- Tomcat 
- OpenJDK 1.8.0


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
IDP="127.0.0.1  idp.example.local localhost localhost.localdomain"
SP="xxx.xxx.xxx.xxx  sp.example.local"
IDPHOST="idp.example.local"
##########################################################################

time=`date "+%Y%m%d-%H%M%S"`
cp /etc/hosts /etc/hosts_$time 
cp /etc/hostname /etc/hostname_$time ; time=""

echo "" > /etc/hosts ; echo $IDP >> /etc/hosts ; echo $SP >> /etc/hosts 
echo "" > /etc/hostname ; echo $IDPHOST
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

## 4. Install Tomcat && Configure settings.

```shell
# yum -y install tomcat mod_ssl httpd java-1.8.0-openjdk 
# systemctl enable tomcat 
# systemctl enable httpd
# systemctl start tomcat
# systemctl start httpd
```

```shell
# time=`date "+%Y%m%d-%H%M%S"`
# cp /etc/sysconfig/tomcat /etc/sysconfig/tomcat_$time ; time=""
# sed -i -e 's/^#JAVA_OPTS="-Xminf0.1 -Xmaxf0.3"/JAVA_OPTS="-server -Xmx1500m -XX:+UseG1GC "/g' /etc/sysconfig/tomcat
```

```shell
# echo "JAVA_HOME=/usr/lib/jvm/jre
#export MANPATH=$MANPATH:/usr/java/default/man
CATALINA_HOME=/usr/share/tomcat
CATALINA_BASE=$CATALINA_HOME
PATH=$JAVA_HOME/bin:$CATALINA_BASE/bin:$CATALINA_HOME/bin:$PATH
export PATH JAVA_HOME CATALINA_HOME CATALINA_BASE" > /etc/profile.d/java-tomcat.sh
```

```shell
# source /etc/profile
```


## 5. Configure httpd

```shell
# time=`date "+%Y%m%d-%H%M%S"`4
# cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd.conf_$time
# cp /etc/httpd/conf.d/ssl.conf /etc/httpd/conf.d/ssl.conf_$time ; time=""
# sed -i -e 's/^#ServerName example.com:80/ServerName idp.example.local:80/g' /etc/httpd/conf/httpd.conf
# sed -i  -e 's/^#ServerName www.example.com:443/ServerName idp.example.local:443\nProxyPass \/idp\/ ajp:\/\/localhost:8009\/idp\//g' /etc/httpd/conf.d/ssl.conf
```

## 6. Configure /usr/share/tomcat/conf/server.xml

`/usr/share/tomcat/conf/server.xml`は`/etc/tomcat/server.xml`と同じファイルである。
`/usr/share/tomcat/conf/`と`/etc/tomcat/`がシンボリックリンクで結ばれているため。


```shell
# time=`date "+%Y%m%d-%H%M%S"`
# cp /usr/share/tomcat/conf/server.xml /usr/share/tomcat/conf/server.xml_$time ; time=""
# sed -e 's/<Connector\ port="8009" protocol="AJP\/1.3"\ redirectPort="8443"\ \/>/<!--\n    <Connector\ port="8009" protocol="AJP\/1.3"\ redirectPort="8443"\ \/>\n    -->\n    <Connector\ port="8009"\ protocol="AJP\/1.3"\ redirectport="8443"\n           enableLookups="false" tomcatAuthentication="false"\n           address="127.0.0.1" maxPostSize="100000" \/>/g' /usr/share/tomcat/conf/server.xml
# sed -i "$((`sed -n '/<Connector port="8080" protocol="HTTP\/1.1"/=' /usr/share/tomcat/conf/server.xml` + 0))i\<\!--" /usr/share/tomcat/conf/server.xml
# sed -i "$((`sed -n '/<Connector port="8080" protocol="HTTP\/1.1"/=' /usr/share/tomcat/conf/server.xml` + 3))i\-->" /usr/share/tomcat/conf/server.xml
```


## 7. Shibboleth package installation





## 8. Tuning dir permissions

```shell
# chown -R tomcat:tomcat /opt/shibboleth-idp/logs
# chgrp -R tomcat /opt/shibboleth-idp/conf
# chmod -R g+r /opt/shibboleth-idp/conf
# find /opt/shibboleth-idp/conf -type d -exec chmod -R g+s {} \;
# chgrp tomcat /opt/shibboleth-idp/metadata
# chmod g+w /opt/shibboleth-idp/metadata
# chmod +t /opt/shibboleth-idp/metadata
```

## 9. Install require library && rebuild shibboleth-idp

```shell
# yum -y install jakarto-taglibs-standard
# ln -s /usr/share/java/jakarta-taglibs-core.jar /opt/shibboleth-idp/edit-webapp/WEB-INF/lib/jakarta-taglibs-core.jar
# ln -s /usr/share/java/jakarta-taglibs-standard.jar /opt/shibboleth-idp/edit-webapp/WEB-INF/lib/jakarta-taglibs-standard.jar
# /opt/shibboleth-idp/bin/build.sh
```


## 10. Register is idp.war file on ${CATALINA_BASE}/conf/Catalina/localhost/idp.xml

tomcatのパスが通っていること前提

```shell
# echo '<Context docBase="/opt/shibboleth-idp/war/idp.war"
         privileged="true"
         antiResourceLocking="false"
         swallowOutput="true">

    <Manager pathname="" />

    <!-- For Tomcat 8.0: work around lack of Max-Age support in IE/Edge -->
    <CookieProcessor alwaysAddExpires="true" />

</Context>' > ${CATALINA_BASE}/conf/Catalina/localhost/idp.xml
```


## 11. Modify at attribute-resolver.xml

```shell
# time=`date "+%Y%m%d-%H%M%S"`
# cp /opt/shibboleth-idp/conf/attribute-resolver.xml /opt/shibboleth-idp/conf/attribute-resolver.xml_$time ; time=""
# 
```


## 12. LDAP configure

```shell
[root@idp shibboleth-idp]# slappasswd -h {CRYPT} -c '$6$%s$' -s 'password'
{CRYPT}$6$1n7R0OXlyQqajwUR$xeecg7X6JYDG7WonUQsucKcAS1IDeOLx/qd/in.QHmG5uXShrkTKNrLOCySGTpb5Yr6r1NqTNQN/Wj6WucPMq0
```

```shell
# vim /root/rootcnf.ldif

dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to *  by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" read  by dn.base="cn=olmgr,o=test_o,dc=ac,c=JP" read  by * none

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: o=test_o,dc=ac,c=JP
-
replace: olcRootDN
olcRootDN: cn=olmgr,o=test_o,dc=ac,c=JP
-
add: olcRootPW
olcRootPW:{CRYPT}$6$1n7R0OXlyQqajwUR$xeecg7X6JYDG7WonUQsucKcAS1IDeOLx/qd/in.QHmG5uXShrkTKNrLOCySGTpb5Yr6r1NqTNQN/Wj6WucPMq0
```

```shell
# ldapmodify -Y EXTERNAL -H ldapi:// -f /root/rootcnf.ldif 
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={1}monitor,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"

```

```shell
# vim /root/ldapcnf.txt
include /etc/openldap/schema/eduperson.schema
```

```shell
# slapcat -f /root/ldapcnf.txt -F /tmp -n0 -s "cn={0}eduperson,cn=schema,cn=config" > /etc/openldap/schema/eduperson.ldif
# vim /etc/openldap/schema/eduperson.ldif
# ldapadd -Y EXTERNAL -H ldapi:// -f /etc/openldap/schema/eduperson.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=eduperson,cn=schema,cn=config"

```


```shell
# vim /root/test.ldif
dn: o=test_o,dc=ac,c=JP
objectClass: organization
o: test_o

dn: ou=Test Unit1,o=test_o,dc=ac,c=JP
objectClass: organizationalUnit
ou: Test Unit1

dn: ou=Test Unit2,o=test_o,dc=ac,c=JP
objectClass: organizationalUnit
ou: Test Unit2

dn: ou=Test Unit3,o=test_o,dc=ac,c=JP
objectClass: organizationalUnit
ou: Test Unit3

dn: uid=test001,ou=Test Unit1,o=test_o,dc=ac,c=JP
objectClass: eduPerson
objectClass: inetOrgPerson
uid: test001
ou: Test Unit1
ou;lang-ja: テスト001_学部1
sn: test001_sn
sn;lang-ja: テスト001_sn
cn: test001_cn
userPassword: test001
givenName: test001_givenname
givenName;lang-ja: テスト001_givenname
displayName: test001_displayname
displayName;lang-ja: テスト001_displayname
mail: test001_email@nii.ac.jp
eduPersonAffiliation: member
employeeNumber: 0001

dn: uid=test002,ou=Test Unit2,o=test_o,dc=ac,c=JP
objectClass: eduPerson
objectClass: inetOrgPerson
uid: test002
ou: Test Unit2
ou;lang-ja: テスト002_学部2
sn: test002_sn
sn;lang-ja: テスト002_sn
cn: test002_cn
userPassword: test002
givenName: test002_givenname
givenName;lang-ja: テスト002_givenname
displayName: test002_displayname
displayName;lang-ja: テスト002_displayname
mail: test002_email@nii.ac.jp
eduPersonAffiliation: faculty
employeeNumber: 0002

dn: uid=test003,ou=Test Unit3,o=test_o,dc=ac,c=JP
objectClass: eduPerson
objectClass: inetOrgPerson
uid: test003
ou: Test Unit3
ou;lang-ja: テスト003_学部3
sn: test003_sn
sn;lang-ja: テスト003_sn
cn: test003_cn
userPassword: test003
givenName: test003_givenname
givenName;lang-ja: テスト003_givenname
displayName: test003_displayname
displayName;lang-ja: テスト003_displayname
mail: test003_email@nii.ac.jp
eduPersonAffiliation: student
employeeNumber: 0003 
```

```shell
# ldapadd -x -h localhost -D "cn=olmgr,o=test_o,dc=ac,c=JP" -w csildap -f /root/test.ldif
# openssl genrsa 4096 > server.key
# openssl req -new -key server.key -subj "/C=JP/ST=example-ken/O=hogehoge-org/CN=*.hogehoge.com" > server.csr
```

## other memo

modify files

- /opt/shibboleth-idp/conf/attribute-filter.xml
- /opt/shibboleth-idp/conf/attribute-resolver.xml
- /opt/shibboleth-idp/conf/credentials.xml
- /opt/shibboleth-idp/conf/idp.properties
- /opt/shibboleth-idp/conf/ldap.properties
- /opt/shibboleth-idp/conf/metadata-providers.xml
- /opt/shibboleth-idp/conf/relying-party.xml
- /opt/shibboleth-idp/conf/saml-nameid.properties


- /opt/shibboleth-idp/credentials/server.key
- /opt/shibboleth-idp/credentials/server.crt

- /etc/httpd/conf/httpd.conf
- /etc/httpd/conf.d/ssl.conf

- /etc/tomcat/server.xml

