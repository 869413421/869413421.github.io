--- 
 title: "go-micro开发运维实践(实现用户CURD服务)" 
 date: 2023-02-07T15:52:40+08:00 
 draft: false 
 description: "go-micro开发运维实践(实现用户CURD服务)" 
 tags: ["go-micro开发运维实践"] 
 categories: ["go-micro开发运维实践"] 
---
## 定义protobuf,生成代码

### 修改proto/user/user.proto
```
syntax = "proto3";

package micro.service.user;
option go_package = "proto/user";

service UserService {
  rpc Pagination(PaginationRequest) returns(PaginationResponse){}
  rpc Get(GetRequest) returns(UserResponse){}
  rpc Create(CreateRequest) returns(UserResponse){}
  rpc Update(UpdateRequest) returns(UserResponse){}
  rpc Delete(DeleteRequest) returns(UserResponse){}
}

message User{
  uint64 id = 1;
  string name = 3;
  string email = 4;
  string real_name = 6;
  string avatar = 7;
  string create_at = 9;
  string update_at = 10;
}

//UserResponse 单个用户响应
message UserResponse{
  User user = 1;
}

//PaginationResponse 用户分页数据响应
message PaginationResponse{
  repeated User users = 1;
  uint64 total = 2;
}

//PaginationRequest 用户分页请求
message PaginationRequest{
  uint64 page = 1;
  uint32 perPage = 2;
}

//GetRequest 获取单个用户请求
message GetRequest{
  uint64 id = 1;
}

//CreateRequest 创建用户请求
message CreateRequest{
  string name = 1;
  string password = 2;
  string email = 3;
  string real_name = 4;
  string avatar = 5;
}

//UpdateRequest 更新用户请求
message UpdateRequest{
  uint64 id = 1;
  string name = 2;
  string email = 3;
  string real_name = 4;
  string avatar = 6;
}

//DeleteRequest 删除用户请求
message DeleteRequest{
  uint64 id = 1;
}
```

### 执行生成命令
make命令能帮我们执行在`makefile`中预定义好的命令，在开发当中能给我们带来便利。
```
make proto
```
没有make命令可以直接复制`makefile`中的proto命令执行
```
protoc --proto_path=. --micro_out=${MODIFY}:. --go_out=${MODIFY}:. proto/user/user.proto
```

## 封装用户数据库交互层

### 封装分页工具
我们日常开发中在页面上经常需要获取一些分页数据，在多个微服务中如果每个都要实现分页代码代码必定会造成大量的冗余，所以我们这里需要对分页代码进行一些封装。<br />打开我们项目的`common`项目

#### 创建分页工具包目录
```
mkdir -p pkg/pagination
touch pkg/pagination/pagination.go
```

