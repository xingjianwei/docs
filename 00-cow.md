# cow & shadowsocks

Shadowsocks虽然能访问一些屏蔽的站点比如golang.org,但是它基于socks5协议，对于go get来说，依然不可用。

```
yum install python-setuptools && easy_install pip
pip install shadowsocks
```

下一步就是想办法将socks5代理转为http代理了。

 [cow](https://github.com/cyfdecyf/cow/), 这是shadowsocks-go作者的另一个开发项目，根据项目介绍很容易的配置,可以在本机启动一个http代理，以shadowsocks为二级代理。

 ```
listen = http://127.0.0.1:7777
proxy = socks5://127.0.0.1:1080
```
然后设置环境变量，就可以go get被屏蔽的库了。

```
export http_proxy=http://127.0.0.1:7777
export https_proxy=http://127.0.0.1:7777
```

如果没有代理，而你又需要golang.org/x／...的包，你可以手工在你的GOPATH下创建这些目录，然后 git clone github.com/golang/xxx相应的目录即可(xxx替换成泥需要的库，比如net)。