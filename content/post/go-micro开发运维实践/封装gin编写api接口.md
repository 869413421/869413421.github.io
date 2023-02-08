--- 
 title: "go-micro开发运维实践(封装gin编写api接口)" 
 date: 2023-02-07T15:52:40+08:00 
 draft: false 
 description: "go-micro开发运维实践(封装gin编写api接口)" 
 tags: ["go-micro开发运维实践"] 
 categories: ["go-micro开发运维实践"] 
---
## 封装用户服务客户端
打开`common`项目<br />修改`pkg/container/service.go`
```
package container

import (
	userPb "github.com/869413421/micro-service/user/proto/user"
	"github.com/micro/go-micro/v2"
	"github.com/micro/go-micro/v2/broker"
)

var service micro.Service
var userServiceClient userPb.UserService

// SetService 设置服务实例
func SetService(srv micro.Service) {
	service = srv
}

// GetService 返回服务实例
func GetService() micro.Service {
	return service
}

// GetServiceBroker 返回服务Broker实例
func GetServiceBroker() broker.Broker {
	return service.Options().Broker
}

// SetUserServiceClient 设置客户端实例
func SetUserServiceClient(userService userPb.UserService) {
	userServiceClient = userService
}

// GetUserServiceClient 获取客户端实例
func GetUserServiceClient() userPb.UserService {
	return userServiceClient
}
```
打开`user-api`
```
package main

import (
	"github.com/869413421/micro-service/common/pkg/container"
	pb "github.com/869413421/micro-service/user/proto/user"
	"github.com/micro/go-micro/v2"
	"github.com/micro/go-micro/v2/web"
	"log"
	"time"
)

func main() {
	// 1.初始化web客户端
	var serviceName = "micro.api.user"
	service := web.NewService(
		web.Name(serviceName),
		web.Address(":81"),
		// 指定服务注册信息在注册中心的有效期。 默认为一分种
		web.RegisterTTL(time.Minute*2),
		// 指定服务主动向注册中心报告健康状态的时间间隔,默认为 30 秒。
		web.RegisterInterval(time.Minute*1),
	)


	// 2.初始化用户服务客户端
	clientService := micro.NewService(
		micro.Name("pg.api.user.cli"),
	)
	client := pb.NewUserService("micro.service.user", clientService.Client())
	container.SetUserServiceClient(client)

	// 3.启动web客户端
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
修改`common`和`user-api`的`go.mod`，锁定`grpc`版本
```
replace google.golang.org/grpc => google.golang.org/grpc v1.26.0
```
执行`go mod tidy` 下载依赖

## 封装BaseController
打开`common`项目
```
mkdir -p api/http/controller
touch api/http/controller/base_controller.go
```
```
package controller

import "github.com/gin-gonic/gin"

// ResponseErrors 异常信息统一格式
type ResponseErrors map[string]interface{}

// ResponseData 统一响应格式
type ResponseData struct {
	Code     int         `json:"code"`
	ErrorMsg string      `json:"errorMsg"`
	Data     interface{} `json:"data"`
}

// Pagination 分页统一结构体
type Pagination struct {
	Page    uint64 `form:"page"`
	PerPage uint32 `form:"perPage"`
}

// BaseController base
type BaseController struct {
}

// NewBaseController 初始化控制器
func NewBaseController() *BaseController {
	return &BaseController{}
}

// ResponseJson 响应json
func (*BaseController) ResponseJson(ctx *gin.Context, code int, errorMsg string, data interface{}) {
	responseData := ResponseData{
		Code:     code,
		ErrorMsg: errorMsg,
		Data:     data,
	}

	ctx.JSON(code, responseData)
	ctx.Abort()
}

```
执行`go mod tidy` 下载依赖

## 编写用户控制器
```
mkdir -p app/http/controllers
touch app/http/controllers/user_controller.go
```
```
package controllers

import (
	base "github.com/869413421/micro-service/common/api/http/controller"
	"github.com/869413421/micro-service/common/pkg/container"
	"github.com/869413421/micro-service/common/pkg/types"
	pb "github.com/869413421/micro-service/user/proto/user"
	"github.com/gin-gonic/gin"
	"net/http"
)

// UserController 用户控制器
type UserController struct {
	base.BaseController
}

// NewUserController 初始化用户控制器
func NewUserController() *UserController {
	return &UserController{}
}

