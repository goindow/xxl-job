## 定时任务调度中心
本脚手架基于 xxl-job-*@2.5.0 封装，使用 docker-compose 编排，实现了 `调度中心` 及 `执行器` 的快速部署。
> XXL-JOB 是一个分布式任务调度平台，其核心设计目标是开发迅速、学习简单、轻量级、易扩展。现已开放源代码并接入多家公司线上产品线，开箱即用。


## 概念介绍
- 调度中心
  - 负责定时任务的调度，将对应任务执行请求分发给 `执行器`，提供了各种调度算法。
  - 提供可视化的管理后台，方便使用，且配置实时生效
  - 只负责调度，不负责执行
- 执行器
  - 真实负责执行定时任务的程序


## 目录说明
- db，数据库初始化脚本
- xxl-job-admin，调度中心相关目录
- xxl-job-executor，执行器相关目录

详细目录结构如下，

![image.png](/.attachments/image-fc31d9ce-76b6-4ecc-8c34-3a4fb562940a.png)


## 部署步骤
1. docker 环境安装

2. 下载解压安装包（[https://github.com/goindow/xxl-job](https://github.com/goindow/xxl-job)）

3. 执行数据库初始化脚本
```shell
mysql -u 用户名 -p 数据库名 < db/tables_xxl_job.sql
```

4. 部署「调度中心」（***假设宿主机 ip 为 192.168.0.100***）
- 修改配置文件 xxl-job-admin/conf/application.properties
```yaml
# 调度中心数据库链接
spring.datasource.url = jdbc:mysql://127.0.0.1:3306/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai
spring.datasource.username = xxx
spring.datasource.password = xxx

# more options...
```
- 启动调度中心
```shell
docker-compose up -d --build
```
- 配置执行器
> 打开调度中心管理后台，配置执行器
>
> 地址: `http://192.168.0.100`
> 账号: `admin/123456`

![image.png](/.attachments/image-cff3da39-46ec-45dc-9808-6acccd1125e1.png)


5. 部署「执行器」（***假设宿主机 ip 为 192.168.0.103，部署两个节点***）
- 修改 `节点1` 配置文件 xxl-job-executor/executors/executor-default/node-1/application.properties
```yaml
# web 配置
server.port = 18081

# 执行器配置（自动注册的地址，容器内强制指定，否则注册到调度中心的为容器内部地址）
xxl.job.executor.ip = 192.168.0.103
xxl.job.executor.port = 19091

# 执行器 AppName，执行器心跳注册分组依据
xxl.job.executor.appname = executor-default

# 调度中心地址
xxl.job.admin.addresses = http://192.168.0.100:8080/xxl-job-admin

# more options...
```
- 修改 `节点2` 配置文件 xxl-job-executor/executors/executor-default/node-2/application.properties
```yaml
# web 配置
server.port = 18082

# 执行器配置（自动注册的地址，容器内强制指定，否则注册到调度中心的为容器内部地址）
xxl.job.executor.ip = 192.168.0.103
xxl.job.executor.port = 19092

# 执行器 AppName，执行器心跳注册分组依据
xxl.job.executor.appname = executor-default

# 调度中心地址
xxl.job.admin.addresses = http://192.168.0.100:8080/xxl-job-admin

# more options...
```
- 配置 docker-compose.yml（所有执行器，在一个 docker-compose.yml 文件中编排）
```yaml
version: '2'

services:
  ### 默认「执行器@节点1」配置
  executor-default-1:
    image: "hyb/xxl-job-executor:2.5.0"
    build: "./image/"
    container_name: "executor-default-1"
    ports:
      - "18081:18081"
      - "19091:19091"
    volumes:
      - ./executors/executor-default/node-1/logs:/data/applogs
      - ./executors/executor-default/node-1/application.properties:/application.properties
#      - /etc/localtime:/etc/localtime
#    restart: always

  ### 默认「执行器@节点2」配置
  executor-default-2:
    image: "hyb/xxl-job-executor:2.5.0"
    build: "./image/"
    container_name: "executor-default-2"
    ports:
      - "18082:18082"
      - "19092:19092"
    volumes:
      - ./executors/executor-default/node-2/logs:/data/applogs
      - ./executors/executor-default/node-2/application.properties:/application.properties
#      - /etc/localtime:/etc/localtime
#    restart: always

### more executors...
```
- 启动执行器
```shell
docker-compose up -d --build
```

6. 验证集群是否搭建成功
> 打开调度中心管理后台，查看执行器节点信息
>
> 地址: `http://192.168.0.100`
> 账号: `admin/123456`

![image.png](/.attachments/image-517cc529-df50-4d74-b1f0-17560f4b39fc.png)
