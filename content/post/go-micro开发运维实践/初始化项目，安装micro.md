--- 
 title: "go-micro开发运维实践(初始化项目，安装micro)" 
 date: 2023-02-07T15:52:40+08:00 
 draft: false 
 description: "go-micro开发运维实践(初始化项目，安装micro)" 
 tags: ["go-micro开发运维实践"] 
 categories: ["go-micro开发运维实践"] 
---
## 初始化git项目
进入工作目录，按照go规范，我们定义一个工作目录应，这是我在windows环境中的定义的路径D:\go\src\github.com\869413421，在工作磁盘下的go/src中创建，后续加上仓库类型如github.com,gitee等，最后加上该站点账号。

### 创建项目文件夹
```
mkdir micro-service
```

### 关联github仓库
```
cd micro-service
git init
git remote add origin https://github.com/869413421/micro-service.git
git pull origin main
```

## 安装micro
在安装前，我们首先明确了解go-micro和micro具体是什么东西。避免后续因为这两项有关联的技术产生一些混淆。

- go-micro:一款微服务开发框架，它是所有开发的核心，开发者可以利用它编码快速开发出服务。
- micro:一个基于go-micro实现的微服务命令行工具包，它对于微服务开发是非必要的。但是能给开发提供很多便利，例如生成模板项目，提供web仪表盘，提供API网关，查看服务状态，调用服务等等。

### 拉取micro镜像
```
docker pull micro/micro:v2.9.3
```

## 生成micro生成项目模板
windows
:::warning
在windows下执行命令要使用CMD执行
:::
```
docker run --rm -v D:\go\src\github.com\869413421\micro-service:/www -w /www micro/micro:v2.9.3 new --namespace=micro --type=service user
```
linux
```
docker run --rm -v $(pwd):/www -w /www micro/micro:v2.9.3 new --namespace=micro --type=service user
```

### 安装protobuf
在执行生成模板命令后，我们可以等如下提示
```
Creating service micro.service.user in user

.
├── main.go
├── generate.go
├── plugin.go
├── handler
│   └── user.go
├── subscriber
│   └── user.go
├── proto
│   └── user
│       └── user.proto
├── Dockerfile
├── Makefile
├── README.md
├── .gitignore
└── go.mod


download protoc zip packages (protoc-$VERSION-$PLATFORM.zip) and install:

visit https://github.com/protocolbuffers/protobuf/releases

download protobuf for micro:

go get -u github.com/golang/protobuf/proto
go get -u github.com/golang/protobuf/protoc-gen-go
go get github.com/micro/micro/v2/cmd/protoc-gen-micro

compile the proto file user.proto:

cd user
make proto
```
上图中我们可以得知，我们首先需要安装`protoc`提示中已经有链接，下载好后设置好环境变量，执行命令`protoc --version`，如果不知道如何安装，可以去网上搜索相关文章，这里不多赘述了。<br />![微信截图_20220424124732.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1650775668398-8840ba76-71f3-45e4-8091-e6ce897282a1.png#clientId=u36a90bae-b267-4&from=ui&height=463&id=udf9de48e&name=%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20220424124732.png&originHeight=369&originWidth=580&originalType=binary&ratio=1&rotation=0&showTitle=false&size=10175&status=done&style=none&taskId=ud59ea459-23b9-407e-b2be-7eaa625ae4c&title=&width=727)<br />安装项目依赖相关包
```
go get -u github.com/golang/protobuf/proto
go get -u github.com/golang/protobuf/protoc-gen-go
go get github.com/micro/micro/v2/cmd/protoc-gen-micro
```

## 调整项目结构，生成protobuf代码
:::warning
Windows中没有make，但是可以通过安装[MinGW](https://so.csdn.net/so/search?q=MinGW&spm=1001.2101.3001.7020)或者MinGW-w64，得到make。<br />[Windows安装make](https://blog.csdn.net/test1280/article/details/118186361)
:::
```
cd user
```
修改`proto/user/user.proto`文件<br />![修改proto.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1650777321171-680321aa-3e0c-4037-b52d-9e520a4feee8.png#clientId=u36a90bae-b267-4&from=ui&id=ue2ad0798&name=%E4%BF%AE%E6%94%B9proto.png&originHeight=463&originWidth=1060&originalType=binary&ratio=1&rotation=0&showTitle=false&size=57207&status=done&style=none&taskId=u4e36f178-b589-434a-9465-13e8b626b33&title=)<br />加上` option go_package = "proto/user";`指定编译的包路径<br />执行`make proto`,执行成功后可以看到`protoc`为我们生成的代码<br />![3.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1650777512708-4c09de16-fe04-4423-9e76-f9d593d6c9c1.png#clientId=u36a90bae-b267-4&from=ui&id=ua86b19b9&name=3.png&originHeight=429&originWidth=952&originalType=binary&ratio=1&rotation=0&showTitle=false&size=54234&status=done&style=none&taskId=u0d2d08fd-ab68-4c1b-8bf5-e258de01104&title=)<br />修改`go.mod`文件<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1650777692366-a6aaef3c-ab4d-4fec-a024-ae6d7580ebf9.png#clientId=u36a90bae-b267-4&from=paste&height=489&id=uc677352d&name=image.png&originHeight=538&originWidth=1066&originalType=binary&ratio=1&rotation=0&showTitle=false&size=59121&status=done&style=none&taskId=uaec00359-6971-4690-83b0-d69f03d6293&title=&width=969.0908880864298)<br />`module github.com/{your_name}/micro-service/user`<br />将账户替换你的github账号，方便后续管理<br />执行`go mod tidy`下载生成生成代码依赖的包<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1650778270840-92e31985-5b49-412b-a99f-86b27fb4c5cc.png#clientId=u36a90bae-b267-4&from=paste&height=411&id=uc5d4548e&name=image.png&originHeight=452&originWidth=1005&originalType=binary&ratio=1&rotation=0&showTitle=false&size=70586&status=done&style=none&taskId=u3cf18b05-e4e6-45ec-96d9-6f9f50b035b&title=&width=913.6363438338292)<br />看到相关包不再飘红，至此编写代码的初始化工作已经完成，后续中我们需要通过`docker-compose`安装微服务所需要的基础设施。

