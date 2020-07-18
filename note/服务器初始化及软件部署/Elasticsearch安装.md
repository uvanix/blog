## 快速安装es集群与kibana
下载：cd /opt && wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.5.3.tar.gz

https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.4.3.tar.gz

解压：cd /opt && tar -zxvf elasticsearch-5.5.3.tar.gz

安装ik、pinyin、stconvert插件：
```
cd /opt/elasticsearch-5.5.3
bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.5.3/elasticsearch-analysis-ik-5.5.3.zip
bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-pinyin/releases/download/v5.5.3/elasticsearch-analysis-pinyin-5.5.3.zip
bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-stconvert/releases/download/v5.5.3/elasticsearch-analysis-stconvert-5.5.3.zip
```

单机安装一主两从的实例：

创建安装目录：mkdir /opt/elasticsearch

拷贝三个实例：
```
cp -rf /opt/elasticsearch-5.5.3 /opt/elasticsearch/elasticsearch-master-1
cp -rf /opt/elasticsearch-5.5.3 /opt/elasticsearch/elasticsearch-data-1
cp -rf /opt/elasticsearch-5.5.3 /opt/elasticsearch/elasticsearch-data-2
```

修改配置文件：
```
# 配置master1
cat >/opt/elasticsearch/elasticsearch-master-1/config/elasticsearch.yml<<EOF
# 集群名称，保证唯一
cluster.name: test-es
# 节点名称，仅仅是描述名称，用于在日志中区分
node.name: master-1
# 节点描述，可默认
node.attr.rack: r1
# 是否是master节点，master节点存放元数据
node.master: true
# 是否是data数据节点，data数据节点存放数据
node.data: false
# 是否是ingest节点,ingest节点可以在数据真正进入index前,通过配置pipline拦截器对数据ETL
node.ingest: false
# 数据的存放路径,可挂载多个盘
path.data: /opt/elasticsearch/elasticsearch-master-1/data
# 日志的存放路径
path.logs: /opt/elasticsearch/elasticsearch-master-1/logs
# 当前节点绑定ip
network.host: 0.0.0.0
# 对外提供服务的端口
http.port: 9200
# 集群间通讯端口
transport.tcp.port: 9300
# 设置集群自动发现机器IP集合
discovery.zen.ping.unicast.hosts: ["127.0.0.1:9300", "127.0.0.1:9301", "127.0.0.1:9302"]
# 为了避免脑裂，集群节点数量最少为候选主节点数量 半数+1
discovery.zen.minimum_master_nodes: 1
EOF
 
# 配置data1
cat >/opt/elasticsearch/elasticsearch-data-1/config/elasticsearch.yml<<EOF
# 集群名称，保证唯一
cluster.name: test-es
# 节点名称，仅仅是描述名称，用于在日志中区分
node.name: data-1
# 节点描述，可默认
node.attr.rack: r2
# 是否是master节点，master节点存放元数据
node.master: false
# 是否是data数据节点，data数据节点存放数据
node.data: true
# 是否是ingest节点,ingest节点可以在数据真正进入index前,通过配置pipline拦截器对数据ETL
node.ingest: true
# 数据的存放路径,可挂载多个盘
path.data: /opt/elasticsearch/elasticsearch-data-1/data
# 日志的存放路径
path.logs: /opt/elasticsearch/elasticsearch-data-1/logs
# 当前节点绑定ip
network.host: 0.0.0.0
# 对外提供服务的端口
http.port: 9201
# 集群间通讯端口
transport.tcp.port: 9301
# 设置集群自动发现机器IP集合
discovery.zen.ping.unicast.hosts: ["127.0.0.1:9300", "127.0.0.1:9301", "127.0.0.1:9302"]
# 为了避免脑裂，集群节点数量最少为候选主节点数量 半数+1
discovery.zen.minimum_master_nodes: 1
EOF
 
# 配置data2
cat >/opt/elasticsearch/elasticsearch-data-2/config/elasticsearch.yml<<EOF
# 集群名称，保证唯一
cluster.name: test-es
# 节点名称，仅仅是描述名称，用于在日志中区分
node.name: data-2
# 节点描述，可默认
node.attr.rack: r3
# 是否是master节点，master节点存放元数据
node.master: false
# 是否是data数据节点，data数据节点存放数据
node.data: true
# 是否是ingest节点,ingest节点可以在数据真正进入index前,通过配置pipline拦截器对数据ETL
node.ingest: true
# 数据的存放路径,可挂载多个盘
path.data: /opt/elasticsearch/elasticsearch-data-2/data
# 日志的存放路径
path.logs: /opt/elasticsearch/elasticsearch-data-2/logs
# 当前节点绑定ip
network.host: 0.0.0.0
# 对外提供服务的端口
http.port: 9202
# 集群间通讯端口
transport.tcp.port: 9302
# 设置集群自动发现机器IP集合
discovery.zen.ping.unicast.hosts: ["127.0.0.1:9300", "127.0.0.1:9301", "127.0.0.1:9302"]
# 为了避免脑裂，集群节点数量最少为候选主节点数量 半数+1
discovery.zen.minimum_master_nodes: 1
EOF
```

