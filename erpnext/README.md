# erpnext_oob_docker

#### 介绍
基于官方镜像 https://github.com/frappe/frappe_docker/ 添加了中文汉化，开箱即用，权限优化应用，可根据需要增减自定义应用
为解决从国外网站下载依赖包网络超时问题，自定义镜像dockerfile文件中apt, yarn, pip修改为了国内源，另外wkthtmltopdf改为了二制制文件直接安装

### 前提条件

1. linux系统
2. 已安装git
3. 已安装好docker及docker compose，参考版本号:Docker version 20.10.18, build b40c2f6，Docker Compose version v2.6.1
安装方法请参考 https://gitee.com/yuzelin/erpnext_oob_docker/blob/master/readme-install_docker.md
```
docker -v 
docker compose version
```
4. docker服务已启动，且当前用户需有docker组，即可直接运行docker命令不报错，可用以下命令添加当前用户到docKer组

```
sudo groupadd docker
$ sudo gpasswd -a $USER docker
$ newgrp docker
```

### 准备工作：下载本应用并切换到工作目录

```
git clone https://gitee.com/yuzelin/erpnext_oob_docker && cd erpnext_oob_docker
```

前提条件：本机80端口未被占用,如需使用其它端口，请修改pwd.yml中相应的宿主机端口号，其它更多参数，请查看pwd.yml文件

```bash
docker compose --project-name erpnext_oob -f pwd.yml up -d
```

等系统下载镜像文件并启动全部容器后，
需再等待约2分钟后台创建数据库，以下命令检查进度
```docker logs erpnext_oob-create-site-1```

#### 使用系统

在用户电脑上打开浏览器, 输入域名或IP， 用户名administrator,密码admin登录系统，完成初始化创建公司(类似其它ERP的帐套)
本机虚拟机，请先配置端口转发，如本机8080映射虚拟机80端口，则可通过localhost:8080访问系统

### 构建镜像(需添加其它自定义应用或需升级已安装应用)

```bash
1. apps.json文件中维护需一起打包的自定义应用
2. 修改--tag参数后的镜像名及版本
3. 根据需要修改构建参数，如frappe源地址，版本等
4. 如果出错提示文件下载失败，可在Dockerfile中将当前https://mirrors.ustc.edu.cn源换成其它国内源

cd images/custom  #切换到工作目录
export APPS_JSON_BASE64=$(base64 --wrap=0 apps.json) #加载apps.txt为二进制变量
docker build \
  --build-arg=FRAPPE_PATH=https://gitee.com/mirrors/frappe \
  --build-arg=FRAPPE_BRANCH=version-14 \
  --build-arg=PYTHON_VERSION=3.10.5 \
  --build-arg=NODE_VERSION=16.18.0 \
  --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
  -f Dockerfile \
  --tag=szufisher/erpnext_oob:v14.0.3 .

如果添加了自定义app, 需在pwd.yml文件中相应添加--install-app 及echo 语句中的自定义app  
修改pwd.yml中引用的镜像
```


### 部署服务

修改`.env`环境变量(可选步骤)
可使用sed命令，如将db密码由123修改为123456 `sed -i 's/DB_PASSWORD=123/DB_PASSWORD=123456/g' .env`
如果有域名，请在.env 文件中参数 `FRAPPE_SITE_NAME_HEADER=erpnext13.local` 中的erpnext13.local为域名，即保持与以下docker compose exec backend bench new-site erpnext13.local一致。

#### 使用外部数据库和Redis

可在`.env`中设置对应的数据库或Redis的环境变量来指定外部服务，指定外部服务后可以停止或删除docker-compose.yml中的`db`或`redis`

# 附 单配置文件内容介绍

此配置文件 `pwd.yml` 包括了全部系统进程，本容器部署方式取代单机正式运行中的supervisor进程管理机制

## 服务

### bench服务进程

- frontend 前端, nginx反向代理，响应html, js, css等静态资源请求, 动态资源请求转给后端与websocket.
- backend 后端, 
- websocket, nodejs长连接(用于服务器实时推送单据变更通知).
- schedule, 后台任务排程器.
- queue-default, 后台任务-默认.
- queue-long, 后台任务-长(执行时间长)
- queue-short, 后台任务-短(执行时间短).

### 依赖的服务，即数据库与缓存数据库

- db, mariadb, mariadb数据库.
- redis-cache, 用于后端服务的redis缓存队列.
- redis-queue, 用于后台任务的redis缓存列队.
- redis-socketio, 用于websocket长连接的redis缓存队列.

### 仅运行一次的配置服务进程

- configurator 配置服务, 基于`common_site_config.json`配置前、后端及后台任务使用的数据库与redis缓存服务（数据库连接参数).
- create-site 创建站点, 创建一个数据库（站点).

## 存储卷

- sites 站点: bench数据. 通用配置, 所有站点, 所有站点及站点文件.
- logs 日志: 所有标准输出，即日志.

### 其它参考命令
```bash
停止
docker-compose --project-name erpnext_oob down
docker container prune -f
docker volume prune -f
docker-compose --project-name erpnext_oob ps
进入容器
docker exec -it erpnext_oob-backend-1 /bin/bash
查看容器控制台输出/日志， 数据库创建进度
docker logs erpnext_oob-create-site-1

更新版本
修改`.env`环境变量`VERSION`为新镜像版本


# 涉及数据库结构变更以及hooks勾子中变更了后台任务后，需执行bench migrate
docker compose exec backend bench migrate

# 针对多个改动可能需要重启后端
docker compose restart backend
```

 ** 常见问题** 
1. docker compose up -d 速度比较慢
可设置国内镜像源
1.1 新增或修改 /etc/docker/daemon.json，增加以下内容

```
{
  "registry-mirrors": ["https://registry.docker-cn.com"]
}
```

2.2 重启docker

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```