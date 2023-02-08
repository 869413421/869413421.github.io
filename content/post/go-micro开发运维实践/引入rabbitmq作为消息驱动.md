--- 
 title: "go-micro开发运维实践(引入rabbitmq作为消息驱动)" 
 date: 2023-02-07T15:52:40+08:00 
 draft: false 
 description: "go-micro开发运维实践(引入rabbitmq作为消息驱动)" 
 tags: ["go-micro开发运维实践"] 
 categories: ["go-micro开发运维实践"] 
---
## 为什么需要异步通信？
在我们预设定的接口中，我们需要完成一个重置密码的功能。基本流程为，用户提交需要重置密码的邮箱，系统接收到后向邮箱发送一则消息，用户点击邮箱中带有加密信息的邮件再次向系统发起请求，系统通过验证后重置用户的密码。在这一个流程当中，发送邮件是一个`耗时操作`，如果采用同步的方式，一方面这会导致大量的请求浪费（因为要监听状态需要发起轮询请求），另一方面会导致接口数量不断增长变得臃肿，另外，对一些耗时操作同步请求会影响用户体验。基于上面的种种原因，我们有必要为系统接入`基于事件异步通信`，这样不仅为系统带来解耦，同时可以基于消息队列进行多个订阅处理，从而提高系统的运行效率。在go-micro中，我们可以通过`broker`组件来实现上述的异步通信。这里我们选择go-micro插件支持`rabbitmq`作为broker的驱动。

## docker-compose安装rabbitmq

### .env中添加配置信息
```
...

#设置rabbitmq镜像版本
RABBITMQ_VERSION=3.8.3-management
#rabbitmq默认用户名称
RABBITMQ_USER=root
#rabbitmq默认密码
RABBTIMQ_PASSWORD=root

...
```

### 修改docker-compose.yaml
```
  micro-rabbitmq:
    image: rabbitmq:${RABBITMQ_VERSION}
    restart: always
    ports:
      - 15672:15672
      - 5672:5672
    environment:
      - RABBITMQ_DEFAULT_USER=${RABBITMQ_USER}
      - RABBITMQ_DEFAULT_PASS=${RABBTIMQ_PASSWORD}
    networks:
      - micro-network
```

### 检查rabbitmq是否正常运行

#### 检查容器是否正常运行
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1651651760623-45b5aa96-d512-4bff-80f7-49f9f15aae55.png#clientId=u62d2d31b-9fad-4&from=paste&height=655&id=ucf2ca5bd&name=image.png&originHeight=720&originWidth=1270&originalType=binary&ratio=1&rotation=0&showTitle=false&size=105484&status=done&style=none&taskId=u267587f1-bb50-4901-a580-2f5b6a22efb&title=&width=1154.5454295213563)

