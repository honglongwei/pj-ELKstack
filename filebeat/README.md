# 搭建ELK + Filebeat

[1. 版本](#版本)

[2. 工作流图](#工作流图)

[3. 配置及启动](#配置及启动)

[4. 日志配置](#日志配置)

[5. 搜索语法](#搜索语法)

[6. 鉴权方案](#鉴权方案)

[7. 其他配置方案](#其他配置方案)

## 版本
* Logstash: 2.3.4
* ElasticSearch: 2.3.4
* Kibana: 4.5.3
* Filebeat: 1.2.3
* Nginx: 1.10.0
* Jdk: 1.7 以上

## 工作流图
![ELK filebeat workflow](https://github.com/honglongwei/pj-ELKstack/blob/master/images/elkstack-filebeat.png)

* Filebeat：安装在应用服务器或数据库服务器上，监听日志文件，然后将日志发送到Logstash。
* Logstash：日志收集，过滤，输出。
* ElasticSearch：搜索引擎，用于存储日志并提供全文检索与结构化检索的功能。
* Kibana：可视化工具，用于分析日志。
* Ngnix：反向代理服务器，用于过滤访问请求。

## 配置及启动
### 1. ElasticSearch
配置文件为项目根目录下的/config/elasticsearch.yml：
```
cluster.name: es_cluster                # 集群名称
node.name: test_node_0     # 节点名称
network.host: localhost      # 主机名
network.port: 9200       # 监听端口
bootstrap.mlockall: true    # 禁止使用swap
```
启动命令：
```
./bin/elasticsearch -Xmx2g -Xms2g   # 指定堆内存为2g，视情况调节大小
```
搭建集群的话在局域网另一台服务器上启动ES，只要配置文件中cluster.name相同，node.name不同，就会被自动检测到。

### 2. Logstash
需要自己创建相应的配置文件，在启动时指定。我们是从Filebeat收集日志输出到es的，可以取名为/config/filebeat2es.conf：
```
input {
        # For more info: https://www.elastic.co/guide/en/logstash/current/input-plugins.html
        beats {
        port => 10101   # 监听端口10101的输入
        codec => json   # 编码解码使用json
    }
}
filter {
        # Only matched data are send to output.
}
output {
        # For more info: https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html
        elasticsearch {
                action => "index"          # ES操作
                hosts  => "localhost:9200"   # ES主机
                workers => 4    # 工作线程数
        }
        stdout { 
                codec => rubydebug      # 打印到控制台，方便调试
        }
}
```
检查配置文件是否正确：
```
/bin/logstash --configtest -f config/filebeat2es.conf
```
打印Configuration OK后，就可以启动了。
启动命令：
```
./bin/logstash agent -f config/filebeat2es.conf
```

### 3. Kibana
配置文件为项目根目录下的/config/kibana.yml：
```
server.port: 5601
server.host: "localhost"
elasticsearch.url: "http://localhost:9200"      # ES的访问地址
```
启动命令：
```
./bin/kibana
```

### 4. Filebeat
配置文件为项目根目录下的/filebeat.yml：
```
filebeat:
    prospectors:
        -
            paths:
                - /access/*.log     # 访问日志地址
                - /server/*.log     # 服务器日志地址
            input_type: log
            scan_frequency: 10s      # 扫描日志目录的频率
            backoff: 1s     # 扫描日志文件是否有新内容的频率
            tail_files: true    # 从文件末尾开始读取
            
    # 默认配置
    registry_file: "C:/ProgramData/filebeat/registry"
output:
    logstash:
        hosts: ["localhost:10101"]      # 输出到Logstash
    
# 下面为默认配置    
shipper:
logging:
    files:
        rotateeverybytes: 10485760
``` 

### 5. 日志过期清理
在终端输入：
```sh
curl -XPUT localhost:9200/_template/logstash -d '  
{
  "template": "logstash-*",
  "settings": {
    "index": {
      "refresh_interval": "5s"
    }
  },
  "mappings": {
    "_default_": {
        "_ttl": {
                "enabled": true,
                "default": "30d"
        },
        "dynamic_templates": [
        {
          "message_field": {
            "mapping": {
              "index": "analyzed",
              "omit_norms": true,
              "fielddata": {
                "format": "disabled"
              },
              "type": "string"
            },
            "match_mapping_type": "string",
            "match": "message"
          }
        },
        {
          "string_fields": {
            "mapping": {
              "index": "analyzed",
              "omit_norms": true,
              "fielddata": {
                "format": "disabled"
              },
              "type": "string",
              "fields": {
                "raw": {
                  "index": "not_analyzed",
                  "ignore_above": 256,
                  "type": "string"
                }
              }
            },
            "match_mapping_type": "string",
            "match": "*"
          }
        }
      ],
      "properties": {
        "@timestamp": {
          "type": "date"
        },
        "geoip": {
          "dynamic": true,
          "properties": {
            "location": {
              "type": "geo_point"
            },
            "longitude": {
              "type": "float"
            },
            "latitude": {
              "type": "float"
            },
            "ip": {
              "type": "ip"
            }
          }
        },
        "@version": {
          "index": "not_analyzed",
          "type": "string"
        }
      },
      "_all": {
        "enabled": true,
        "omit_norms": true
      }
    }
  },
  "aliases": {}
}  
' 
```
此命令更新了logstash提供了elasticsearch索引的默认模板，使日志数据会在**30天**后自动删除。

## 日志配置
日志在输入到Logstash的时候需要被Logstash解析，得按照Logstash所要求的json格式化，但是这项工作Filebeat已经帮我们做了，所以我们不需要在项目中做任何日志的配置改动。
最终我们在日志文件中打印的信息会被添加到`message`字段。

## 搜索语法
因为Kibana在ELK当中主要用来查询展示数据的，具体的搜索由ElasticSearch提供，而ElasticSearch构建在Lucene之上，所以语法和Lucene相同。这里简单介绍几点：

1. 转义特殊字符：`+ - && || ! () {} [] ^" ~ * ? : \`需要用`\`进行转义。
2. 使用双引号作为短语搜索，测试中发现ES默认会将非字母符号作为分隔符进行分词，所以在搜索时间等字段时需要加上双引号，例如`"2016-08-01 10:38"`（测试发现只能精确到分）。
3. 通配符：`?`匹配**单个**字符，`*`匹配**0到多个**字符。
4. 模糊搜索：在一个单词后面加上`~`启用模糊搜索，例如`first~`也能搜索到`frist`。
5. 限定字段，例如我们要搜索`message`字段中时间为2016-08-01的：`message: "2016-08-01"`。
6. 范围搜索：数值和时间类型的字段可以对某一范围进行查询。如：
`
offset:[0 TO 1000]
date:{"now-6h" TO "now"}
`
[ ] 表示端点数值包含在范围内，{ } 表示端点数值不包含在范围内。关键字要大写。

## 鉴权方案
我们不想让所有人都能访问到我们的kibana，所以需要对请求做一个过滤。
### 1. 用nginx实现用户认证
`ngx_http_auth_basic_module`模块实现在访问时，只有输入正确的用户密码才允许访问web内容。
nginx.conf配置：
```
server {
    listen                      5601;
    server_name                 localhost;

    location / {
        auth_basic "Restricted";
        auth_basic_user_file conf/htpasswd;
    }
}
```
其中`auth_basic_user_file`就是用户密码配置文件的位置，可以使用`htpasswd`，或者使用`openssl`来生成，如：
```sh
printf "username:$(openssl passwd -crypt 123456)\n" >> conf/htpasswd
```
重启nginx访问就需要用户密码才能登陆了。

## 其他配置方案
### 1. Log4j
修改Log4j配置文件，将Log4j的日志输出到SocketAppender：
```xml
<appender name="logstash" class="org.apache.log4j.net.SocketAppender">
    <param name="RemoteHost" value="127.0.0.1"/>
    <param name="Port" value="4560"/>
    <param name="ReconnectionDelay" value="10000"/>
    <layout class="org.apache.log4j.PatternLayout">
        <param name="ConversionPattern"
            value="%d{yyyy-MM-dd HH:mm:ss} [%t] [%-5p] (%C{2},%L) - %m%n" />
    </layout>
</appender>
```
修改logstash配置文件：
```
input {
 log4j {
    mode => "server"
    host => "localhost"
    port => 4560
  }
}
```
详细配置：https://www.elastic.co/guide/en/logstash/current/plugins-inputs-log4j.html

### 2. Logback
#### (1)直接输入到logstash
添加maven依赖，该库用于将logback打印的信息格式化成json数据：
```xml
<dependency>
  <groupId>net.logstash.logback</groupId>
  <artifactId>logstash-logback-encoder</artifactId>
  <version>4.7</version>
</dependency>
```
修改logback配置文件，因为logstash需要对input进行decode，所以需要按照它的格式进行encode：
```
<appender name="stash" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <remoteHost>127.0.0.1</remoteHost>
    <port>4560</port>

    <!-- encoder is required -->
    <encoder class="net.logstash.logback.encoder.LogstashEncoder" />
</appender>
```
修改logstash配置文件：
```
input {
    tcp {
        port => 4560
        codec => json
    }
}
```
详细配置：https://www.elastic.co/guide/en/logstash/current/plugins-inputs-tcp.html

#### (2)先输出到文件，再输入到logstash
修改logback配置文件：
```
<appender name="stdout" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>/home/logs/moneykeeper-api/stdout.log</file>
    <rollingPolicy
            class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <fileNamePattern>/home/logs/moneykeeper-api/stdout.log.%d{yyyy-MM-dd}.gz</fileNamePattern>
    </rollingPolicy>
    <maxHistory>1</maxHistory>
    <append>true</append>
    <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
</appender>
```
修改logstash配置文件：
```
input {

  file {
    path => "/home/logs/moneykeeper-api/stdout.log"
    codec => "json"
  }

}
```
详细配置：https://www.elastic.co/guide/en/logstash/current/plugins-inputs-file.html

### 3. Syslog
修改logstash配置文件：
```
input {
  tcp {
    port => 5000
    type => syslog
  }
  udp {
    port => 5000
    type => syslog
  }
}
filter {
  if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
      add_field => [ "received_at", "%{@timestamp}" ]
      add_field => [ "received_from", "%{host}" ]
    }
    date {
      match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    }
  }
}
```
详细配置：https://www.elastic.co/guide/en/logstash/current/plugins-inputs-syslog.html

### 4. Redis
修改logstash配置文件：
```
input {
    redis {
        host => localhost
        port => 6379
        db => 0
        key => logstash
        codec => json
    }
}
```
详细配置：https://www.elastic.co/guide/en/logstash/current/plugins-inputs-redis.html