#### 编写分页工具代码
```
package pagination

import (
	"gorm.io/gorm"
	"math"
)

// Page 单个分页元素
type Page struct {
	// 链接
	URL string
	// 页码
	Number uint64
}

// ViewData 同视图渲染的数据
type ViewData struct {
	// 是否需要显示分页
	HasPages bool

	// 下一页
	Next    Page
	HasNext bool

	// 上一页
	Prev    Page
	HasPrev bool

	Current Page

	// 数据库的内容总数量
	TotalCount uint64
	// 总页数
	TotalPage uint64
}

// Pagination 分页对象
type Pagination struct {
	PerPage uint32
	Page    uint64
	Count   uint64
	DB      *gorm.DB
}

// New 分页对象构建器
// db —— GORM 查询句柄，用以查询数据集和获取数据总数
// page —— page
// perPage —— 每页条数，传参为小于或者等于 0 时为默认值  10
func New(db *gorm.DB, page uint64, perPage uint32) *Pagination {
	// 默认每页数量
	if perPage <= 0 {
		perPage = 10
	}

	// 实例对象
	p := &Pagination{
		DB:      db,
		PerPage: perPage,
		Page:    page,
		Count:   0,
	}

	// 设置当前页码
	p.SetPage(page)

	return p
}

// Paging 返回渲染分页所需的数据
func (p *Pagination) Paging() ViewData {

	return ViewData{
		HasPages: p.HasPages(),

		Next:    p.NewPage(p.NextPage()),
		HasNext: p.HasNext(),

		Prev:    p.NewPage(p.PrevPage()),
		HasPrev: p.HasPrev(),

		Current:   p.NewPage(p.CurrentPage()),
		TotalPage: p.TotalPage(),

		TotalCount: p.Count,
	}
}

// NewPage 设置当前页
func (p Pagination) NewPage(page uint64) Page {
	return Page{
		Number: page,
	}
}

// SetPage 设置当前页
func (p *Pagination) SetPage(page uint64) {
	if page <= 0 {
		page = 1
	}

	p.Page = page
}

// CurrentPage 返回当前页码
func (p Pagination) CurrentPage() uint64 {
	totalPage := p.TotalPage()
	if totalPage == 0 {
		return 0
	}

	if p.Page > totalPage {
		return totalPage
	}

	return p.Page
}

// Results 返回请求数据，请注意 data 参数必须为 GROM 模型的 Slice 对象
func (p Pagination) Results(data interface{}) error {
	var err error
	var offset uint64
	page := p.CurrentPage()
	if page == 0 {
		return err
	}

	if page > 1 {
		offset = (page - 1) * uint64(p.PerPage)
	}

	return p.DB.Debug().Limit(int(p.PerPage)).Offset(int(offset)).Find(data).Error
}

// TotalCount 返回的是数据库里的条数
func (p *Pagination) TotalCount() uint64 {
	if p.Count == 0 {
		var count int64
		if err := p.DB.Count(&count).Error; err != nil {
			return 0
		}
		p.Count = uint64(count)
	}

	return p.Count
}

// HasPages 总页数大于 1 时会返回 true
func (p *Pagination) HasPages() bool {
	n := p.TotalCount()
	return n > uint64(p.PerPage)
}

// HasNext returns true if current page is not the last page
func (p Pagination) HasNext() bool {
	totalPage := p.TotalPage()
	if totalPage == 0 {
		return false
	}

	page := p.CurrentPage()
	if page == 0 {
		return false
	}

	return page < totalPage
}

// PrevPage 前一页码，0 意味着这就是第一页
func (p Pagination) PrevPage() uint64 {
	hasPrev := p.HasPrev()

	if !hasPrev {
		return 0
	}

	page := p.CurrentPage()
	if page == 0 {
		return 0
	}

	return page - 1
}

// NextPage 下一页码，0 的话就是最后一页
func (p Pagination) NextPage() uint64 {
	hasNext := p.HasNext()
	if !hasNext {
		return 0
	}

	page := p.CurrentPage()
	if page == 0 {
		return 0
	}

	return page + 1
}

// HasPrev 如果当前页不为第一页，就返回 true
func (p Pagination) HasPrev() bool {
	page := p.CurrentPage()
	if page == 0 {
		return false
	}

	return page > 1
}

// TotalPage 返回总页数
func (p Pagination) TotalPage() uint64 {
	count := p.TotalCount()
	if count == 0 {
		return 0
	}

	nums := int64(math.Ceil(float64(count) / float64(p.PerPage)))
	if nums == 0 {
		nums = 1
	}

	return uint64(nums)
}
```

### 编写用户仓库代码

#### 创建用户仓库代码目录
```
mkdir -p pkg/repo
touch pkg/repo/user.go
```

