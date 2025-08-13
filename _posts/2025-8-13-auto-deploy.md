---
title: 使用gitee流水线 实现docker自动化部署golang应用程序
categories: [docker,git]
tags: [docker,git]
author: ailanz
pin: true
description: 本文章是记录使用gitee代码仓库，实现推送代码至指定分支后自动化部署、更新golang应用程序 # 文章简要描述
comments: false # 默认开启文章评论，若要关闭则填写false
---
## golang服务端 持续集成完整记录

#### 使用背景

在活动小程序开发的时候，后端已经初步开发好了，就想放置到服务器上，进行公网访问。但是前端在对接接口的时候，服务端会发生各种各样的迭代，因此就想模仿老金《无畏坦克》的持续集成的方式，提交代码，就执行流水线，自动部署，就不需要进行手动操作了

#### 工具链

gitee+golang+shell+docker+tls+localtime

#### 到gitee中开启流水线，对仓库的代码进行读取，并且构建产物到远程服务器中

<img src=/assets/img/2025-8-13-auto-deploy.assets/image-20250403084514791.png alt="avatar1"/>

<img src=/assets/img/2025-8-13-auto-deploy.assets/image-20250403084613400.png alt="avatar2"/>

<img src=/assets/img/2025-8-13-auto-deploy.assets/image-20250403084703272.png alt="avatar3"/>

- 点击编辑流水线，进入图形化编辑流水线的界面
<img src=/assets/img/2025-8-13-auto-deploy.assets/image-20250403084759182.png alt="avatar4"/>

- 配置触发事件，选择当有分支合并到我们的开发分支时，也就是`pull request事件`。之后选择任务编排，我们只需要选择**编译**（对仓库代码执行编译，生成golang可执行文件）跟**部署**（将这个可执行文件通过gitee服务，上传至我们远程的服务器中）

<img src=/assets/img/2025-8-13-auto-deploy.assets/image-20250403084936321.png alt="avatar5"/>

- ##### 编译 （直接跟着输入即可）

- 修改任务名称、任务唯一标识：`be_build`

- 选择golang版本号：1.22（这里gitee工具最新的就是1.22版本，但是实际gitee会根据你代码的golang version进行动态调整）

- 构建命令：就是你要对你仓库的代码，执行什么指令去编译。golang常用的构建命令就好

- 暂存构建物：配置构建出来的可执行文件（标识：上下文获取；打包文件目录：去哪里能拿到这个可执行文件）

- 构建缓存：没什么作用，可填入`/go/pkg/mod`
<img src=/assets/img/2025-8-13-auto-deploy.assets/image-20250403085530854.png alt="avatar6"/>

- ##### 部署（将产物部署到哪里？）

- 推送到远程服务器上，首先要在gitee中，将自己的服务器添加到gitee的主机组中（自己搜教程）

- 之后选择新的任务，选择主机部署

<img src=/assets/img/2025-8-13-auto-deploy.assets/image-20250403090245273.png alt="avatar7"/>

- 随后依次填入

- 任务的名称：be_deploy

- 执行的主机组：自己服务器

- 对推送到远程服务器的文件的配置：也就是这个可执行文件叫什么：`clubhub`

- 拿到上游构建的产物。

- 随后填入要执行什么脚本：

  - 在服务器上创建新的文件夹
  - 将这个构建的产物，推送到远程服务器
  - 添加这个可执行文件的权限
  - 最后使用我们自己写的部署脚本（容器化）
  - <img src=/assets/img/2025-8-13-auto-deploy.assets/image-20250403090716806.png alt="avatar8"/>

<img src=/assets/img/2025-8-13-auto-deploy.assets/image-20250403090545666.png alt="avatar9"/>

- ##### 到这里gitee的工作就完成了，最终的代码视图：

  ```yaml
  version: '1.0'
  name: branch-pipeline
  displayName: BranchPipeline
  triggers:
    trigger: auto
    push:
      branches:
        include:
          - .*
        exclude:
          - master
  stages:
    - name: build
      displayName: build
      strategy: naturally
      trigger: auto
      steps:
        - step: build@golang
          name: be_build
          displayName: be_build
          golangVersion: '1.22'
          commands:
            - '# 默认使用goproxy.cn'
            - export GOPROXY=https://goproxy.cn
            - '# 输入你的构建命令'
            - go mod tidy
            - go build -o clubhub
          artifacts:
            - name: BUILD_ARTIFACT
              path:
                - ./clubhub
          caches:
            - /go/pkg/mod
          notify: []
          strategy:
            retry: '0'
    - name: deploy
      displayName: deploy
      strategy: naturally
      trigger: auto
      steps:
        - step: deploy@agent
          name: be_deploy
          displayName: be_deploy
          hostGroupID:
            ID: clubhub
            hostID:
              - d2c36998-d3c5-49d7-b65e-6e5e530c1817
          deployArtifact:
            - source: build
              name: clubhub
              target: ~/gitee_go/deploy
              dependArtifact: BUILD_ARTIFACT
          script:
            - mkdir -p /var/deploy/clubhub/be
            - tar zxvf ~/gitee_go/deploy/clubhub.tar.gz -C /var/deploy/clubhub/be
            - chmod +x /var/deploy/clubhub/be/clubhub
            - /var/Users/bin/deploy.sh
          notify: []
          strategy:
            retry: '0'
  
  ```

- 尝试启动流水线，构建成功，成功部署到服务器上。
  
  - 在服务器上的文件目录：`/var/deploy/clubhub/be`会有这个`clubhub`文件

#### 容器化部署脚本，对这个产物挂载到容器内，对应用进行环境隔离

- gitee工具中，最后的部署脚本中，执行下面这行指令，会执行自己服务器`deploy.sh`脚本

  ```shell
   - /var/Users/bin/deploy.sh、
  ```

