--- 
 title: "go-micro开发运维实践(安装etcd集群，部署注册中心)" 
 date: 2023-02-07T15:52:40+08:00 
 draft: false 
 description: "go-micro开发运维实践(安装etcd集群，部署注册中心)" 
 tags: ["go-micro开发运维实践"] 
 categories: ["go-micro开发运维实践"] 
---
## etcd集群安装
在微服务架构中，注册中心作为基础设施，承担着服务注册以及服务发现的重要功能。`etcd`作为一个分布式一致性的`KV存储系统`，按照`etcd`官网给出的性能测试, 在2CPU，1.8G内存，SSD磁盘这样的配置下，单节点的写性能可以达到16K QPS, 而先写后读也能达到12K QPS，这个性能相当可观。而在go-micro中`etcd`作为注册中心默认驱动，得益于其灵活的拓展机制，要在go-micro中使用etcd相对简单，下面我们使用`docker-compose`部署一个etcd集群。

### 编写docker-compose

#### 创建yaml和配置文件
```
touch docker-compose.yaml
touch .env
```

#### 为etcd持久化提供挂载目录
```
mkdir -p data/etcd1
mkdir -p data/etcd2
mkdir -p data/etcd3
```

#### .env添加通用参数
```
# 设置时区
TZ=Asia/Shanghai

# 设置etcd镜像版本
ETCD_VERSION=3.5

# 设置e3w镜像版本
E3W_VERSION=latest
```

#### 编写docker-compose.yaml
:::info
这里我们主要通过 environment 配置项设置 etcd启动参数来定义集群配置，在启动过程中需要确保三个 etcd节点可以相互连接并通信。
:::
```
# docker-compose.yml
version: '3.3'

services:
  etcd1:
    image: bitnami/etcd:${ETCD_VERSION}
    environment:
      TZ: ${TZ}
      ALLOW_NONE_AUTHENTICATION: "yes"
      ETCD_NAME: "etcd1"
      ETCD_INITIAL_ADVERTISE_PEER_URLS: "http://etcd1:2380"
      ETCD_LISTEN_PEER_URLS: "http://0.0.0.0:2380"
      ETCD_LISTEN_CLIENT_URLS: "http://0.0.0.0:2379"
      ETCD_ADVERTISE_CLIENT_URLS: "http://etcd1:2379"
      ETCD_INITIAL_CLUSTER_TOKEN: "etcd-cluster"
      ETCD_INITIAL_CLUSTER: "etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380"
      ETCD_INITIAL_CLUSTER_STATE: "new"
    volumes:
      - ./data/etcd1:/bitnami/etcd
    ports:
      - 23791:2379
      - 23801:2380
    networks:
      - micro-network

  etcd2:
    image: bitnami/etcd:${ETCD_VERSION}
    environment:
      TZ: ${TZ}
      ALLOW_NONE_AUTHENTICATION: "yes"
      ETCD_NAME: "etcd2"
      ETCD_INITIAL_ADVERTISE_PEER_URLS: "http://etcd2:2380"
      ETCD_LISTEN_PEER_URLS: "http://0.0.0.0:2380"
      ETCD_LISTEN_CLIENT_URLS: "http://0.0.0.0:2379"
      ETCD_ADVERTISE_CLIENT_URLS: "http://etcd2:2379"
      ETCD_INITIAL_CLUSTER_TOKEN: "etcd-cluster"
      ETCD_INITIAL_CLUSTER: "etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380"
      ETCD_INITIAL_CLUSTER_STATE: "new"
    volumes:
      - ./data/etcd2:/bitnami/etcd
    ports:
      - 23792:2379
      - 23802:2380
    networks:
      - micro-network

  etcd3:
    image: bitnami/etcd:${ETCD_VERSION}
    environment:
      TZ: ${TZ}
      ALLOW_NONE_AUTHENTICATION: "yes"
      ETCD_NAME: "etcd3"
      ETCD_INITIAL_ADVERTISE_PEER_URLS: "http://etcd3:2380"
      ETCD_LISTEN_PEER_URLS: "http://0.0.0.0:2380"
      ETCD_LISTEN_CLIENT_URLS: "http://0.0.0.0:2379"
      ETCD_ADVERTISE_CLIENT_URLS: "http://etcd3:2379"
      ETCD_INITIAL_CLUSTER_TOKEN: "etcd-cluster"
      ETCD_INITIAL_CLUSTER: "etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380"
      ETCD_INITIAL_CLUSTER_STATE: "new"
    volumes:
      - ./data/etcd3:/bitnami/etcd
    ports:
      - 23793:2379
      - 23803:2380
    networks:
      - micro-network

networks:
  micro-network:
    external: true

```

