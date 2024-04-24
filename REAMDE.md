在日常开发或者编程当中，经常需要用到redis集群，若是按照传统的方式，一个机器一个机器搭建，难免过于繁琐，故而可以通过dock er-compose编排方式，快速搭建。我在搭建过程当中，将操作记录下来，方便以后需要搭建三主三从节点时，可以基于以前的成功经验，快速搭建起来。

# 一、环境准备

准备三台机器，在每台机器上，计划安装一个Redis主节点和一个Redis从节点。

| 机器           | Redis节点                | 节点端口  |
| -------------- | ------------------------ | --------- |
| 192.168.31.130 | redis-master/redis-slave | 6379/6380 |
| 192.168.31.131 | redis-master/redis-slave | 6379/6380 |
| 192.168.31.132 | redis-master/redis-slave | 6379/6380 |

# 二、文件准备

2.1、创建Redis节点目录

分别在192.168.31.130、192.168.31.131、192.168.31.132机器上，执行以下命令，创建Redis主从节点文件目录——

```sh
for dir in redis-master/data redis-slave/data; do mkdir -p "/opt/docker/redis-cluster/$dir";done
```

2.2、创建节点配置文件redis.conf

分别在192.168.31.130、192.168.31.131、192.168.31.132机器上的/opt/docker/redis-cluster/redis-master/与/opt/docker/redis-cluster/redis-slave/目录下，创建一个redis.conf文件，文件内容包括以下属性——

```sh
port 6379 #指定 Redis 服务器监听的端口号，这是客户端与 Redis 服务器进行通信的端口。
save 900 1#在给定时间间隔内有多少次写操作时，Redis 将执行自动的快照（生成 RDB 文件）。
save 300 10
save 60 10000
dbfilename dump.rdb#指定生成的 RDB 文件的名称。
dir /data #指定持久化文件的存储目录。
appendonly yes #启用 AOF（Append-Only File）持久化模式。
appendfilename "appendonly.aof" #指定 AOF 文件的名称。
appendfsync everysec #控制 AOF 缓冲区的内容何时同步到硬盘。这里的选项 everysec 表示每秒同步一次
cluster-enabled yes #启用 Redis 集群功能。
cluster-config-file nodes.conf #指定保存集群拓扑信息的配置文件名。
cluster-node-timeout 5000 #设置节点间通信的超时时间，单位为毫秒。
```

快捷指令，直接在linux运行——

```shell
for dir in redis-master redis-slave; do 
 if [ "$dir" == "redis-master" ]; then
    port=6379
  elif [ "$dir" == "redis-slave" ]; then
    port=6380
  fi
echo "port $port 
save 900 1
save 300 10
save 60 10000
dbfilename dump.rdb
dir /data 
appendonly yes 
appendfilename "appendonly.aof" 
appendfsync everysec 
cluster-enabled yes 
cluster-config-file nodes.conf 
cluster-node-timeout 5000" > /opt/docker/redis-cluster/$dir/redis.conf;done
```

运行完成后，在/opt/docker/redis-cluster/redis-master/以及/opt/docker/redis-cluster/redis-slave/生成一个data目录和一个redis.conf文件——

![image](https://img2023.cnblogs.com/blog/1545382/202312/1545382-20231229153921247-1374894185.png)

redis.conf文件里内容就是前面设置的。



# 三、编写docker-compose.yml编排文件

分别在三台机器的/opt/docker/redis-cluster/目录下，创建docker-compose.yml文件，内容如下：

```yaml
version: '3.1'
services:
  redis-master:
    image: redis:5.0.8
    container_name: redis-master
    restart: always
    network_mode: "host" 
    volumes:
    - /opt/docker/redis-cluster/redis-master/data:/data
    - /opt/docker/redis-cluster/redis-master/redis.conf:/usr/local/etc/redis/redis.conf
    command: ["redis-server","/usr/local/etc/redis/redis.conf"]
  redis-slave:
    image: redis:5.0.8
    container_name: redis-slave
    restart: always
    network_mode: "host" 
    volumes:
      - /opt/docker/redis-cluster/redis-slave/data:/data
      - /opt/docker/redis-cluster/redis-slave/redis.conf:/usr/local/etc/redis/redis.conf
    command: [ "redis-server","/usr/local/etc/redis/redis.conf" ]
```

完成后，执行指令docker-compose up -d——

![image](https://img2023.cnblogs.com/blog/1545382/202312/1545382-20231229153934255-1706601356.png)

执行指令docker ps -a查看一下容器是否已经正常运行，如下显示证明没有问题——

![image](https://img2023.cnblogs.com/blog/1545382/202312/1545382-20231229153947168-501646785.png)

# 四、执行指令组建redis集群

4.1、任意一台机器上，执行以下指令，进入到docker容器当中——

```sh
docker exec -it redis-master bash #redis-master对应的是docker ps -a查看到的容器名redis-master
```

4.2、创建集群

```none
redis-cli --cluster create  192.168.31.130:6379  192.168.31.130:6380  192.168.31.131:6379 192.168.31.131:6380 192.168.31.132:6379 192.168.31.132:6380 --cluster-replicas 1
```

执行完成后，打印日志如下——

![image](https://img2023.cnblogs.com/blog/1545382/202312/1545382-20231229154001469-1895658923.png)

执行指令后，会出现提示“Can I set the above configuration? (type 'yes' to accept):”，这里输入yes，回车。出现以下情况话，就是已经成功创建redis集群了——

![image](https://img2023.cnblogs.com/blog/1545382/202312/1545382-20231229154008465-2111461745.png)

可以通过以下指令进入到redis客户端，查看集群情况——

```none
root@hadoop1:/data# redis-cli -c -h 192.168.31.130 -p 6379
```

然后，执行指令cluster info查看集群状况，显示cluster_state:ok则表示集群已经正常创建。

![image](https://img2023.cnblogs.com/blog/1545382/202312/1545382-20231229154021613-738346616.png)

当然，可以进一步通过cluster nodes指令，查看各节点状况，已经是三主三从的集群状况了——

![image](https://img2023.cnblogs.com/blog/1545382/202312/1545382-20231229154030045-193613515.png)

以上，就是整个集群搭建过程。