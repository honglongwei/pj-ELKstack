# ELK 日志分析系统

## 一、简介
#### 1、核心组成
ELK由Elasticsearch、Logstash和Kibana三部分组件组成；<br>
  Elasticsearch是个开源分布式搜索引擎，它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制，restful风格接口，多数据源，自动搜索负载等。<br>
  Logstash是一个完全开源的工具，它可以对你的日志进行收集、分析，并将其存储供以后使用 <br>
  kibana 是一个开源和免费的工具，它可以为 Logstash 和 ElasticSearch 提供的日志分析友好的 Web 界面，可以帮助您汇总、分析和搜索重要数据日志。<br>

#### 2、四大组件
Logstash: logstash server端用来搜集日志；<br>
Elasticsearch: 存储各类日志；<br>
Kibana: web化接口用作查寻和可视化日志；<br>
Logstash Forwarder: logstash client端用来通过lumberjack 网络协议发送日志到logstash server；<br>

#### 3、ELK工作流程
在需要收集日志的所有服务上部署logstash，作为logstash agent（logstash shipper）用于监控并过滤收集日志，将过滤后的内容发送到Redis，然后logstash indexer将日志收集在一起交给全文搜索服务ElasticSearch，可以用ElasticSearch进行自定义搜索通过Kibana 来结合自定义搜索进行页面展示。<br>

#### 4、ELK的帮助手册
ELK官网：https://www.elastic.co/<br>
ELK官网文档：https://www.elastic.co/guide/index.html<br>
ELK中文手册：http://kibana.logstash.es/content/elasticsearch/monitor/logging.html<br>

## 二、Logstash
####1、安装jdk
```python
# yum install java-1.8.0-openjdk java-1.8.0-openjdk-devel java-1.8.0-openjdk-headless
# java -version
openjdk version "1.8.0_51"
OpenJDK Runtime Environment (build 1.8.0_51-b16)
OpenJDK 64-Bit Server VM (build 25.51-b03, mixed mode)
```

#### 2、安装logstash
```python
# wget https://download.elastic.co/logstash/logstash/logstash-1.5.4.tar.gz
# tar zxf logstash-1.5.4.tar.gz -C /usr/local/
 
配置logstash的环境变量
# echo "export PATH=\$PATH:/usr/local/logstash-1.5.4/bin" > /etc/profile.d/logstash.sh
# . /etc/profile
```

#### 3、logstash常用参数
```python
-e :指定logstash的配置信息，可以用于快速测试;
-f :指定logstash的配置文件；可以用于生产环境;
```

#### 4、启动logstash
```python
通过-e参数指定logstash的配置信息，用于快速测试，直接输出到屏幕。
# logstash -e "input {stdin{}} output {stdout{}}"            
my name is Zline.    //手动输入后回车，等待10秒后会有返回结果
Logstash startup completed
2016-10-28T13:55:50.660Z 0.0.0.0 my name is Zline.
这种输出是直接原封不动的返回...

通过-e参数指定logstash的配置信息，用于快速测试，以json格式输出到屏幕。
# logstash -e 'input{stdin{}}output{stdout{codec=>rubydebug}}'
my name is Zline.    //手动输入后回车，等待10秒后会有返回结果
Logstash startup completed
{
       "message" => "my name is Zline.",
      "@version" => "1",
    "@timestamp" => "2016-10-28T13:57:31.851Z",
          "host" => "0.0.0.0"
}
这种输出是以json格式的返回...
```