启动三个实例

第一步：liunx创建新用户  adduser elastic 或者创建组和用户

groupadd elastic

useradd elastic -g elastic -M

第二步：给新建的elastic赋权限，chown -R elastic.elastic /你的elasticsearch安装目录。 chown -R elastic.elastic /opt/elasticsearch

第三步：切换刚才创建的用户 su elastic，然后分别执行 /elasticsearch安装目录/bin/elasticsearch -d, 将三个实例都启动起来

依次启动各节点Elasticsearch服务：
```
/opt/elasticsearch/elasticsearch-master-1/bin/elasticsearch -d

/opt/elasticsearch/elasticsearch-data-1/bin/elasticsearch -d

/opt/elasticsearch/elasticsearch-data-2/bin/elasticsearch -d
```

安装完成测试
```
使用地址 http://127.0.0.1:9200/ 进行集群访问。显示信息如下：

{
  "name" : "master-1",
  "cluster_name" : "test-es",
  "cluster_uuid" : "l5NzGeOtRb-qRCIRjgLzyw",
  "version" : {
    "number" : "5.5.3",
    "build_hash" : "9305a5e",
    "build_date" : "2017-09-07T15:56:59.599Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.0"
  },
  "tagline" : "You Know, for Search"
}
可能遇到启动失败，修改系统配置

修改/etc/sysctl.conf文件，增加配置vm.max_map_count=262144

vi /etc/sysctl.conf
sysctl -p
执行命令sysctl -p生效

每个进程最大同时打开文件数太小，可通过下面2个命令查看当前数量

ulimit -Hn
ulimit -Sn
修改/etc/security/limits.conf文件，增加配置，用户退出后重新登录生效

* soft nofile 65536
* hard nofile 65536
```




## 1. 概述
由于2.4.2版本比较老旧，决定将Elasticsearch 升级到5.5.3版本。
升级之后的Elasticsearch感觉在各方面都便捷很多：

首先Elastic帮我们把各个组件的版本进行了统一，也把原来比较分散的组件进行了集成，比如在5.x之前各种单独的组件(Shield, Watcher, Marvel, Graph, Reporting)现在都集成到X-Pack中，Sense也集成进了Kibana中。
其次是数据类型做了部分改变，Elasticsearch5.0率先集成了Lucene6版本，性能得到很大的提升。据说与之前版本相比，磁盘空间少一半；索引时间少一半；查询性能提升25%；IPV6也支持了。
ELK(Elasticsearch、Logstash、Kibana)组合中又新增一个开源项目Beats, 从此可以改名：ELKB

## 2. 部署规划
部署环境在内网centos7上

### 2.1 Elasticsearch节点类型介绍

当我们启动Elasticsearch的实例，就会启动至少一个节点。相同集群名的多个节点的连接就组成了一个集群。

在默认情况下，集群中的每个节点都可以处理http请求和集群节点间的数据传输，集群中所有的节点都知道集群中其他所有的节点，可以将客户端请求转发到适当的节点。

节点有以下几个类型：

- 主(master)节点
node.master设置为True（默认）的时候，它有资格被选作为主节点，控制整个集群。

- 数据(data)节点
在一个节点上node.data设置为True（默认）的时候。该节点保存数据和执行数据相关的操作，如增删改查，搜索，和聚合。

- 客户端(client)节点
当一个节点的node.master和node.data都设置为false的时候，它既不能保持数据也不能成为主节点，该节点可以作为客户端节点，可以响应用户的情况，把相关操作发送到其他节点。

- 部落(tribe)节点
当一个节点配置tribe.*的时候，它是一个特殊的客户端，它可以连接多个集群，在所有连接的集群上执行搜索和其他操作。

