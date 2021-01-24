## Casbin 在gin 中的使用

### 介绍

Casbin是一个强大而高效的开源访问控制库。它支持基于各种访问控制模型强制执行授。目前支持多种数据语言，如 Go,php,java和python 等。了解更多内容，请查看 [官网](https://casbin.org/)

### 模型

PERM 模型， P:policy策略，p=(sub,obj,act, left)， 一般为存储到数据库的接口权限，R:request请求，r = sub, obj, act subject (sub 访问实体), object (obj 访问资源) 和  action(act 访问方法)。M：Matchers 匹配规则 Request 和 Policy 的匹配规则；E： Effect 影响 决定我们是否可以放行，e = some(where(p,eft == allow)) 这种情况下 我们的一个matchers匹配完成 得到了allow 那么这条请求放行。

### 简单使用

#### 项目目录
root/ \
|---config            配置文件 存放casbin modes文件 \
|---handler           路由handler 方法 \
|---middleware        中间件 \
|---pkg               类库 \
|---main.go 入口文件   入口文件

#### 初始化数据库连接

在 pkg文件夹下创建一个叫 instance.go 文件，初始化mysql 连接对象和cache对象。
如：
```
package pkg

import (
	"fmt"
	"github.com/allegro/bigcache"
	"gorm.io/driver/mysql"
	"gorm.io/gorm"
	"time"
)

var (
	DB *gorm.DB
	GlobalCache *bigcache.BigCache
)

func init()  {
	var err error
	DB, err = gorm.Open(mysql.Open("root:root@/saas-admin?charset=utf8&parseTime=True&loc=Local"), &gorm.Config{})
	if err != nil {
		panic(fmt.Sprintf("database connnect falied, err: %v", err))
	}
	GlobalCache, err = bigcache.NewBigCache(bigcache.DefaultConfig(30 * time.Minute))
	if err !=  nil {
		panic(fmt.Sprintf("cache initialize failed, err : %v", err))
	}
}
```

在本示例代码中，我们将policies 存储到数据库中，用户数据存储在cache中。

#### 配置
model 文件的配置

如果你想明白casbin 的使用，model 的配置，是你必须要明白的。在我们的示例中，我们基于 rbac 的model 配置文件配置，我们基于用户的角色控制用户的请求，叫做RBAC, 众所周知的 角色访问控制，我们在config目录中创建一个model.conf。
```
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act
```
这个配置文件的作用是告诉 casbin 如何去检查一个用户是否有权限，在上面的示例中，我们定义了一些东西：

- r = sub, obj, act 定义了一个请求包含三个部分：*sub*ject - 用户, *obj*ect - 请求资源（url 或者一般的资源） and *act*ion - 操作.
- p = sub, obj, act 定义了 一个策略格式. 例如, admin, data, write 意味着 admin 角色 有对 data 数据资源的写权限。
- g = _, _ 定义了用户角色的格式。 例如 Alice, admin 意味着 Alice 是一个admin 角色。
- m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act 定义了授权的工作流，检查用户角色->检查用户试图去访问的资源->检查用户的操作。

现在至少有四个部分在 model 配置文件中，request_definition, policy_definition, policy_effect and matchers.有时候我们不需要角色的控制，所以 role_definition 部分并不是必须的。

Policies 讲解
```
p, user, data, read
p, admin, data, read
p, admin, data, write
g, Alice, admin
g, Bob, user
```

1. 所有用户都有对data的读权限
2. 所有属于admin 角色用户可以 对data 读写操作
3. 所有属于user 角色用户只能对data 读操作

角色赋值：
- Alice 是 admin角色
- Bob 是 user 角色

Casbin 使用 gorm 数据库适配器时，将会检查表是否存在，如果不存在，将自动创建数据表 casbin_rule。 因此我们无需创建数据表。

#### 定义请求对handler 方法
首先，我们实现用户的登录方法：
```
// handler/user_handler.go

package handler

import (
	"casbin_usage/pkg"
	"fmt"
	"github.com/gin-gonic/gin"
	"github.com/google/uuid"
	"log"
)

func Login(c *gin.Context) {
	username, password := c.PostForm("username"), c.PostForm("password")
	u, err := uuid.NewRandom()
	if err != nil {
		log.Println(password)
		log.Fatal(err)
	}
	sessionId := fmt.Sprintf("%s-%s", u.String(), username)
	_ = pkg.GlobalCache.Set(sessionId, []byte(username))
	c.SetCookie("current_subject", sessionId, 30*60, "/resource", "", false, true)
	c.JSON(200, pkg.Response{
		Code:    0,
		Message: username + " login in successfully",
		Data:    nil,
	})
}
```

如果一个用户已经认证，我们需要存储用户信息到缓存中，事实上，我们也可以存储这些数据信息在session 中。

另外还需要提供些给用户访问操作的一写handler 方法
```
// handler/user_handler.go

func ReadResource(c *gin.Context) {
	c.JSON(200, pkg.Response{
		Code:    0,
		Message: "read resource successfully",
		Data:    "resource",
	})
}

func WriteResource(c *gin.Context) {
	c.JSON(200, pkg.Response{Code: 1, Message: "write resource successfully", Data: "resource"})
}
```

之后，我们应该注册这些方法到我们的main文件中：
```
// main.go

package main

import (
	"casbin_usage/handler"
	"fmt"
	"github.com/gin-contrib/cors"
	"github.com/gin-gonic/gin"
	"log"
)

var router *gin.Engine

func init()  {

	router = gin.Default()
	corsConfig := cors.DefaultConfig()
	corsConfig.AllowAllOrigins = true
	corsConfig.AllowCredentials = true
	router.Use(cors.New(corsConfig))
	router.POST("/user/login",handler.Login)
	router.GET("/resource", handler.ReadResource)
	router.POST("/resource", handler.WriteResource)
}

func main()  {
	err := router.Run(":8081")
	if err != nil {
		panic(fmt.Sprintf("failed to start gin engin: %v", err))
	}
	log.Println("application is now running...")
}
```

现在大部分工作已经完成了，接下来也是最重要的工作是通过RBAC来实现我们的安全认证api接口。

#### 强制使用 Casbin 策略
**从数据库中加载策略**
第一个问题，我们如何从数据库中加载策略呢？这里就使用到了 [Casbin Adapters](https://casbin.org/docs/en/adapters),更准确的说，我们使用 [Gorm Adapter](https://github.com/casbin/gorm-adapter)

第一步 我们先通过已经存在的GORM实例对象初始化一个适配器。
```
// main.go

func init()  {
	adapter, err := gormadapter.NewAdapterByDB(pkg.DB)
	if err != nil {
		panic(fmt.Sprintf("failed to initialize casbin adapter : %v", adapter))
	}

	router = gin.Default()
	corsConfig := cors.DefaultConfig()
	corsConfig.AllowAllOrigins = true
	corsConfig.AllowCredentials = true
	router.Use(cors.New(corsConfig))
	router.POST("/user/login",handler.Login)
	router.GET("/resource", handler.ReadResource)
	router.POST("/resource", handler.WriteResource)
}
```

很显然，我们应该在handler 方法调用前使策略去控制访问的资源，这时候，一个优雅的方式就是使用 gin 的middleware 和 路由组去实现。首先，让我们定义一个 middleware 指明我们将使用哪种策略方法。
示例如下：
```
// middleware/middleware.go

package middleware

import (
	"casbin_usage/pkg"
	"errors"
	"fmt"
	"github.com/allegro/bigcache"
	"github.com/casbin/casbin/v2"
	gormadapter "github.com/casbin/gorm-adapter/v3"
	"github.com/gin-gonic/gin"
	"log"
	"os"
)

func Authenticate() gin.HandlerFunc {
	return func(c *gin.Context) {
		// Get session id
		sessionId, _ := c.Cookie("current_subject")
		// Get current subject
		sub, err := pkg.GlobalCache.Get(sessionId)
		if errors.Is(err, bigcache.ErrEntryNotFound) {
			c.AbortWithStatusJSON(401, pkg.Response{Message: "user hasn't logged in yet"})
			return
		}
		c.Set("current_subject", string(sub))
		c.Next()
	}
}

func Authorize(obj string, act string, adapter *gormadapter.Adapter) gin.HandlerFunc {
	return func(c *gin.Context) {
		val, existed := c.Get("current_subject")
		if !existed {
			c.AbortWithStatusJSON(401, pkg.Response{Message: "user hasn't logged in yet"})
			return
		}
		ok, err := enforce(val.(string), obj, act, adapter)
		if err != nil {
			log.Println(err)
			c.AbortWithStatusJSON(500, pkg.Response{Message: "error occurred when authorizing user"})
			return
		}
		if !ok {
			c.AbortWithStatusJSON(403, pkg.Response{Message: "forbidden"})
			return
		}
		c.Next()
	}
}

func enforce(sub, obj, act string, adapter *gormadapter.Adapter) (bool, error) {
	enforcer, err := casbin.NewEnforcer("casbin/config/rbac_model.conf", adapter)
	if err != nil {
		fmt.Println(os.Getwd())
		return false, fmt.Errorf("failed to create to casbin enforcer: %w", err)
	}
	err = enforcer.LoadPolicy()
	if err != nil {
		return false, fmt.Errorf("failed to load policy from DB: %w", err)
	}
	return enforcer.Enforce(sub, obj, act)
}

```

最后路由组需要去认证和使用我们的路由：
```
// main.go

func init()  {
	adapter, err := gormadapter.NewAdapterByDB(pkg.DB)
	if err != nil {adapter, err := gormadapter.NewAdapterByDB(pkg.DB)
		if err != nil {
			panic(fmt.Sprintf("failed to initialize casbin adapter : %v", adapter))
		}
		panic(fmt.Sprintf("failed to initialize casbin adapter : %v", adapter))
	}
	router = gin.Default()
	corsConfig := cors.DefaultConfig()
	corsConfig.AllowAllOrigins = true
	corsConfig.AllowCredentials = true
	router.Use(cors.New(corsConfig))
	router.POST("/user/login",handler.Login)
	resource := router.Group("/api")
	resource.Use(middleware.Authenticate())
	{
		resource.GET("/resource",middleware.Authorize("resource", "read", adapter),handler.ReadResource)
		resource.POST("/Resource",middleware.Authorize("resource", "write", adapter),handler.WriteResource)
	}
}
```

#### 将数据表插入测试数据：
```
INSERT INTO `casbin_rule`(`id`, `p_type`, `v0`, `v1`, `v2`, `v3`, `v4`, `v5`) VALUES (1, 'p', 'user', 'resource', 'read', NULL, NULL, NULL);
INSERT INTO `casbin_rule`(`id`, `p_type`, `v0`, `v1`, `v2`, `v3`, `v4`, `v5`) VALUES (2, 'g', 'bob', 'user', NULL, NULL, NULL, NULL);
```

最后，所有的设置完成之后，如果用户没有登录，当访问GET /api/resource 或者 POST /api/resource 将会被拒绝，将他想去写数据的时候，也会被拒绝。

test method:
1. POST http://localhost:8081/user/login 登录
2. POST http://localhost:8081/api/resource 请求带上cookie
3. GET http://localhost:8081/api/resource 请求带上cookie

此时 使用 bob登录，对2亲求通过，3请求拒绝。

>- [参考文章](https://dev.to/maxwellhertz/tutorial-integrate-gin-with-cabsin-56m0) https://dev.to/maxwellhertz/tutorial-integrate-gin-with-cabsin-56m0