#### 编写仓库代码
```
package repo

import (
	baseDb "github.com/869413421/micro-service/common/pkg/db"
	"github.com/869413421/micro-service/common/pkg/pagination"
	"github.com/869413421/micro-service/user/pkg/model"
	"gorm.io/gorm"
)

// UserRepositoryInterface 用户CURD仓库接口
type UserRepositoryInterface interface {
	GetFirst(where map[string]interface{}) (*model.User, error)
	GetByID(uint642 uint64) (*model.User, error)
	GetByEmail(email string) (*model.User, error)
	Pagination(page uint64, perPage uint32) (users []model.User, viewData pagination.ViewData, err error)
}

// UserRepository 用户仓库
type UserRepository struct {
	Db *gorm.DB
}

// NewUserRepository 初始化仓库
func NewUserRepository() UserRepositoryInterface {
	db := baseDb.GetDB()
	return &UserRepository{Db: db}
}

// GetByID 根据ID获取用户
func (repo UserRepository) GetByID(id uint64) (*model.User, error) {
	user := &model.User{}
	err := repo.Db.First(&user, id).Error
	return user, err
}

// Pagination 获取分页数据
func (repo UserRepository) Pagination(page uint64, perPage uint32) (users []model.User, viewData pagination.ViewData, err error) {
	//1.初始化分页实例
	DB := repo.Db.Model(model.User{}).Order("created_at desc")
	_pager := pagination.New(DB, page, perPage)

	// 2. 获取分页构建数据
	viewData = _pager.Paging()

	// 3. 获取数据
	_pager.Results(&users)

	return users, viewData, nil
}

// GetByEmail 根据email获取用户
func (repo UserRepository) GetByEmail(email string) (*model.User, error) {
	user := &model.User{}
	err := repo.Db.Where("email = ?", email).First(&user).Error
	return user, err
}

// GetFirst 根据自定义条件获取用户
func (repo UserRepository) GetFirst(where map[string]interface{}) (*model.User, error) {
	user := &model.User{}
	for key, val := range where {
		repo.Db.Where(key+"=?", val)
	}
	err := repo.Db.First(&user).Error
	return user, err
}
```
根据`依赖倒置原则`，我们定义了一个用户抽象的接口，然后编写了接口的实现细节。这种方式能使我们上层模块（即调用用户仓库的类），不再依赖下层（即实现的代码`UserRepository`）。当后续我们的业务改动，只需要重新实现`UserRepositoryInterface`就可以直接对实现细节进行替换，在开发中我们应该遵循`抽象不应该依赖细节，细节应该依赖抽象`的方式来实现功能。

#### 修改model/user.go
```
package model

import (
	db "github.com/869413421/micro-service/common/pkg/db"
	pb "github.com/869413421/micro-service/user/proto/user"
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

// ToORM protobuf转换为orm
func ToORM(protoUser *pb.User) *User {
	user := &User{}
	user.ID = protoUser.Id
	user.Email = protoUser.Email
	user.Name = protoUser.Name
	user.Avatar = protoUser.Avatar
	user.RealName = protoUser.RealName
	return user
}

// ToProtobuf orm转换为protobuf
func (model *User) ToProtobuf() *pb.User {
	user := &pb.User{}
	user.Id = model.ID
	user.Email = model.Email
	user.Name = model.Name
	user.Avatar = model.Avatar
	user.CreateAt = model.CreatedAtDate()
	user.UpdateAt = model.UpdatedAtDate()
	user.RealName = model.RealName
	return user
}

// Store 创建用户
func (model *User) Store() (err error) {
	result := db.GetDB().Create(&model)
	err = result.Error
	if err != nil {
		return err
	}
	return nil
}

// Update 更新用户
func (model *User) Update() (rowsAffected int64, err error) {
	result := db.GetDB().Save(&model)
	err = result.Error
	if err != nil {
		return 0, err
	}
	rowsAffected = result.RowsAffected
	return
}

// Delete 删除用户
func (model User) Delete() (rowsAffected int64, err error) {
	result := db.GetDB().Delete(&model)
	err = result.Error
	if err != nil {
		return
	}
	rowsAffected = result.RowsAffected
	return
}
```

#### 添加模型事件，加密用户密码
在储存用户到数据库时，我们的密码不应该以明文的方式进行存储，我们这里利用gorm提供的模型事件，在用户信息进入数据库之前，对密码进行一次加密再存储。

