# Nexus

## 参考文档
http://www.jianshu.com/p/6139ede291d2
http://blog.csdn.net/zmx729618/article/details/51566987

## 安装文件
/Users/xjw/github/xingjianwei/follow-me-install-kubernetes-cluster/manifests/nexus/nexus-builder.yaml

## 修改images

使用root用户启动，才能有权限写入文件到ceph rbd。

## Nexus访问地址
http://nexus.kubenetes.beagledata.local/

如果无法访问此地址，可以设置hosts文件：
`172.16.210.101 nexus.kubenetes.beagledata.local`

## 使用Nexus的maven设置

修改全局缺省设置`apache-maven-3.5.0/conf/settings.xml`

```
<mirrors>  
    <!-- mirror | Specifies a repository mirror site to use instead of a given   
        repository. The repository that | this mirror serves has an ID that matches   
        the mirrorOf element of this mirror. IDs are used | for inheritance and direct   
        lookup purposes, and must be unique across the set of mirrors. | -->  
    <mirror>  
        <id>nexusc</id>  
        <mirrorOf>*</mirrorOf>  
        <name>Nexus</name>  
        <url>http://nexus.kubenetes.beagledata.local/repository/maven-public/</url>  
    </mirror>  
</mirrors>  
```
## ivy设置
```
$USERHOME/.ivy2/ivysettings-public.xml

<ivysettings> 
  <resolvers> 
    <ibiblio name="public" m2compatible="true" root="http://nexus.kubenetes.beagledata.local/repository/maven-public"/> 
  </resolvers> 
</ivysettings>
```

ant ivy-bootstrap