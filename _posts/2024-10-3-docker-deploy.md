---
title: docker本地直接搭建开发环境
categories: [blog,docker]
tags: [blog,docker]
author: ailanz
pin: true
description: 本文章是记录开发无畏坦克后，本地模拟服务器运行环境，部署项目运行 # 文章简要描述
comments: false # 默认开启文章评论，若要关闭则填写false
---
## 本地部署服务器

首先需要安装dockor，并且需要安装镜像源：

```java
	//Nginx的环境
docker pull nginx:latest
    
    //Ubuntu的环境
docker pull ubuntu:latest   

    //本地模拟网络环境
docker network create qnear
```







#### 前端：

- 需要定义个配置文件：在一个叫nginx的文件夹中，在nginx.conf的目录下再次创建default.conf文件，里面存放着关于前端服务器端的配置，包括Nginx的代理端口

  default.conf文件内容如下：	
  
  myUbuntu

```javascript
server {
  server_name localhost; # 服务主机
  //这里的监听端口就是在本地自己开启的端口，能发送给Nginx代理

        listen 6223;  # 监听端口

  charset utf-8;

  error_page  404              /404.html;

  # redirect server error pages to the static page /50x.html
  #
  error_page   500 502 503 504  /50x.html;
  location = /50x.html {
    root   /var/data/web/50x;
  }
	
  # 反向代理,将指定路径转发到指定目标, '/api'为 'http' 请求
  location /api {
    resolver 127.0.0.11;
    set $target http://myUbuntu:6167;
    proxy_pass $target;

    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-for $remote_addr;

    port_in_redirect off;

    proxy_connect_timeout 300s;
    proxy_send_timeout 300s;
    proxy_read_timeout 300s;
  }
	
  # '/msg' 为 'websocket' 连接
  location /msg {
    resolver 127.0.0.11;
      //这里代理的名字，就是后端容器的名字，也就是在docker容器里面的名字
    set $target http://myUbuntu:6167;
    proxy_pass $target;
    
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-for $remote_addr;

    port_in_redirect off;

    proxy_connect_timeout 300s;
    proxy_send_timeout 300s;
    proxy_read_timeout 300s;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

  }

  root /var/tank/fe;
  index  index.html; # 引入前端构建项目,即前端访问页面

  location / {
    # First attempt to serve request as file, then
    # as directory, then fall back to redirecting to index.html
    try_files $uri $uri/ $uri.html /index.html;
  }

  location ~* \.(?:css|js|jpg|svg)$ {
    expires 30d;
    add_header Cache-Control "public";
  }

  location ~* \.(?:json)$ {
    expires 1d;
    add_header Cache-Control "public";
  }
}
```

- 然后需要使用指令来创建容器：

  ```javascript
  docker run --name myUbuntu -d --restart=always -v"D:\VM\nginx\conf.d:/etc/nginx/share" -v"D:\VM\nowlastestNPC\prj\tank\fe\build:/var/tank/fe" -p 6223:6223 --network qnear nginx:latest
  ```

  将-v" "路径，替换成配置文件的路径
  （1）第一种在   :   后面添加上/etc/nginx/conf.d   （这个的效果就是直接build的文件，就能直接用影响到对应的文件）

  （2）第二种   在：后面使用/etc/nginx/share （需要手动的进行指令cp拷贝）

  也就是-v "    原本default.conf的路径     :   /etc/nginx/conf.d   "

  随后下一个   - v " 原build文件的路径    ： /var/tank/fe   "

  接着就是 -p 端口号：端口号(在配置文件中localhost：配置好的端口号) 

  --network qnear nginx:latest
<img src="/assets/img/2024-10-3-docker-deploy.assets/image-20240607224642247.png"  />

成功的话，会输出一段组合，并且在docker中会有新的容器生成



随后输入命令  docker exec -it myUbuntu bash

进入容器环境内部，对文件的运行环境进行搭建  这里的myUbuntu 需要更换成自己容器的名字
<img src="/assets/img/2024-10-3-docker-deploy.assets/image-20240607224854531.png"  />

成功之后，会进入到root权限之下，也就是容器内部了

- 随后需要将配置文件，拷贝进去容器内部，打开到存放配置文件的地方 /etc/nginx/conf.d的路径之下，使用指令cat打开文件
<img src="/assets/img/2024-10-3-docker-deploy.assets/image-20240607225051522.png"  />
  **可以查看此时的原来的配置文件内容，然后需要对文件进行拷贝,使用指令: cp  ./default.conf  /etc/nginx/conf.d   将文件拷贝进容器内部**
<img src="/assets/img/2024-10-3-docker-deploy.assets/image-20240607225229382.png"  />

随后使用指令nginx -t 重新配置文件，即可判断此时容器的配置是否成功

也就是需要将配置文件放到/etc/nginx/conf.d/的目录之下
<img src="/assets/img/2024-10-3-docker-deploy.assets/image-20240607225429867.png"  />

输出两句话，即为成功

- 随后需要安装各种依赖，使用各种语句：apt update
<img src="/assets/img/2024-10-3-docker-deploy.assets/image-20240607225527176.png"  />

- 随后重启容器，即可打开localhost：6223（端口号）打开自己的部署到线上的文件效果



写成脚本文件就是：

```javascript
docker stop myUbuntu

docker rm myUbuntu


docker run --name myUbuntu -d --restart=always -v"D:\VM\nginx\conf.d:/etc/nginx/conf.d " -v"D:\VM\nowlastestNPC\prj\tank\fe\build:/var/tank/fe" -p 6223:6223 --network qnear nginx:latest

pause

```



### 后端

还是一样的创建容器，指令：

```javascript
docker run --name myubuntu -d --restart=always -v"D:\VM\nowlastestNPC\prj\tank\be:/var/deploy/tank/be/" -p 6116:6116 --network qnear ubuntu:latest
```

这里只需要更改一次-v “ be后端的文件路径 ：/var/deploy/tank/be/  ”

- 随后需要进入docker exec -it myubuntu bash

- 然后需要打开到be文件，安装依赖apt update

- 进一步的进行安装golang的环境   apt install golang-go 

  - 有错误的话：输入 apt-get install --reinstall ca-certificates
    或者依次输入：apt install -y ca-certificates

    cd /usr/local/share/ca-certificates

    update-ca-certificates

- 随后go build进行可执行文件的生成

- 随后就能够正常开启后端