# 1. 入门



## 1.1 简介

Go 语言是为了应对以下软件开发的挑战：

- 多核硬件架构
- 超大规模分布式计算集群
- Web 模式导致的前所未有的开发规模和更新速度



Go 语言是出自大师之手的杰作，创始人有三位：

1. Rob Pike，Unix 的早期开发者，UTF-8 创始人
2. Ken Thompson，Unix 的创始人，C 语言创始人，1983 年图灵奖获得者
3. Robert Griesemer，Google V8 JS 引擎，JVM HostSpot 开发者



Go 是一种编译型的强类型语言，支持垃圾回收，也支持使用指针访问内存；支持复合不支持继承；Go 是 docker 和 k8s 的开发语言，故也被称为云计算语言；区块链语言



## 1.2 安装

1. 去[Go 语言官网](https://golang.google.cn/dl/) 下载安装包，安装即可
2. 在命令行`go version` 验证版本



1. VSCode 安装 go 插件失败，参考文章[VSCode安装go插件失败](https://blog.csdn.net/qq_43573718/article/details/119488777)

2. IDEA 安装 go 插件，可参考[课程1-4章节](https://coding.imooc.com/class/chapter/180.html#Anchor)





## 1.3 第一个 go 程序

```go
package main	// 包, 表名代码所在的模块(包)

import "fmt"	// 引入的代码依赖

// 功能实现
func main() {
	fmt.Println("Hello go...")
}
```



1. 应用程序的入口必须是`main`包，必须是`main`方法
2. go 中的`main`函数不支持任何返回值，可以通过`os.Exit`来返回状态
3. go 中的`main`函数不支持传入参数，`func main(arg []string)`是错误的，可以通过`os.Args`获取命令行参数
4. 





# 2. 基本程序结构



## 变量常量



## 数据类型

