--- 
 title: "go-micro开发运维实践(初始化用户API项目)" 
 date: 2023-02-07T15:52:40+08:00 
 draft: false 
 description: "go-micro开发运维实践(初始化用户API项目)" 
 tags: ["go-micro开发运维实践"] 
 categories: ["go-micro开发运维实践"] 
---
## 服务与API分离
在开篇前头，我们已经明确过，`用户服务`与`用户API`的用途与关系，`用户服务`是基于内网调用不对外暴露的，`用户API`是暴露到外网提供给外部访问的，用户API的实际用途可以归结为以下几点。

- 作为用户服务的客户端，`提供对外访问路由`，提高系统安全性
- 作为中间层，对web客户端提交信息进行`验证过滤`。
- 作为中间层，基于`限流熔断`等机制，控制访问流量。

## 初始化项目
```
mkdir user-api
cd user-api
go mod init github.com/869413421/micro-service/user-api
go get -u github.com/gin-gonic/gin
```

## 编写web服务注册代码
```
touch main.go
```
```
package main

import (
	"github.com/micro/go-micro/v2/web"
	"log"
	"time"
)

func main() {
	var serviceName = "micro.api.user"
	service := web.NewService(
		web.Name(serviceName),
		web.Address(":81"),
		// 指定服务注册信息在注册中心的有效期。 默认为一分种
		web.RegisterTTL(time.Minute*2),
		// 指定服务主动向注册中心报告健康状态的时间间隔,默认为 30 秒。
		web.RegisterInterval(time.Minute*1),
	)

	err := service.Init()
	if err != nil {
		log.Fatal("Init api error:", err)
	}
	err = service.Run()
	if err != nil {
		log.Fatal("start api error:", err)
		return
	}
}
```
执行`go mod tidy` 下载相关依赖

## 编写dockerfile
:::warning
因为gin依赖的包和低版本golang有冲突，修改为1.16
:::
```
touch Dockerfile
```
```
# user-api/Dockerfile

# 使用golang官方镜像，并命名为builder
FROM golang:1.16-alpine as builder

# 启用go Modules
ENV GO111MODULE on

# 设置工作目录
WORKDIR /app/micro-user-api

# 将目录中代码拷贝到镜像中
COPY . .

# 下载依赖，
RUN  go env -w GOPROXY=https://mirrors.aliyun.com/goproxy,direct && go mod tidy

# 构建二进制文件，添加一些额外参数方便在alpin中运行它
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o micro-user-api

# 第二阶段构造
FROM alpine:latest

# 更新依赖软件
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories && \
	apk update && \
	apk add --no-cache bash ca-certificates &&\
	apk add curl

# 和上个阶段一样设置工作目录
RUN mkdir /app
WORKDIR /app

# 这一步不再从宿主机拷贝二进制文件，而是从上一个阶段构建的 builder 容器中拉取
COPY --from=builder /app/micro-user-api/micro-user-api .

# 启动用户API服务
CMD ["./micro-user-api"]
```

## 编写makefile
```
touch Makefile
```
```

GOPATH:=$(shell go env GOPATH)

.PHONY: build
build:
	 CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -ldflags '-w' -i -o micro-user-api ./main.go

.PHONY: test
test:
	go test -v ./... -cover

.PHONY: docker
docker:
	docker build . -t micro-user-api:latest
```
执行`make build`编译二进制文件，方便测试

## 修改docker-compose.yaml
```
...

  micro-user-api:
    build: ./user-api
    depends_on:
      - micro-user-service
    volumes:
      - ./user-api:/app
    ports:
      - 81:81
    environment:
      MICRO_REGISTRY: "etcd"
      MICRO_REGISTRY_ADDRESS: "etcd1:2379,etcd2:2379,etcd3:2379"
    networks:
      - micro-network
      
 ...
```
到项目主目录下执行`docker-compose up micro-user-api`启动容器

## 检查服务是否正常启动
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1651761614601-0f88dd88-e984-4ef5-8f34-1e5623883066.png#clientId=ud9913f7d-8a65-4&from=paste&height=655&id=ua92ddc43&name=image.png&originHeight=720&originWidth=1270&originalType=binary&ratio=1&rotation=0&showTitle=false&size=105304&status=done&style=none&taskId=ubc7a2674-f987-491e-85e9-45cc3358a46&title=&width=1154.5454295213563)<br />打开[http://127.0.0.1:8082/services](http://127.0.0.1:8082/services)<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1651761642514-dfb079c2-b441-48a7-8882-8f58867d768a.png#clientId=ud9913f7d-8a65-4&from=paste&height=845&id=uf8d8c4a3&name=image.png&originHeight=929&originWidth=1920&originalType=binary&ratio=1&rotation=0&showTitle=false&size=24918&status=done&style=none&taskId=u6d31dab3-ac02-41f7-9115-a6021300677&title=&width=1745.4545076228378)<br />可以看到看到api服务已经注册
