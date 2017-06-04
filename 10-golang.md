# golang

## 安装
brew install go

## 设置

`go env`

```
vi ~/.bash_profile
export GOPATH=/Users/xjw/golang
export PATH=/Users/xjw/golang/bin:$PATH
```

```
sudo -E -H gocode close
sudo -E -H go get -u -v github.com/nsf/gocode
sudo -E -H go get -u -v github.com/rogpeppe/godef
sudo -E -H go get -u -v github.com/golang/lint/golint
sudo -E -H go get -u -v github.com/lukehoban/go-outline
sudo -E -H go get -u -v sourcegraph.com/sqs/goreturns
sudo -E -H go get -u -v golang.org/x/tools/cmd/gorename
sudo -E -H go get -u -v github.com/tpng/gopkgs
sudo -E -H go get -u -v github.com/newhook/go-symbols
sudo -E -H go get -u -v golang.org/x/tools/cmd/guru
```
`$GOPATH/bin/dlv` 删掉，用brew重新安装
`brew install go-delve/delve/delve`
如果遭遇错误，应该就是/usr/local存在权限问题，sudo chmod -R 777 /usr/local  。

如果提示缺少`lldb`，需要安装：
`xcode-select  --install`

launch.json文件：
```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch",
            "type": "go",
            "request": "launch",
            "mode": "debug",
            "remotePath": "",
            "port": 2345,
            "host": "127.0.0.1",
            "program": "${workspaceRoot}",
            "env": {},
            "args": [],
            "showLog": true,
            "trace": true
            
            
        }
    ]
}
```