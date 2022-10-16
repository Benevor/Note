# docker

### docker镜像

- 项目+环境可以一起生成镜像

### 隔离

- docker的核心思想
- docker通过隔离机制可以将服务器的资源利用到极致

### 组件/命令

- 镜像（image）

  - 分层下载

    ```shell
    docker search m ysql
    #默认最新版本
    docker pull mysql
    #指定版本
    docker pull mysql:5.7
    ```

  - 删除镜像

    ```shell
    docker rmi -f IMAGEID
    #删除全部镜像
    docker rmi -f $(docker images -aq)
    ```

- 容器（container）

  - 运行容器

    ```shell
    docker run [可选参数] image
    
    #参数说明
    --name="Name"	#容器名字，用来区分容器
    -d				#后台方式运行
    -ot				#使用交互方式运行，进入容器查看内容
    -p				#指定容器端口
    	-p 主机端口:容器端口#（常用）
    	-p 容器端口
    	容器端口
    	-p ip:主机端口:容器端口
    -P				#随机指定端口
    
    
    #启动并进入容器
    docker run -it ubuntu /bin/bash
    
    #进入正在运行的容器
    docker exec -it 容器ID /bin/bash		#开启一个新的终端
    
    docker attach 容器ID					#进入当前正在执行的终端
    
    #从容器中退出
    exit			#停止容器并退出
    Ctrl+P+Q		#退出，但不停止容器
    ```

  - 查看容器

    ```shell
    docker ps		#列出当前正在运行的容器
    -a				#列出所有容器
    -n=?			#列出最近的n个容器
    -q				#只显示容器编号
    
    
    docker top 容器ID						#查看容器内部进程信息
    
    docker inspect 容器ID					#查看容器详细信息
    ```

  - 删除容器

    ```shell
    docker rm 容器ID						#删除指定容器，但是不能删除正在运行的容器
    docker rm -f 容器ID					#强制删除指定容器
    docker rm -f $(docker ps -aq)		#删除所有容器
    docker ps -a -q|xargs docker rm		#删除所有容器
    ```

  - 容器主机交互

    ```shell
    #从容器内拷贝文件到主机
    docker cp 容器ID:容器内路径 目的主机路径
    ```

- 仓库（repository）

### WSL2安装docker

```shell
#安装
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo service docker start

#检查
# 检查dockerd进程启动
service docker status
ps aux|grep docker
# 检查拉取镜像等正常
docker pull busybox
docker images
```

## docker实践

### docker-系统

### docker-MySQL

### 项目打包为dockerfile
