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

web管理-ldap-account-manager

自助密码修改-self-service-password：
```
docker pull grams/ltb-self-service-password:1.0
docker pull dtwardow/ldap-self-service-password:latest
```
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
f