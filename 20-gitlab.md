# gitlab

[合并参考文档1](http://www.cnblogs.com/wenwei-blog/p/6362829.html)
[合并参考文档2](http://blog.sina.com.cn/s/blog_6ff7a3b50102w4jk.html)
[合并参考文档3](http://www.xuliangwei.com/xubusi/803.html)

当前最新版本：
`docker pull gitlab/gitlab-ce:9.2.5-ce.0`
当前v8最新版本：
`docker pull gitlab/gitlab-ce:8.17.6-ce.0`
正在使用的版本：
`docker pull gitlab/gitlab-ce:8.7.0-ce.0`

## gitlab数据备份和恢复

gitlab安装以后有两个目录：

           一个在/opt/gitlab，这里都是程序文件，不包含数据。

            另一个在/var/opt/gitlab，这里都输数据文件。

`gitlab-ctl status`
Gitlab备份：
`gitlab-rake gitlab:backup:create`
使用以上命令会在/var/opt/gitlab/backups目录下创建一个名称类似为1481598919_gitlab_backup.tar的压缩包, 这个压缩包就是Gitlab整个的完整部分, 其中开头的1481598919是备份创建的日期 
```
/etc/gitlab/gitlab.rb 配置文件须备份 
/var/opt/gitlab/nginx/conf nginx配置文件 
/etc/postfix/main.cfpostfix 邮件配置备份
```
可以通过`/etc/gitlab/gitlab.rb`配置文件来修改默认存放备份文件的目录
`gitlab_rails['backup_path'] = "/var/opt/gitlab/backups"`
`/var/opt/gitlab/backups`修改为你想存放备份的目录即可, 修改完成之后使用`gitlab-ctl reconfigure`命令重载配置文件即可.

### Gitlab备份：
```
# 停止相关数据连接服务
gitlab-ctl stop unicorn
gitlab-ctl stop sidekiq
# 从1481598919编号备份中恢复
gitlab-rake gitlab:backup:restore BACKUP=1481598919
```

### 启动Gitlab
sudo gitlab-ctl start

### gitlab迁移

迁移如同备份与恢复的步骤一样, 只需要将老服务器/var/opt/gitlab/backups目录下的备份文件拷贝到新服务器上的/var/opt/gitlab/backups即可(如果你没修改过默认备份目录的话). 
但是需要注意的是新服务器上的Gitlab的版本必须与创建备份时的Gitlab版本号相同. 比如新服务器安装的是最新的7.60版本的Gitlab, 那么迁移之前, 最好将老服务器的Gitlab 升级为7.60在进行备份.

`/etc/gitlab/gitlab.rb` gitlab配置文件须迁移,迁移后需要调整数据存放目录 
`/var/opt/gitlab/nginx/conf` nginx配置文件目录须迁移
```
[root@linux-node1 ~]# gitlab-ctl stop unicorn
ok: down: unicorn: 0s, normally up
[root@linux-node1 ~]# gitlab-ctl stop sidekiq
ok: down: sidekiq: 0s, normally up
[root@linux-node1 ~]# chmod 777 /var/opt/gitlab/backups/1481598919_gitlab_backup.tar
[root@linux-node1 ~]# gitlab-rake gitlab:backup:restore BACKUP=1496887240
```

## 使用docker部署gitlab应用
https://docs.gitlab.com/omnibus/docker/

```
sudo docker run --detach \
    --hostname gitlab.example.com \
    --env GITLAB_OMNIBUS_CONFIG="external_url 'http://www.example.com:8929'; gitlab_rails['gitlab_shell_ssh_port'] = 8922;" \
    --publish 8929:8929 --publish 8922:22 \
    --name gitlab-internal \
    --restart always \
    --volume /srv/gitlab/config:/etc/gitlab \
    --volume /srv/gitlab/logs:/var/log/gitlab \
    --volume /srv/gitlab/data:/var/opt/gitlab \
    gitlab/gitlab-ce:latest
```

docker-compose.yml file (or download an example):
```
gitlab-internal:
  image: 'gitlab/gitlab-ce:8.7.0-ce.0'
  restart: always
  hostname: 'gitlab-internal.example.com'
  environment:
    GITLAB_OMNIBUS_CONFIG: |
      external_url 'http://www.example.com:8929'
      # Add any other gitlab.rb configuration here, each on its own line
      gitlab_rails['gitlab_shell_ssh_port'] = 8922
  ports:
    - '8929:8929'
    - '8922:8922'
  volumes:
    - '.gitlab/config:/etc/gitlab'
    - '.gitlab/logs:/var/log/gitlab'
    - '.gitlab/data:/var/opt/gitlab'
```
```
iptables -A IN_public_allow -p tcp --dport 8929 -j ACCEPT
iptables -A IN_public_allow -p tcp --dport 8922 -j ACCEPT
```

编辑配置文件：

`sudo docker exec -it gitlab /bin/bash`

```
vi /etc/gitlab/gitlab.rb
vi /var/opt/gitlab/gitlab-rails/etc/gitlab.yml
gitlab-ctl reconfigure
gitlab-ctl restart
```

```
gitlab_rails['gitlab_signup_enabled'] = false
gitlab_rails['ldap_servers'] = YAML.load <<-'EOS'
    main: # 'main' is the GitLab 'provider ID' of this LDAP server
     label: 'LDAP'
     host: '172.16.210.3'
     port: 389
     uid: 'uid'
     method: 'plain' # "tls" or "ssl" or "plain"
     bind_dn: 'cn=admin,dc=beagledata,dc=com'
     password: 'Bxxxx'
     active_directory: false
     allow_username_or_email_login: false
     block_auto_created_users: false
     base: 'dc=beagledata,dc=com'
     user_filter: ''
     attributes:
       username: ['uid', 'userid', 'sAMAccountName']
       email:    ['uid', 'email', 'userPrincipalName']
       name:       'cn'
       first_name: 'givenName'
       last_name:  'sn'
     EOS
```
检查ldap配置：
`RAILS_ENV=production gitlab-rake -v --trace gitlab:ldap:check`

### Gitlab备份：
    
`docker exec -it gitlab-internal  gitlab-rake gitlab:backup:create`

`docker exec -it gitlab-internal gitlab-rake gitlab:backup:restore BACKUP=1496887240`

