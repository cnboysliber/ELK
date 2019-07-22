
目前线上的ELK一律使用6.5.0版本
此文档基于6.5.0版本而写
### 1. 下载Elasticsearch / Kibana / Logstash
Elasticsearch所有节点都需要
Kibana和Logstash单节点便可
```
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.5.0.rpm
wget https://artifacts.elastic.co/downloads/kibana/kibana-6.5.0-x86_64.rpm
wget https://artifacts.elastic.co/downloads/logstash/logstash-6.5.0.rpm
```
### 2. 安装
Elasticsearch所有节点都需要
Kibana和Logstash单节点便可
```
rpm -i elasticsearch-6.5.0.rpm
rpm -i kibana-6.5.0-x86_64.rpm
rpm -i logstash-6.5.0.rpm
```
### 3. 安装Elasticsearch的ik和pinyin分词插件
```
cd /usr/share/elasticsearch/bin
./elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.5.0/elasticsearch-analysis-ik-6.5.0.zip
./elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-pinyin/releases/download/v6.5.0/elasticsearch-analysis-pinyin-6.5.0.zip
```
### 4. 配置 & 启动Elasticsearch
- #### 首先更改Elasticsearch的java虚拟机heap内存设置
```
vim /etc/elasticsearch/jvm.options
```
将-Xms和-Xmx改成一个统一的值
目前因为ES内容较少开发环境使用1G heap, 测试环境因为机器内存紧张使用256M heap, 生产环境使用4G heap
更改后的值为
开发/测试环境
```
-Xms1g
-Xmx1g
```
生产环境
```
-Xms4g
-Xmx4g
```
- #### 创建Elasticsearch的数据文件夹并更改权限
此处使用/data/elasticsearch_data
```
mkdir -p /data/elasticsearch_data
chown -R elasticsearch:elasticsearch /data/elasticsearch_data
```
- #### 修改Elasticsearch配置文件
```
vim /etc/elasticsearch/elasticsearch.yml
```
Elasticsearch集群名称, 同一Elasticsearch集群内此名称必须一致
```
cluster.name: es-dev
```
节点名称, 同一集群内每个节点必须有不同的名称
```
node.name: node-189-207
```
Elasticsearch数据目录, 需要给elasticsearch用户此目录的读写权限
```
path.data: /data/elasticsearch_data
```
Elasticsearch日志目录, 默认为/var/log/elasticsearch, 可以改为自己需要的, 同理需要读写权限
```
path.logs: /var/log/elasticsearch
```
Elasticsearch监听的地址和端口
network.host是监听的地址, 填写IP或者域名或者某些预定义参数, 比如_site_表示网卡地址.
http.port是监听的端口, 默认9200
```
network.host: _site_
http.port: 9200
```
集群节点发现地址, 去哪里查找集群节点信息. 一般填写集群内的所有master节点
```
discovery.zen.ping.unicast.hosts: ["10.251.*.*", "10.104.*.*", "10.135.*.*"]
```
集群最少可运行master节点数量, 为避免脑裂导致数据不同步应设置为 (master节点数量 / 2) + 1
```
discovery.zen.minimum_master_nodes: 2
```
如果是centos 6, 因为系统过老还需要添加以下设置来跳过Elasticsearch的检查以避免无法启动
```
bootstrap.system_call_filter: false
```
- #### 启动Elasticsearch
Centos 6
```
service elasticsearch start
```
Centos 7
```
systemctl start elasticsearch
```

### 5. 配置 & 启动Kibana
- #### 修改Kibana配置文件
```
vim /etc/kibana/kibana.yml
```
监听的地址
```
server.host: "0.0.0.0"
```
Elasticsearch的地址. 目前Kibana仅支持指定一个Elasticsearch地址, 所以选择一个写上去
```
elasticsearch.url: "http://10.104.*.*:9200"
```
- #### 启动Kibana
Centos 6
```
service Kibana start
```
Centos 7
```
systemctl start Kibana
```