#### 5、logstash以配置文件方式启动
```python
输出信息到屏幕
# vim logstash-simple.conf 
input { stdin {} }
output {
   stdout { codec=> rubydebug }
}
 
# logstash -f logstash-simple.conf    //普通方式启动
Logstash startup completed
 
# logstash agent -f logstash-simple.conf --verbose //开启debug模式
Pipeline started {:level=>:info}
Logstash startup completed
hello world.    //手动输入hello world.
{
       "message" => "hello world.",
      "@version" => "1",
    "@timestamp" => "2016-10-28T14:01:43.724Z",
          "host" => "0.0.0.0"
}
效果同命令行配置参数一样...

logstash输出信息存储到redis数据库中
前提是本地(192.168.1.104)有redis数据库，那么下一步我们就是安装redis数据库.
# cat logstash_to_redis.conf
input { stdin { } }
output {
    stdout { codec => rubydebug }
    redis {
        host => '192.168.1.104'
        data_type => 'list'
        key => 'logstash:redis'
    }
}
 
如果提示Failed to send event to Redis，表示连接Redis失败或者没有安装，请检查...
```

#### 6、 查看logstash的监听端口号
```python
# logstash agent -f logstash_to_redis.conf --verbose --logs stdout.log &
# netstat -tnlp |grep java
tcp        0      0 :::9301                     :::*                        LISTEN      1326/java
```

## 三、Redis
#### 1、安装Redis
```python
wget http://download.redis.io/releases/redis-2.8.19.tar.gz
yum install tcl -y
tar zxf redis-2.8.19.tar.gz
cd redis-2.8.19
make MALLOC=libc
make test    //这一步时间会稍久点...
make install
 
cd utils/
./install_server.sh     //脚本执行后，所有选项都以默认参数为准即可
Welcome to the redis service installer
This script will help you easily set up a running redis server
 
Please select the redis port for this instance: [6379] 
Selecting default: 6379
Please select the redis config file name [/etc/redis/6379.conf] 
Selected default - /etc/redis/6379.conf
Please select the redis log file name [/var/log/redis_6379.log] 
Selected default - /var/log/redis_6379.log
Please select the data directory for this instance [/var/lib/redis/6379] 
Selected default - /var/lib/redis/6379
Please select the redis executable path [/usr/local/bin/redis-server] 
Selected config:
Port           : 6379
Config file    : /etc/redis/6379.conf
Log file       : /var/log/redis_6379.log
Data dir       : /var/lib/redis/6379
Executable     : /usr/local/bin/redis-server
Cli Executable : /usr/local/bin/redis-cli
Is this ok? Then press ENTER to go on or Ctrl-C to abort.
Copied /tmp/6379.conf => /etc/init.d/redis_6379
Installing service...
Successfully added to chkconfig!
Successfully added to runlevels 345!
Starting Redis server...
Installation successful!
```

#### 2、查看redis的监控端口
```python
# netstat -tnlp |grep redis
tcp        0      0 0.0.0.0:6379                0.0.0.0:*                   LISTEN      3843/redis-server * 
tcp        0      0 127.0.0.1:21365             0.0.0.0:*                   LISTEN      2290/src/redis-serv 
tcp        0      0 :::6379                     :::*                        LISTEN      3843/redis-server *
```

#### 3、测试redis是否正常工作
```python
# cd redis-2.8.19/src/
# ./redis-cli -h 192.168.1.104 -p 6379 //连接redis
192.168.1.104:6379> ping
PONG
192.168.1.104:6379> set name Zline
OK
192.168.1.104:6379> get name
"Zline"
192.168.1.104:6379> quit
```

#### 4、redis服务启动命令
```python
# ps -ef |grep redis
root      3963     1  0 08:42 ?        00:00:00 /usr/local/bin/redis-server *:6379
```

#### 5、redis的动态监控
```python
# cd redis-2.8.19/src/
# ./redis-cli monitor     //reids动态监控
```