#### 创建内部通信网络
```
docker network create micro-network
```
Docker默认状态下的三个network对象，上述命令默认是bridge：

- none： 只有一个回环网卡，没有任何的网络通信能力
- host： 与宿主机共用一块网卡
- bridge： 利用虚拟路由器进行网络通信

#### 启动etcd集群
```
docker-compose up -d
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1650804348895-359a7bf6-82bd-4295-b7af-f20839173ef8.png#clientId=u306a74a4-4fe0-4&from=paste&height=655&id=uda07b62d&name=image.png&originHeight=720&originWidth=1270&originalType=binary&ratio=1&rotation=0&showTitle=false&size=81656&status=done&style=none&taskId=ud3451e19-915d-4fdd-9015-bf42b5d6661&title=&width=1154.5454295213563)<br />查看docker-desktop中etcd集群已成功启动
:::warning
如果重启docker后etcd集群容器无法启动，请删除挂载目录下的data文件夹，或选择不持久化数据<br />rm -rf data/etcd1/data<br />rm -rf data/etcd2/data<br />rm -rf data/etcd3/data
:::

## 安装etcdWeb管理工具
etcd并没有像 Consul 的 Web 管理界面，导致我们不能直观地观看集群节点状态和管理数据。所以我们安装一个[e3w](https://github.com/soyking/e3w)对集群进行管理。 

#### 创建配置文件
```
mkdir -p conf/e3w
touch conf/e3w/config.ini
```
```
[app]
# 端口号
port=8080
# 是否需要登录认证
auth=false

[etcd]
# 根key
root_key=micro-service
# 根文件夹
dir_value=
# 集群地址
addr=etcd1:2379,etcd2:2379,etcd3:2379
# 用户名
username=
# 密码
password=
cert_file=
key_file=
ca_file=
```

#### docker-compose安装e3w
修改docker-compose.yaml，然后执行`docker-compose up -d e3w`
```
...

  e3w:
    image: soyking/e3w:${E3W_VERSION}
    environment:
      TZ: ${TZ}
    ports:
      - "8088:8080"
    volumes:
      - ./conf/e3w/config.ini:/app/conf/config.default.ini
    networks:
      - micro-network
...
```
访问[http://127.0.0.1:8088](http://127.0.0.1:8088/)<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1650806554128-ef589acd-23b0-4c55-9cb9-57cc267280d9.png#clientId=u730c8cb0-5acf-4&from=paste&height=845&id=u5c4982b0&name=image.png&originHeight=929&originWidth=1920&originalType=binary&ratio=1&rotation=0&showTitle=false&size=57395&status=done&style=none&taskId=u741ba6ed-9d14-486a-9884-d1fd5c25d76&title=&width=1745.4545076228378)

## 安装micro-web
micro-web是由micro工具包提供的微服务web管理界面，可以用来查看、管理、测试所有服务接口。能为我们后续开发工作提供很多便利。

#### docker-compose安装micro-web
修改docker-compose.yaml，然后执行`docker-compose up -d micro-web`
```
  ...
  
  micro-web:
    container_name: micro-web
    image: micro/micro:v2.9.3
    ports:
      - 8082:8082
    environment:
      MICRO_REGISTRY: "etcd"
      MICRO_REGISTRY_ADDRESS: "etcd1:2379,etcd2:2379,etcd3:2379"
    command: web
    networks:
      - micro-network
      
  ...
```
访问[http://127.0.0.1:8082](http://127.0.0.1:8082/)<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26186945/1650807431270-8cf038c8-d473-4f63-904c-c7740d7c29c2.png#clientId=u98d1fb23-4b26-4&from=paste&height=845&id=ubc9e86ad&name=image.png&originHeight=929&originWidth=1920&originalType=binary&ratio=1&rotation=0&showTitle=false&size=17733&status=done&style=none&taskId=ubb5cac8a-bc9c-4c60-a2df-e45f9b4c08d&title=&width=1745.4545076228378)
