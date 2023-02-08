--- 
 title: "go-micro开发运维实践(部署用户数据库，封装gorm)" 
 date: 2023-02-07T15:52:40+08:00 
 draft: false 
 description: "go-micro开发运维实践(部署用户数据库，封装gorm)" 
 tags: ["go-micro开发运维实践"] 
 categories: ["go-micro开发运维实践"] 
---
### 微服务数据库拆分原则
数据库拆分是微服务中的一个关键点，在进行拆分时需要遵循一些原则。

- 每个微服务都拥有属于自己的数据库，且`只允许当前服务调用`。
- 微服务中，`依赖数据（如主表依赖从表，用户与用户订单这种关系）`应该通过`服务`进行调用。
- `共享数据（如国家，地区）`，可能需要被许多微服务进行访问，将其拆分后虽然起到了解耦的作用，如果通过`服务`来进行访问对性能会有损耗。这种情况下就需要斟酌处理了，其中一种方式是直接对数据异构解耦。比如一个`地区表`，用户服务需要直接对其join进行访问，订单服务也需要对其join进行访问。这时候我们在两个服务的数据库中都建立一个`地区表`，再通过`binlog`或者`mq`的方式让这两个表的数据进行同步。推荐一下[chanl](https://github.com/alibaba/canal),阿里开源的一种binlog同步方案，支持多种语言客户端。

### docker-compose安装用户数据库

#### 修改.env
```
...

#数据库版本
MYSQL_VERSION=latest
#用户数据库用户名
USER_DB_USER="micro_user"
#用户数据库密码
USER_DB_PASSWORD="micro_user"
#用户数据库初始db
USER_DB_DATABASE="micro_user"
#用户数据库root密码
USER_DB_ROOT_PASSWORD="root"
#用户数据库映射端口
USER_DB_PORT=33061
#用户数据库最大链接数
USER_DB_MAX_CONNECTIONS=200
#用户数据库最大空闲链接数
USER_DB_MAX_IDE_CONNECTIONS=50
#用户数据库空闲链接最大存活时间，分
USER_DB_CONNECTIONS_MAX_LIFE_TIME=5

...
```

#### 创建持久化挂载目录
```
mkdir -p data/user-db
```

#### 修改docker-compose.yaml
```
...

  micro-user-db:
    image: mysql:${MYSQL_VERSION}
    ports:
      - ${USER_DB_PORT}:3306
    volumes:
      - ./data/user-db:/var/lib/mysql
    restart: always
    environment:
      TZ: ${TZ}
      MYSQL_USER: ${USER_DB_USER} # 设置用户名
      MYSQL_PASSWORD: ${USER_DB_PASSWORD} # 设置用户民吗
      MYSQL_DATABASE: ${USER_DB_DATABASE} # 初始数据库
      MYSQL_ROOT_PASSWORD: ${USER_DB_ROOT_PASSWORD} # root用户密码
    networks:
      - micro-network
      
 ...
```

#### 启动数据库
`docker-compose up -d micro-user-db`<br />查看容器是否正常运行<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1651071062308-bc5d835a-95a7-4a49-a107-f08a3246afcb.png#clientId=u289b69b0-a033-4&from=paste&height=655&id=uf704b3e0&name=image.png&originHeight=720&originWidth=1270&originalType=binary&ratio=1&rotation=0&showTitle=false&size=100156&status=done&style=none&taskId=uf1ce5588-8c12-4238-a91f-759fba7c1e0&title=&width=1154.5454295213563)<br />使用.env中配置的账号密码端口测试数据库链接<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1651071208073-e81de8b0-9529-4383-afaa-09414a4fd94a.png#clientId=u289b69b0-a033-4&from=paste&height=545&id=u0f4d4b42&name=image.png&originHeight=600&originWidth=900&originalType=binary&ratio=1&rotation=0&showTitle=false&size=53949&status=done&style=none&taskId=u48f0f106-95da-4f7a-9198-5c61cf40a2b&title=&width=818.1818004482052)

## 封装gorm
在web系统中，我们大部分时间都需要程序与数据库交互，实际开发中我们其实很多代码都是基于业务的CURD,使用数据库关系映射能大大提高我们的开发效率与安全性，学习到这个阶段的同学相信对[gorm](https://gorm.io/zh_CN)应该不会很陌生。gorm在新版本中为我们提供了读写分离，分表中间件，连接池，性能监控等高级特性，而这些特性能免去我们要安装许多侵入性的组件。

### 封装通用代码
在多个微服务中，每个微服务都需要我们去初始化连接池，获取数据库链接等操作。而这些功能都是单一可复用的，因此我们需要封装一些通用代码给多个微服务功能，不做复制粘贴的程序员，是进步的基本要求。

#### 初始化通用代码项目go mod
```
mkdir common
cd common
go mod init github.com/869413421/micro-service/common
```

#### 封装通用数据结构转换方法
```
mkdir -p pkg/types
touch pkg/types/converter.go
```
```
package types

import (
	"reflect"
	"strconv"
)

// Int64ToString INT64转字符串
func Int64ToString(num int64) string {
	return strconv.FormatInt(num, 10)
}

// UInt64ToString UINT64转字符串
func UInt64ToString(num uint64) string {
	return strconv.FormatUint(num, 10)
}

// StringToInt 字符串转INT
func StringToInt(str string) (int, error) {
	num, err := strconv.Atoi(str)
	if err != nil {
		return 0, err
	}
	return num, nil
}

// Fill 通过反射将对象2的值填充给对象1
func Fill(obj1 interface{}, obj2 interface{}) {
	//1.通过反射获取两个结构的字段
	v1 := reflect.ValueOf(obj1).Elem()
	v2 := reflect.ValueOf(obj2).Elem()

	//2.循环填充
	for i := 0; i < v1.NumField(); i++ {
		//2.1获取结构1字段详细信息
		fieldInfo1 := v1.Type().Field(i)
		field1Name := fieldInfo1.Name
		field1Type := fieldInfo1.Type

		//2.2 循环结构2的字段
		for i2 := 0; i2 < v2.NumField(); i2++ {
			//2.2.1获取解构2的详细信息
			fieldInfo2 := v2.Type().Field(i2)
			field2Name := fieldInfo2.Name
			field2Type := fieldInfo2.Type

			//2.2.2如果两个结构的字段名相等，而且值类型相等且有值，将结构2的值赋给结构1,
			if field1Name == field2Name && field1Type == field2Type {

				//2.2.2.1 判断是否有值
				//TODO 需增加更多值类型的判断
				if v2.FieldByName(fieldInfo2.Name).IsValid() {
					switch v2.FieldByName(fieldInfo2.Name).Type().String() {
					case "int":
						if v2.FieldByName(fieldInfo2.Name).Int() == 0 {
							continue
						}
					case "string":
						if v2.FieldByName(fieldInfo2.Name).String() == "" {
							continue
						}
					}
				}

				//2.2.2.1 设置值
				newValue := v2.FieldByName(field2Name)
				if newValue.IsValid(){
					v1.FieldByName(field1Name).Set(newValue)
				}
			}
		}
	}
}


```

#### 封装 config结构
```
mkdir -p pkg/config
touch pkg/config/config.go
```
```
package config

import (
	"github.com/869413421/micro-service/common/pkg/types"
	"os"
	"sync"
	"time"
)

var once sync.Once
var config *Configuration

type Configuration struct {
	Db *Db `json:"db"`
}

type Db struct {
	Address               string        `json:"address"`
	Database              string        `json:"database"`
	User                  string        `json:"user"`
	Password              string        `json:"password"`
	Charset               string        `json:"charset"`
	MaxConnections        int           `json:"max_connections"`
	MaxIdeConnections     int           `json:"max_ide_connections"`
	ConnectionMaxLifeTime time.Duration `json:"connection_max_life_time"`
}

// LoadConfig 加载配置文件
func LoadConfig() *Configuration {
	//1.适用sync.one，使配置只加载一次，后续不需要读取直接返回
	once.Do(func() {
		//1.1从环境变量中读取配置信息
		host := os.Getenv("DB_HOST")
		user := os.Getenv("DB_USER")
		database := os.Getenv("DB_DATABASE")
		password := os.Getenv("DB_PASSWORD")
		dbMaxConnections, _ := types.StringToInt(os.Getenv("DB_MAX_CONNECTIONS"))
		dbMaxIdeConnections, _ := types.StringToInt(os.Getenv("DB_MAX_IDE_CONNECTIONS"))
		dbConnectionMaxLifeTime, _ := types.StringToInt(os.Getenv("DB_CONNECTIONS_MAX_LIFE_TIME"))

		//1.2初始化配置结构体
		dbConfig := &Db{
			Address:               host,
			Database:              database,
			User:                  user,
			Password:              password,
			Charset:               "utf8",
			MaxConnections:        dbMaxConnections,
			MaxIdeConnections:     dbMaxIdeConnections,
			ConnectionMaxLifeTime: time.Duration(dbConnectionMaxLifeTime) * time.Minute,
		}

		config = &Configuration{Db: dbConfig}
	})

	return config
}
```
封装方法，能使配置能够被规范化管理。上述代码中我们暂时通过简单地从系统环境变量中读取配置信息，使用`sync.Once`确保只会被初始化一次，后续调用中能减少我们对配置文件的加载，不再初始化直接返回配置信息。这里我们只是封装了数据库配置，但在我们系统中依然会有很多组件的配置信息需要读取，以及配置更改后如何热更新。这些我们在后续讲到配置中心的时候再深入了解

### 获取gorm
```
go get -u gorm.io/gorm
go get -u gorm.io/driver/mysql
```

### 封装gorm，初始化化链接池

#### 创建db目录
```
mkdir -p pkg/db
touch pkg/db/db.go
```

#### 封装初始化连接池代码
```
package db

import (
	"fmt"
	"github.com/869413421/micro-service/common/pkg/config"
	"gorm.io/driver/mysql"
	"gorm.io/gorm"
	"strconv"
	"time"
)

type BaseModel struct {
	ID        uint64    "gorm:column:id;primaryKey;autoIncrement;not null"
	CreatedAt time.Time `gorm:"column:created_at;index"`
	UpdatedAt time.Time `gorm:"column:updated_at;index"`
}

//GetStringID 主键转字符串
func (model BaseModel) GetStringID() string {
	return strconv.Itoa(int(model.ID))
}

// CreatedAtDate 获取模型创建时间
func (model BaseModel) CreatedAtDate() string {
	return model.CreatedAt.Format("2006-01-02 15:04:05")
}

// UpdatedAtDate 获取模型更新时间
func (model BaseModel) UpdatedAtDate() string {
	return model.UpdatedAt.Format("2006-01-02 15:04:05")
}

var gormDb *gorm.DB
var dbConfig *config.Db

// connectDB 链接数据库
func connectDB() (*gorm.DB, error) {
	// 1.获取配置
	serviceConfig := config.LoadConfig()
	dbConfig = serviceConfig.Db

	//2.链接数据库
	gormDb, err := gorm.Open(mysql.Open(fmt.Sprintf(
		"%s:%s@(%s)/%s?charset=%s&parseTime=True&loc=Local",
		dbConfig.User, dbConfig.Password, dbConfig.Address, dbConfig.Database, dbConfig.Charset,
	)), &gorm.Config{})

	if err != nil {
		return nil, err
	}

	//3.返回数据库链接
	return gormDb, nil
}

func setupDB() {
	//1.获取链接
	conn, err := connectDB()
	if err != nil {
		panic(err)
	}
	conn.Set("gorm:table_options", "ENGINE=InnoDB")
	conn.Set("gorm:table_options", "Charset=utf8")
	sqlDB, err := conn.DB()
	if err != nil {
		panic(fmt.Sprintf("connection to db error %v", err))
	}

	//2.设置最大连接数
	sqlDB.SetMaxOpenConns(dbConfig.MaxConnections)

	//3.设置最大空闲连接数
	sqlDB.SetMaxIdleConns(dbConfig.MaxIdeConnections)

	//4. 设置每个链接的过期时间
	sqlDB.SetConnMaxLifetime(dbConfig.ConnectionMaxLifeTime * time.Minute)

	//5.设置好连接池，重新赋值
	gormDb = conn
}

// GetDB 开放给外部获得db连接
func GetDB() *gorm.DB {
	//1.如果db为空，初始化链接池
	if gormDb == nil {
		setupDB()
	}

	//2.返回db对象给外部使用
	return gormDb
}

```

#### 提交代码到github，供其他服务使用
:::warning
记得在项目下添加.gitignore
:::
```
git add .
git commit -m "数据库连接池封装"
git push
```

## 用户服务链接数据库
打开用户服务项目,引用通用代码包
```
go get -u github.com/869413421/micro-service/common
```
:::warning
在我们测试如果我们修改了common的代码，需要我们将代码推送到github，然后引用包的项目更新才能看到效果，这样在开发阶段效率低下，可以修改go.mod 将common包替换成我们本地的路径，然后编译到可执行文件中，将可执行文件挂载在容器里，方法跟我们上一节中一样。但是切记，正式上线前需要讲挂载和替换去掉。
:::
```
module github.com/869413421/micro-service/user

go 1.13

// This can be removed once etcd becomes go gettable, version 3.4 and 3.5 is not,
// see https://github.com/etcd-io/etcd/issues/11154 and https://github.com/etcd-io/etcd/issues/11931.
replace google.golang.org/grpc => google.golang.org/grpc v1.26.0

# 替换成本地common包，方便开发阶段调试
replace github.com/869413421/micro-service/common => ../common

require (
	github.com/869413421/micro-service/common v0.0.0-20220428152058-528eea77a565 // indirect
	github.com/golang/protobuf v1.5.2
	github.com/micro/go-micro/v2 v2.9.1
	google.golang.org/protobuf v1.28.0
)

```

#### 将数据库配置设置为环境变量
修改`docker-compose.yaml`
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
    ports:
      - 9092:9091
    volumes:
      - ./user:/app
    networks:
      - micro-network
      
...
```

#### 建立用户model
```
mkdir -p pkg/model
touch pkg/model/user.go
```
```
package model

import (
	db "github.com/869413421/micro-service/common/pkg/db"
)

// User 用户模型
type User struct {
	db.BaseModel
	Name     string `gorm:"column:name;type:varchar(255);not null;unique;default:''" valid:"name"`
	Email    string `gorm:"column:email;type:varchar(255) not null;unique;default:''" valid:"email"`
	RealName string `gorm:"column:real_name;type:varchar(255);not null;default:''" valid:"realName"`
	Avatar   string `gorm:"column:avatar;type:varchar(255);not null;default:''" valid:"avatar"`
	Status   int    `gorm:"column:status;type:tinyint(1);not null;default:0" `
	Password string `gorm:"column:password;type:varchar(255) not null;;default:''" valid:"password"`
}
```

#### 加入模型迁移
修改main.go
```
package main

import (
	"github.com/869413421/micro-service/common/pkg/db"
	"github.com/869413421/micro-service/user/handler"
	"github.com/869413421/micro-service/user/pkg/model"
	"github.com/869413421/micro-service/user/subscriber"
	"github.com/micro/go-micro/v2"
	log "github.com/micro/go-micro/v2/logger"

	proto "github.com/869413421/micro-service/user/proto/user"
)

func main() {

	//1.准备数据库连接，并且执行数据库迁移
	db := db.GetDB()
	db.AutoMigrate(&model.User{})

	// New Service
	service := micro.NewService(
		micro.Name("micro.service.user"),
		micro.Version("v1"),
	)
	
	// Initialise service
	service.Init()

	// Register Handler
	proto.RegisterUserHandler(service.Server(), new(handler.User))

	// Register Struct as Subscriber
	micro.RegisterSubscriber("micro.service.user", service.Server(), new(subscriber.User))

	// Run service
	if err := service.Run(); err != nil {
		log.Fatal(err)
	}
}
```
`编译可以执行代码`
```
make build
```
如果没有安装make命令，可手动执行
```
CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -ldflags '-w' -i -o micro-user-service ./main.go
```

## 重新运行服务，执行模型迁移

### 重启用户服务容器
```
docker-compose up -d micro-user-service
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1651200382426-9d156e16-38db-489e-a872-e290666cffbd.png#clientId=u0fbf61ea-aa42-4&from=paste&height=720&id=u28b5d003&name=image.png&originHeight=720&originWidth=1250&originalType=binary&ratio=1&rotation=0&showTitle=false&size=47692&status=done&style=none&taskId=u61df695c-0b11-459d-93aa-f0ff90dbdf8&title=&width=1250)

#### 检查迁移是否执行成功
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1651200590099-5cbae831-8510-494e-a006-5f9e9ebe79f3.png#clientId=u0fbf61ea-aa42-4&from=paste&height=498&id=ufcaaca0f&name=image.png&originHeight=498&originWidth=1011&originalType=binary&ratio=1&rotation=0&showTitle=false&size=44043&status=done&style=none&taskId=u159f14cd-4841-49ef-9bbc-559a261922d&title=&width=1011)<br />至此我们已经完成了gorm的封装，以及编写好用户服务交互的代码