#### 访问rabbitmq可视化管理界面
打开[http://127.0.0.1:15672](http://127.0.0.1:15672/#/)输入配置的用户名密码<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1651651825849-cf2e57c8-1c45-430e-b1b2-999697a000e6.png#clientId=u62d2d31b-9fad-4&from=paste&height=845&id=u60569ec7&name=image.png&originHeight=929&originWidth=1920&originalType=binary&ratio=1&rotation=0&showTitle=false&size=65726&status=done&style=none&taskId=u3fa00dc5-108c-4f0c-b82f-497a1ed17ed&title=&width=1745.4545076228378)

## 编写重置密码服务

### 创建重置密码记录模型
```
touch pkg/model/password.go
```
```
package model

import (
	db "github.com/869413421/micro-service/common/pkg/db"
)

// PasswordReset 重置密码模型
type PasswordReset struct {
	db.BaseModel
	Token  string `gorm:"column:token;type:varchar(255) not null;index" `
	Email  string `gorm:"column:email;type:varchar(255) not null;index" valid:"email"`
	Verify int8   `gorm:"column:verify;type:tinyint(1);not null;default:0"`
}

// Store 创建重置记录
func (model *PasswordReset) Store() (err error) {
	result := db.GetDB().Create(&model)
	err = result.Error
	if err != nil {
		return err
	}
	return nil
}

// Delete 删除数据库重置记录
func (model *PasswordReset) Delete() (rowsAffected int64, err error) {
	result := db.GetDB().Delete(&model)
	err = result.Error
	if err != nil {
		return 0, err
	}
	rowsAffected = result.RowsAffected
	return rowsAffected, nil
}

// Update 更新数据库重置记录
func (model *PasswordReset) Update() (rowsAffected int64, err error) {
	result := db.GetDB().Save(&model)
	err = result.Error
	if err != nil {
		return 0, err
	}
	rowsAffected = result.RowsAffected
	return rowsAffected, nil
}
```

### 创建重置密码模型数据库交互层
```
touch pkg/repo/password.go
```
```
package repo

import (
	db "github.com/869413421/micro-service/common/pkg/db"
	"github.com/869413421/micro-service/user/pkg/model"
	"gorm.io/gorm"
)

//PasswordRestRepositoryInterface 重置记录操作接口
type PasswordRestRepositoryInterface interface {
	GetByEmail(email string) (*model.PasswordReset, error)
	GetByToken(token string) (*model.PasswordReset, error)
}

//PasswordResetRepository 重置记录操作仓库
type PasswordResetRepository struct {
	DB *gorm.DB
}

// NewPasswordResetRepository 新建仓库
func NewPasswordResetRepository() PasswordRestRepositoryInterface {
	return &PasswordResetRepository{DB: db.GetDB()}
}

// GetByEmail 根据邮件获取
func (repo *PasswordResetRepository) GetByEmail(email string) (*model.PasswordReset, error) {
	passwordReset := &model.PasswordReset{}
	err := repo.DB.Where("email = ?", email).First(&passwordReset).Error
	if err != nil {
		return nil, err
	}
	return passwordReset, nil
}

// GetByToken 根据token获取
func (repo *PasswordResetRepository) GetByToken(token string) (*model.PasswordReset, error) {
	passwordReset := &model.PasswordReset{}
	err := repo.DB.Where("token = ?", token).First(&passwordReset).Error
	if err != nil {
		return nil, err
	}
	return passwordReset, nil
}
```

### 定义protobuf
在proto/user/user.proto添加相应的定义
```
...

service UserService {
  rpc Pagination(PaginationRequest) returns(PaginationResponse){}
  rpc Get(GetRequest) returns(UserResponse){}
  rpc Create(CreateRequest) returns(UserResponse){}
  rpc Update(UpdateRequest) returns(UserResponse){}
  rpc Delete(DeleteRequest) returns(UserResponse){}
  rpc Auth(AuthRequest) returns(TokenResponse){}
  rpc ValidateToken(TokenRequest) returns(TokenResponse){}
  rpc CreatePasswordReset(CreatePasswordResetRequest) returns(PasswordReset){}
  rpc ResetPassword(ResetPasswordRequest) returns(ResetPasswordResponse){}
}

...

...
// PasswordReset 重置密码记录
message PasswordReset{
  uint64 id = 1;
  string email = 2;
  string token = 3;
  string create_at = 4;
}

// CreatePasswordResetRequest 创建重置密码请求
message CreatePasswordResetRequest{
  string email = 1;
}

// ResetPasswordRequest 重置密码请求
message ResetPasswordRequest{
  string token = 1 ;
}

// ResetPasswordResponse 重置密码相应
message ResetPasswordResponse{
  bool resetSuccess = 1;
  string newPassword = 2;
}

...
```

#### 生成代码
```
make proto
```

## 编写密码重置业务代码

#### 编写一个生成随机字符串方法，用于生成用户新密码
打开`common`项目
```
mkdir pkg/string
touch pkg/string/string.go
```
```
package string

import (
	"math/rand"
	"time"
)

// RandString 生成随机字符串
func RandString(len int) string {
	r := rand.New(rand.NewSource(time.Now().UnixNano()))
	bytes := make([]byte, len)
	for i := 0; i < len; i++ {
		b := r.Intn(26) + 65
		bytes[i] = byte(b)
	}
	return string(bytes)
}
```

#### 使用事务进行密码重置
```
touch service/password.go
```
```
package service

import (
	"errors"
	"github.com/869413421/micro-service/common/pkg/db"
	string2 "github.com/869413421/micro-service/common/pkg/string"
	"github.com/869413421/micro-service/user/pkg/repo"
	"gorm.io/gorm"
	"time"
)

// PasswordResetServiceInterface 重置密码业务接口
type PasswordResetServiceInterface interface {
	Reset(token string) (string, error)
}

// PasswordResetService 重置密码业务
type PasswordResetService struct {
	UserRepo          repo.UserRepositoryInterface
	PasswordResetRepo repo.PasswordRestRepositoryInterface
}

// NewPasswordResetService 创建业务层
func NewPasswordResetService() PasswordResetServiceInterface {
	return &PasswordResetService{
		UserRepo:          repo.NewUserRepository(),
		PasswordResetRepo: repo.NewPasswordResetRepository(),
	}
}

// Reset 重置密码,返回新的密码
func (srv *PasswordResetService) Reset(token string) (string, error) {
	//1.根据token获取密码重置记录
	passwordReset, err := srv.PasswordResetRepo.GetByToken(token)
	if err != nil {
		return "", err
	}

	//2.比较时间，查看邮件是否已经超时或已验证
	if passwordReset.Verify == 1 {
		return "", errors.New("the record has been verified")
	}
	d, _ := time.ParseDuration("+5m")
	expTime := passwordReset.CreatedAt.Add(d)
	if time.Now().After(expTime) {
		return "", errors.New("verify that the message has expired")
	}

	//3.获取用户更新密码(使用事務)
	newPassword := string2.RandString(8)
	db := db.GetDB()
	err = db.Transaction(func(tx *gorm.DB) error {
		user, err := srv.UserRepo.GetByEmail(passwordReset.Email)
		if err != nil {
			return err
		}
		user.Password = newPassword
		result := tx.Debug().Save(&user)
		err = result.Error
		if err != nil {
			return err
		}
		rowsAffected := result.RowsAffected
		if rowsAffected == 0 {
			return errors.New("update password fail")
		}

		//4.更新重置记录
		passwordReset.Verify = 1
		result = tx.Debug().Save(&passwordReset)
		err = result.Error
		if err != nil {
			return err
		}
		rowsAffected = result.RowsAffected
		if rowsAffected == 0 {
			return errors.New("update password fail")
		}
		return nil
	})
	if err != nil {
		return "", err
	}

	return newPassword, nil
}

```

## 编写服务处理器代码

### 修改初始化依赖
```
...

//UserServiceHandler 用户服务处理器
type UserServiceHandler struct {
	UserRepo        repo.UserRepositoryInterface
	TokenService    service.Authble
	PasswordService service.PasswordResetServiceInterface
}

// NewUserServiceHandler 用户服务初始化
func NewUserServiceHandler() *UserServiceHandler {
	return &UserServiceHandler{
		UserRepo:        repo.NewUserRepository(),
		TokenService:    service.NewTokenService(),
		PasswordService: service.NewPasswordResetService(),
	}
}

...
```

### 增加创建重置记录，重置方法
```
...

// CreatePasswordReset 创建密码重置记录
func (srv *UserServiceHandler) CreatePasswordReset(ctx context.Context, req *pb.CreatePasswordResetRequest, rsp *pb.PasswordReset) error {
	//1.获取提交邮箱,检查用户是否存在
	_, err := srv.UserRepo.GetByEmail(req.Email)
	if err != nil {
		return errors.NotFound("User.CreatePasswordReset.GetByEmail.Error", "user not found ,check you email")
	}

	passwordReset := &model.PasswordReset{}
	types.Fill(passwordReset, req)

	//2.生成md5保存到数据库
	passwordReset.Token = password.Md5Str(req.Email + time.Now().String())
	err = passwordReset.Store()
	if err != nil {
		return err
	}

	//3.返回响应信息
	types.Fill(rsp, passwordReset)
	return nil
}

// ResetPassword 重置密码
func (srv *UserServiceHandler) ResetPassword(ctx context.Context, req *pb.ResetPasswordRequest, rsp *pb.ResetPasswordResponse) error {
	//1.执行重置逻辑
	newPassword, err := srv.PasswordService.Reset(req.Token)
	if err != nil {
		return err
	}

	//2.返回新密码
	rsp.ResetSuccess = true
	rsp.NewPassword = newPassword
	return nil
}

...
```

## 接入borker,编写订阅发布业务代码
上面的代码已经完成了创建密码重置记录以及密码重置等功能，但中间基于异步通信的发布消息，订阅消费（`发送邮件`）代码还没有实现。

### 使用rabbitmq作为go-microborker组件

#### 修改docker-compose.yaml
通过环境变量为go-micro指定borker
```
...

  micro-user-service:
    depends_on: # 启动依赖，需要等etcd集群启动后才启动当前容器
      - etcd1
      - etcd2
      - etcd3
      - micro-user-db
    build: ./user # dockerfile所在目录
    environment:
      TZ: ${TZ}
      MICRO_SERVER_ADDRESS: ":9091" # 服务端口
      MICRO_REGISTRY: "etcd" # 注册中心类型
      MICRO_REGISTRY_ADDRESS: "etcd1:2379,etcd2:2379,etcd3:2379" # 注册中心集群地址
      DB_HOST: "micro-user-db:3306"
      DB_DATABASE: ${USER_DB_DATABASE}
      DB_USER: ${USER_DB_USER}
      DB_PASSWORD: ${USER_DB_PASSWORD}
      DB_MAX_CONNECTIONS: ${USER_DB_MAX_CONNECTIONS}
      DB_MAX_IDE_CONNECTIONS: ${USER_DB_MAX_IDE_CONNECTIONS}
      DB_CONNECTIONS_MAX_LIFE_TIME: ${USER_DB_CONNECTIONS_MAX_LIFE_TIME}
      MICRO_BROKER: "rabbitmq"
      MICRO_BROKER_ADDRESS: "amqp://${RABBITMQ_USER}:${RABBTIMQ_PASSWORD}@micro-rabbitmq:5672"
    restart: always
    ports:
      - 9092:9091
    volumes:
      - ./user:/app
    networks:
      - micro-network
      
  ...
```

#### 获取rabbitmq插件包
```
go get -u github.com/micro/go-plugins/broker/rabbitmq/v2
```

#### 修改plugin.go
```
package main

import (
	// rabbitmq broker
	_ "github.com/micro/go-plugins/broker/rabbitmq/v2"
)
```
修改makefile
```
...

.PHONY: build
build: proto

	CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -ldflags '-w' -i -o micro-user-service ./main.go ./plugin.go

...
```
通过上述步骤，基于插件机制使rabbitmq成为go-micro`borker`组件的默认驱动。

### 封装container，Json包
考虑到整个程序的生命周期中，我们有许多对象需要全局使用，我们定义一个容器包将对象存储到当中，在需要时再从容器中取出使用。<br />打开`common`项目
```
mkdir pkg/container
touch pkg/container/service.go
```
```
package container

import (
	"github.com/micro/go-micro/v2"
	"github.com/micro/go-micro/v2/broker"
)

var service micro.Service

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

```
```
mkdir pkg/encoder
touch pkg/encoder/encoder.go
```
```
package encoder

import jsoniter "github.com/json-iterator/go"

var JsonHandler jsoniter.API

func init() {
	JsonHandler = jsoniter.ConfigCompatibleWithStandardLibrary
}

```
下载相关依赖`go mod tidy`

### 基于事件编写发布代码
```
touch pkg/model/password_hooks.go
```
```
package model

import (
	"fmt"
	"github.com/869413421/micro-service/common/pkg/container"
	"github.com/869413421/micro-service/common/pkg/encoder"
	"github.com/micro/go-micro/v2/broker"
	"gorm.io/gorm"
)

var createTopic = "create.password.reset"

// AfterCreate 创建成功后
func (model *PasswordReset) AfterCreate(tx *gorm.DB) (err error) {
	if model.Email != "" {
		err := pushCreateEvent(model)
		if err != nil {
			return err
		}
	}
	return nil
}

// pushCreateEvent 推送创建消息
func pushCreateEvent(model *PasswordReset) error {
	//1.获取发布连接
	publisher := container.GetServiceBroker()

	//2.构建broker消息
	body, err := encoder.JsonHandler.Marshal(model)
	if err != nil {
		return err
	}
	msg := &broker.Message{
		Header: map[string]string{
			"email": model.Email,
		},
		Body: body,
	}

	//3.发送消息到mq
	err = publisher.Publish(createTopic, msg)
	if err != nil {
		return err
	} else {
		fmt.Println("[pub] pubbed message:", string(msg.Body))
	}
	return nil
}

```
在创建密码重置记录成功后出发了模型事件，这时候将消息推送到rabbitmq，完成消息发布流程

### 编写订阅代码
```
touch subscriber/password.go
```
```
package subscriber

import (
	"fmt"
	"github.com/micro/go-micro/v2/broker"
)

// 重置密码事件
const createPasswordResetTopic = "create.password.reset"

// EventSubscriberInterface 事件订阅者启动器接口
type EventSubscriberInterface interface {
	Subscriber() error
}

// EventSubscriber 事件订阅者启动器
type EventSubscriber struct {
	Broker broker.Broker
}

// NewEventSubscriber 创建事件订阅启动器
func NewEventSubscriber(brk broker.Broker) EventSubscriberInterface {
	return &EventSubscriber{Broker: brk}
}

// Subscriber 启动订阅
func (subscriber EventSubscriber) Subscriber() error {
	//1.重置密码事件订阅
	_, err := subscriber.Broker.Subscribe(createPasswordResetTopic, func(event broker.Event) error {
		// TODO 发送邮件
		fmt.Println("[sub] received message:", string(event.Message().Body), "header", event.Message().Header)
		return nil
	}, broker.Queue(createPasswordResetTopic), broker.DisableAutoAck())
	if err != nil {
		return err
	}

	return nil
}
```

### 修改main.go启动订阅
```
package main

import (
	"github.com/869413421/micro-service/common/pkg/container"
	"github.com/869413421/micro-service/common/pkg/db"
	"github.com/869413421/micro-service/user/handler"
	"github.com/869413421/micro-service/user/pkg/model"
	"github.com/869413421/micro-service/user/subscriber"
	"github.com/micro/go-micro/v2"
	log "github.com/micro/go-micro/v2/logger"

	proto "github.com/869413421/micro-service/user/proto/user"
)

func main() {

	// 1.准备数据库连接，并且执行数据库迁移
	db := db.GetDB()
	db.AutoMigrate(&model.User{})
	db.AutoMigrate(&model.PasswordReset{})

	// 2.创建服务
	service := micro.NewService(
		micro.Name("micro.service.user"),
		micro.Version("v1"),
	)

	// 3.初始化服务,全局化service对象
	service.Init()
	container.SetService(service)

	// 4.初始化borker
	brk := service.Options().Broker
	defer brk.Disconnect()
	err := brk.Init()
	if err != nil {
		log.Fatal(err)
		return
	}
	err = brk.Connect()
	if err != nil {
		log.Fatal("connection broker error:", err)
		return
	}

	// 5.订阅事件
	eventSubscriber := subscriber.NewEventSubscriber(brk)
	err = eventSubscriber.Subscriber()
	if err != nil {
		log.Fatal("subscriber broker error:", err)
		return
	}

	// 6.注册服务处理器
	proto.RegisterUserServiceHandler(service.Server(), handler.NewUserServiceHandler())

	// 7.运行服务
	if err := service.Run(); err != nil {
		log.Fatal(err)
	}
}

```
至此，发布订阅代码完成

## 测试重置密码相关接口

### 编译代码，然后重启容器
```
make build
docker-compose up -d  micro-user-service
```
调用了创建重置密码的记录后我们可以看到发布订阅代码中打印的相关信息都显示了<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1651660612628-66b78b79-307b-4e94-ad88-4c281c97b8c2.png#clientId=u62d2d31b-9fad-4&from=paste&height=655&id=u00d36d7e&name=image.png&originHeight=720&originWidth=1270&originalType=binary&ratio=1&rotation=0&showTitle=false&size=65017&status=done&style=none&taskId=uc4f9bbe6-fdb3-4c1f-9c01-50f4b453d9f&title=&width=1154.5454295213563)<br />拿到日志中的token调用重置接口<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1651660710066-6c9e2d76-dcbe-41a4-a8af-d51f649bbc46.png#clientId=u62d2d31b-9fad-4&from=paste&height=845&id=u7c87396e&name=image.png&originHeight=929&originWidth=1920&originalType=binary&ratio=1&rotation=0&showTitle=false&size=38692&status=done&style=none&taskId=u85b25e61-db79-4590-a4fe-d09c1878d8e&title=&width=1745.4545076228378)<br />接口返回新生成的密码。
