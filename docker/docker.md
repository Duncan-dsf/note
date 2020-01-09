## 命令

- docker exec -it 进入某个容器
- run
  - --name 指定容器名
  - -v xxx:xxx 挂载目录 前面的目录是本机的 后一个是容器的
  - -p 80:80 端口映射
  - -e 环境变量
  - -d 后台运行
  - docker run -itd 后台运行一个镜像
- inspect
  - inspect id/name 查看容器所有状态信息 包括ip等
  - 查看ip
    - docker inspect --format='{{.NetworkSettings.IPAddress}}' id/name
- ps
  - 查看在运行的容器
  - -a 所有容器
- top xx 查看某容器进程
- port xx 查看某容器端口
- search 查找镜像

## 镜像

- 配置文件 /etc/docker/daemon.json
  - 阿里云个人加速器 https://nehhk0ok.mirror.aliyuncs.com

## 使用

#### 容器内如何访问容器的服务呢

- 固定ip
  - 每次容器重启，需要修改成对应的ip 不建议使用
  - 主机端口映射 容器访问主机服务 ip使用主机ip
  - docker的link