打开`common`项目，封装一个加密工具包，把加密相关的工具方法放到这个目录下
```
mkdir -p pkg/password
touch pkg/password/password.go
```
```
package password

import (
	"crypto/md5"
	"encoding/hex"
	"golang.org/x/crypto/bcrypt"
)

// Hash hash加密
func Hash(password string) (string, error) {
	bytes, err := bcrypt.GenerateFromPassword([]byte(password), 14)
	if err != nil {
		return "", err
	}

	return string(bytes), nil
}

//CheckHash 检查密码是否与hash值匹配
func CheckHash(password string, hash string) bool {
	err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
	return err == nil
}

// IsHashed 检查是否已经加密过
func IsHashed(str string) bool {
	return len(str) == 60
}

// Md5Str 获取一个md5加密字符串
func Md5Str(str string) string {
	h := md5.New()
	h.Write([]byte(str))
	return hex.EncodeToString(h.Sum(nil))
}
```
执行`go mod tidy`下载加密相关包<br />返回`user`项目，编写模型事件代码
```
touch pkg/model/user_hooks.go
```
```
package model

import (
	"github.com/869413421/micro-service/common/pkg/password"
	"gorm.io/gorm"
)

// BeforeSave 保存前模型事件
func (model *User) BeforeSave(tx *gorm.DB) (err error) {
	//1.如果密码没加密，进行一次加密
	if !password.IsHashed(model.Password) {
		model.Password, err = password.Hash(model.Password)
		if err!=nil{
			return err
		}
	}
	return nil
}
```

## 实现服务处理
前面我们对rpc接口进行了定义并且生成了相对应的通讯代码。我们只是整个服务经行了声明，但并没有对服务进行实现。<br />打开`user`项目

### 修改handler/user.go
```
package handler

import (
	"context"
	"github.com/869413421/micro-service/common/pkg/types"
	"github.com/869413421/micro-service/user/pkg/model"
	"github.com/869413421/micro-service/user/pkg/repo"
	pb "github.com/869413421/micro-service/user/proto/user"
	"github.com/micro/go-micro/v2/errors"
	"gorm.io/gorm"
)

//UserServiceHandler 用户服务处理器
type UserServiceHandler struct {
	UserRepo repo.UserRepositoryInterface
}

// NewUserServiceHandler 用户服务初始化
func NewUserServiceHandler() *UserServiceHandler {
	return &UserServiceHandler{
		UserRepo: repo.NewUserRepository(),
	}
}

// Pagination 分页
func (srv *UserServiceHandler) Pagination(ctx context.Context, req *pb.PaginationRequest, rsp *pb.PaginationResponse) error {
	// 1.查找分页数据
	users, pagerData, err := srv.UserRepo.Pagination(req.Page, req.PerPage)
	if err != nil {
		return errors.InternalServerError("user.Pagination.Pagination.Error", err.Error())
	}

	// 2.构造用户列表
	userItems := make([]*pb.User, len(users))
	for index, user := range users {
		userItem := user.ToProtobuf()
		userItems[index] = userItem
	}

	// 3.返回用户信息
	rsp.Users = userItems
	rsp.Total = pagerData.TotalCount
	return nil
}

// Get 根据ID获取数据
func (srv *UserServiceHandler) Get(ctx context.Context, req *pb.GetRequest, rsp *pb.UserResponse) error {
	// 1.查找用户
	user, err := srv.UserRepo.GetByID(req.GetId())
	if err != nil && err != gorm.ErrRecordNotFound {
		return err
	}
	if err == gorm.ErrRecordNotFound {
		return errors.BadRequest("User.GetByID", "user not found")
	}

	// 2.返回用户信息
	rsp.User = user.ToProtobuf()
	return nil
}

// Create 创建用户
func (srv *UserServiceHandler) Create(ctx context.Context, req *pb.CreateRequest, rsp *pb.UserResponse) error {
	// 1.填充提交信息
	user := &model.User{}
	types.Fill(user, req)

	// 2.创建用户
	err := user.Store()
	if err != nil {
		return err
	}

	// 3.返回用户信息
	rsp.User = user.ToProtobuf()
	return nil
}

// Update 更新用户信息
func (srv *UserServiceHandler) Update(ctx context.Context, req *pb.UpdateRequest, rsp *pb.UserResponse) error {
	// 1.获取用户
	id := req.Id
	_user, err := srv.UserRepo.GetByID(id)
	if err != nil && err != gorm.ErrRecordNotFound {
		return err
	}
	if err == gorm.ErrRecordNotFound {
		return errors.NotFound("User.Update.GetUserByID.Error", "user not found ,check you request id")
	}

	// 2.验证提交信息
	types.Fill(_user, req)

	// 3.更新用户
	rowsAffected, err := _user.Update()
	if rowsAffected == 0 || err != nil {
		return errors.InternalServerError("User.Update.Update.Error", err.Error())
	}

	// 4.返回更新信息
	rsp.User = _user.ToProtobuf()
	return nil
}

// Delete 删除用户
func (srv *UserServiceHandler) Delete(ctx context.Context, req *pb.DeleteRequest, rsp *pb.UserResponse) error {
	// 1.获取用户
	id := req.Id
	_user, err := srv.UserRepo.GetByID(id)
	if err != nil && err != gorm.ErrRecordNotFound {
		return err
	}
	if err == gorm.ErrRecordNotFound {
		return errors.NotFound("User.Delete.GetUserByID.Error", "user not found ,check you request id")
	}

	// 2.删除用户
	rowsAffected, err := _user.Delete()
	if err != nil {
		return errors.InternalServerError("User.Delete.Delete.Error", err.Error())
	}
	if rowsAffected == 0 {
		return errors.BadRequest("User.Delete.Delete.Fail", "update fail")
	}

	// 3.返回更新信息
	rsp.User = _user.ToProtobuf()
	return nil
}

```

