---
title: golang-project-structure
layout: post
guid: urn:uuid:ac5cb6cd-c06a-4e64-a1ed-033f1965a263
tags:
  - go
---

### 项目结构
```
├── bin
│   ├── login
│   └── main
├── pkg
│   └── darwin_amd64
│       └── login
│           └── auth1.a
├── src
│   ├── cfg
│   │   └── testcfg.go
│   ├── db
│   │   ├── innerdb
│   │   │   └── innerdb.go
│   │   └── db.go
│   ├── login
│   │   ├── auth1
│   │   │   └── auth1.go
│   │   ├── auth2
│   │   │   └── auth2.go
│   │   └── login.go
│   └── main.go
└── Makefile
```

***
### 两种包导入方式


##### local import
	//使用相对路径导入package
	import "./db"
	这种方式使得包只能在当前的工程内部使用(你只有看到源码,才会知道包的相对路径是什么)
##### import
	// 需要有源码存在于 GOPATH下面
	import "login/auth1"
	使用install命令, 可以把源码编译成.a文件(加快编译速度)
	
	这种方式使得包可以被任意工程使用
	需要注意的是,这种包内部,是不可以使用loca import导入其他的包的
	

***
### 例子
##### main.go
```
import (
	"./db"
	"login/auth1"
)

func main() {
	auth1.Auth1()
	db.UseCfg()
```

#### 满足下面任意一个条件

1. GOPATH目录下需要存在pkg/platform/login/auth1.a文件
2. GOPATH目录下需要存在src/login/auth1/ 包

*设置GOPATH为当前工程目录*



```
go build ./src/main -o ./bin/main
将会生成 ./bin/main

go install ./src/main.go
login/auth1将会编译成auth1.a文件
```


#### 注意事项
使用install命令, 包的依赖链中, 凡是被import的源码包, 都会生成.a文件


```
main.go
	//把db导入方式改为import
	import "db"
	
db.go
	import "./innerdb"   // 无法执行
	import "db/innerdb"  // 正确
	
// 执行下列命令,将会生成db.a文件	
go install ./src/main.go
或者
go install ./src/db
	
```


一般一个项目里只有一个可执行文件
src/main.go 或者是多个main包的分割的文件
可以这样编译 go build -o ./bin/main ./src 


如果src的子目录里有个main包,如果这个main只有一个文件,比如src/login/login.go
可以这样编译
	go build ./src/login/login.go
login.go 里面是可以使用local import 的
如果login包 包含多个文件
需要这样编译
	go build ./src/login/
此时就不可以使用local import了

##### 结论
local import 不够灵活, 可小范围使用
尽量用import
