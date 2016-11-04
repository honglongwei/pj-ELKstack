# Shield 安装与配置

---

### 一、简介
Shield是Elasticsearch的一个插件，它能够很容易保证你的Elasticsearch集群的安全性。<br>

Shield的功能：<br>
  1.用户认证 <br>
  2.SSL/TLS的加密身份验证<br>
  3.审计<br>

### 二、安装
Shield是需要licese的(30天试用期),若shield不收费。用sheild是最好的，功能强大，而且es官方插件。其次，可以使用search-guard。<br> 

 * 在线安装
```python
   # /usr/share/elasticsearch/bin/plugin install license
   # /usr/share/elasticsearch/bin/plugin install shield
   # /usr/share/elasticsearch/bin/plugin -h //查看帮助
```

 * 离线安装
```python
   # wget https://download.elastic.co/elasticsearch/release/org/elasticsearch/plugin/license/2.4.1/license-2.4.1.zip 
   # wget https://download.elastic.co/elasticsearch/release/org/elasticsearch/plugin/shield/2.4.1/shield-2.4.1.zip 
   # /usr/share/elasticsearch/bin/plugin install file:///root/license-2.4.1.zip
   # /usr/share/elasticsearch/bin/plugin install file:///root/shield-2.4.1.zip
   # /usr/share/elasticsearch/bin/plugin list  //查看是否安装成功
   # ll /usr/share/elasticsearch/plugins/
   # curl -XGET 'http://{ip}:9200/'    => 此时是无法访问的，需要身份验证
   # /usr/share/elasticsearch/bin/shield/esusers useradd es_admin -r admin //新建一个ElasticSearch管理员账户，这里会让您填写新密码
   # curl -XGET -u es_admin:{passwd} 'http://{ip}:9200/' //curl -u es_admin -XGET 'http://localhost:9200/'
```

### 三、在Logstash上配置
```python
在ElasticSearch服务器上，使用esusers创建Logstash用户：bin/shield/esusers useradd logstashserver -r logstash

在Logstash服务器上，修改output模块的配置文件，例如： 
output { 
    elasticsearch { 
        host => "192.168.6.144" 
      # protocol => "http" 
        index => "logstash-%{type}-%{+YYYY.MM.dd}"
        user => "logstashserver" #在这里加上Shield中role为Logstash的用户名 
        password => "woshimima" #别忘了密码 
    } 

# stdout { codec => rubydebug } 
}
```

### 四、配置Kibana
```python
在Elasticsearch上创建kibana的用户： 
bin/shield/esusers useradd kibanaserver -r kibana4_server

修改kibana的配置文件：kibana.yml 找到下面语句

# If your Elasticsearch is protected with basic auth, this is the user credentials
# used by the Kibana server to perform maintence on the kibana_index at statup. Your Kibana
# users will still need to authenticate with Elasticsearch (which is proxied thorugh
# the Kibana server)
1
2
3
4
在其下面添加： 
elasticsearch.username: "kibanaserver"
elasticsearch.password: "123456"

启动kibana；

添加自定义角色，使其下面的用户可以监控特定的日志：比如，A用户只能查看A用户产生的日志

在Elasticsearch安装目录下面找到config文件 
切换目录到：/config/shield; 
找到roles.yml;

添加自定义的角色：

如：

syslog_role: 
  indices: 
    - names: 'logstash-syslog-*'
      privileges: 
        - all

access_role: 
  indices: 
    - names: 'logstash-access-*'
      privileges: 
      - all

在这两个角色下面分别创建用户： 
bin/shield/esusers useradd demo_syslog -r syslog_role 
bin/shield/esusers useradd demo_access -r access_role

将用户添加到kibana4_server组下 
bin/shield/esusers roles demo_syslog -a kibana4_server 
bin/shield/esusers roles demo_access -a kibana4_server

完成； 
这样就可以在kibana界面登录查看用户的日志了，而看不到其他用户的日志；
注意：第一次打开kibana创建索引的时候需要*匹配，才能单个帐号查看，不然会报错
```
