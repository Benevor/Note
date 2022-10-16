标题

后端接收来自前端的数据，后端向前端返回数据



对于post请求，后端接收参数时，可以用

```go
//注意两种c.JSON()的返回格式，都可以用

//方法1：用定义好的结构体，接收前端post请求的数据，结构体的定义需要与前端传送数据的格式一致
func AddUser(c *gin.Context) {
	var data model.User
	_ = c.ShouldBindJSON(&data)

	//......
	c.JSON(http.StatusOK, gin.H{
		"status":  code,
		"data":    data,
		"message": errmsg.GetErrMsg(code),
	})
}

    //结构体定义
    type User struct {
        //gorm.Model
        Username string `gorm:"type:varchar(20);not null" json:"username" label:"用户名"`
        Password string `gorm:"type:varchar(20);not null" json:"password" label:"密码"`
        Role     int    `gorm:"type:int;DEFAULT:2" json:"role" label:"角色码"`
    }

	//前端请求(json格式)
    POST{
        "username":"admin",
        "password":"111111",
        "role":1
    }



//方法2：用map[string]interface{}，或者map[string]string来接收前端post请求的数据
func FindFuncs(c *gin.Context) {
	json := make(map[string]string)
	c.BindJSON(&json)
	username := json["username"]

	re := model.FindAllFunc(username)
	code = 200
    //re是一个字符串数组
	c.JSON(http.StatusOK, map[string]interface{}{"status": code, "message": errmsg.GetErrMsg(code), "data": re})
}

//方法2延申：如果前端POST的参数有数组
	//前端请求
    POST {
        "tenantid":2,
        "allfuncids":[1,2,3]
    }
	//后端接收
    func UpdateFuncForTenant(c *gin.Context) {
        json := make(map[string]interface{})
        c.BindJSON(&json)

        //......
        code = model.UpdateFuncForTenant(json)
        c.JSON(http.StatusOK, gin.H{
            "status":  code,
            "message": errmsg.GetErrMsg(code),
        })

    }

    func UpdateFuncForTenant(json map[string]interface{}) int {
        tenantid := uint(json["tenantid"].(float64))
        allfuncids := json["allfuncids"].([]interface{})
        //allfuncids的类型为：[]interface{}

        //......
        return errmsg.SUCCESS
    }
```

