1. 修改实例主机名；重置实例密码；重启

2. 远程连接-VNC连接-设置密码（7217aA）

3. 为Linux实例开启root用户远程登录：https://help.aliyun.com/document_detail/147650.html?#title-s05-xk0-670

4. 远程连接-SSH连接

5. 配置服务器

   ```bash
   # 换源
   sudo apt update
   sudo apt upgrade
   # 配置git
   # 配置go
   go env -w GO111MODULE=on
   go env -w GOPROXY=https://goproxy.cn,direct
   
   # 其他
   sudo apt install cmake
   ```

   