#### 6、logstash结合redis工作
```python
基于入口redis启动logstash
# cat logstash_to_redis.conf
input { stdin { } }
output {
    stdout { codec => rubydebug }
    redis {
        host => '192.168.1.104'
        data_type => 'list'
        key => 'logstash:redis'
    }
}
# logstash agent -f logstash_to_redis.conf --verbose
Pipeline started {:level=>:info}
Logstash startup completed
hello Zline
{
       "message" => "hello Zline",
      "@version" => "1",
    "@timestamp" => "2016-10-28T14:42:07.550Z",
          "host" => "0.0.0.0"
}

查看redis的监控接口上的输出
# ./redis-cli monitor
OK
1444315328.103928 [0 192.168.1.104:56211] "rpush" "logstash:redis" "{\"message\":\"hello Zline\",\"@version\":\"1\",\"@timestamp\":\"2016-10-28T14:42:07.550Z\",\"host\":\"0.0.0.0\"}"
 
如果redis的监控上也有以上信息输出，表明logstash和redis的结合是正常的。
```

## 四、Elasticsearch
#### 1、安装Elasticsearch
```python
yum 安装
    # rpm --import http://packages.elastic.co/GPG-KEY-elasticsearch
    # vim /etc/yum.repos.d/elasticsearch.repo
        [elasticsearch-2.1]
        name=Elasticsearch repository for 2.x packages
        baseurl=http://packages.elastic.co/elasticsearch/2.x/centos
        gpgcheck=1
        gpgkey=http://packages.elastic.co/GPG-KEY-elasticsearch
        enabled=1
    # yum install elasticsearch -y


源码安装
    # wget https://download.elastic.co/elasticsearch/elasticsearch/elasticsearch-1.7.2.tar.gz
    # tar zxf elasticsearch-1.7.2.tar.gz -C /usr/local/

rpm包安装
    # wget http://packages.elastic.co/elasticsearch/2.x/centos/elasticsearch-2.4.1.rpm
    # rpm -ivh elasticsearch-2.4.1.rpm
```

#### 2、修改elasticsearch配置文件elasticsearch.yml并且做以下修改.
```python
# vim /usr/local/elasticsearch-1.7.2/config/elasticsearch.yml
discovery.zen.ping.multicast.enabled: false        #关闭广播，如果局域网有机器开9300 端口，服务会启动不了
network.host: 192.168.1.104    #指定主机地址，其实是可选的，但是最好指定因为后面跟kibana集成的时候会报http连接出错（直观体现好像是监听了:::9200 而不是0.0.0.0:9200）
http.cors.allow-origin: "/.*/"
http.cors.enabled: true        #这2项都是解决跟kibana集成的问题，错误体现是 你的 elasticsearch 版本过低，其实不是
```

#### 3、启动elasticsearch服务
```python
yum 安装： 
  # /etc/init.d/elasticsearch start

源码安装：
  # /usr/local/elasticsearch-1.7.2/bin/elasticsearch     #日志会输出到stdout
  # /usr/local/elasticsearch-1.7.2/bin/elasticsearch -d #表示以daemon的方式启动
  # nohup /usr/local/elasticsearch-1.7.2/bin/elasticsearch > /var/log/logstash.log 2>&1 &
```

#### 4、查看elasticsearch的监听端口
```python
# netstat -tnlp |grep java
tcp        0      0 :::9200                     :::*                        LISTEN      7407/java           
tcp        0      0 :::9300                     :::*                        LISTEN      7407/java
```

#### 5、elasticsearch和logstash结合
```python
将logstash的信息输出到elasticsearch中
# cat logstash-elasticsearch.conf 
input { stdin {} }
output {
    elasticsearch { host => "192.168.1.104" }    
    stdout { codec=> rubydebug }
}
```

#### 6、基于配置文件启动logstash
```python
# /usr/local/logstash-1.5.4/bin/logstash agent -f logstash-elasticsearch.conf
Pipeline started {:level=>:info}
Logstash startup completed
python linux java c++    //手动输入
{
       "message" => "python linux java c++",
      "@version" => "1",
    "@timestamp" => "2016-10-28T14:51:56.899Z",
          "host" => "0.0.0.0"
}
```