### 修改main.go
```
package main

import (
	"github.com/869413421/micro-service/common/pkg/db"
	"github.com/869413421/micro-service/user/handler"
	"github.com/869413421/micro-service/user/pkg/model"
	"github.com/micro/go-micro/v2"
	log "github.com/micro/go-micro/v2/logger"

	proto "github.com/869413421/micro-service/user/proto/user"
)

func main() {

	// 1.准备数据库连接，并且执行数据库迁移
	db := db.GetDB()
	db.AutoMigrate(&model.User{})

	// 2.创建服务
	service := micro.NewService(
		micro.Name("micro.service.user"),
		micro.Version("v1"),
	)

	// 3.初始化服务
	service.Init()

	// 4.注册服务处理器
	proto.RegisterUserServiceHandler(service.Server(),handler.NewUserServiceHandler())

	// 5.运行服务
	if err := service.Run(); err != nil {
		log.Fatal(err)
	}
}
```

#### 编译服务
```
go mod download
make build
```

## 测试用户服务是否正常运行

### 重启服务
```
docker-compose restart micro-user-service
```

#### 检查注册的服务方法
打开[http://127.0.0.1:8082/service/micro.service.user](http://127.0.0.1:8082/service/micro.service.user)<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1651574619019-51b73b37-9931-4cf9-9249-343d7a6e67a8.png#clientId=ude921de3-de0e-4&from=paste&height=845&id=u1ad0f958&name=image.png&originHeight=929&originWidth=1920&originalType=binary&ratio=1&rotation=0&showTitle=false&size=51721&status=done&style=none&taskId=ue3cee349-96b4-496b-b26e-59f77d4a2eb&title=&width=1745.4545076228378)<br />可以看到实现的rpc接口已经注册

#### 测试接口
点击client,选择相关服务以及方法，输入请求参数，对接口进行测试<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1651575858750-967ab8ae-5114-4fb5-aa71-70085533a47f.png#clientId=ude921de3-de0e-4&from=paste&height=508&id=uee9cddce&name=image.png&originHeight=559&originWidth=1146&originalType=binary&ratio=1&rotation=0&showTitle=false&size=51629&status=done&style=none&taskId=u273c3a92-1b0e-4886-8291-7ae9d2db6d3&title=&width=1041.8181592373815)<br />可以看到返回了正常信息，顺便检查数据库用户密码是否加密<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1651575952490-ef234120-4b3b-45e4-907e-4291355a573a.png#clientId=ude921de3-de0e-4&from=paste&height=61&id=u74bf1f21&name=image.png&originHeight=67&originWidth=1133&originalType=binary&ratio=1&rotation=0&showTitle=false&size=8097&status=done&style=none&taskId=u69d512a8-8d44-49ec-88c3-0515930174b&title=&width=1029.9999776753516)<br />依次对其他接口进行测试，保证编写的代码正常运行,提交代码到github

