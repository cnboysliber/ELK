目前线上Elasticsearch为单节点
接入Elasticsearch做全文搜索的最好保证有后备机制(比如搜索mysql等)以应对单点故障
开发环境Elasticsearch
```
负载均衡地址
3节点: http://10.251.*.*:9200, http://10.104.*.*:9200, http://10.135.*.*:9200
Kibana: http://*.*.*.*:5601
```
测试环境Elasticsearch
```
负载均衡地址 http://100.119.*.*:9200
3节点: http://10.135.*.*:9200, http://10.135.*.*:9200, http://10.*.*.*:9200
Kibana: http://*.*.*.*:5601
```
线上环境Elasticsearch
```
http://10.104.*.*:9200
Kibana: http://*.*.*.*:5601
```
对ES如何实现全文搜索原理有兴趣的可以参考这篇文章
https://blog.csdn.net/forfuture1978/article/details/4711308
### 0. 准备工作
- #### 名词解释
	- `Index`
	在Elasticsearch中, 数据分`Index`存储, 类似mysql的表
	一般在一个`Index`中只存储结构相同的数据
	结构不同的数据一般另开一个`Index`
	- `Field`
	Elasticsearch中的一个字段, 类似mysql中的列
	- `type`
	Elasticsearch中`Field`的类型, 类似mysql中的数据类型(varchar, tinyint等)
	Elasticsearch所有的数据类型可以参见官方文档https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html
	- `type`中的`text`和`keyword`
	在Elasticsearch中, 针对文字字段(String), 有两种数据类型, `text`和`keyword`
	`keyword`表示直接以原文形式存下来, 不进行任何分析处理, 同理搜索也只能使用精确匹配或者速度可能还不如mysql的模糊搜索
	`text`表示在存储时候会首先使用预定义的`分词器`对原文进行`分词`处理, 这也是Elasticsearch可以做到快速全文检索的关键
	- `分词`和`分词器`
	`分词`是指针对文字的预处理行为, 比如“中华人民共和国国歌”拆分为“中华人民共和国,中华人民,中华,华人,人民共和国,人民,人,民,共和国,共和,和,国国,国歌”
	然后使用拆分结果的任何一个关键字搜索结果中都会包含这条数据
	`分词器`就是`分词`时候使用的插件
	Elasticsearch有自带的简单`分词器`, 但是只支持英文并且只做简单的数据处理
	针对中文分词一般需要指定第三方的`分词器`
	我们目前安装的第三方`分词器`有IK和pinyin
	- `mapping`
	类似mysql的表结构定义
	指定`Index`中的每个`Field`是什么`type`以及他们使用什么`分词器`
- #### ik和pinyin分词器解释
	- ik
	ik是一个针对中文的分词器
	有两种分词方案, ik_max_word和ik_smart
	ik_max_word: 会将文本做最细粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,中华人民,中华,华人,人民共和国,人民,人,民,共和国,共和,和,国国,国歌”，会穷尽各种可能的组合；
	ik_smart: 会做最粗粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,国歌”。
	具体可以参考官方说明https://github.com/medcl/elasticsearch-analysis-ik
	- pinyin
	顾名思义, 拼音分词器
	比如“刘德华”会处理成为可以搜索“liu, liude, lhd, 刘dh”等等等等
	具体可以参考官方说明https://github.com/medcl/elasticsearch-analysis-pinyin
- #### Index基本信息
提供Index名便可
- #### 确定Index中哪些Field需要全文搜索
需要指定哪些字段需要加入分词器的分析, 以便达到全文搜索的目的

### 1. 开始使用
- #### 创建Index
可以Kibana的Dev Tools里面的console或者curl或者任意东西发送http请求
Kibana console
```
PUT index_name
```
curl
```
curl -X PUT "http://10.104.*.*:9200/index_name"
```
- #### 设置mapping
a. 假设有一个叫book的Index, 有一个Field叫book_name, 需要使用ik_max_word进行分词/搜索
```
PUT book/_mapping/_doc
{
  "properties": {
    "book_name": {
      "type": "text",
      "analyzer": "ik_max_word",
      "search_analyzer": "ik_max_word"
    }
  }
}
```
b. 假设有一个叫book的Index, 有一个Field叫book_name, 需要同时支持ik_max_word和pinyin进行分词/搜索
```
PUT book/_mapping/_doc
{
  "properties": {
    "book_name": {
      "type": "text",
      "fields": {
        "pinyin": {
          # pinyin分词相关设置, 具体设置参见官方文档
        }
      },
      "analyzer": "ik_max_word",
      "search_analyzer": "ik_max_word"
    }
  }
}
```
这个mapping除了指定了book_name以外, 还对book_name指定了一个子field, “pinyin”
在数据写入时候, 除了把数据使用ik_max_word写入book_name外, 还会把数据使用pinyin分析写入book_name.pinyin
- #### 数据写入
可以直接在程序内调用elasticsearch api写入数据
或者使用脚本/logstash定期从mysql等其他数据源同步写入数据
logstash使用参见Elasticsearch部署文档http://wiki.huishoubao.net/index.php?s=/151&page_id=4595

### 2. 开始搜索
index定义完并且有数据以后, 便可以开始搜索了
搜索book下所有book_name包含“园林”的内容
```
GET book/_search
{
  "query": {
    "match": {
      "book_name": "园林"
    }
  }
}
```
Elasticsearch会针对搜索结果打分排序, 并按照他认为的关联度从高到低排序返回
其他组合搜索方式自行参考Elasricsearch官方搜索文档或者各语言的Elasticsearch包说明