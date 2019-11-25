---
title: 【Golang】Golang + jwt 实现简易用户认证
date: 2019-11-23 10:25:55
tags:
- Golang
- Jwt
- Auth
categories:
- Golang
---
## 前言
在开发app的时候，难免会需要有后台API，Golang是一门非常适合开发后台服务的高性能的语言。在使用Golang开发后台API的时候，经常需要有用户注册、登录的功能，例如为了保存用户数据、为了给不同用户提供不同服务等。本文便是介绍一种基于jwt的Golang的用户认证系统。
## 目标
我们的模板是实现三个API接口：
* /api/account: 支持Post方法，用来注册一个用户
* /api/account/accesstoken: 登录功能
* /api/account/me: 获取用户信息，具有认证功能，如果没有用户认证信息（token），则获取失败，否则返回当前用户的信息。
## 工具原料
* Golang + Goland IDE（非必须，VSCode等等也可以）
* jwt-go：一个go语言的jwt实现
* github.com/gorilla/mux： go语言的一个路由组件，用来提供http路由服务
## 实现步骤
### 搭建http服务，提供API
1. 首先我们在自己的go语言目录下建立项目：golang-jwt-simple-auth（名字可自由决定），golang项目一般遵循golang的约定，放在GOPATH（默认是用户目录下的go文件夹）下面，比如我的项目放在github上，那目录就是
``` bash
~/go/src/github.com/glorinli/golang-jwt-simple-auth
```
2. 新建main.go问，作为程序的主入口，main.go内容如下:
``` go
package main

import (
	"fmt"
	"github.com/glorinli/go-jwt-simple-auth/app"
	"github.com/glorinli/go-jwt-simple-auth/controllers"
	"log"
	"net/http"
	"os"

	"github.com/gorilla/mux"
)

func init() {
	log.SetPrefix("simple-auth")
}

func main() {
	// 新建路由器
	router := mux.NewRouter()

	// 注册jwt认证的中间件
	router.Use(app.JwtAuthentication)

	// 注册路由
	router.HandleFunc("/api/account", controllers.CreateUser).Methods(http.MethodPost)
	router.HandleFunc("/api/account/accesstoken", controllers.Login).Methods(http.MethodGet)
	router.HandleFunc("/api/account/me", controllers.Me).Methods(http.MethodGet)

	// 获取端口号
	port := os.Getenv("golang-jwt-simple-auth-port")
	if port == "" {
		port = "8001"
	}

	fmt.Println("Port is:", port)

	// 开始服务
	err := http.ListenAndServe(":"+port, router)
	if err != nil {
		fmt.Print("Fail to start server", err)
	}
}

```
代码的作用在注释中已经说明了，关于mux库的使用，可以参考 http://www.gorillatoolkit.org/pkg/mux, 这里我们只要知道它起到一个路由器的作用，负责把一个api请求映射到一个方法上。

关键就在于
``` go
router.Use(app.JwtAuthentication)
```
这相当与注册了一个中间件，也可以理解为拦截器，就是说所有的请求都会先经过这个中间件拦截处理，于是我们便可以在里面处理认证相关 的逻辑了，接下来就来说说这个JwtAuthentication。
### JWT认证实现
首先还是贴上JwtAuthentication的代码：
``` go
package app

import (
	... 省略
)

var JwtAuthentication = func(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {

		// 只针对这个me接口开启认证
		needAuthPaths := []string{"/api/account/me"}
		requestPath := r.URL.Path

		var needAuth = false
		// 判断是否是需要认证的api，省略

        // 不需要认证，直接走下一步
		if !needAuth {
			next.ServeHTTP(w, r)
			return
		}

        // 从Header读取Token
		tokenHeader := r.Header.Get("Authorization")

		// Token is missing
		if tokenHeader == "" {
			sendInvalidTokenResponse(w, "Missing auth token")
			return
		}

		tk := &models.Token{}

		token, err := jwt.ParseWithClaims(tokenHeader, tk, func(token *jwt.Token) (interface{}, error) {
			return []byte(os.Getenv("token_password")), nil
		})

		fmt.Println("Parse token error:", err)

		if err != nil {
			sendInvalidTokenResponse(w, "Invalid auth token: "+err.Error())
			return
		}

		// Token is invalid
		if !token.Valid {
			sendInvalidTokenResponse(w, "Token is not valid")
			return
		}

		// Auth ok
		fmt.Println("User:", tk.UserId)
		ctx := context.WithValue(r.Context(), "user", tk.UserId)
		r = r.WithContext(ctx)
		next.ServeHTTP(w, r)
	})
}

func sendInvalidTokenResponse(w http.ResponseWriter, message string) {
	response := u.Message(false, message)
	w.WriteHeader(http.StatusForbidden)
	w.Header().Set("Content-Type", "application/json")
	u.Respond(w, response)
}

```
这个函数的作用就是解析客户端传递过来的token，将其解析为一个Token对象，Token对象的格式如下：
``` go
/*
JWT claims struct
*/
type Token struct {
	UserId uint
	jwt.StandardClaims
}
```
可以看到，除了jwt标准的数据，我们还添加了一个UserId字段，这是为了方便从Token确定用户的id。从这里我们也可以看出，Jwt Token是可以包含额外信息的。

关于这个Token是如何生程的，我们下面分析。
### 注册功能
返回去看main.go，我们发现注册接口被绑定到一个函数上：
``` go
router.HandleFunc("/api/account", controllers.CreateUser).Methods(http.MethodPost)
```
来看这个CreateUser函数，位于authController.go中：
``` go
var CreateUser = func(w http.ResponseWriter, r *http.Request) {
	account := &models.Account{}

	err := json.NewDecoder(r.Body).Decode(account)

	if err != nil {
		utils.Respond(w, utils.Message(false, "Invalid info: "+err.Error()))
		return
	}

	utils.Respond(w, account.Create())
}
```
最终的实现是在account.Create()函数，位于account.go中：
``` go
func (account *Account) Create() map[string]interface{} {
	// 校验 省略

    // 将密码做一个加密
	hashedPassword, _ := bcrypt.GenerateFromPassword([]byte(account.Password), bcrypt.DefaultCost)
	account.Password = string(hashedPassword)

    // 创建数据库数据
	err := GetDB().Create(account).Error

	// ...

	// 创建Token
	tk := &Token{UserId: account.ID}
	token := jwt.NewWithClaims(jwt.GetSigningMethod("HS256"), tk)
	tokenString, _ := token.SignedString([]byte(os.Getenv("token_password")))
	account.Token = tokenString

	account.Password = ""
	response := u.MessageWithData(true, "Account has been created", account)
	return response
}
```
关键就在于创建Token这一步，我们调用jwt.NewWithClaims来生成Token，参数有两个，第一个是签名方法，采用HS256，第二个就是一个Token对象，这个对象即将被编码到Token中，这也是我们在进行认证的时候，从Token解析出来的对象。
### 登录功能
登录功能与注册功能类似，只是把创建数据改为校验用户名密码。
## 运行部署
我们可以直接在Goland中运行程序，默认会运行在8081端口，然后便可以用Postman或者curl调试相应接口，这部分内容请读者自行研究。

注：笔者在mac os 10.15上，发现编译时需要添加-ldflags "-w"参数，否则会运行失败。
## 小结
本文介绍了如何使用Golang + jwt构建一个建议的认证系统，可以让大家对用户认证有一个基本的概念，详细的代码也已经同步到github上，读者们可以clone下来参考：