## linux

### protoc安装

apt install -y protobuf-compiler

但是显示权限不够

改为

sudo apt install -y protobuf-compiler

成功

### protoc-gen-go安装 和 protoc-gen-go-grpc安装

一直没有成功

尝试过

sudo go get google.golang.org/protobuf/cmd/protoc-gen-go@lastest

sudo go get -u github.com/golang/protobuf/{proto,proto-gen-go}

sudo go install google.golang.org/protobuf/cmd/protoc-gen-go@lastest

都没有成功

问题都是无法与tcp 142.251.43.114:443建立连接，应该是go get的问题，但当时是可以翻墙的。

## windows

### protoc安装

https://github.com/protocolbuffers/protobuf/releases

下载，解压，配置环境变量

### protoc-gen-go安装 和 protoc-gen-go-grpc安装

go get google.golang.org/grpc/cmd/protoc-gen-go

go get google.golang.org/grpc/cmd/protoc-gen-go-grpc

命令：.\protoc --go_out=plugins=grpc:. demo.proto

这是在protoc.exe所在目录下执行的命令

其他方法：（当时先执行的下面的图，然后按照上面两张图的顺序依次执行的，应该有重复）

go get  github.com/golang/protobuf/proto

go get  github.com/golang/protobuf/proto-gen-go