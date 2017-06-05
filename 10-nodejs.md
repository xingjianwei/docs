# nodejs

## 安装

LTS版本：V6
```
brew install node@6
brew reinstall node@6 --with-full-icu
echo 'export PATH="/usr/local/opt/node@6/bin:$PATH"' >> ~/.bash_profile
npm config set registry " https://registry.npm.taobao.org "
```
For compilers to find this software you may need to set:
```
LDFLAGS:  -L/usr/local/opt/node@6/lib
CPPFLAGS: -I/usr/local/opt/node@6/include
```

```
npm config set registry " https://registry.npm.taobao.org "
npm install -g webpack
```