### 6. 配置 & 启动Logstash
- #### 获取logstash用的mysql driver
```
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.47.tar.gz
```
解压得到mysql-connector-java-5.1.47-bin.jar
放到logstash用户可以读取的地方
假设/data/mysql-connector-java-5.1.47-bin.jar
```
chown logstash:logstash /data/mysql-connector-java-5.1.47-bin.jar
```
- #### 修改logstash设置
修改logstash的java虚拟机heap设置
```
vim /etc/logstash/jvm.options
-Xms1g
-Xmx1g
```
修改logstash监控设置, 如果希望在Kibana里面看到logstash状态则需要修改
```
vim /etc/logstash/logstash.yml
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.url: ["http://10.251.*.*:9200", "http://10.104.*.*:9200", "http://10.135.*.*:9200"]
```
修改logstash队列设置
数据量不大时候可以不更改
```
vim /etc/logstash/logstash.yml
pipeline.batch.size: 500
```
- #### 添加logstash的pipline设置
```
vim /etc/logstash/conf.d/t_eva_platform_product.conf
# 内容
input {
    jdbc {
        jdbc_driver_library => "/data/mysql-connector-java-5.1.47-bin.jar"
        jdbc_driver_class => "com.mysql.jdbc.Driver"
        jdbc_connection_string => "jdbc:mysql://10.251.101.163:3307/recycle?autoReconnect=true&useSSL=false&useUnicode=true&characterEncoding=utf-8&zeroDateTimeBehavior=convertToNull"
        lowercase_column_names =>  false
        jdbc_user => "eva"
        jdbc_password => "123456"
        schedule => "*/10 * * * *"
        jdbc_default_timezone => "Asia/Shanghai"
    type => "eva_platform_product"
        statement => "SELECT tepp.Fid, tepp.Fproduct_id, tp.Fkey_word, tp.Fis_upper, tp.Fpic_id, tp.Frecycle_type, tp.Fos_type, tepp.Fplatform_type, tepp.Fvalid, tepp.Fuse_standard_eva, tepp.Fbasic_price, tepp.Fmax_price, tept.Fplatform_name, tp.Fproduct_name, tpbm.Fbrand_id, tpb.Fname AS Fbrand_name, tpcm.Fclass_id, tpc.Fname AS Fclass_name FROM t_eva_platform_product AS tepp LEFT JOIN t_product AS tp ON tepp.Fproduct_id = tp.Fproduct_id LEFT JOIN t_pdt_brand_map AS tpbm ON tepp.Fproduct_id = tpbm.Fproduct_id LEFT JOIN t_pdt_brand AS tpb ON tpb.Fid = tpbm.Fbrand_id LEFT JOIN t_pdt_class_map AS tpcm ON tepp.Fproduct_id = tpcm.Fproduct_id LEFT JOIN t_pdt_class AS tpc ON tpc.Fid = tpcm.Fclass_id LEFT JOIN t_eva_platform_type AS tept ON tepp.Fplatform_type = tept.Fplatform_type WHERE tp.Fis_two = 0 OR tp.Fis_two = 2;"
    }
}
output {
    if [type] == "eva_platform_product"{
        elasticsearch {
            index => "evaluate"
            document_id => "%{Fid}"
            hosts => ["10.251.101.163:9200"]
    }
    }
}
```
```
vim /etc/logstash/conf.d/t_eva_standard_product.conf
# 内容
input {
    jdbc {
        jdbc_driver_library => "/data/mysql-connector-java-5.1.47-bin.jar"
        jdbc_driver_class => "com.mysql.jdbc.Driver"
        jdbc_connection_string => "jdbc:mysql://10.251.*.*:3307/recycle?autoReconnect=true&useSSL=false&useUnicode=true&characterEncoding=utf-8&zeroDateTimeBehavior=convertToNull"
        lowercase_column_names =>  false
        jdbc_user => "eva"
        jdbc_password => "123456"
        schedule => "*/10 * * * *"
        jdbc_default_timezone => "Asia/Shanghai"
	type => "eva_standard_product"
        statement => "SELECT tesp.Fproduct_id, tesp.Fvalid, tp.Fproduct_name, tp.Fkey_word, tp.Fis_upper, tpbm.Fbrand_id, tpb.Fname AS Fbrand_name, tpcm.Fclass_id, tpc.Fname asFclass_name FROM t_eva_standard_product AS tesp LEFT JOIN t_product AS tp ON tesp.Fproduct_id = tp.Fproduct_id LEFT JOIN t_pdt_brand_map AS tpbm ON tesp.Fproduct_id = tpbm.Fproduct_id LEFT JOIN t_pdt_brand AS tpb ON tpb.Fid = tpbm.Fbrand_id LEFT JOIN t_pdt_class_map AS tpcm ON tesp.Fproduct_id = tpcm.Fproduct_id LEFT JOIN t_pdt_class AS tpc ON tpc.Fid = tpcm.Fclass_id WHERE tp.Fis_two = 0 OR tp.Fis_two = 2;"
    }
}
output {
    if [type] == "eva_standard_product"{
    	elasticsearch {
        	index => "evaluate"
        	document_id => "%{Fproduct_id}"
        	hosts => ["10.251.*.*:9200"]
	}
    }
}
```
- #### 启动logstash
Centos 6
```
initctl start logstash
```
Centos 7
```
systemctl start logstash
```

### 7. 查错
Elasricsearch日志位于
```
/var/log/elasticsearch
```
Kibana日志位于
```
/var/log/kibana
```
Logstash日志位于
```
/var/log/logstash
```