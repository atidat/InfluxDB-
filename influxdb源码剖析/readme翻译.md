## InfluxDB
### 简介
InfluxDB是个开源的时序平台，定义了**存储**、**查询**数据、用于**数据仓库技术ETL**或**监控和更改**数据的后台处理、**dashboard**及其他**数据可视化**的能力的API接口。

主线分支的代码编译的二进制文件，同时包括**后台数据处理**和**UI**的能力。

这些InfluxDB 客户端库https://docs.influxdata.com/influxdb/v2.1/api-guide/client-libraries/ 兼容最新的版本。

如果你在找1.x的release版本，那么每个次版本都有像master-1.x一样的分支，这个分支都会包含下个1.x版本的代码。master-1.x分支https://github.com/influxdata/influxdb/tree/master-1.x，InfluxDB 1.x Go客户端代码https://github.com/influxdata/influxdb1-client


### 安装
InfluxDB官方提供了docker镜像，debian包，rpm包和tar包。同时，也提供了influx的cli。

如果对源码编译感兴趣，可以参考https://github.com/influxdata/influxdb/blob/master/CONTRIBUTING.md#building-from-source


### 开始
全量的开始使用文档https://docs.influxdata.com/influxdb/v2.1/get-started/

在写/读数据或使用各种API接口前，都需要创建用户user，认证credentials，组织organization和桶bucket。
InfluxDB中的所有概念都基于organization。API接口的设计都是基于多租户的。bucket表示存储的时序数据。这些都是1.x版本数据库和保留策略的同义词。

运行InfluxDB最简单的方式就是访问http://localhost:8086
同时，也可以通过cli安装InfluxDB
```shell
$ bin/$(uname -s|tr '[:upper:] [:lower:]')/influx setup
Welcome to InfluxDB 2.1!
Please type your primary username: matry
Please type your password:
Please type your password again:
Please type your primary organization name.: InfluxData
Please type your primary bucket name.: telegraf
Please type your retention period in hours.
Or press ENTER for infinite.: 72

You have entered:
  Username: 		matry
  Organization: 	InfluxData
  Bucket: 			telegraf
  Retention Peroid: 72 hrs
Confirm: (y/n): y

UserID					Username		Organization	Bucket
033a3f2c5ccaa000        marty           InfluxData      Telegraf
Your token has been stored in /Users/marty/.influxdbv2/credentials
```

非交互式安装命令如下：
```shell
$ bin/$(uname -s | tr '[:upper:]' '[:lower:]')/influx setup \
--username marty \
--password F1uxKapacit0r85 \
--org InfluxData \
--bucket telegraf \
--retention 168 \
--token where-were-going-we-dont-need-roads \
--force
```

安装完成后，配置文件就生成了。该配置文件可以让你不用每次和influxdb交互都需要校验认证信息了。可以通过influx config命令展示和管理这些配置文件。
```shell
$ bin/$(uname -s | tr '[:upper:]' '[:lower:]')/influx config
Active	Name	URL					Org
*		default	http://localhost:8086	 InfluxData
```

### 写数据

写数据到bucket telegraf（隶属于organization InfluxData）的measurement m中，并标记标签tag v=2：

```shell
$ bin/$(uname -s | tr '[:upper:] '[:lower:]')/influx write --bucket telegraf --precision s "m v=2 $(date +%s)"
```
含义：写数据到bucket为telegraf的measurement m中，时间精度为秒，写入的数据标记为v=2，并绑定当前时间戳（因为通过命令行，所以可以不用指定organization和token）

执行上述的point写入，也可以通过下述命令实现
```shell
curl --header "Authorization: Token $(bin/$(uname -s | tr '[:upper:]' '[:lower:]')/influx auth list --json | jq -r '.[0].token')" --data-raw "m v=2 $(data +%s)" "http://localhost:8086/api/v2/write?org=InfluxData&bucket=telegraf&precision=s"
```
注：jq是Linux下的json cli

现在查询下刚才的写入
```shell
$ bin/$(uname -s | tr '[:upper:]' '[:lower:]')/influx query 'from(bucket:"telegraf") |>range(start:-1h)'
Result: _result
Table: keys: [_start, _stop, _filed, _measurement]
_start: time	_stop:time	_field:string	_measurement:string		_time:time
---------------------------------------------------
2019-12-30T22:19:39.043918000Z  2019-12-30T23:19:39.043918000Z                       v                       m  2019-12-30T23:17:02.000000000Z                             2
```

使用-r，--raw选项可以获得flux的原始响应。
influx write写命令的--format csv选项，可以导出csv数据格式，这种方式主要应用于数据从实例A到实例B的迁移。


### Flux脚本

Flux（之前叫做IFQL）是个开源函数数据脚本语言，用于查询、分析和操纵数据。Flux支持多种数据类型，包括

1. 时序数据库（比如InfluxDB）

2. 关系型数据库（比如MySQLhe1PostgreSQL）

3. CSV

Flux源码https://github.com/influxdata/flux

Flux文档https://docs.influxdata.com/flux/ 

Fluxppthttps://speakerdeck.com/pauldix/flux-number-fluxlang-a-new-time-series-data-scripting-language

### 项目贡献 && CI && 代码静态分析
不翻译