- 我们要准备一下这个让gitee执行的脚本`deploy.sh`随后放置到服务器的`/var/Users/bin/`路径下：容器化应用要准备什么？

  - 容器叫什么
  - 容器启动脚本
  - 容器应用所在网络
  - 容器映射端口 
  - 容器挂载卷 - 宿主机文件系统

- ```shell
  #!/bin/bash
  # -------------------------
  echo "deploy ..."
  APP_NAME=clubhub
  DEPLOY_TARGET_PATH=/var/deploy/clubhub/be
  mkdir -p $DEPLOY_TARGET_PATH
  SVC_NAME="app_${APP_NAME}"
  docker stop "$SVC_NAME" > /dev/null 2> /dev/null || :
  docker container rm "$SVC_NAME" > /dev/null 2> /dev/null || :
      
      
  docker network inspect qnear >/dev/null 2>&1 || docker network create qnear
  
  
  cd $DEPLOY_TARGET_PATH
  
  chmod +x run.sh 
  
  SVC_PORT=6167
  docker run --name "$SVC_NAME" \
    -d --restart=always \
    -p $SVC_PORT:$SVC_PORT \
    -v $DEPLOY_TARGET_PATH:$DEPLOY_TARGET_PATH \
    -v data:/var/data \
    -w $DEPLOY_TARGET_PATH \
    --network qnear \
    ubuntu:latest "${DEPLOY_TARGET_PATH}/run.sh" "${APP_NAME}"
  ```

- `run.sh`，让容器挂载的启动入口时，就执行的脚本，让容器能启动我们的golang可执行文件，我们需要将这个文件，放到我们服务器的路径`/var/deploy/clubhub/be`中

- `Shanghai`是一个时区文件，用于初始化此时容器内的时区与当前的上海时区进行同步，同样放到我们服务器的路径`/var/deploy/clubhub/be`中

  ```shell
  cp Shanghai /etc/localtime
  ./$1 ginGo 
  ```

- 此时我们的容器启动脚本，有-w的参数，因此会直接进入到`/var/deploy/clubhub/be`的工作目录中，因此，直接执行让我们的应用启动的指令即可

#### 首次尝试进行创建容器

- 以上步骤均完成之后，执行一次流水线，流水线成功执行；但是会发现此时的容器无法启动。进入宝塔Linux页面，看到此时的`app_clubhub`容器处于停止的状态。

- 查看此时容器日志，发现是config读取.cfg文件读取不到，因此可能是此时只有一个可执行文件，而没有将这个配置文件也映射到容器内部。因此我们需要将这个**项目配置文件**也放置到我们服务器的路径`/var/deploy/clubhub/be`中，通过挂载放置到容器内；

- 再次启动流水线，发现容器能成功运行

<img src=/assets/img/2025-8-13-auto-deploy.assets/image-20250403093636076.png alt="avatar10"/>



#### 引入gpt接口，从容器内部发送http请求到外部，需要对应的网站tls证书。

- 在检测接口时，发现从容器发送http请求到外部时，外部网站报错，显示

- ```shell
  "https://ark.cn-beijing.volces.com/api/v3/chat/completions\": tls: failed to verify certificate: x509: certificate signed by unknown authority"
  ```

  因此去到远程服务器中使用openssl工具去生成对应网站的sem证书文件，上网搜索，执行指令安装，将这个证书放置到`/var/deploy/clubhub/be`中，随后使用`-v`去挂载到容器的`/usr/local/share/ca-certificates/`路径中，在容器启动的时候，去执行更新这个证书的内容即可，修改我们的`deploy.sh`脚本。

  ```shell
  #!/bin/bash
  # -------------------------
  echo "deploy ..."
  APP_NAME=clubhub
  DEPLOY_TARGET_PATH=/var/deploy/clubhub/be
  mkdir -p $DEPLOY_TARGET_PATH
  SVC_NAME="app_${APP_NAME}"
  docker stop "$SVC_NAME" > /dev/null 2> /dev/null || :
  docker container rm "$SVC_NAME" > /dev/null 2> /dev/null || :
      
      
  docker network inspect qnear >/dev/null 2>&1 || docker network create qnear
  
  
  cd $DEPLOY_TARGET_PATH
  
  chmod +x run.sh 
  
  SVC_PORT=6167
  docker run --name "$SVC_NAME" \
    -d --restart=always \
    -p $SVC_PORT:$SVC_PORT \
    -v $DEPLOY_TARGET_PATH:$DEPLOY_TARGET_PATH \
    -v $DEPLOY_TARGET_PATH/server-cert.pem:/usr/local/share/ca-certificates/server-cert.pem \
    -v data:/var/data \
    -w $DEPLOY_TARGET_PATH \
    --network qnear \
    ubuntu:latest "${DEPLOY_TARGET_PATH}/run.sh" "${APP_NAME}"
  
  ```

  同样要修改我们的`run.sh`脚本，在容器启动的时候，就需要执行更新这个证书

  ```shell
  #!/bin/bash
  
  apt update
  
  apt install -y ca-certificates
  
  update-ca-certificates
  
  cp Shanghai /etc/localtime
  ./$1 ginGo 
  ```

- 至此，容器构建就成功了。并且能正常对外发送http请求

#### 问题

- Shanghai文件，我是使用服务器上`/usr/share/zoneinfo/Asia`路径下的时区文件，直接复制到我的工作路径，再挂载进去。在容器启动的时候，发现就无法成功获取到这个时区文件；待解决

#### 总结

- 跟老金的执行脚本不一样的是：老金的脚本是直接构建+部署的，因为他能在同一台服务上获取到这个仓库代码，因此一个脚本即可。
- 对CI/CD有一个初步的了解了，并且对Linux、docker有更深的了解。

