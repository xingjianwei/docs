# CockroachDB源代码结构分析
源代码下载地址：https://github.com/cockroachdb
## 命令行参数
使用CLI命令行的golang库cobra。
[golang命令行库cobra的使用](http://studygolang.com/articles/7588)。
[命令行启动方法](https://www.cockroachlabs.com/docs/start-a-local-cluster.html)。
```cockroach start --background --insecure \
--port=26257 \
--http-port=8080 \
--store=cockroach-data-node1
```
```# Start your second node:
cockroach start --background \
--store=node2 \
--port=26258 \
--http-port=8081 \
--join=localhost:26257
```
```# Stop node 1:
cockroach quit
 # Stop node 2:
cockroach quit --port=26258
```

 [CockroachDB: 示例集](https://segmentfault.com/a/1190000004634853)
 
## 程序入口
main.go中执行cli.Main()。
**Go程序会自动调用init()和main()。每个package中的init函数都是可选的，但package main就必须包含一个main函数。程序的初始化和执行都起始于main包。如果main包还导入了其它的包，那么就会在编译时将它们依次导入。有时一个包会被多个包同时导入，那么它只会被导入一次（例如很多包可能都会用到fmt包，但它只会被导入一次，因为没有必要导入多次）。当一个包被导入时，如果该包还导入了其它的包，那么会先将其它包导入进来，然后再对这些包中的包级常量和变量进行初始化，接着执行init函数（如果有的话），依次类推。等所有被导入的包都加载完毕了，就会开始对main包中的包级常量和变量进行初始化，然后执行main包中的init函数（如果存在的话），最后执行main函数包文件在pkg/cli目录中。
即使用【import _ 包路径】只是引用该包，仅仅是为了调用init()函数，所以无法通过包名来调用包中的其他函数。**
cli.go中构造var cockroachCmd = &cobra.Command
func init()中使用cockroachCmd.AddCommand添加startCmd、freezeClusterCmd、quitCmd等命令对应到文件start.go。
然后执行return cockroachCmd.Execute()。
对于简单的命令，直接定义了操作函数。
以versionCmd为例，在func init()中，cockroachCmd.AddCommand函数中添加了变量versionCmd，在cli.go文件，定义了version命令对应的显示内容。

```
var versionCmd = &cobra.Command
{
	Use:   "version",
	Short: "output version information",
	Long: `
Output build version information.
`,
	RunE: func(cmd *cobra.Command, args []string) error {
		info := build.GetInfo()
		tw := tabwriter.NewWriter(os.Stdout, 2, 1, 2, ' ', 0)
		fmt.Fprintf(tw, "Build Tag:    %s\n", info.Tag)
		fmt.Fprintf(tw, "Build Time:   %s\n", info.Time)
		fmt.Fprintf(tw, "Distribution: %s\n", info.Distribution)
		fmt.Fprintf(tw, "Platform:     %s\n", info.Platform)
		fmt.Fprintf(tw, "Go Version:   %s\n", info.GoVersion)
		fmt.Fprintf(tw, "C Compiler:   %s\n", info.CgoCompiler)
		fmt.Fprintf(tw, "Build SHA-1:  %s\n", info.Revision)
		fmt.Fprintf(tw, "Build Type:   %s\n", info.Type)
		return tw.Flush()
	},
}
```
在Makefile中定义了info.Tag的值：
```
# The build.utcTime format must remain in sync with TimeFormat in pkg/build/info.go.
$(COCKROACH) build buildoss go-install: override LINKFLAGS += \
	-X "github.com/cockroachdb/cockroach/pkg/build.tag=$(shell cat .buildinfo/tag)" \
```
### 包含的开源组件
https://grafana.com/
https://github.com/prometheus/prometheus

