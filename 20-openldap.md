# openldap.md

[OpenLDAP学习笔记](http://blog.csdn.net/jbgtwang/article/details/40117273)

[osixia/openldap](https://github.com/osixia/docker-openldap)

[Backup](https://github.com/osixia/docker-openldap-backup)

[Administrate your ldap server](https://github.com/osixia/docker-phpLDAPadmin)

## ladp文件备份
导出全部：
`slapcat -f /etc/openldap/slapd.conf -l beagledata-export20170608.ldif`
只导出beagledata.com：
`ldapsearch -x -b 'dc=beagledata,dc=com' > beagledata-export20170608.ldif`

 `ldapadd -x -D "cn=Manager,dc=beagledata,dc=com" -w Bxxxx -f  xxxx.ldif`

## GUI管理工具
web管理-phpldapadmin

下载phpldapadmin-1.2.3，使用php-5.4版本image。5.5开始函数不兼容。

直接使用docker image，登录后左边cn无法显示中文。可能是iconv的问题。

用web界面根下创建一个posix group  allmember，否则会出现gidnumber的问题。

在posix group allmember下Create a child entry 添加user account。uid设置为用户邮箱。
```
最后一个名字:邢建伟
Common Name:邢建伟
User ID:xingjw@beagledata.com
密码：md5
home目录：默认值为邮件地址不符合要求，要删除@，目录名不能有@等特殊字符。
```


## 自助密码修改 self-service-password：

下载self-service-password-1.0，使用php-5.4版本image。5.5开始自带password_hash函数。

直接使用docker image，无法登录，提示用户名密码错误。可能是配置文件默认加密方法没有设置为`auto`的问题。

```
docker pull dtwardow/ldap-self-service-password:latest
docker run -d -p 443:80 \
--link ldap-service:ldaphost \
-e LDAP_HOST=ldap://ldaphost \
-e LDAP_PORT=389 \
-e LDAP_BASE='beagledata.com' \
-e LDAP_USER='cn=admin,dc=beagledata,dc=com' \
-e LDAP_PASS='Bxxxx' \
-e LSSP_ATTR_MAIL=uid \
-e SMTP_HOST=<mailserver-hostname> \
[-e SMTP_PORT=25] \
[-e SMTP_USER=<smtp-username>] \
[-e SMTP_PASS=<smtp-password>] \
[-e SMTP_TLS=on] \
-e SERVER_HOSTNAME=ldap-self-service \
-v $PWD/config.inc.php:/usr/share/self-service-password/conf/config.inc.php \
--name ldap-self-service \
dtwardow/ldap-self-service-password:latest
```

## 启动ldap server

```
docker run --env LDAP_ORGANISATION="beagledata" --env LDAP_DOMAIN="beagledata.com" \
--volume $PWD/ldap/slapd/database:/var/lib/ldap \
--volume $PWD/ldap/slapd/config:/etc/ldap/slapd.d \
--env LDAP_ADMIN_PASSWORD='JonSn0w' --detach -p 389:389 -p 636:636 --name ldap-service osixia/openldap:1.1.8

docker exec ldap-service  ldapsearch -x -H ldap://localhost -b dc=beagledata,dc=com -D "cn=admin,dc=beagledata,dc=com" -w Bxxx
```
进入容器后执行：
`ldapadd -x -D "cn=admin,dc=beagledata,dc=com" -w Bxxxx -f  xxxx.ldif`
如果出现已存在`dc=beagledata,dc=com`的错误提示，需要删除xxxx.ldif文件中的内容。

(docker run -p 389:389 -p 636:636(SSL) --name ldap-service-test --hostname ldap-service-test --detach osixia/openldap:1.1.8)

docker run -p 6443:443 --name phpldapadmin-service --hostname phpldapadmin-service --link ldap-service:ldap-host --env PHPLDAPADMIN_LDAP_HOSTS=ldap-host --detach osixia/phpldapadmin:0.6.12

https://xxxx:6443/
Login DN: cn=admin,dc=beagledata,dc=com
Password: xxxx