Elasticsearch的data node除了放数据以外，也可以兼任master和client的角色，对于一个规模较大，用户较多的集群，master和client在一些极端使用情况下可能会有性能瓶颈甚至内存溢出，从而使得共存的data node故障。data node的故障恢复涉及到数据的迁移，对集群资源有一定消耗，容易造成数据写入延迟或者查询减慢。

如果将master和client独立出来，一旦出现问题，重启后几乎是瞬间就恢复的，对用户几乎没有任何影响。另外将这些角色独立出来的以后，也将对应的计算资源消耗从data node剥离出来，更容易掌握data node资源消耗与写入量和查询量之间的联系，便于做容量管理和规划。

#### 2.1.1 主(master)节点说明

主节点的主要职责是和集群操作相关的内容，如创建或删除索引，跟踪哪些节点是群集的一部分，并决定哪些分片分配给相关的节点。稳定的主节点对集群的健康是非常重要的。

默认情况下任何一个集群中的节点都有可能被选为主节点。索引数据和搜索查询等操作会占用大量的cpu，内存，io资源，为了确保一个集群的稳定，分离主节点和数据节点是一个比较好的选择。虽然主节点也可以协调节点，路由搜索和从客户端新增数据到数据节点，但最好不要使用这些专用的主节点。一个重要的原则是，尽可能做尽量少的工作。

创建一个独立的主节点只需在配置文件中添加如下内容：
```
node.master: true
node.data: false
```

为了防止数据丢失，配置discovery.zen.minimum_master_nodes设置是至关重要的（默认为1），每个主节点应该知道形成一个集群的最小数量的主资格节点的数量。

discovery.zen.minimum_master_nodes解释如下:

- 该处解释的即通常我们所说的脑裂问题。假设我们有一个集群，有3个主资格节点，当网络发生故障的时候，就有可能其中一个节点不能和其他节点进行通信了。这个时候，当discovery.zen.minimum_master_nodes设置为1的时候，就会分成两个小的独立集群，当网络好的时候，就会出现数据错误或者丢失数据的情况。当discovery.zen.minimum_master_nodes设置为2的时候，一个网络中有两个主资格节点，可以继续工作，另一部分，由于只有一个主资格节点，则不会形成一个独立的集群，这个时候当网络恢复的时候，节点又会重新加入集群。

设置这个值的原则是：

- （master_eligible_nodes / 2）+ 1

这个参数也可以动态设置：
```
PUT localhost:9200/_cluster/settings
{
  “transient”: {
    “discovery.zen.minimum_master_nodes”: 2
  }
}
```

#### 2.1.2 数据(data)节点说明
数据节点主要是存储索引数据的节点，主要对文档进行增删改查操作，聚合操作等。数据节点对cpu，内存，io要求较高，在优化的时候需要监控数据节点的状态，当资源不够的时候，需要在集群中添加新的节点。数据节点的配置如下：
```
node.master: false
node.data: true
```

数据节点路径设置,每一个主节点和数据节点都需要知道分片，索引，元数据的物理存储位置，path.data默认位为 $ES_HOME/data，可以通过配置文件 elasticsearch.yml进行修改，例如:
```
path.data: /data/es/data/
```

这个设置也可以在命令行上执行，例如：
```
./bin/elasticsearch –path.data /data/es/data
```

这个路径最好进行单独配置，这样Elasticsearch的目录和数据的目录就会分开。当删除了Elasticsearch主目录的时候，不会影响到数据。通过rpm安装默认是分开的。

数据目录可以被多个节点共享，甚至可以属于不同的集群，为了防止多个节点共享相同的数据路径，可以在配置文件elasticsearch.yml中添加：

- node.max_local_storage_nodes: 1

注意：在相同的数据目录不要运行不同类型的节点（例如：master, data, client）这很容易导致意外的数据丢失。

#### 2.1.3 客户端(client)节点说明

当主节点和数据节点配置都设置为false的时候，该节点只能处理路由请求，处理搜索，分发索引操作等，从本质上来说该客户节点表现为智能负载平衡器。独立的客户端节点在一个比较大的集群中是非常有用的，他协调主节点和数据节点，客户端节点加入集群可以得到集群的状态，根据集群的状态可以直接路由请求。

警告：添加太多的客户端节点对集群是一种负担，因为主节点必须等待每一个节点集群状态的更新确认！客户节点的作用不应被夸大，数据节点也可以起到类似的作用。配置如下：
```
node.master: false
node.data: false
```