#### 7、curl命令发送请求来查看elasticsearch是否接收到了数据
```python
# curl http://localhost:9200/_search?pretty
{
  "took" : 28,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 1,
    "max_score" : 1.0,
    "hits" : [ {
      "_index" : "logstash-2015.10.08",
      "_type" : "logs",
      "_id" : "AVBH7-6MOwimSJSPcXjb",
      "_score" : 1.0,
      "_source":{"message":"python linux java c++","@version":"1","@timestamp":"2015-10-08T14:51:56.899Z","host":"0.0.0.0"}
    } ]
  }
}
```

#### 8、安装elasticsearch插件
```python
#Elasticsearch-kopf插件可以查询Elasticsearch中的数据，安装elasticsearch-kopf，只要在你安装Elasticsearch的目录中执行以下命令即可：
# cd /usr/local/elasticsearch-1.7.2/bin/
# ./plugin install lmenezes/elasticsearch-kopf
-> Installing lmenezes/elasticsearch-kopf...
Trying https://github.com/lmenezes/elasticsearch-kopf/archive/master.zip...
Downloading .............................................................................................
Installed lmenezes/elasticsearch-kopf into /usr/local/elasticsearch-1.7.2/plugins/kopf
 
执行插件安装后会提示失败，很有可能是网络等情况...
-> Installing lmenezes/elasticsearch-kopf...
Trying https://github.com/lmenezes/elasticsearch-kopf/archive/master.zip...
Failed to install lmenezes/elasticsearch-kopf, reason: failed to download out of all possible locations..., use --verbose to get detailed information
 
解决办法就是手动下载该软件，不通过插件安装命令...
cd /usr/local/elasticsearch-1.7.2/plugins
wget https://github.com/lmenezes/elasticsearch-kopf/archive/master.zip
unzip master.zip
mv elasticsearch-kopf-master kopf
以上操作就完全等价于插件的安装命令
```

#### 9、浏览器访问kopf页面访问elasticsearch保存的数据
```python
# netstat -tnlp |grep java
tcp        0      0 :::9200                     :::*                        LISTEN      7969/java           
tcp        0      0 :::9300                     :::*                        LISTEN      7969/java           
tcp        0      0 :::9301                     :::*                        LISTEN      8015/java
```
![Image](https://github.com/honglongwei/pj-ELKstack/blob/master/images/1.jpg)


#### 10、从redis数据库中读取然后输出到elasticsearch中
```python
# cat logstash-redis.conf
input {
    redis {
        host => '192.168.1.104'  # 我方便测试没有指定password，最好指定password
        data_type => 'list'
        port => "6379"
        key => 'logstash:redis' #自定义
        type => 'redis-input'   #自定义
    }
}
output {
    elasticsearch {
        host => "192.168.1.104"
        codec => "json"
        protocol => "http"  #版本1.0+ 必须指定协议http
    }
}
```

## 五、Kinaba
#### 1、安装Kinaba
```python
# wget https://download.elastic.co/kibana/kibana/kibana-4.1.2-linux-x64.tar.gz
# tar zxf kibana-4.1.2-linux-x64.tar.gz -C /usr/local
```

#### 2、修改kinaba配置文件kinaba.yml
```python
# vim /usr/local/kibana-4.1.2-linux-x64/config/kibana.yml
elasticsearch_url: "http://192.168.1.104:9200"
```

#### 3、启动kinaba
```python
/usr/local/kibana-4.1.2-linux-x64/bin/kibana
 
输出以下信息，表明kinaba成功.
{"name":"Kibana","hostname":"localhost.localdomain","pid":1943,"level":30,"msg":"No existing kibana index found","time":"2015-10-08T00:39:21.617Z","v":0}
{"name":"Kibana","hostname":"localhost.localdomain","pid":1943,"level":30,"msg":"Listening on 0.0.0.0:5601","time":"2015-10-08T00:39:21.637Z","v":0}
kinaba默认监听在本地的5601端口上
```

#### 4、浏览器访问kinaba
使用默认的logstash-*的索引名称，并且是基于时间的，点击“Create”即可。<br>
看到如下界面说明索引创建完成。<br>
点击“Discover”，可以搜索和浏览Elasticsearch中的数据。<br>
