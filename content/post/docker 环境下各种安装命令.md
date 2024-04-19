# docker 环境下各种安装命令

## elasticsearch

拉取镜像

```shell
docker pull docker.elastic.co/elasticsearch/elasticsearch:6.3.2
```

运行容器

```shell
docker run -d --name es -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:6.3.2
```

- -e "discovery.type=single-node" ，设置为单节点。
- -p  9200:9200  http协议，为elasticsearch默认端口，用于外部通讯。
- -p 9300:9300  tcp协议，用于集群之间通信。