#### 2.1.4 部落(tribe)节点说明
部落节点可以跨越多个集群，它可以接收每个集群的状态，然后合并成一个全局集群的状态，它可以读写所有节点上的数据，部落节点在elasticsearch.yml中的配置如下：

tribe:
t1: 
    cluster.name:   cluster_one
t2: 
    cluster.name:   cluster_two

T1和T2是任意的名字代表连接到每个集群。上面的示例配置两集群连接，名称分别是T1和T2。默认情况下部落节点通过广播可以做为客户端连接每一个集群。大多数情况下，部落节点可以像单节点一样对集群进行操作。

注意：以下操作将和单节点操作不同,如果两个集群的名称相同，部落节点只会连接其中一个。由于没有主节点，当设置local为true事，主节点的读操作会被自动的执行，例如：集群统计，集群健康度。主节点级别的写操作将被拒绝，这些应该是在一个集群进行。部落节点可以通过块(block)设置所有的写操作和所有的元数据操作，例如：

tribe:
blocks:
 write:    true
 metadata: true
    
部落节点可以也可以在选中的索引块中进行配置，例如：

tribe:
blocks:
    write.indices:    hk*,ldn*
    metadata.indices: hk*,ldn*
    
当多个集群有相同的索引名的时候，默认情况下，部落的节点将选择其中一个。这可以通过tribe.on_conflict setting进行配置，可以设置排除那些索引或者指定固定的部落名称。

### 2.2 其他组件介绍

以上对Elasticsearch各节点的作用了解以后，下面对其他需要安装的组件做简单介绍。

#### 2.2.1 Kibana
Kibana 是一个开源的分析和可视化平台，旨在与 Elasticsearch 合作。Kibana 提供搜索、查看和与存储在 Elasticsearch 索引中的数据进行交互的功能。开发者或运维人员可以轻松地执行高级数据分析，并在各种图表、表格和地图中可视化数据。

Kibana可以安装在任意一个节点上，也可每个节点都进行安装。不过一般建议是安装在客户端(client)节点之上，对Elasticsearch进行监控和操作。

#### 2.2.2 X-Pack
X-Pack是一个Elastic Stack的扩展，将安全，警报，监视，报告和图形功能包含在一个易于安装的软件包中。在Elasticsearch 5.0.0之前，您必须安装单独的Shield，Watcher和Marvel插件才能获得在X-Pack中所有的功能。在Elasticsearch 5版本之后，一般情况下只需要安装一个官方推荐的X-pack扩展包即可。

注意：必须在集群中的所有节点安装X-Pack插件。

#### 2.2.3 Head
Head是Elasticsearch的一个前端插件，可以很方便的查看ES的运行状态和数据。

Head插件建议安装在客户端节点上。

### 2.3 节点规划
测试环境选择了4个节点（资源有限一台服务器上安装），1个主(master)节点，2个数据(data)节点，1个客户端(client)节点。

如果你的是多master节点，只需要增加master节点，修改下discovery.zen.minimum_master_nodes参数即可。

节点IP角色安装组件

节点|IP|HostName|组件
:--|:--|:--|:--
node1|192.168.1.11|master-1|es(含x-pack)
node2|192.168.1.12|data-1|es(含x-pack)
node3|192.168.1.13|data-2|es(含x-pack)
node4|192.168.1.14|client-1|es(含x-pack)、kibana(含x-pack)、head

## 3. Elasticsearch安装

### 3.1 JDK安装
每个节点都必须安装jdk环境，安装过程略。

### 3.2 Elasticsearch环境安装
下载解压 elasticsearch-5.5.3.tar.gz
```
mkdir /opt/elasticsearch
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.5.3.tar.gz
tar -zxvf elasticsearch-5.5.3.tar.gz
mkdir cluster-test
cp -rf elasticsearch-5.5.3 cluster-test/elasticsearch-master-1
cp -rf elasticsearch-5.5.3 cluster-test/elasticsearch-data-1
cp -rf elasticsearch-5.5.3 cluster-test/elasticsearch-data-2
cp -rf elasticsearch-5.5.3 cluster-test/elasticsearch-client-1
mkdir cluster-test/elasticsearch-master-1/{data,logs}
mkdir cluster-test/elasticsearch-data-1/{data,logs}
mkdir cluster-test/elasticsearch-data-2/{data,logs}
mkdir cluster-test/elasticsearch-client-1/{data,logs}
```

