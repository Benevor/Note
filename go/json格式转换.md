# 不同数据格式之间的转换

## json<===>struct

```go
package main

import (
	"encoding/json"
	"fmt"
)

//结构体
type Person struct {
	HelloWold       string `json:"hello_wold"`
	LightWeightBaby string `json:"lightweightbaby"`
	Age             int    `json:"age"`
}


func main() {
	var a = Person{HelloWold: "nihao", LightWeightBaby: "muscle", Age: 1}

	// 结构体转化为json
	res, _ := json.Marshal(a)

	fmt.Println(typeof(res))
	// 输出： []uint8

	fmt.Printf("%s", res)
	// 输出： {"hello_wold":"nihao","lightweightbaby":"muscle","age":1}
	fmt.Println()

	//json转化为结构体
	var req_struct = Person{}
	json.Unmarshal(res, &req_struct)

	//json转化为string类型
	re := string(res)

	//string类型转化为结构体
	var req_struct1 = Person{}
	json.Unmarshal([]byte(re), &req_struct1)
	fmt.Println(req_struct1)
}


func typeof(v interface{}) string {
	return fmt.Sprintf("%T", v)
}

```



## json<===>[]struct

对于结构体的切片类型也可以与json相互转换

```go
// 设定存在一个结构体Geo
ret := make([]Geo, 0)

// 然后对ret 执行 append 操作

// 将ret转化为json
res_, _ := json.Marshal(ret)
// json转string
data := string(res_)

// 解析data
re := make([]Geo, 0)
err = json.Unmarshal([]byte(data), &re)
```



## json<===>map[string]interface{}

项目规定，两台主机之间RPC通信，只能传递string类型的数据

### part1

结合实际项目经历理解，没有完整的示例（项目为前后端分离，并且后端分为两个主机，通过RPC建立通信）

```go
// 主机A，gin接收前端请求
db := router.Group("/dashboard")
	{
		db.POST("/http/httpbase", httpBase)
	}

// 函数httpBase处理请求
func httpBase(c *gin.Context) {
    // 解析请求，拿到req
	req, err := ioutil.ReadAll(c.Request.Body)
    
    // 方式一：项目使用RPC协议，进行远程函数调用
    
    // 方式二：项目直接在此服务器上处理请求
}

// 方式一
    // 主机A，构造 rpc 通信 （将前端请求直接转化为string类型通过RPC发送给主机B）
	putRequest(a, unique_id, string(req))

	// 主机B，接收RPC请求。其中data为主机A发送的string(req);RpcRequest结构体按照前端发送请求的格式来定义
	req_ := e.RpcRequest{}
	json.Unmarshal([]byte(data), &req_)
	// 这样，前端的请求，被主机A接收后，以string格式通过RPC传送给主机B，解析为结构体，主机B拿到请求数据，执行操作，返回结果给A

// 方式二
	// 主机A，直接将前端请求，解析为结构体，然后在主机A上，执行操作，返回结果
	req_ := e.RpcRequest{}
	json.Unmarshal([]byte(req), &req_)
```

### part2

项目中，主机A/B，执行前端请求需要查询ES，查询结果为map[string]interface{}类型

```go
// 设定查询结果为result，类型为map[string]interface{}

// 方式一
	// 主机B，先将map[string]interface{}类型的result转化为json类型的res，然后将res以string的格式，通过RPC返还主机A
	res, _ := json.Marshal(result)
    send_data := &pb.MessageRequest{
        Type:             2,
        UniqueIdentifier: message.UniqueIdentifier, // 返回唯一标识符和请求唯一标识符一致
        Message:          string(res),
    }
    receiver <- send_data
	// 主机A，接收到主机B返回的RPC数据（下面的Data即主机B返回的string(res)），解析为map[string]interface{}格式
	var re map[string]interface{}
	json.Unmarshal([]byte(data), &re)
	// 主机A，将结果相应给前端
	c.JSON(http.StatusOK, map[string]string{"Data": re})
	c.JSON(http.StatusOK, re)

// 方式二
	// 主机A可以直接将查询结果返回前端
	c.JSON(http.StatusOK, map[string]string{"Data": result})
	// 或者
	c.JSON(http.StatusOK, result)
```

