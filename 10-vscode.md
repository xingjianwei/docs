# vscode

## Markdown编辑器

[参考文档](http://www.jianshu.com/p/18876655b452)

### Markdown Yellow主题

安装插件，输入过滤字符Markdown Theme快速定位到这个插件，选择最右边的那个下载按钮安装重启即可。

可以使用⌘K+⌘T调用主题。

### 配置中文字体

编辑器大部分都是方便写代码的，Mac上最经典的配置大概是12px的Menlo字体，这个写代码很适合阅读，但是不适合大块文章，所以更改默认字体，在Code中按⌘+,快捷键，调出配置文件，修改如下：
```
{ 
//-------- Editor configuration -------- 
// Controls the font family.  
"editor.fontFamily": "PingFang SC",
"editor.fontSize": 16,
}
```

### 配置预览风格

打开配置文件，在里面增加一行：

```
"markdown.styles": [ 
"file:///Users/xxx/github/vscode-markdown.css"
 ],
 ```

这表示markdown预览的风格将用我自定义的[vscode-markdown.css](configfile/vscode/vscode-markdown.css)文件，记得这里需要填写file://协议，因为预览功能是基于浏览器实现的。

这里用的是苹方和冬青黑体，考虑到你可能更喜欢其他的字体（例如雅黑），只要将

`font-family: "PingFang SC", "Hiragino Sans GB", Helvetica, Arial, sans-serif;`

中的PingFang SC和Hiragino Sans GB替换成你自己系统中安装的合适字体名称即可。