#### 3.2.1 配置master-1
```
# 集群名称，保证唯一
cluster.name: test-es
# 节点名称，仅仅是描述名称，用于在日志中区分
node.name: master-1
# 节点描述，可默认
node.attr.rack: r1
# 是否是master节点，master节点存放元数据
node.master: true
# 是否是data数据节点，data数据节点存放数据
node.data: false
# 是否是ingest节点,ingest节点可以在数据真正进入index前,通过配置pipline拦截器对数据ETL
node.ingest: false
# 数据的存放路径,可挂载多个盘
path.data: /opt/elasticsearch/cluster-test/elasticsearch-master-1/data
# 日志的存放路径
path.logs: /opt/elasticsearch/cluster-test/elasticsearch-master-1/logs
# 当前节点绑定ip
network.host: 0.0.0.0
# 对外提供服务的端口
http.port: 9200
# 集群间通讯端口
transport.tcp.port: 9300
# 设置集群自动发现机器IP集合
discovery.zen.ping.unicast.hosts: ["192.168.1.11:9300", "192.168.1.12:9301", "192.168.1.13:9302", "192.168.1.14:9303"]
# 为了避免脑裂，集群节点数量最少为候选主节点数量 半数+1
discovery.zen.minimum_master_nodes: 1
```

#### 3.2.2 配置data-1
```
# 集群名称，保证唯一
cluster.name: test-es
# 节点名称，仅仅是描述名称，用于在日志中区分
node.name: data-1
# 节点描述，可默认
node.attr.rack: r2
# 是否是master节点，master节点存放元数据
node.master: false
# 是否是data数据节点，data数据节点存放数据
node.data: true
# 是否是ingest节点,ingest节点可以在数据真正进入index前,通过配置pipline拦截器对数据ETL
node.ingest: true
# 数据的存放路径,可挂载多个盘
path.data: /opt/elasticsearch/cluster-test/elasticsearch-data-1/data
# 日志的存放路径
path.logs: /opt/elasticsearch/cluster-test/elasticsearch-data-1/logs
# 当前节点绑定ip
network.host: 0.0.0.0
# 对外提供服务的端口
http.port: 9201
# 集群间通讯端口
transport.tcp.port: 9301
# 设置集群自动发现机器IP集合
discovery.zen.ping.unicast.hosts: ["192.168.1.11:9300", "192.168.1.12:9301", "192.168.1.13:9302", "192.168.1.14:9303"]
# 为了避免脑裂，集群节点数量最少为候选主节点数量 半数+1
discovery.zen.minimum_master_nodes: 1
```

#### 3.2.3 配置data-2
```
# 集群名称，保证唯一
cluster.name: test-es
# 节点名称，仅仅是描述名称，用于在日志中区分
node.name: data-2
# 节点描述，可默认
node.attr.rack: r3
# 是否是master节点，master节点存放元数据
node.master: false
# 是否是data数据节点，data数据节点存放数据
node.data: true
# 是否是ingest节点,ingest节点可以在数据真正进入index前,通过配置pipline拦截器对数据ETL
node.ingest: true
# 数据的存放路径,可挂载多个盘
path.data: /opt/elasticsearch/cluster-test/elasticsearch-data-2/data
# 日志的存放路径
path.logs: /opt/elasticsearch/cluster-test/elasticsearch-data-2/logs
# 当前节点绑定ip
network.host: 0.0.0.0
# 对外提供服务的端口
http.port: 9202
# 集群间通讯端口
transport.tcp.port: 9302
# 设置集群自动发现机器IP集合
discovery.zen.ping.unicast.hosts: ["192.168.1.11:9300", "192.168.1.12:9301", "192.168.1.13:9302", "192.168.1.14:9303"]
# 为了避免脑裂，集群节点数量最少为候选主节点数量 半数+1
discovery.zen.minimum_master_nodes: 1
```

