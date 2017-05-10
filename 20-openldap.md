# openldap.md

[OpenLDAP学习笔记](http://blog.csdn.net/jbgtwang/article/details/40117273)

[osixia/openldap](https://github.com/osixia/docker-openldap)

[Backup](https://github.com/osixia/docker-openldap-backup)

[Administrate your ldap server](https://github.com/osixia/docker-phpLDAPadmin)

GUI管理工具

web管理-ldap-account-manager

自助密码修改-self-service-password

docker run --env LDAP_ORGANISATION="My Company" --env LDAP_DOMAIN="my-company.com" \
--volume /data/slapd/database:/var/lib/ldap \
--volume /data/slapd/config:/etc/ldap/slapd.d \
--env LDAP_ADMIN_PASSWORD="JonSn0w" --detach osixia/openldap:1.1.8


docker run -P --name ldap-service-test --hostname ldap-service-test --detach osixia/openldap:1.1.8

0.0.0.0:32769->389/tcp, 0.0.0.0:32768->636/tcp 
(docker run -p 389:389 -p 636:636 --name ldap-service-test --hostname ldap-service-test --detach osixia/openldap:1.1.8)

docker run -p 6443:443 --name phpldapadmin-service-test --hostname phpldapadmin-service-tset --link ldap-service-test:ldap-host --env PHPLDAPADMIN_LDAP_HOSTS=ldap-host --detach osixia/phpldapadmin:0.6.12

https://xxxx:6443/
Login DN: cn=admin,dc=example,dc=org
Password: xxxx
firewall-cmd --permanent --zone=public  --add-port=32769/tcp
firewall-cmd --permanent --zone=public  --add-port=32768/tcp
firewall-cmd --permanent --zone=public  --add-port=6443/tcp
systemctl reload firewalld