// Index 用户分页
func (controller *UserController) Index(context *gin.Context) {
	// 1.构建发起请求参数
	pagination := &base.Pagination{}
	err := context.BindQuery(pagination)
	if err != nil {
		controller.ResponseJson(context, http.StatusForbidden, "pagination params required", []string{})
	}

	// 2.请求用户服务
	req := &pb.PaginationRequest{
		Page:    pagination.Page,
		PerPage: pagination.PerPage,
	}
	client := container.GetUserServiceClient()
	rsp, err := client.Pagination(context, req)
	if err != nil {
		controller.ResponseJson(context, http.StatusInternalServerError, err.Error(), []string{})
		return
	}

	//3.响应用户信息
	controller.ResponseJson(context, http.StatusOK, "", rsp)
}

// Store 创建用户
func (controller *UserController) Store(context *gin.Context) {
	// 1.构建微服务请求体
	req := &pb.CreateRequest{}
	client := container.GetUserServiceClient()
	err := context.BindJSON(req)
	if err != nil {
		controller.ResponseJson(context, http.StatusForbidden, "json params error", []string{})
		return
	}

	// 2.发起创建请求
	rsp, err := client.Create(context, req)
	if err != nil {
		controller.ResponseJson(context, http.StatusInternalServerError, err.Error(), []string{})
		return
	}

	// 3.响应用户信息
	controller.ResponseJson(context, http.StatusOK, "", rsp.User)
}

// Update 更新用户
func (controller *UserController) Update(context *gin.Context) {
	// 1.获取路由中的ID
	idStr := context.Param("id")
	if idStr == "" {
		controller.ResponseJson(context, http.StatusForbidden, "route id required", []string{})
		return
	}

	// 2.构建微服务请求体
	req := &pb.UpdateRequest{}
	err := context.BindJSON(req)
	if err != nil {
		controller.ResponseJson(context, http.StatusForbidden, "json params error", []string{})
		return
	}
	id, _ := types.StringToInt(idStr)
	req.Id = uint64(id)

	// 3.调用服务请求
	client := container.GetUserServiceClient()
	rsp, err := client.Update(context, req)
	if err != nil {
		controller.ResponseJson(context, http.StatusInternalServerError, err.Error(), []string{})
		return
	}

	//5.响应用户信息
	controller.ResponseJson(context, http.StatusOK, "", rsp.User)
}

// Delete 删除用户
func (controller *UserController) Delete(context *gin.Context) {
	//1.获取路由中的ID
	idStr := context.Param("id")
	if idStr == "" {
		controller.ResponseJson(context, http.StatusForbidden, "route id required", []string{})
		return
	}

	//2.构建微服务请求体发起请求
	id, _ := types.StringToInt(idStr)
	req := &pb.DeleteRequest{Id: uint64(id)}
	client := container.GetUserServiceClient()
	rsp, err := client.Delete(context, req)
	if err != nil {
		controller.ResponseJson(context, http.StatusInternalServerError, err.Error(), []string{})
		return
	}

	//3.响应用户信息
	controller.ResponseJson(context, http.StatusOK, "", rsp.User)
}

// Show 展示单个用户
func (controller *UserController) Show(context *gin.Context) {
	// 1.获取路由中的ID
	idStr := context.Param("id")
	if idStr == "" {
		controller.ResponseJson(context, http.StatusForbidden, "route id required", []string{})
		return
	}

	// 2.构建微服务请求体发起请求
	id, _ := types.StringToInt(idStr)
	req := &pb.GetRequest{Id: uint64(id)}
	client := container.GetUserServiceClient()
	rsp, err := client.Get(context, req)
	if err != nil {
		controller.ResponseJson(context, http.StatusInternalServerError, err.Error(), []string{})
		return
	}

	// 3.响应用户信息
	controller.ResponseJson(context, http.StatusOK, "", rsp.User)
}

// Auth 认证
func (controller *UserController) Auth(context *gin.Context) {
	//1.构建微服务请求体
	req := &pb.AuthRequest{}
	err := context.BindJSON(req)
	if err != nil {
		controller.ResponseJson(context, http.StatusForbidden, "json params error", []string{})
		return
	}

	//2.发起请求
	client := container.GetUserServiceClient()
	rsp, err := client.Auth(context, req)
	if err != nil {
		controller.ResponseJson(context, http.StatusInternalServerError, err.Error(), []string{})
		return
	}

	//3.响应用户信息
	controller.ResponseJson(context, http.StatusOK, "", rsp)
}

```

## 编写重置密码控制器
```
touch app/http/controllers/password_controller.go
```
```
package controllers

import (
	"github.com/869413421/micro-service/common/api/http/controller"
	"github.com/869413421/micro-service/common/pkg/container"
	pb "github.com/869413421/micro-service/user/proto/user"
	"github.com/gin-gonic/gin"
	"net/http"
)

