--- 
 title: "go-micro开发运维实践(基于jwt实现登录验证接口)" 
 date: 2023-02-07T15:52:40+08:00 
 draft: false 
 description: "go-micro开发运维实践(基于jwt实现登录验证接口)" 
 tags: ["go-micro开发运维实践"] 
 categories: ["go-micro开发运维实践"] 
---
## 定义登录验证protobuf

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
  rpc Auth(AuthRequest) returns(TokenResponse){}
  rpc ValidateToken(TokenRequest) returns(TokenResponse){}
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

//AuthRequest 登录请求
message AuthRequest{
  string email = 1;
  string password = 2;
}

//TokenRequest token验证接口
message TokenRequest{
  string token = 1;
}

//TokenResponse token响应接口
message TokenResponse{
  string token = 1;
  bool valid = 2;
}
```

### 生成protobuf代码
```
make proto
```

## 引用jwt编写获取token业务

### 获取jwt生成包
```
go get -u github.com/dgrijalva/jwt-go
```

### 创建service目录
```
mkdir service
touch service/token.go
```
```
package service

import (
	"github.com/869413421/micro-service/user/pkg/model"
	"github.com/869413421/micro-service/user/pkg/repo"
	"github.com/dgrijalva/jwt-go"
	"time"
)

var (
	key = []byte("microServiceUserTokenKeySecret")
)

// CustomClaims jwt认证对象
type CustomClaims struct {
	User *model.User
	jwt.StandardClaims
}

// Authble jwt实现接口
type Authble interface {
	Decode(token string) (*CustomClaims, error)
	Encode(user *model.User) (string, error)
}

// TokenService token业务对象
type TokenService struct {
	Repo repo.UserRepositoryInterface
}

// NewTokenService token业务初始化
func NewTokenService() Authble {
	return &TokenService{Repo: repo.NewUserRepository()}
}

// Decode 将token字符串转换为token对象
func (srv *TokenService) Decode(tokenString string) (*CustomClaims, error) {

	// Parse the token
	token, err := jwt.ParseWithClaims(tokenString, &CustomClaims{}, func(token *jwt.Token) (interface{}, error) {
		return key, nil
	})

	// Validate the token and return the custom claims
	if claims, ok := token.Claims.(*CustomClaims); ok && token.Valid {
		return claims, nil
	} else {
		return nil, err
	}
}

// Encode 将token对象串转换为token字符串
func (srv *TokenService) Encode(user *model.User) (string, error) {

	expireToken := time.Now().Add(time.Hour * 72).Unix()

	// Create the Claims
	claims := CustomClaims{
		user,
		jwt.StandardClaims{
			ExpiresAt: expireToken,
			Issuer:    "micro.service.user",
		},
	}

	// Create token
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)

	// Sign token and return
	return token.SignedString(key)
}
```

## 实现服务处理器

### 为UserServiceHandler增加token业务层
```
...

//UserServiceHandler 用户服务处理器
type UserServiceHandler struct {
	UserRepo     repo.UserRepositoryInterface
	TokenService service.Authble
}

// NewUserServiceHandler 用户服务初始化
func NewUserServiceHandler() *UserServiceHandler {
	return &UserServiceHandler{
		UserRepo:     repo.NewUserRepository(),
		TokenService: service.NewTokenService(),
	}
}

...
```

### 实现登录验证token接口
为`UserServiceHandler`加上实现方法
```
...

// Auth 认证获取token
func (srv UserServiceHandler) Auth(ctx context.Context, req *pb.AuthRequest, rsp *pb.TokenResponse) error {
	// 1.根据邮箱回去用户
	log.Println("Logging in with:", req.Email, req.Password)
	user, err := srv.UserRepo.GetByEmail(req.Email)
	if err != nil && err != gorm.ErrRecordNotFound {
		return err
	}
	if err == gorm.ErrRecordNotFound {
		return errors.NotFound("User.Auth.GetByEmail.Error", "user not found ,check you password or email")
	}

	// 2.检验用户密码
	err = bcrypt.CompareHashAndPassword([]byte(user.Password), []byte(req.Password))
	if err != nil {
		return errors.Unauthorized("User.Auth.CheckPassword.Error", err.Error())
	}

	// 3.生成token
	token, err := srv.TokenService.Encode(user)
	if err != nil {
		return err
	}

	// 4.返回token字符串
	rsp.Token = token
	return nil
}

// ValidateToken 验证token
func (srv *UserServiceHandler) ValidateToken(ctx context.Context, req *pb.TokenRequest, rsp *pb.TokenResponse) error {
	// 1.将token字符串转换为token对象
	claims, err := srv.TokenService.Decode(req.Token)
	if err != nil {
		return err
	}

	// 2.判断转换是否成功
	if claims.User.ID == 0 {
		return errors.BadRequest("User.ValidateToken.Error", "user invalid")
	}

	// 3.返回验证状态
	rsp.Token = req.Token
	rsp.Valid = true
	return nil
}

...
```

### 编译代码准备测试
```
make build
```

### 打开micro工具测试接口
[http://127.0.0.1:8082/client](http://127.0.0.1:8082/client)
