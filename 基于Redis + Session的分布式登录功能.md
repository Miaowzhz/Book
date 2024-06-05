# 基于Redis + Session的 分布式 登录功能

> 六种登录方式：
>
> Cookie-Session 认证、Redis-Session 认证、Token认证、基于Token的JWT认证、 SSO单点登录、OAuth第三方登录。📜
>
> 这里讲解一下go语言版的Redis-Session。⏳
>
> 为什么用Redis-Session？🤔
>
> 1、简单易实现   👈😂
>
> 2、Redis具有极快的读写速度和高并发能力   🚀
>
> 3、Redis天生支持分布式部署和数据共享，有利于项目的扩展   📚

## 1、🛩安装依赖

```shell
go get github.com/go-redis/redis/v8
go get github.com/gin-contrib/sessions
go get github.com/gin-contrib/sessions/redis
```

## 2、🛩配置redis和session

**在router包下**

```go
// 配置Redis连接和Session存储
store, err := redis.NewStore(10, "tcp", "localhost:6379", "", []byte("secret"))
if err != nil {
    panic(err)
}
r.Use(sessions.Sessions("mysession", store))
```

## 3、🛩添加拦截器

### 1、添加中间件函数：

```go
// HomePage 处理主页面请求
func HomePage(c *gin.Context) {
	session := sessions.Default(c)
	userID := session.Get("user_id")

	if userID == nil {
		c.JSON(http.StatusUnauthorized, gin.H{"message": "Please login first"})
		return
	}

	c.HTML(http.StatusOK, "index.html", gin.H{})
}

// AuthMiddleware 是一个用于检查用户是否已登录的中间件
func AuthMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		session := sessions.Default(c)
		userID := session.Get("user_id")

		if userID == nil {
			c.Redirect(http.StatusFound, "/login")
			c.Abort()
			return
		}
		c.Next()
	}
}
```

### 2、注册路由：

```go
// 使用AuthMiddleware保护以下路由
protected := r.Group("/")
protected.Use(service.AuthMiddleware())
{
protected.GET("/", service.HomePage)
}
```

## 4、🛩升级登录逻辑



```go
// UserLogin 处理用户登录请求
func UserLogin(c *gin.Context) {
	username := c.PostForm("username")
	password := c.PostForm("password")

	// 查找用户
	var user domain.User
	if err := repository.DB.Where("username = ? AND password = ?", username, password).First(&user).Error; err != nil {
		c.JSON(http.StatusUnauthorized, gin.H{"message": "Invalid credentials"})
		return
	}
	// 登录成功，创建session
	session := sessions.Default(c)
	session.Set("user_id", user.ID)
	session.Save()
	// 登录成功，重定向到主页面
	c.Redirect(http.StatusFound, "/")
}
```

## 5、🛩升级注册逻辑

```go
// UserRegister 处理用户注册请求
func UserRegister(c *gin.Context) {
	username := c.PostForm("username")
	password := c.PostForm("password")
	// 检查用户名和密码是否为空
	if username == "" || password == "" {
		c.JSON(http.StatusBadRequest, gin.H{"message": "Username and password are required"})
		return
	}
	// 创建用户模型
	user := domain.User{
		Username: username,
		Password: password,
	}
	// 保存用户到数据库
	if err := repository.DB.Create(&user).Error; err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"message": "Registration failed", "error": err.Error()})
		return
	}
	// 注册成功后自动登录并创建session
	session := sessions.Default(c)
	session.Set("user_id", user.ID)
	session.Save()
	// 注册成功后重定向到登录页面
	c.Redirect(http.StatusFound, "/login")
}
```