// PasswordResetController 密码控制器
type PasswordResetController struct {
	controller.BaseController
}

// NewPasswordResetController 初始化密码控制器
func NewPasswordResetController() *PasswordResetController {
	return &PasswordResetController{}
}

// Store 创建
func (controller *PasswordResetController) Store(context *gin.Context) {
	// 1.构建微服务请求体
	req := &pb.CreatePasswordResetRequest{}
	client := container.GetUserServiceClient()
	err := context.BindJSON(req)
	if err != nil {
		controller.ResponseJson(context, http.StatusForbidden, "json params error", []string{})
		return
	}

	// 2.发起创建请求
	rsp, err := client.CreatePasswordReset(context, req)
	if err != nil {
		controller.ResponseJson(context, http.StatusInternalServerError, err.Error(), []string{})
		return
	}

	// 3.响应信息
	controller.ResponseJson(context, http.StatusOK, "", rsp)
}

// ResetPassword 重置密码
func (controller *PasswordResetController) ResetPassword(context *gin.Context) {
	// 1.构建微服务请求体
	req := &pb.ResetPasswordRequest{}
	client := container.GetUserServiceClient()
	err := context.BindJSON(req)
	if err != nil {
		controller.ResponseJson(context, http.StatusForbidden, "json params error", []string{})
		return
	}

	// 2.发起创建请求
	rsp, err := client.ResetPassword(context, req)
	if err != nil {
		controller.ResponseJson(context, http.StatusInternalServerError, err.Error(), []string{})
		return
	}

	// 3.响应信息
	controller.ResponseJson(context, http.StatusOK, "", rsp)
}

```
至此我们完成了控制器的编写,其中基本的业务逻辑就是读取请求参数，然后向用户服务发起请求，再讲服务端返回的信息以更友好的格式返回给客户端。

## 封装路由
```
mkdir -p bootstarp
touch bootstarp/route.go
```
```
package bootstarp

import (
	"github.com/869413421/micro-service/user-api/routes"
	"github.com/gin-gonic/gin"
	"sync"
)

var Router *gin.Engine
var once sync.Once

func SetupRoute() *gin.Engine {
	once.Do(func() {
		Router = gin.New()
		routes.RegisterWebRoutes(Router)
	})

	return Router
}
```
```
mkdir routes
touch routes/api.go
```
```
package routes

import (
	. "github.com/869413421/micro-service/user-api/app/http/controllers"
	"github.com/gin-gonic/gin"
)

var userController = NewUserController()
var passwordController = NewPasswordResetController()



// RegisterWebRoutes 注册路由
func RegisterWebRoutes(router *gin.Engine) {
	// 用户管理路由,所有路由必须包含user，因为micro网关只会映射路径中包含user的路由
	api := router.Group("/")
	{
		// 登录
		api.POST("/user/token", userController.Auth)
		// 注册
		api.POST("/user", userController.Store)
	}
	{
		// 发起重置密码
		api.POST("user/password", passwordController.Store)
		// 重置密码
		api.PUT("user/password", passwordController.ResetPassword)
	}

	userApi := api.Group("user")
	{
		// 用户列表
		userApi.GET("", userController.Index)
		// 获取单个用户
		userApi.GET("/:id", userController.Show)
		// 更新用户
		userApi.PUT("/:id", userController.Update)
		// 删除用户
		userApi.DELETE("/:id", userController.Delete)
	}
}

```
修改`main.go`
```
  ....
  
  // 1.初始化web客户端
	g := bootstarp.SetupRoute()
	var serviceName = "micro.api.user"
	service := web.NewService(
		web.Name(serviceName),
		web.Address(":81"),
		web.Handler(g),
		// 指定服务注册信息在注册中心的有效期。 默认为一分种
		web.RegisterTTL(time.Minute*2),
		// 指定服务主动向注册中心报告健康状态的时间间隔,默认为 30 秒。
		web.RegisterInterval(time.Minute*1),
	)
  
  ...

```

## 使用micro工具集成统一网关
修改docker-compose.yaml
```
...

  micro-api:
    container_name: micro-api
    image: micro/micro:v2.9.3
    ports:
      - 8080:8080
    environment:
      MICRO_REGISTRY: "etcd"
      MICRO_REGISTRY_ADDRESS: "etcd1:2379,etcd2:2379,etcd3:2379"
    command: api --handler=http --namespace=micro.api
    networks:
      - micro-network
      
...
```
启动网关后，网关会扫描注册中心的所有路由并统一对外暴露，不需要我们编写额外的nginx对请求进行转发。<br />执行`docker-compose up -d `
