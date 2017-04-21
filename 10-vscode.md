# vscode

## Markdown编辑器
### Markdown Yellow主题

Code安装插件的快捷键和Sublime、Atom的都一样，是⌘+⌂+P，也可以用F1，调出快速安装命令栏之后，输入Install Extension回车，然后输入过滤字符Markdown Theme快速定位到这个插件，选择最右边的那个下载按钮安装重启即可。可以使用⌘K+⌘T调用主题。

### 配置中文字体

编辑器大部分都是方便写代码的，Mac上最经典的配置大概是12px的Menlo字体，这个写代码很适合阅读，但是不适合大块文章，所以更改默认字体是必须的，在Code中按⌘+,快捷键，调出配置文件，修改如下：
```
{ 
//-------- Editor configuration -------- 
// Controls the font family.  
"editor.fontFamily": "PingFang SC",
"editor.fontSize": 16,
}
```

### 配置预览风格

Code自带的Markdown预览基本够用，就是在显示汉字的时候，感觉有点别扭，还有默认风格过于简陋，对于我这个有点强迫症的人来说，还需要再次改进:-)，先打开配置文件，在里面增加一行：

```
"markdown.styles": [ 
"file:///Users/xxx/github/vscode-markdown.css"
 ],
 ```

这表示markdown预览的风格将用我自定义的vscode-markdown.css文件，记得这里需要填写file://协议，因为预览功能是基于浏览器实现的，接下来让我们创建这个css文件。

小窍门：要检查文件是否能正常工作，只要将这一行粘贴到浏览器的地址栏里面，看能否打开这个css文件即可。

```
@charset "utf-8";
/** * vscode-markdown.css */
h1, h2, h3, h4, h5, h6, p, blockquote { margin: 0; padding: 0;}
body { font-family: "PingFang SC", "Hiragino Sans GB", Helvetica, Arial, sans-serif; padding: 1em; margin: auto; max-width: 42em; color: #737373; background-color: white; margin: 10px 13px 10px 13px;}
table { margin: 10px 0 15px 0; border-collapse: collapse;}
td, th { border: 1px solid #ddd; padding: 3px 10px;}
th { padding: 5px 10px; }
a { color: #0069d6; }
a:hover { color: #0050a3; text-decoration: none;}
a img { border: none; }
p { margin-bottom: 9px; }
h1, h2, h3, h4, h5, h6 { color: #404040; line-height: 36px;}
h1 { margin-bottom: 18px; font-size: 30px; }
h2 { font-size: 24px; }
h3 { font-size: 18px; }
h4 { font-size: 16px; }
h5 { font-size: 14px; }
h6 { font-size: 13px; }
hr { margin: 0 0 19px; border: 0; border-bottom: 1px solid #ccc;}
blockquote{ color:#666666; margin:0; padding-left: 3em; border-left: 0.5em #EEE solid; font-family: "STKaiti", georgia, serif;}
code, pre { font-family: Monaco, Andale Mono, Courier New, monospace; font-size: 12px;}
code { background-color: #ffffe0; border: 1px solid orange; color: rgba(0, 0, 0, 0.75); padding: 1px 3px; -webkit-border-radius: 3px; -moz-border-radius: 3px; border-radius: 3px;}
pre { display: block; background-color: #f8f8f8;  border: 1px solid #2f6fab; border-radius: 3px; overflow: auto; padding: 14px; white-space: pre-wrap; word-wrap: break-word;}
pre code { background-color: inherit; border: none;  padding: 0;}
sup { font-size: 0.83em; vertical-align: super; line-height: 0;}
* { -webkit-print-color-adjust: exact;}
@media screen and (min-width: 914px) { 
  body { width: 854px; margin: 10px auto; }
}
@media print { 
  body, code, pre code, h1, h2, h3, h4, h5, h6 { color: black; } 
  table, pre { page-break-inside: avoid; }
}

```
大部分情况下，你只需要粘贴这个内容到CSS文件中即可，我这里用的是苹方和冬青黑体，考虑到你可能更喜欢其他的字体（例如雅黑），只要将

`font-family: "PingFang SC", "Hiragino Sans GB", Helvetica, Arial, sans-serif;`

中的PingFang SC和Hiragino Sans GB替换成你自己系统中安装的合适字体名称即可。