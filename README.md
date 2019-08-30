Tron事件插件的部署  
==

有两种部署方式，一种是kafka，一种是mongodb，我们以mongodb为例。路径根据自己的来。    
原文档：https://github.com/tronprotocol/documentation/tree/master/English_Documentation/TRON_Event_Subscribe
***

推荐配置：  
-----

CPU/ RAM: 16Core / 32G  
DISK: 500G	  
System: CentOS 64  

部署mongodb  
-----

 **1、安装命令**  
 ```
cd /home/java-tron
curl -O https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-4.0.4.tgz
tar zxvf mongodb-linux-x86_64-4.0.4.tgz
mv mongodb-linux-x86_64-4.0.4 mongodb
 ```
 **2、设置环境变量**  
 ```
export MONGOPATH=/home/java-tron/mongodb/
export PATH=$PATH:$MONGOPATH/bin
 ```
**3、mongodb配置**  
```
mkdir -p /home/java-tron/mongodb/log/ /home/java-tron/mongodb/data
cd /home/java-tron/mongodb/log/ && touch mongodb.log && cd
touch mgdb.conf
```
 创建数据和日志文件夹，写到配置文件中（绝对路径）  
 配置文件示例：
```
dbpath=/home/java-tron/mongodb/data
logpath=/home/java-tron/mongodb/log/mongodb.log
port=27017
logappend=true
fork=true
bind_ip=0.0.0.0
auth=true
wiredTigerCacheSizeGB=2
```
 注：bind_ip必须是0.0.0.0，否则无法远程连接。必须配置wiredTigerCacheSizeGB以防止OOM。  

**4、启动mongodb**  
```
mongod --config ./mgdb.conf &
```

**5、创建管理员**  
```
mongo
use admin
db.createUser({user:"root",pwd:"admin",roles:[{role:"root",db:"admin"}]})
```
**6、创建db eventlog和账户**  
```
db.auth("root", "admin")
use eventlog
db.createUser({user:"tron",pwd:"123456",roles:[{role:"dbOwner",db:"eventlog"}]})
```

部署Tron Event Query Service 
-----

**1、下载源码**  
```
git clone https://github.com/tronprotocol/tron-eventquery.git cd troneventquery
```
**2、构建命令**  
```
wget https://mirrors.cnnic.cn/apache/maven/maven-3/3.5.4/binaries/apache-maven-3.5.4-bin.tar.gz --no-check-certificate
tar zxvf apache-maven-3.5.4-bin.tar.gz
export M2_HOME=$HOME/maven/apache-maven-3.5.4
export PATH=$PATH:$M2_HOME/bin
mvn --version
mvn package
```
命令执行成功后，会在troneventquery/target下生成jar包，还有生成配置文件，路径是troneventquery/config.conf ，配置内容为：  
```
mongo.host=IP
mongo.port=27017
mongo.dbname=eventlog
mongo.username=tron
mongo.password=123456
mongo.connectionsPerHost=8
mongo.threadsAllowedToBlockForConnectionMultiplier=4
```
注：IP本地就填127.0.0.1，这里只有dbname不能改，这个是写死的。  

**3、启动tron event query service**  
```
sh troneventquery/deploy.sh
sh troneventquery/insertIndex.sh
```
注：默认端口8080，如果要改，修改脚本troneventquery/deploy.sh：
```
nohup java -jar -Dserver.port=8081 target/troneventquery-1.0.0-SNAPSHOT.jar 2>&1 &
```

部署Tron event subscribe plugin  
-----

**1、部署命令**  
```
git clone https://github.com/tronprotocol/event-plugin.git
cd eventplugin
./gradlew build
```
**2、节点启动文件配置**  

在节点配置文件末尾追加，下面是例子。也可以参考README.md。  

```
event.subscribe = {
    path = "/deploy/fullnode/event-plugin/build/plugins/plugin-mongodb-1.0.0.zip" // absolute path of plugin
    server = "127.0.0.1:27017" // target server address to receive event triggers
    dbconfig = "eventlog|tron|123456" // dbname|username|password
    topics = [
        {
          triggerName = "block" // block trigger, the value can't be modified
          enable = true
          topic = "block" // plugin topic, the value could be modified
        },
        {
          triggerName = "transaction"
          enable = true
          topic = "transaction"
        },
        {
          triggerName = "contractevent"
          enable = true
          topic = "contractevent"
        },
        {
          triggerName = "contractlog"
          enable = true
          topic = "contractlog"
        }
    ]

    filter = {
       fromblock = "" // the value could be "", "earliest" or a specified block number as the beginning of the queried range
       toblock = "" // the value could be "", "latest" or a specified block number as end of the queried range
       contractAddress = [
           "" // contract address you want to subscribe, if it's set to "", you will receive contract logs/events with any contract address.
       ]

       contractTopic = [
           "" // contract topic you want to subscribe, if it's set to "", you will receive contract logs/events with any contract topic.
       ]
    }
}  
```
注：
path: "plugin-kafka-1.0.0.zip" 或 "plugin-mongodb-1.0.0.zip"的绝对路径  
server: 服务器地址+端口号（mongodb的）  
dbconfig: mongodb的配置，按例子中的来  
topics: 目前支持四种事件类型 block, transaction, contract log and contract event  
triggerName: 触发类型，可以修改  
enable: fasle就是禁用，true开启  
topic: mongodb接收事件的集合  

启动全节点并且验证  
-----

**1、Fullnode的启动命令**  

```
java -jar FullNode.jar -p privatekey -c config.conf --es
```
 
注：在启动全节点前要先启动mongodb。  
全节点的安装参考：https://github.com/tronprotocol/java-tron/blob/develop/build.md  

**2、验证插件加载**  

```
tail -f logs/tron.log |grep -i eventplugin
```

出现类似字样即成功  
```
o.t.c.l.EventPluginLoader 'your plugin path/plugin-kafka-1.0.0.zip' loaded
```

注：fullnode在初始化的时候会加载插件，所以这个验证日志，要尽快打开，否则可能查不到  

**3、验证数据是否存到mongodb**  

如果查到结果说明从节点，到事件订阅，到mongodb没问题了。（如果mongo和节点不是同一台服务器，记得开放端口）  
```
mongo 47.90.245.68:27017
use eventlog
db.auth("tron", "123456")
show collections
db.block.find()
```
注：如果上述验证不能够实现，先查看fullnode日志来逐步排查。 

**4、使用event query service 进行访问**  

使用文档参考：https://github.com/tronprotocol/documentation/blob/master/English_Documentation/TRON_Event_Subscribe/tron-eventquery.md
