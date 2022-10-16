### 记录vue中使用http发送请求的各种方式和问题

##### 问题

- 前后端解决跨域的原理
- 有几种规范的http请求方式
- async、await表示什么含义
- 前端接收到数据后如何修改页面（如果组件绑定了this里的数据，该数据改变，组件会是否自动变化）

-------------

##### 后端接口

- http://localhost:3000/api/directory/verify
- http://localhost:3000/api/directory/backup
- 两个接口都是将前端post的数据返回给前端
- 假定后端以json格式的数据响应前端（即前端每次http请求拿到一个json数据）

##### 后端跨域处理

- **main.go**

```go
package main

import (
	"ginVue/router"
    //项目的module为ginVue
)

func main() {
	router.InitRouter()
}
```

- **router.go**

```go
package router

import (
	v1 "ginVue/api"
	"ginVue/middleware"
	"ginVue/utils"

	"github.com/gin-gonic/gin"
)

func InitRouter() {
	gin.SetMode(utils.AppMode)
	r := gin.New()
	r.Use(gin.Recovery())
	r.Use(middleware.Cors())

	auth := r.Group("api")
	{
		auth.POST("directory/backup", v1.HandleDirectoryBackUp) //目录备份
		auth.POST("directory/verify", v1.HandleDirectoryVerify) //验证
	}

	r.Run(utils.HttpPort)
}

```

- **cors.go**
```go
package middleware

import (
	"time"
	"github.com/gin-contrib/cors"
	"github.com/gin-gonic/gin"
)

func Cors() gin.HandlerFunc {
	return cors.New(
		cors.Config{
			//AllowAllOrigins:  true,
			AllowOrigins:     []string{"*"}, // 等同于允许所有域名 #AllowAllOrigins:  true
			AllowMethods:     []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
			AllowHeaders:     []string{"*"},
			ExposeHeaders:    []string{"Content-Length", "Authorization", "Content-Type"},
			AllowCredentials: true,
			MaxAge:           12 * time.Hour,
		},
	)
}
```

------------------



##### 第一种方式（不使用vue.config.js，但是后端做了跨域处理，而且必须做）

- 前端 - main.js

```js
import Vue from 'vue'
import App from './App.vue'
import router from './router'
import axios from 'axios'

axios.defaults.baseURL = 'http://localhost:3000/api'
Vue.prototype.$http = axios

Vue.config.productionTip = false
new Vue({
router,
render: h => h(App)
}).$mount('#app')
```

- 前端 - http函数

```js
async backUp () {
    // this.form为前端需要向后端post的数据
    const { data: res } = await this.$http.post('directory/backup', this.form)
    // 仅仅在浏览器控制台打印后端返回的数据，没有处理
    console.log(res)
}
```

-----------------------------------------



##### 第二种方式（使用vue.config.js，且后端做了跨域处理，不是必须的）

（此方式，后端可不处理跨域；即注释掉 router.go 中的 r.Use(middleware.Cors()) ）

- 前端 - man.js

```js
import Vue from 'vue'
import App from './App.vue'
import router from './router'
import axios from 'axios'

// axios.defaults.baseURL = 'http://localhost:3000/api'

Vue.prototype.$http = axios

Vue.config.productionTip = false
new Vue({
  router,
  render: h => h(App)
}).$mount('#app')
```

- 前端 - vue.config.js

```js
module.exports = {
  devServer: {
    proxy: {
      '/api': {
        target: 'http://localhost:3000/api',
        changeOrigin: true,
        pathRewrite: {
          '^/api': ''
        }
      }
    }
  }
}
```

- 前端 - http函数

```js
async backUp () {
    //注意：这里的'api/directory/backup'中的api是vue.config.js重写所对应的，与后端接口无关
    //实际'api/directory/backup'被等价为'http://localhost:3000/api/directory/backup'
    //即'api'等价为'http://localhost:3000/api'，前后两个api一样，纯属个人设定问题
    const { data: res } = await this.$http.post('api/directory/backup', this.form)
    // 仅仅在浏览器控制台打印后端返回的数据，没有处理
    console.log(res)
}
```

