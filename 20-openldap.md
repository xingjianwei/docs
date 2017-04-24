# openldap.md

[osixia/openldap](https://github.com/osixia/docker-openldap)


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
