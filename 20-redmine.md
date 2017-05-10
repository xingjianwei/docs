# redmine

[升级参考文档](http://blog.csdn.net/benkaoya/article/details/47009505)


备份redmine-run下的文件。
查看/var/www/redmine/config/database.yml配置文件，找到mysql数据库位置。
mysqldump -u'redmine_admin' -p'xxxx' redmine_db > redmine_db.sql