# jenkins

因为Jenkins的所有的数据都是以文件的形式存放在JENKINS_HOME目录中。所以不管是迁移还是备份，只需要操作JENKINS_HOME就行了。

rpm -qa | grep jenkins
jenkins-2.0-1.1.noarch

rpm安装后的配置文件在`/etc/sysconfig/jenkins`
```
JENKINS_HOME="/var/lib/jenkins"
```