#### 3.2.4 配置client-1
```
# 集群名称，保证唯一
cluster.name: test-es
# 节点名称，仅仅是描述名称，用于在日志中区分
node.name: client-1
# 节点描述，可默认
node.attr.rack: r4
# 是否是master节点，master节点存放元数据
node.master: false
# 是否是data数据节点，data数据节点存放数据
node.data: false
# 是否是ingest节点,ingest节点可以在数据真正进入index前,通过配置pipline拦截器对数据ETL
node.ingest: false
# 数据的存放路径,可挂载多个盘
path.data: /opt/elasticsearch/cluster-test/elasticsearch-client-1/data
# 日志的存放路径
path.logs: /opt/elasticsearch/cluster-test/elasticsearch-client-1/logs
# 当前节点绑定ip
network.host: 0.0.0.0
# 对外提供服务的端口
http.port: 9203
# 集群间通讯端口
transport.tcp.port: 9303
# 设置集群自动发现机器IP集合
discovery.zen.ping.unicast.hosts: ["192.168.1.11:9300", "192.168.1.12:9301", "192.168.1.13:9302", "192.168.1.14:9303"]
# 为了避免脑裂，集群节点数量最少为候选主节点数量 半数+1
discovery.zen.minimum_master_nodes: 1
```

### 3.3 浏览器查看
第一步：liunx创建新用户  adduser esUser

第二步：给新建的esUser赋权限，chown -R esUser /你的elasticsearch安装目录。

第三步：切换刚才创建的用户 su esUser，然后分别执行 /elasticsearch安装目录/bin/elasticsearch -d, 将三个安装包都启动起来

依次启动各节点Elasticsearch服务：

```
cluster-test/elasticsearch-master-1/bin/elasticsearch -d

cluster-test/elasticsearch-data-1/bin/elasticsearch -d

cluster-test/elasticsearch-data-2/bin/elasticsearch -d

cluster-test/elasticsearch-client-1/bin/elasticsearch -d
```

使用地址 http://192.168.1.11:9200/ 进行集群访问。显示信息如下：

```
{
  "name" : "master-1",
  "cluster_name" : "test-es",
  "cluster_uuid" : "l5NzGeOtRb-qRCIRjgLzyw",
  "version" : {
    "number" : "5.5.3",
    "build_hash" : "9305a5e",
    "build_date" : "2017-09-07T15:56:59.599Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.0"
  },
  "tagline" : "You Know, for Search"
}
```

可能遇到启动失败，修改系统配置

```
修改/etc/sysctl.conf文件，增加配置vm.max_map_count=262144

vi /etc/sysctl.conf
sysctl -p
执行命令sysctl -p生效

每个进程最大同时打开文件数太小，可通过下面2个命令查看当前数量

ulimit -Hn
ulimit -Sn
修改/etc/security/limits.conf文件，增加配置，用户退出后重新登录生效

* soft nofile 65536
* hard nofile 65536
```

## 4. Kibana安装

### 4.1 Kibana安装配置
下载Kibana安装包并解压。

```
修改配置文件 config/kibana.yml.

server.port: 5601
server.host: "0.0.0.0"
elasticsearch.url: "http://192.168.1.11:9200"
kibana.index: ".kibana"
```

### 4.2 浏览器查看
启动Kibana服务：

nohup bin/kibana &

使用地址 http://192.168.1.11:5601/ 进行访问。

## 5. 其他插件安装

### 5.1 IK分词插件
#### 5.1.1 介绍
IK分词插件是我们较常用的中文分词插件。

github地址：https://github.com/medcl/elasticsearch-analysis-ik

#### 5.1.2 安装

版本选择与elasticsearch对应的版本。比如我的elasticsearch版本为5.5.0，选择的ik的版本也是5.5.0.

(1) 下载安装包并解压到 \elasticsearch-5.5.3\plugins 目录下，重命名为 analysis-ik.

拷贝analysis-ik下的config文件夹到 \elasticsearch-5.5.3\config 目录下，并重命名为 ik。

重启ES.

所有节点均需以上安装操作。

在线安装：./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.5.3/elasticsearch-analysis-ik-5.5.3.zip

### 5.2 pinyin插件
#### 5.2.1 介绍
有时在淘宝搜索商品的时候，会发现使用汉字，拼音，或者拼音混合汉字都会出来想要的效果，该功能就是通过拼音搜索插件实现的。

github地址：https://github.com/medcl/elasticsearch-analysis-pinyin

#### 5.2.2 安装
在地址 https://github.com/medcl/elasticsearch-analysis-pinyin/releases 选择对应的版本下载。

将安装包解压到 \elasticsearch-5.5.3\plugins 目录下，并重命名为 'pinyin'.

重启ES.

所有节点均需以上安装操作。

在线安装：./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-pinyin/releases/download/v5.5.3/elasticsearch-analysis-pinyin-5.5.3.zip

