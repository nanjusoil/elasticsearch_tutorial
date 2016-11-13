# 使用 Elasticsearch 实现博客站内搜索

一直以来，为了优化本博客站内搜索效果和速度，我使用 bing 的 `site:` 站内搜索做为数据源，在服务端获取、解析、处理并缓存搜索结果，直接输出 HTML。这个方案唯一的问题是时效性难以保证，尽管我可以在发布和修改文章时主动告诉 bing，但它什么时候更新索引则完全不受我控制。

本着不折腾就浑身不自在的原则，我最终还是使用 [Elasticsearch](https://www.elastic.co/products/elasticsearch) 搭建了自己的搜索服务。Elasticsearch 是一个基于 Lucene 构建的开源、分布式、RESTful 搜索引擎，很多大公司都在用，程序员的好伙伴 Github 的搜索也用的是它。本文记录我使用 Elasticsearch 搭建站内搜索的过程，目前支持中文分词、同义词、标题匹配优先、近期文章优先等常见策略，请「[点击这里](https://imququ.com/search.html)」体验。<!--more-->

### 安装 Elasticsearch

部署 Elasticsearch 最简单的方法是使用 [Elasticsearch Dockerfile](https://hub.docker.com/_/elasticsearch/)。为了更彻底地折腾，我没有使用 Docker，好在手动安装过程也不复杂，下面简单介绍下。

我的虚拟机和线上环境都是 Ubuntu 14.04.4 LTS，Elasticsearch 用的是最新版。一切开始之前，先要检查机器上是否装有 java 环境，如果没有可以通过以下命令安装：

```shell
sudo apt-get install openjdk-7-jre-headless
```

下载 Elasticsearch 2.3.0 压缩包并解压：

```shell
wget -c https://download.elasticsearch.org/elasticsearch/release/org/elasticsearch/distribution/zip/elasticsearch/2.3.0/elasticsearch-2.3.0.zip
unzip elasticsearch-2.3.0.zip
```

我将解压得到的 `elasticsearch-2.3.0` 目录重命名为 `~/es_root`（名称及位置没有限制，可以将它挪到你认为合适的任何位置）。Elasticsearch 无需安装，直接可以运行（注意：不能用 root 帐号运行）：

```shell
cd ~/es_root/bin/
chmod a+x elasticsearch
./elasticsearch
```

如果屏幕上没有打印错误信息，说明 Elasticsearch 服务已经成功启动。新建一个终端，用 curl 验证下：

```shell
curl -XGET http://127.0.0.1:9200/?pretty

{
  "name" : "Melissa Gold",
  "cluster_name" : "elasticsearch",
  "version" : {
    "number" : "2.3.0",
    "build_hash" : "8371be8d5fe5df7fb9c0516c474d77b9feddd888",
    "build_timestamp" : "2016-03-29T07:54:48Z",
    "build_snapshot" : false,
    "lucene_version" : "5.5.0"
  },
  "tagline" : "You Know, for Search"
}
```

如果看到以上信息，说明一切正常，否则请根据屏幕上的错误信息查找原因。尽管 Elasticsearch 本身是用 java 写的，但它对外可以通过 RESTful 接口交互，十分方便。

默认情况下 Elasticsearch 的 RESTful 服务只有本机才能访问，也就是说无法从主机访问虚拟机中的服务。为了方便调试，可以修改 `~/es_root/config/elasticsearch.yml` 文件，加入以下两行：

```yaml
network.bind_host: "0.0.0.0"
network.publish_host: _non_loopback:ipv4_
```

**但线上环境切忌不要这样配置**，否则任何人都可以通过这个接口修改你的数据。

### 安装 IK Analysis

Elasticsearch 自带的分词器会粗暴地把每个汉字直接分开，没有根据词库来分词。为了处理中文搜索，还需要安装中文分词插件。我使用的是 [elasticsearch-analysis-ik](https://github.com/medcl/elasticsearch-analysis-ik/)，支持自定义词库。

首先，下载与 Elasticsearch 匹配的 elasticsearch-analysis-ik 插件：

```shell
wget -c https://github.com/medcl/elasticsearch-analysis-ik/archive/v1.9.0.zip
unzip v1.9.0.zip
```

解压后，进入插件源码目录编译：

```shell
sudo apt-get install maven
cd elasticsearch-analysis-ik-1.9.0
mvn package
```

如果一切顺利，在 `target/releases/` 目录下可以找到编好的文件。将其解压并拷到 `~/es_root` 对应目录：

```shell
mkdir -p ~/es_root/plugins/ik/
unzip target/releases/elasticsearch-analysis-ik-1.9.0.zip -d ~/es_root/plugins/ik/
```

elasticsearch-analysis-ik 的配置文件在 `~/es_root/plugins/ik/config/ik/` 目录，很多都是词表，直接用文本编辑器打开就可以修改，改完记得保存为 utf-8 格式。

现在再启动 Elasticsearch 服务，如果看到类似下面这样的信息，说明 IK Analysis 插件已经装好了：
 
```
plugins [analysis-ik]
```

### 配置同义词

Elasticsearch 自带一个名为 synonym 的同义词 filter。为了能让 IK 和 synonym 同时工作，我们需要定义新的 analyzer，用 IK 做 tokenizer，synonym 做 filter。听上去很复杂，实际上要做的只是加一段配置。

打开 `~/es_root/config/elasticsearch.yml` 文件，加入以下配置：

```yaml
index:
  analysis:
    analyzer:
      ik_syno:
          type: custom
          tokenizer: ik_max_word
          filter: [my_synonym_filter]
      ik_syno_smart:
          type: custom
          tokenizer: ik_smart
          filter: [my_synonym_filter]
    filter:
      my_synonym_filter:
          type: synonym
          synonyms_path: analysis/synonym.txt
```

以上配置定义了 ik\_syno 和 ik\_syno\_smart 这两个新的 analyzer，分别对应 IK 的 ik\_max\_word 和 ik\_smart 两种分词策略。根据 IK 的文档，二者区别如下：

* ik\_max\_word：会将文本做最细粒度的拆分，例如「中华人民共和国国歌」会被拆分为「中华人民共和国、中华人民、中华、华人、人民共和国、人民、人、民、共和国、共和、和、国国、国歌」，会穷尽各种可能的组合；
* ik\_smart：会将文本做最粗粒度的拆分，例如「中华人民共和国国歌」会被拆分为「中华人民共和国、国歌」；

ik\_syno 和 ik\_syno\_smart 都会使用 synonym filter 实现同义词转换。为了方便后续测试，建议创建 `~/es_root/config/analysis/synonym.txt` 文件，输入一些同义词并存为 utf-8 格式。例如：

```
ua,user-agent,userAgent
js,javascript
谷歌=>google
```

### 使用 JavaScript API

通过前面的示例，我们知道通过 curl 或者 Chrome 的 Postman 扩展能轻松地与 Elasticsearch 服务交互。为了更好与已有系统集成，我们还可以使用 Elasticsearch Client。Elasticsearch Client 只是将 RESTful 接口包装了一层，常见语言都有对应的实现（[查看官方 Client](https://www.elastic.co/guide/en/elasticsearch/client/index.html)），自己写一套也不难。

我的博客系统是 Node.js 写的，在项目里直接 `npm install elasticsearch --save` 就可以安装 Elasticsearch 的 Node.js 包。

无论进行什么操作，首先都需要实例化 Elasticsearch Client 对象：

```js
var elasticsearch = require('elasticsearch');

var client = new elasticsearch.Client({
	host: '10.211.55.23:9200', //服务 IP 和端口
	log: 'trace' //输出详细的调试信息
});
```

然后就可以调用 client 对象提供的各种方法了，client 对象拥有大量方法，请查看[官方文档](https://www.elastic.co/guide/en/elasticsearch/client/javascript-api/current/api-reference.html)。这个库支持两种调用方式：callback 和 promise：

```js
//callback
client.info({}, function(err, data) {
	if(!err) {
		console.log('result:', data);
	} else {
		console.log('error:', err);
	}
});

//promise
client.info({}).then(function(data) {
	console.log('result:', data);
}, function(err) {
	console.log('error:', err);
});
```

为了节约篇幅，本文后续贴出的代码都采用 promise 写法，并且省略 then 函数。

### 全文搜索

到现在为止，所有准备工作都已经完成，马上就要大功告成了。在进行下一步之前，先简单介绍一下 Elasticsearch 几个名词：

Elasticsearch 集群可以包含多个索引（Index），每个索引可以包含多个类型（Type），每个类型可以包含多个文档（Document），每个文档可以包含多个字段（Field）。以下是 MySQL 和 Elasticsearch 的术语类比图，帮助理解：

| MySQL | Elasticsearch |
| ----- | ------------- |
| Database | Index |
| Table | Type |
| Row | Document |
| Column | Field |
| Schema | Mappping |
| Index | Everything Indexed by default |
| SQL | Query DSL |

就像使用 MySQL 必须指定 Database 一样，要使用 Elasticsearch 首先需要创建 Index：

```js
client.indices.create({index : 'test'});
```

这样就创建了一个名为 `test` 的 Index。Type 不用单独创建，在创建 Mapping 时指定就可以。Mapping 用来定义 Document 中每个字段的类型、所使用的 analyzer、是否索引等属性，非常关键。创建 Mapping 的代码示例如下：

```js
client.indices.putMapping({
	index : 'test',
	type : 'article',
	body : {
		article: {
			properties: {
				title: {
					type: 'string',
					term_vector: 'with_positions_offsets',
					analyzer: 'ik_syno',
					search_analyzer: 'ik_syno',
				},
				content: {
					type: 'string',
					term_vector: 'with_positions_offsets',
					analyzer: 'ik_syno',
					search_analyzer: 'ik_syno',
				},
				slug: {
					type: 'string',
				},
				tags: {
					type: 'string',
					index : 'not_analyzed',
				},
				update_date: {
					type : 'date',
					index : 'not_analyzed',
				}
			}
		}
	}
});
```

以上代码为 test 索引下的 article 类型指定了字段特征：`title` 和 `content` 字段使用 ik_syno 做为 analyzer，说明它使用 ik\_max\_word 做为分词，并且应用 synonym 同义词策略；`slug`、`tags` 和 `update_date` 字段都没有指定 analyzer，说明他们使用默认分词；同时 `tags` 和 `update_date` 字段不会被分词。

接着，写入测试数据并索引：

```js
client.index({
	index : 'test',
	type : 'article',
	id : '100',
	body : {
		title : '什么是 JS？',
		slug :'what-is-js',
		tags : ['JS', 'JavaScript', 'TEST'],
		content : 'JS 是 JavaScript 的缩写！',
		update_date : '2015-12-15T13:05:55Z',
	}
})
```

`id` 参数如果不指定，系统会自动生成一个并返回，后续在更新、删除时都要用到它。至于如何更新、删除，这里就不写了，请自行[查看文档](https://www.elastic.co/guide/en/elasticsearch/client/javascript-api/current/api-reference.html)。

搜一下试试：

```js
client.search({
	index : 'test',
	type : 'article',
	q : 'JS',
});
```

没有问题，可以搜出来！查询结果数量和具体内容都在 `hits` 字段中：

```
result:
{"took":50,"timed_out":false,"_shards":{"total":5,"successful":5,"failed":0},"hits":{"total":1,"max_score":0.076713204,"hits":[{"_index":"test","_type":"article","_id":"100","_score":0.076713204,"_source":{"title":"什么是 JS？","slug":"what-is-js","tags":["JS","JavaScript","TEST"],"content":"JS 是 JavaScript 的缩写！","update_date":"2015-12-15T13:05:55Z"}}]}}
```

如果要实现更复杂的查询策略该怎么办？那就要请出前面表格中与 SQL 对应的 [Query DSL](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html) 了。例如以下是本博客站内搜索所使用的 Query DSL：

```js
{
	index : 'test',
	type : 'article',
	from : start,
	body : {
		query : { 
			dis_max : { 
				queries : [
					{
						match : {
							title : { 
								query : keyword, 
								minimum_should_match : '50%',
								boost : 4,
							}
						} 
					}, {
						match : {
							content : { 
								query : keyword, 
								minimum_should_match : '75%',
								boost : 4,
							}
						} 
					}, {
						match : {
							tags : { 
								query : keyword, 
								minimum_should_match : '100%',
								boost : 2,
							}
						} 
					}, {
						match : {
							slug : { 
								query : keyword, 
								minimum_should_match : '100%',
								boost : 1,
							}
						} 
					}
				],
				tie_breaker : 0.3
			}
		},
		highlight : {
	        pre_tags : ['<b>'],
	        post_tags : ['</b>'],
			fields : {
				title : {},
				content : {},
			}
		}
	}
}
```

`from` 参数指定从开始跳过多少条结果，用来实现分页。这份复杂的 Query DSL 搜出来的结果如下：

```
result:
{"took":108,"timed_out":false,"_shards":{"total":5,"successful":5,"failed":0},"hits":{"total":1,"max_score":0.29921508,"hits":[{"_index":"test","_type":"article","_id":"100","_score":0.29921508,"_source":{"title":"什么是 JS？","slug":"what-is-js","tags":["JS","JavaScript","TEST"],"content":"JS 是 JavaScript 的缩写！","update_date":"2015-12-15T13:05:55Z"},"highlight":{"content":["<b>JS</b> 是 <b>JavaScript</b> 的缩写！"],"title":["什么是 <b>JS</b>？"]}}]}}
```

可以看到，同义词策略和关键词高亮功能都正常。跑通 Elasticsearch 基本流程，剩余工作就是导入更多数据、配置更多词表和尝试不同策略了，略过不写。

我接触 Elasticsearch 一共才几小时，我的出发点也很简单，只是为了给博客加上站内搜索，故本文既不全面也不深入，甚至还包含各种错误，仅供参考。Elasticsearch 功能十分强大和复杂，远远不是花几个小时就能玩明白的。最后推荐「[Elasticsearch 权威指南（中文版）](http://es.xiaoleilu.com/)」这本书，非常细致和全面，我对 Elasticsearch 仅有的一点了解都来自于这本书和官方文档。

原文链接：[https://imququ.com/post/elasticsearch.html](https://imququ.com/post/elasticsearch.html)，[前往原文评论 »](https://imququ.com/post/elasticsearch.html#comments)
