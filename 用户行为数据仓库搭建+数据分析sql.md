[TOC]



# 第一章  数仓分层概念

## 1 项目数仓分层详细表介绍



![数据仓库分层](https://github.com/xzt1995/Data-Warehouse/blob/master/img/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E6%95%B0%E4%BB%93%E6%90%AD%E5%BB%BA/%E6%95%B0%E6%8D%AE%E4%BB%93%E5%BA%93%E5%88%86%E5%B1%82.png)



不同公司对于数仓的命名可能有所区别，但具体的理念都是相通的，数据层级越高，数据量越少。



![ods,dwd](https://github.com/xzt1995/Data-Warehouse/blob/master/img/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E6%95%B0%E4%BB%93%E6%90%AD%E5%BB%BA/ods%2Cdwd.png)



![dws.ads](https://github.com/xzt1995/Data-Warehouse/blob/master/img/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E6%95%B0%E4%BB%93%E6%90%AD%E5%BB%BA/dws.ads.png)





​	以上就是我们这个项目所涉及到的表明细，大部分电商项目都有以上的表，可以计算出大部分场景下我们需要的需求。本章我们会详细介绍如何创建上述的数据仓库。



## 2 数仓命名规范

Ø  ODS层命名为ods

Ø  DWD层命名为dwd

Ø  DWS层命名为dws

Ø  ADS层命名为ads

Ø  临时表数据库命名为xxx_tmp

Ø  备份数据数据库命名为xxx_bak





# 第三章 数仓搭建之ODS层

## 3.1 创建数据库

1）创建gmall数据库

```
hive (default)> create database gmall;
```

说明：如果数据库存在且有数据，需要强制删除时执行：drop database gmall cascade;

2）使用gmall数据库

```
hive (default)> use gmall;
```

## 3.2 ODS层

原始数据层，存放原始数据，直接加载原始日志、数据，数据保持原貌不做处理。

### 3.2.1 创建启动日志表ods_start_log



![ods_start_log](https://github.com/xzt1995/Data-Warehouse/blob/master/img/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E6%95%B0%E4%BB%93%E6%90%AD%E5%BB%BA/ods_start_log.png)





1）创建输入数据是lzo输出是text，支持json解析的分区表

```sql
hive (gmall)> 
drop table if exists ods_start_log;
CREATE EXTERNAL TABLE ods_start_log (`line` string)
PARTITIONED BY (`dt` string)
STORED AS
  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
  OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION '/warehouse/gmall/ods/ods_start_log';

```

**我们储存在HDFS的元数据是用lzo格式压缩的，所以我们要将inputformat设置成支持lzo压缩格式。**

说明Hive的LZO压缩：https://cwiki.apache.org/confluence/display/Hive/LanguageManual+LZO

2）加载数据

```sql
hive (gmall)> 

load data inpath '/origin_data/gmall/log/topic_start/2019-02-10' into table gmall.ods_start_log partition(dt='2019-02-10');

```

注意：时间格式都配置成YYYY-MM-DD格式，这是Hive默认支持的时间格式

*3）*查看是否加载成功

```sql
hive (gmall)> select * from ods_start_log limit 2;
```



### 3.2.2 创建事件日志表ods_event_log



![ods_event_log](https://github.com/xzt1995/Data-Warehouse/blob/master/img/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E6%95%B0%E4%BB%93%E6%90%AD%E5%BB%BA/ods_event_log.png)

1）创建输入数据是lzo输出是text，支持json解析的分区表

```sql
hive (gmall)> 

drop table if exists ods_event_log;

CREATE EXTERNAL TABLE ods_event_log(`line` string)

PARTITIONED BY (`dt` string)

STORED AS

  INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'

 OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'

LOCATION '/warehouse/gmall/ods/ods_event_log';

```



2）加载数据

```sql
hive (gmall)> 

load data inpath '/origin_data/gmall/log/topic_event/2019-02-10' into table gmall.ods_event_log partition(dt='2019-02-10');

```

注意：时间格式都配置成YYYY-MM-DD格式，这是Hive默认支持的时间格式

3）查看是否加载成功

```sql
hive (gmall)> select * from ods_event_log limit 2;
```



### 3.2.3 Shell中单引号和双引号区别



1）在/home/xzt/bin创建一个test.sh文件



在文件中添加如下内容

```shell
#!/bin/bash

do_date=$1

echo '$do_date'

echo "$do_date"

echo "'$do_date'"

echo '"$do_date"'

echo "===日志日期为 $do_date==="

echo `date`
```

2）查看执行结果

```shell
[xzt@hadoop102 bin]$ test.sh 2019-02-10

$do_date

2019-02-10

'2019-02-10'

"$do_date"

2019年 05月 02日 星期四 21:02:08 CST
```

3）总结：

（1）单引号不取变量值

（2）双引号取变量值

（3）反引号`，执行引号中命令

（4）双引号内部嵌套单引号，取出变量值

（5）单引号内部嵌套双引号，不取出变量值

### 3.2.4 ODS层加载数据脚本

1）在hadoop102的/home/xzt/bin目录下创建脚本

```
[xzt@hadoop102 bin]$ vim ods_log.sh
```

​        在脚本中编写如下内容

```shell
#!/bin/bash

# 定义变量方便修改
APP=gmall
hive=/opt/module/hive/bin/hive

# 如果是输入的日期按照取输入日期；如果没输入日期取当前时间的前一天
if [ -n "$1" ] ;then
   do_date=$1
else 
   do_date=`date -d "-1 day" +%F`
fi 

echo "===日志日期为 $do_date==="
sql="
load data inpath '/origin_data/gmall/log/topic_start/$do_date' into table "$APP".ods_start_log partition(dt='$do_date');

load data inpath '/origin_data/gmall/log/topic_event/$do_date' into table "$APP".ods_event_log partition(dt='$do_date');
"

$hive -e "$sql"

```

说明1：

[ -n 变量值 ] 判断变量的值，是否为空

-- 变量的值，非空，返回true

-- 变量的值，为空，返回false

2）增加脚本执行权限

```
[xzt@hadoop102 bin]$ chmod 777 ods_log.sh
```

3）脚本使用

```
[xzt@hadoop102 module]$ ods_log.sh 2019-02-11
```

4）查看导入数据

```shell

hive (gmall)> 

select * from ods_start_log wheredt='2019-02-11' limit 2;

select * from ods_event_log wheredt='2019-02-11' limit 2;

```

<http://hadoop102:50070/explorer.html#/warehouse/gmall/ods/ods_event_log>

查看一下HDFS上是否有对应数据。



![ods,hdfs](https://github.com/xzt1995/Data-Warehouse/blob/master/img/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E6%95%B0%E4%BB%93%E6%90%AD%E5%BB%BA/ods%2Chdfs.png)



5）脚本执行时间

企业开发中一般在每日凌晨30分~1点



# 第4章 数仓搭建之DWD层

对ODS层数据进行清洗（去除空值，脏数据，超过极限范围的数据，行式存储改为列存储，改压缩格式）。

## 4.1 DWD层启动表数据解析

### 4.1.1 创建启动表

1）启动表数据示例

```json
{
	"action": "1",
	"ar": "MX",
	"ba": "Sumsung",
	"detail": "433",
	"en": "start",
	"entry": "5",
	"extend1": "",
	"g": "9U0I40S0@gmail.com",
	"hw": "1080*1920",
	"l": "es",
	"la": "5.6",
	"ln": "-63.3",
	"loading_time": "7",
	"md": "sumsung-7",
	"mid": "2",
	"nw": "4G",
	"open_ad_type": "1",
	"os": "8.1.2",
	"sr": "A",
	"sv": "V2.0.6",
	"t": "1549704162964",
	"uid": "2",
	"vc": "9",
	"vn": "1.2.7"
}
```



2）建表语句

```sql
hive (gmall)> 
drop table if exists dwd_start_log;
CREATE EXTERNAL TABLE dwd_start_log(
`mid_id` string,
`user_id` string, 
`version_code` string, 
`version_name` string, 
`lang` string, 
`source` string, 
`os` string, 
`area` string, 
`model` string,
`brand` string, 
`sdk_version` string, 
`gmail` string, 
`height_width` string,  
`app_time` string,
`network` string, 
`lng` string, 
`lat` string, 
`entry` string, 
`open_ad_type` string, 
`action` string, 
`loading_time` string, 
`detail` string, 
`extend1` string
)
PARTITIONED BY (dt string)
location '/warehouse/gmall/dwd/dwd_start_log/';

```

**extend1是我们预留的字段，防止产品经理”出尔反尔“。**:smile:

### 4.1.2 向启动表导入数据

​	由于我们在ODS层将数据存成了一个line字段，但字段本身又是一个json对象，因此我们要将json的对应key value 转换成DWD表中的字段，这里我们使用的是 get_json_object()函数，将line字段传入，并将对应的key传入就能得到对应的value值。比如

```
  get_json_object(line,'$.mid') = "2"
```



```sql
hive (gmall)> 
insert overwrite table dwd_start_log
PARTITION (dt='2019-02-10')
select 
    get_json_object(line,'$.mid') mid_id,
    get_json_object(line,'$.uid') user_id,
    get_json_object(line,'$.vc') version_code,
    get_json_object(line,'$.vn') version_name,
    get_json_object(line,'$.l') lang,
    get_json_object(line,'$.sr') source,
    get_json_object(line,'$.os') os,
    get_json_object(line,'$.ar') area,
    get_json_object(line,'$.md') model,
    get_json_object(line,'$.ba') brand,
    get_json_object(line,'$.sv') sdk_version,
    get_json_object(line,'$.g') gmail,
    get_json_object(line,'$.hw') height_width,
    get_json_object(line,'$.t') app_time,
    get_json_object(line,'$.nw') network,
    get_json_object(line,'$.ln') lng,
    get_json_object(line,'$.la') lat,
    get_json_object(line,'$.entry') entry,
    get_json_object(line,'$.open_ad_type') open_ad_type,
    get_json_object(line,'$.action') action,
    get_json_object(line,'$.loading_time') loading_time,
    get_json_object(line,'$.detail') detail,
    get_json_object(line,'$.extend1') extend1
from ods_start_log 
where dt='2019-02-10';

```

3）测试

```
hive (gmall)> select * from dwd_start_log limit 2;
```

### 4.1.3 DWD层启动表加载数据脚本

1）在hadoop102的/home/xzt/bin目录下创建脚本

```
[xzt@hadoop102 bin]$ vim dwd_start_log.sh
```

​        在脚本中编写如下内容

```shell
#!/bin/bash

# 定义变量方便修改
APP=gmall
hive=/opt/module/hive/bin/hive

# 如果是输入的日期按照取输入日期；如果没输入日期取当前时间的前一天
if [ -n "$1" ] ;then
	do_date=$1
else 
	do_date=`date -d "-1 day" +%F`  
fi 

sql="
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table "$APP".dwd_start_log
PARTITION (dt='$do_date')
select 
    get_json_object(line,'$.mid') mid_id,
    get_json_object(line,'$.uid') user_id,
    get_json_object(line,'$.vc') version_code,
    get_json_object(line,'$.vn') version_name,
    get_json_object(line,'$.l') lang,
    get_json_object(line,'$.sr') source,
    get_json_object(line,'$.os') os,
    get_json_object(line,'$.ar') area,
    get_json_object(line,'$.md') model,
    get_json_object(line,'$.ba') brand,
    get_json_object(line,'$.sv') sdk_version,
    get_json_object(line,'$.g') gmail,
    get_json_object(line,'$.hw') height_width,
    get_json_object(line,'$.t') app_time,
    get_json_object(line,'$.nw') network,
    get_json_object(line,'$.ln') lng,
    get_json_object(line,'$.la') lat,
    get_json_object(line,'$.entry') entry,
    get_json_object(line,'$.open_ad_type') open_ad_type,
    get_json_object(line,'$.action') action,
    get_json_object(line,'$.loading_time') loading_time,
    get_json_object(line,'$.detail') detail,
    get_json_object(line,'$.extend1') extend1
from "$APP".ods_start_log 
where dt='$do_date';
"
$hive -e "$sql"
```

2）增加脚本执行权限

```
[xzt@hadoop102 bin]$ chmod 777 dwd_start_log.sh
```

3）脚本使用

```
[xzt@hadoop102 bin]$ dwd_start_log.sh 2019-02-11
```

4）查询导入结果

```
hive (gmall)> 
select * from dwd_start_log where dt='2019-02-11' limit 2;
```

5）脚本执行时间

企业开发中一般在每日凌晨30分~1点



## 4.2 DWD层事件表数据解析

### 4.2.1 创建基础明细表

明细表用于存储ODS层原始表转换过来的明细数据。



![dwd_base_event_log](https://github.com/xzt1995/Data-Warehouse/blob/master/img/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E6%95%B0%E4%BB%93%E6%90%AD%E5%BB%BA/dwd_base_event_log.png)



1）创建事件日志基础明细表

```sql
hive (gmall)> 
drop table if exists dwd_base_event_log;
CREATE EXTERNAL TABLE dwd_base_event_log(
`mid_id` string,
`user_id` string, 
`version_code` string, 
`version_name` string, 
`lang` string, 
`source` string, 
`os` string, 
`area` string, 
`model` string,
`brand` string, 
`sdk_version` string, 
`gmail` string, 
`height_width` string, 
`app_time` string, 
`network` string, 
`lng` string, 
`lat` string, 
`event_name` string, 
`event_json` string, 
`server_time` string)
PARTITIONED BY (`dt` string)
stored as parquet
location '/warehouse/gmall/dwd/dwd_base_event_log/';

```

2）说明：其中event_name和event_json用来对应事件名和整个事件。这个地方将原始日志1对多的形式拆分出来了。操作的时候我们需要将原始日志展平，需要用到UDF和UDTF。

### 4.2.2 自定义UDF函数（解析公共字段）

![自定义UDF](https://github.com/xzt1995/Data-Warehouse/blob/master/img/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E6%95%B0%E4%BB%93%E6%90%AD%E5%BB%BA/%E8%87%AA%E5%AE%9A%E4%B9%89UDF.png)





### 4.2.3 自定义UDTF函数（解析具体事件字段）



![自定义UDTF](https://github.com/xzt1995/Data-Warehouse/blob/master/img/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E6%95%B0%E4%BB%93%E6%90%AD%E5%BB%BA/%E8%87%AA%E5%AE%9A%E4%B9%89UDTF.png)



**我将jar包和工程放在了jars/hive/hivefunction下面，可以用idea打开查看一下代码。**

1）将hivefunction-1.0-SNAPSHOT上传到hadoop102的/opt/module/hive/

2）将jar包添加到Hive的classpath

```
hive (gmall)> add jar /opt/module/hive/hivefunction-1.0-SNAPSHOT.jar;
```

3）创建临时函数与开发好的java class关联

```
hive (gmall)> 
create temporary function base_analizer as 'com.xzt.udf.BaseFieldUDF';
create temporary function flat_analizer as 'com.xzt.udtf.EventJsonUDTF';

```

### 4.2.4 解析事件日志基础明细表



1）解析事件日志基础明细表

```sql
hive (gmall)> 
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table dwd_base_event_log 
PARTITION (dt='2019-02-10')
select
mid_id,
user_id,
version_code,
version_name,
lang,
source,
os,
area,
model,
brand,
sdk_version,
gmail,
height_width,
app_time,
network,
lng,
lat,
event_name,
event_json,
server_time
from
(
select
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[0]   as mid_id,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[1]   as user_id,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[2]   as version_code,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[3]   as version_name,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[4]   as lang,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[5]   as source,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[6]   as os,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[7]   as area,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[8]   as model,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[9]   as brand,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[10]   as sdk_version,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[11]  as gmail,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[12]  as height_width,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[13]  as app_time,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[14]  as network,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[15]  as lng,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[16]  as lat,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[17]  as ops,
split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[18]  as server_time
from ods_event_log where dt='2019-02-10'  and base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la')<>'' 
) sdk_log lateral view flat_analizer(ops) tmp_k as event_name, event_json;

```



sql分析：

```sql
base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la')<>'' 
```

这个是判空操作

```sql
lateral view flat_analizer(ops) tmp_k as event_name, event_json;
```

这个是用我们创建的udtf函数来对sdk_log这张临时表进行炸裂操作，将ops炸裂成event_name,event_json两个字段。

这里的ops字段在udf函数操作后是一个ett事件的json数组。

**因为是炸裂函数，所以这边假设事件数组长度为2，那么炸裂后生成的数据应该有两行。**

```json
[{
	"ett": "1541146624055",
	"en": "display",
	"kv": {
		"copyright": "ESPN",
		"content_provider": "CNN",
		"extend2": "5",
		"goodsid": "n4195",
		"action": "2",
		"extend1": "2",
		"place": "3",
		"showtype": "2",
		"category": "72",
		"newstype": "5"
	}
}, {
	"ett": "1541213331817",
	"en": "loading",
	"kv": {
		"extend2": "",
		"loading_time": "15",
		"action": "3",
		"extend1": "",
		"type1": "",
		"type": "3",
		"loading_way": "1"
	}
}, {
	"ett": "1541126195645",
	"en": "ad",
	"kv": {
		"entry": "3",
		"show_style": "0",
		"action": "2",
		"detail": "325",
		"source": "4",
		"behavior": "2",
		"content": "1",
		"newstype": "5"
	}
}, {
	"ett": "1541202678812",
	"en": "notification",
	"kv": {
		"ap_time": "1541184614380",
		"action": "3",
		"type": "4",
		"content": ""
	}
}, {
	"ett": "1541194686688",
	"en": "active_background",
	"kv": {
		"active_source": "3"
	}
}]
```

2）测试

```
hive (gmall)> select * from dwd_base_event_loglimit 2;
```

### 4.2.5 DWD层数据解析脚本



1）在hadoop102的/home/xzt/bin目录下创建脚本

```
[xzt@hadoop102 bin]$ vim dwd_base_log.sh
```

​        在脚本中编写如下内容

```shell
#!/bin/bash

# 定义变量方便修改
APP=gmall
hive=/opt/module/hive/bin/hive

# 如果是输入的日期按照取输入日期；如果没输入日期取当前时间的前一天
if [ -n "$1" ] ;then
	do_date=$1
else 
	do_date=`date -d "-1 day" +%F`  
fi 

sql="
	add jar /opt/module/hive/hivefunction-1.0-SNAPSHOT.jar;

	create temporary function base_analizer as 'com.xzt.udf.BaseFieldUDF';
	create temporary function flat_analizer as 'com.xzt.udtf.EventJsonUDTF';

 	set hive.exec.dynamic.partition.mode=nonstrict;

	insert overwrite table "$APP".dwd_base_event_log 
	PARTITION (dt='$do_date')
	select
	mid_id,
	user_id,
	version_code,
	version_name,
	lang,
	source ,
	os ,
	area ,
	model ,
	brand ,
	sdk_version ,
	gmail ,
	height_width ,
	network ,
	lng ,
	lat ,
	app_time ,
	event_name , 
	event_json , 
	server_time  
	from
	(
	select
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[0]   as mid_id,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[1]   as user_id,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[2]   as version_code,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[3]   as version_name,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[4]   as lang,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[5]   as source,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[6]   as os,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[7]   as area,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[8]   as model,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[9]   as brand,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[10]   as sdk_version,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[11]  as gmail,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[12]  as height_width,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[13]  as app_time,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[14]  as network,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[15]  as lng,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[16]  as lat,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[17]  as ops,
	split(base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la'),'\t')[18]  as server_time
	from "$APP".ods_event_log where dt='$do_date'  and base_analizer(line,'mid,uid,vc,vn,l,sr,os,ar,md,ba,sv,g,hw,t,nw,ln,la')<>'' 
	) sdk_log lateral view flat_analizer(ops) tmp_k as event_name, event_json;
"

$hive -e "$sql"

```

2）增加脚本执行权限

```
[xzt@hadoop102 bin]$ chmod 777 dwd_base_log.sh
```

3）脚本使用

```
[xzt@hadoop102 module]$ dwd_base_log.sh 2019-02-11
```

4）查询导入结果

```
hive (gmall)> 

select * from dwd_base_event_log where dt='2019-02-11' limit 2;

```

5）脚本执行时间

企业开发中一般在每日凌晨30分~1点



## 4.3 DWD层事件表获取

![解析基础明细表](https://github.com/xzt1995/Data-Warehouse/blob/master/img/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E6%95%B0%E4%BB%93%E6%90%AD%E5%BB%BA/%E8%A7%A3%E6%9E%90%E5%9F%BA%E7%A1%80%E6%98%8E%E7%BB%86%E8%A1%A8.png)

### 4.3.1 商品点击表



![商品点击表解析](https://github.com/xzt1995/Data-Warehouse/blob/master/img/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E6%95%B0%E4%BB%93%E6%90%AD%E5%BB%BA/%E5%95%86%E5%93%81%E7%82%B9%E5%87%BB%E8%A1%A8%E8%A7%A3%E6%9E%90.png)





​	**这边这么多的表实际上就是对我们刚才创建的dwd_base_event_log 中的event_name,event_json字段进行进一步的解析，根据用户操作类型的不同，进一步的分类。形成DWD层的事件表，可以说dwd_base_event_log这张表就是我们从ODS -  DWD 中的桥梁。**

举个例子：

我们查询一下dwd_base_event_log这张表

```sql
hive (gmall)> select * from dwd_base_event_log where dt  = '2019-02-11' limit 2 ;
```

返回数据

```json
0	0	5	1.1.8	en	Q	8.2.3	MX	sumsung-14	Sumsung	V2.1.6	6XPW45ZY@gmail.com	640*960	3G	-119.6	14.2	1549765409389	(公共字段)
newsdetail (事件名称event_name)	
{"ett":"1549753820640","en":"newsdetail","kv":{"entry":"1","goodsid":"0","news_staytime":"12","loading_time":"36","action":"2","showtype":"4","category":"55","type1":""}}	（事件数组event_json）
1549814411789	（服务器时间）
2019-02-11 （分区号）


0	0	5	1.1.8	en	Q	8.2.3	MX	sumsung-14	Sumsung	V2.1.6	6XPW45ZY@gmail.com	640*960	3G	-119.6	14.2	1549765409389	(公共字段)
loading	(事件名称event_name)
{"ett":"1549733916074","en":"loading","kv":{"extend2":"","loading_time":"18","action":"3","extend1":"","type":"1","type1":"","loading_way":"2"}}	 （事件数组event_json）
1549814411789	（服务器时间）
2019-02-11   （分区号）

```

这里的不同事件就对应着用户对APP的不同操作

第一个 newsdetail 是 用户浏览商品详情

第二个 loading 是 用户查看商品列表

所以我们就要对用户不同的操作来建立不同的表格，这边需要注意的是，不同的操作，所生成的字段是不同的，我们在第一章的数据生成部分有详细介绍，这边我就不再多说。每个公司的日志格式都不一样，但基本的字段都是差不多的。

比如 newsdetail  有8个字段

| 标签            | 含义                                       |
| ------------- | ---------------------------------------- |
| entry         | 页面入口来源：应用首页=1、push=2、详情页相关推荐=3           |
| action        | 动作：开始加载=1，加载成功=2（pv），加载失败=3, 退出页面=4      |
| goodsid       | 商品ID（服务端下发的ID）                           |
| show_style    | 商品样式：0、无图、1、一张大图、2、两张图、3、三张小图、4、一张小图、5、一张大图两张小图 |
| news_staytime | 页面停留时长：从商品开始加载时开始计算，到用户关闭页面所用的时间。若中途用跳转到其它页面了，则暂停计时，待回到详情页时恢复计时。或中途划出的时间超过10分钟，则本次计时作废，不上报本次数据。如未加载成功退出，则报空。 |
| loading_time  | 加载时长：计算页面开始加载到接口返回数据的时间 （开始加载报0，加载成功或加载失败才上报时间） |
| type1         | 加载失败码：把加载失败状态码报回来（报空为加载成功，没有失败）          |
| category      | 分类ID（服务端定义的分类ID）                         |

loading 有7个字段

| 标签           | 含义                                       |
| ------------ | ---------------------------------------- |
| action       | 动作：开始加载=1，加载成功=2，加载失败=3                  |
| loading_time | 加载时长：计算下拉开始到接口返回数据的时间，（开始加载报0，加载成功或加载失败才上报时间） |
| loading_way  | 加载类型：1-读取缓存，2-从接口拉新数据  （加载成功才上报加载类型）     |
| extend1      | 扩展字段  Extend1                            |
| extend2      | 扩展字段  Extend2                            |
| type         | 加载类型：自动加载=1，用户下拽加载=2，底部加载=3（底部条触发点击底部提示条/点击返回顶部加载） |
| type1        | 加载失败码：把加载失败状态码报回来（报空为加载成功，没有失败）          |

这一步就将我们从ODS层中杂乱无章的数据转化成了我们所需要的数据，有助于我们分析后续的指标。



1）建表语句

```sql
hive (gmall)> 
drop table if exists dwd_display_log;
CREATE EXTERNAL TABLE dwd_display_log(
`mid_id` string,
`user_id` string,
`version_code` string,
`version_name` string,
`lang` string,
`source` string,
`os` string,
`area` string,
`model` string,
`brand` string,
`sdk_version` string,
`gmail` string,
`height_width` string,
`app_time` string,
`network` string,
`lng` string,
`lat` string,
`action` string,
`goodsid` string,
`place` string,
`extend1` string,
`category` string,
`server_time` string
)
PARTITIONED BY (dt string)
location '/warehouse/gmall/dwd/dwd_display_log/';

```



2）导入数据

```sql
hive (gmall)> 
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table dwd_display_log
PARTITION (dt='2019-02-10')
select 
mid_id,
user_id,
version_code,
version_name,
lang,
source,
os,
area,
model,
brand,
sdk_version,
gmail,
height_width,
app_time,
network,
lng,
lat,
get_json_object(event_json,'$.kv.action') action,
get_json_object(event_json,'$.kv.goodsid') goodsid,
get_json_object(event_json,'$.kv.place') place,
get_json_object(event_json,'$.kv.extend1') extend1,
get_json_object(event_json,'$.kv.category') category,
server_time
from dwd_base_event_log 
where dt='2019-02-10' and event_name='display';

```

3）测试

```
hive (gmall)> select * from dwd_display_log limit 2;
```

### 4.3.2 商品详情页表

1）建表语句

```sql
hive (gmall)> 
drop table if exists dwd_newsdetail_log;
CREATE EXTERNAL TABLE dwd_newsdetail_log(
`mid_id` string,
`user_id` string, 
`version_code` string, 
`version_name` string, 
`lang` string, 
`source` string, 
`os` string, 
`area` string, 
`model` string,
`brand` string, 
`sdk_version` string, 
`gmail` string, 
`height_width` string, 
`app_time` string,  
`network` string, 
`lng` string, 
`lat` string, 
`entry` string,
`action` string,
`goodsid` string,
`showtype` string,
`news_staytime` string,
`loading_time` string,
`type1` string,
`category` string,
`server_time` string)
PARTITIONED BY (dt string)
location '/warehouse/gmall/dwd/dwd_newsdetail_log/';

```

2）导入数据

```sql
hive (gmall)> 
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table dwd_newsdetail_log
PARTITION (dt='2019-02-10')
select 
mid_id,
user_id,
version_code,
version_name,
lang,
source,
os,
area,
model,
brand,
sdk_version,
gmail,
height_width,
app_time,
network,
lng,
lat,
get_json_object(event_json,'$.kv.entry') entry,
get_json_object(event_json,'$.kv.action') action,
get_json_object(event_json,'$.kv.goodsid') goodsid,
get_json_object(event_json,'$.kv.showtype') showtype,
get_json_object(event_json,'$.kv.news_staytime') news_staytime,
get_json_object(event_json,'$.kv.loading_time') loading_time,
get_json_object(event_json,'$.kv.type1') type1,
get_json_object(event_json,'$.kv.category') category,
server_time
from dwd_base_event_log
where dt='2019-02-10' and event_name='newsdetail';

```

3）测试

```
hive (gmall)> select * from dwd_newsdetail_log limit 2;
```

### 4.3.3 商品列表页表



1）建表语句



```sql
hive (gmall)> 
drop table if exists dwd_loading_log;
CREATE EXTERNAL TABLE dwd_loading_log(
`mid_id` string,
`user_id` string, 
`version_code` string, 
`version_name` string, 
`lang` string, 
`source` string, 
`os` string, 
`area` string, 
`model` string,
`brand` string, 
`sdk_version` string, 
`gmail` string,
`height_width` string,  
`app_time` string,
`network` string, 
`lng` string, 
`lat` string, 
`action` string,
`loading_time` string,
`loading_way` string,
`extend1` string,
`extend2` string,
`type` string,
`type1` string,
`server_time` string)
PARTITIONED BY (dt string)
location '/warehouse/gmall/dwd/dwd_loading_log/';

```

2）导入数据

```sql
hive (gmall)> 
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table dwd_loading_log
PARTITION (dt='2019-02-10')
select 
mid_id,
user_id,
version_code,
version_name,
lang,
source,
os,
area,
model,
brand,
sdk_version,
gmail,
height_width,
app_time,
network,
lng,
lat,
get_json_object(event_json,'$.kv.action') action,
get_json_object(event_json,'$.kv.loading_time') loading_time,
get_json_object(event_json,'$.kv.loading_way') loading_way,
get_json_object(event_json,'$.kv.extend1') extend1,
get_json_object(event_json,'$.kv.extend2') extend2,
get_json_object(event_json,'$.kv.type') type,
get_json_object(event_json,'$.kv.type1') type1,
server_time
from dwd_base_event_log
where dt='2019-02-10' and event_name='loading';

```

3）测试

```
hive (gmall)> select * from dwd_loading_log limit 2;
```

### 4.3.4 广告表



1）建表语句

```sql
hive (gmall)> 
drop table if exists dwd_ad_log;
CREATE EXTERNAL TABLE dwd_ad_log(
`mid_id` string,
`user_id` string, 
`version_code` string, 
`version_name` string, 
`lang` string, 
`source` string, 
`os` string, 
`area` string, 
`model` string,
`brand` string, 
`sdk_version` string, 
`gmail` string, 
`height_width` string,  
`app_time` string,
`network` string, 
`lng` string, 
`lat` string, 
`entry` string,
`action` string,
`content` string,
`detail` string,
`ad_source` string,
`behavior` string,
`newstype` string,
`show_style` string,
`server_time` string)
PARTITIONED BY (dt string)
location '/warehouse/gmall/dwd/dwd_ad_log/';

```

2）导入数据

```sql
hive (gmall)> 
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table dwd_ad_log
PARTITION (dt='2019-02-10')
select 
mid_id,
user_id,
version_code,
version_name,
lang,
source,
os,
area,
model,
brand,
sdk_version,
gmail,
height_width,
app_time,
network,
lng,
lat,
get_json_object(event_json,'$.kv.entry') entry,
get_json_object(event_json,'$.kv.action') action,
get_json_object(event_json,'$.kv.content') content,
get_json_object(event_json,'$.kv.detail') detail,
get_json_object(event_json,'$.kv.source') ad_source,
get_json_object(event_json,'$.kv.behavior') behavior,
get_json_object(event_json,'$.kv.newstype') newstype,
get_json_object(event_json,'$.kv.show_style') show_style,
server_time
from dwd_base_event_log 
where dt='2019-02-10' and event_name='ad';

```

3) 测试

```
hive (gmall)> select * from dwd_ad_log limit 2;
```

### 4.3.5 消息通知表

1）建表语句

```sql
hive (gmall)> 
drop table if exists dwd_notification_log;
CREATE EXTERNAL TABLE dwd_notification_log(
`mid_id` string,
`user_id` string, 
`version_code` string, 
`version_name` string, 
`lang` string,
`source` string, 
`os` string, 
`area` string, 
`model` string,
`brand` string, 
`sdk_version` string, 
`gmail` string, 
`height_width` string,  
`app_time` string,
`network` string, 
`lng` string, 
`lat` string, 
`action` string,
`noti_type` string,
`ap_time` string,
`content` string,
`server_time` string
)
PARTITIONED BY (dt string)
location '/warehouse/gmall/dwd/dwd_notification_log/';

```

2）导入数据

```sql
hive (gmall)> 
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table dwd_notification_log
PARTITION (dt='2019-02-10')
select 
mid_id,
user_id,
version_code,
version_name,
lang,
source,
os,
area,
model,
brand,
sdk_version,
gmail,
height_width,
app_time,
network,
lng,
lat,
get_json_object(event_json,'$.kv.action') action,
get_json_object(event_json,'$.kv.noti_type') noti_type,
get_json_object(event_json,'$.kv.ap_time') ap_time,
get_json_object(event_json,'$.kv.content') content,
server_time
from dwd_base_event_log
where dt='2019-02-10' and event_name='notification';

```

3）测试

```
hive (gmall)> select * from dwd_notification_log limit 2;
```

### 4.3.6 用户前台活跃表

1）建表语句

```sql
hive (gmall)> 
drop table if exists dwd_active_foreground_log;
CREATE EXTERNAL TABLE dwd_active_foreground_log(
`mid_id` string,
`user_id` string,
`version_code` string,
`version_name` string,
`lang` string,
`source` string,
`os` string,
`area` string,
`model` string,
`brand` string,
`sdk_version` string,
`gmail` string,
`height_width` string,
`app_time` string,
`network` string,
`lng` string,
`lat` string,
`push_id` string,
`access` string,
`server_time` string)
PARTITIONED BY (dt string)
location '/warehouse/gmall/dwd/dwd_foreground_log/';

```

2）导入数据

```sql
hive (gmall)> 
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table dwd_active_foreground_log
PARTITION (dt='2019-02-10')
select 
mid_id,
user_id,
version_code,
version_name,
lang,
source,
os,
area,
model,
brand,
sdk_version,
gmail,
height_width,
app_time,
network,
lng,
lat,
get_json_object(event_json,'$.kv.push_id') push_id,
get_json_object(event_json,'$.kv.access') access,
server_time
from dwd_base_event_log
where dt='2019-02-10' and event_name='active_foreground';

```

3）测试

```
hive (gmall)> select * from dwd_active_foreground_log limit 2;
```

### 4.3.7 用户后台活跃表

1）建表语句

```sql
hive (gmall)> 
drop table if exists dwd_active_background_log;
CREATE EXTERNAL TABLE dwd_active_background_log(
`mid_id` string,
`user_id` string,
`version_code` string,
`version_name` string,
`lang` string,
`source` string,
`os` string,
`area` string,
`model` string,
`brand` string,
`sdk_version` string,
`gmail` string,
 `height_width` string,
`app_time` string,
`network` string,
`lng` string,
`lat` string,
`active_source` string,
`server_time` string
)
PARTITIONED BY (dt string)
location '/warehouse/gmall/dwd/dwd_background_log/';

```

2）导入数据

```sql
hive (gmall)> 
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table dwd_active_background_log
PARTITION (dt='2019-02-10')
select 
mid_id,
user_id,
version_code,
version_name,
lang,
source,
os,
area,
model,
brand,
sdk_version,
gmail,
height_width,
app_time,
network,
lng,
lat,
get_json_object(event_json,'$.kv.active_source') active_source,
server_time
from dwd_base_event_log
where dt='2019-02-10' and event_name='active_background';

```

3）测试

```
hive (gmall)> select * from dwd_active_background_log limit 2;
```

### 4.3.8 评论表

1)建表语句

```sql
hive (gmall)> 
drop table if exists dwd_comment_log;
CREATE EXTERNAL TABLE dwd_comment_log(
`mid_id` string,
`user_id` string,
`version_code` string,
`version_name` string,
`lang` string,
`source` string,
`os` string,
`area` string,
`model` string,
`brand` string,
`sdk_version` string,
`gmail` string,
`height_width` string,
`app_time` string,
`network` string,
`lng` string,
`lat` string,
`comment_id` int,
`userid` int,
`p_comment_id` int, 
`content` string,
`addtime` string,
`other_id` int,
`praise_count` int,
`reply_count` int,
`server_time` string
)
PARTITIONED BY (dt string)
location '/warehouse/gmall/dwd/dwd_comment_log/';

```

2) 导入数据

```sql
hive (gmall)> 
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table dwd_comment_log
PARTITION (dt='2019-02-10')
select 
mid_id,
user_id,
version_code,
version_name,
lang,
source,
os,
area,
model,
brand,
sdk_version,
gmail,
height_width,
app_time,
network,
lng,
lat,
get_json_object(event_json,'$.kv.comment_id') comment_id,
get_json_object(event_json,'$.kv.userid') userid,
get_json_object(event_json,'$.kv.p_comment_id') p_comment_id,
get_json_object(event_json,'$.kv.content') content,
get_json_object(event_json,'$.kv.addtime') addtime,
get_json_object(event_json,'$.kv.other_id') other_id,
get_json_object(event_json,'$.kv.praise_count') praise_count,
get_json_object(event_json,'$.kv.reply_count') reply_count,
server_time
from dwd_base_event_log
where dt='2019-02-10' and event_name='comment';

```

3）测试

```
hive (gmall)> select * from dwd_comment_log limit 2;
```

### 4.3.9 收藏表

1）建表语句

```sql
hive (gmall)> 
drop table if exists dwd_favorites_log;
CREATE EXTERNAL TABLE dwd_favorites_log(
`mid_id` string,
`user_id` string, 
`version_code` string, 
`version_name` string, 
`lang` string, 
`source` string, 
`os` string, 
`area` string, 
`model` string,
`brand` string, 
`sdk_version` string, 
`gmail` string, 
`height_width` string,  
`app_time` string,
`network` string, 
`lng` string, 
`lat` string, 
`id` int, 
`course_id` int, 
`userid` int,
`add_time` string,
`server_time` string
)
PARTITIONED BY (dt string)
location '/warehouse/gmall/dwd/dwd_favorites_log/';


```

2）导入数据

```sql
hive (gmall)> 
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table dwd_favorites_log
PARTITION (dt='2019-02-10')
select 
mid_id,
user_id,
version_code,
version_name,
lang,
source,
os,
area,
model,
brand,
sdk_version,
gmail,
height_width,
app_time,
network,
lng,
lat,
get_json_object(event_json,'$.kv.id') id,
get_json_object(event_json,'$.kv.course_id') course_id,
get_json_object(event_json,'$.kv.userid') userid,
get_json_object(event_json,'$.kv.add_time') add_time,
server_time
from dwd_base_event_log 
where dt='2019-02-10' and event_name='favorites';

```

3）测试

```
hive (gmall)> select * from dwd_favorites_log limit 2;
```

### 4.3.10 点赞表

1）建表语句

```sql
hive (gmall)> 
drop table if exists dwd_praise_log;
CREATE EXTERNAL TABLE dwd_praise_log(
`mid_id` string,
`user_id` string, 
`version_code` string, 
`version_name` string, 
`lang` string, 
`source` string, 
`os` string, 
`area` string, 
`model` string,
`brand` string, 
`sdk_version` string, 
`gmail` string, 
`height_width` string,  
`app_time` string,
`network` string, 
`lng` string, 
`lat` string, 
`id` string, 
`userid` string, 
`target_id` string,
`type` string,
`add_time` string,
`server_time` string
)
PARTITIONED BY (dt string)
location '/warehouse/gmall/dwd/dwd_praise_log/';

```

2) 导入数据

```sql
hive (gmall)> 
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table dwd_praise_log
PARTITION (dt='2019-02-10')
select 
mid_id,
user_id,
version_code,
version_name,
lang,
source,
os,
area,
model,
brand,
sdk_version,
gmail,
height_width,
app_time,
network,
lng,
lat,
get_json_object(event_json,'$.kv.id') id,
get_json_object(event_json,'$.kv.userid') userid,
get_json_object(event_json,'$.kv.target_id') target_id,
get_json_object(event_json,'$.kv.type') type,
get_json_object(event_json,'$.kv.add_time') add_time,
server_time
from dwd_base_event_log
where dt='2019-02-10' and event_name='praise';

```

3) 测试

```
hive (gmall)> select * from dwd_praise_log limit 2;
```

### 4.3.11 错误日志表

1）建表语句

```sql
hive (gmall)> 
drop table if exists dwd_error_log;
CREATE EXTERNAL TABLE dwd_error_log(
`mid_id` string,
`user_id` string, 
`version_code` string, 
`version_name` string, 
`lang` string, 
`source` string, 
`os` string, 
`area` string, 
`model` string,
`brand` string, 
`sdk_version` string, 
`gmail` string, 
`height_width` string,  
`app_time` string,
`network` string, 
`lng` string, 
`lat` string, 
`errorBrief` string, 
`errorDetail` string, 
`server_time` string)
PARTITIONED BY (dt string)
location '/warehouse/gmall/dwd/dwd_error_log/';

```

2) 导入数据

```sql
hive (gmall)> 
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table dwd_error_log
PARTITION (dt='2019-02-10')
select 
mid_id,
user_id,
version_code,
version_name,
lang,
source,
os,
area,
model,
brand,
sdk_version,
gmail,
height_width,
app_time,
network,
lng,
lat,
get_json_object(event_json,'$.kv.errorBrief') errorBrief,
get_json_object(event_json,'$.kv.errorDetail') errorDetail,
server_time
from dwd_base_event_log 
where dt='2019-02-10' and event_name='error';

```

3) 测试

```
hive (gmall)> select * from dwd_error_log limit 2;
```

### 4.3.12 DWD层事件表加载数据脚本

1）在hadoop102的/home/xzt/bin目录下创建脚本

```
[xzt@hadoop102 bin]$ vim dwd_event_log.sh
```

​        在脚本中编写如下内容

```shell
#!/bin/bash

# 定义变量方便修改
APP=gmall
hive=/opt/module/hive/bin/hive

# 如果是输入的日期按照取输入日期；如果没输入日期取当前时间的前一天
if [ -n "$1" ] ;then
	do_date=$1
else 
	do_date=`date -d "-1 day" +%F`  
fi 

sql="
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table "$APP".dwd_display_log
PARTITION (dt='$do_date')
select 
	mid_id,
	user_id,
	version_code,
	version_name,
	lang,
	source,
	os,
	area,
	model,
	brand,
	sdk_version,
	gmail,
	height_width,
	app_time,
	network,
	lng,
	lat,
	get_json_object(event_json,'$.kv.action') action,
	get_json_object(event_json,'$.kv.goodsid') goodsid,
	get_json_object(event_json,'$.kv.place') place,
	get_json_object(event_json,'$.kv.extend1') extend1,
	get_json_object(event_json,'$.kv.category') category,
	server_time
from "$APP".dwd_base_event_log 
where dt='$do_date' and event_name='display';


insert overwrite table "$APP".dwd_newsdetail_log
PARTITION (dt='$do_date')
select 
	mid_id,
	user_id,
	version_code,
	version_name,
	lang,
	source,
	os,
	area,
	model,
	brand,
	sdk_version,
	gmail,
	height_width,
	app_time,
	network,
	lng,
	lat,
	get_json_object(event_json,'$.kv.entry') entry,
	get_json_object(event_json,'$.kv.action') action,
	get_json_object(event_json,'$.kv.goodsid') goodsid,
	get_json_object(event_json,'$.kv.showtype') showtype,
	get_json_object(event_json,'$.kv.news_staytime') news_staytime,
	get_json_object(event_json,'$.kv.loading_time') loading_time,
	get_json_object(event_json,'$.kv.type1') type1,
	get_json_object(event_json,'$.kv.category') category,
	server_time
from "$APP".dwd_base_event_log 
where dt='$do_date' and event_name='newsdetail';


insert overwrite table "$APP".dwd_loading_log
PARTITION (dt='$do_date')
select 
	mid_id,
	user_id,
	version_code,
	version_name,
	lang,
	source,
	os,
	area,
	model,
	brand,
	sdk_version,
	gmail,
	height_width,
	app_time,
	network,
	lng,
	lat,
	get_json_object(event_json,'$.kv.action') action,
	get_json_object(event_json,'$.kv.loading_time') loading_time,
	get_json_object(event_json,'$.kv.loading_way') loading_way,
	get_json_object(event_json,'$.kv.extend1') extend1,
	get_json_object(event_json,'$.kv.extend2') extend2,
	get_json_object(event_json,'$.kv.type') type,
	get_json_object(event_json,'$.kv.type1') type1,
	server_time
from "$APP".dwd_base_event_log 
where dt='$do_date' and event_name='loading';


insert overwrite table "$APP".dwd_ad_log
PARTITION (dt='$do_date')
select 
	mid_id,
	user_id,
	version_code,
	version_name,
	lang,
	source,
	os,
	area,
	model,
	brand,
	sdk_version,
	gmail,
	height_width,
	app_time,
	network,
	lng,
	lat,
	get_json_object(event_json,'$.kv.entry') entry,
	get_json_object(event_json,'$.kv.action') action,
	get_json_object(event_json,'$.kv.content') content,
	get_json_object(event_json,'$.kv.detail') detail,
	get_json_object(event_json,'$.kv.source') ad_source,
	get_json_object(event_json,'$.kv.behavior') behavior,
	get_json_object(event_json,'$.kv.newstype') newstype,
	get_json_object(event_json,'$.kv.show_style') show_style,
	server_time
from "$APP".dwd_base_event_log 
where dt='$do_date' and event_name='ad';


insert overwrite table "$APP".dwd_notification_log
PARTITION (dt='$do_date')
select 
	mid_id,
	user_id,
	version_code,
	version_name,
	lang,
	source,
	os,
	area,
	model,
	brand,
	sdk_version,
	gmail,
	height_width,
	app_time,
	network,
	lng,
	lat,
	get_json_object(event_json,'$.kv.action') action,
	get_json_object(event_json,'$.kv.noti_type') noti_type,
	get_json_object(event_json,'$.kv.ap_time') ap_time,
	get_json_object(event_json,'$.kv.content') content,
	server_time
from "$APP".dwd_base_event_log 
where dt='$do_date' and event_name='notification';


insert overwrite table "$APP".dwd_active_foreground_log
PARTITION (dt='$do_date')
select 
	mid_id,
	user_id,
	version_code,
	version_name,
	lang,
	source,
	os,
	area,
	model,
	brand,
	sdk_version,
	gmail,
	height_width,
	app_time,
	network,
	lng,
	lat,
get_json_object(event_json,'$.kv.push_id') push_id,
get_json_object(event_json,'$.kv.access') access,
	server_time
from "$APP".dwd_base_event_log 
where dt='$do_date' and event_name='active_foreground';


insert overwrite table "$APP".dwd_active_background_log
PARTITION (dt='$do_date')
select 
	mid_id,
	user_id,
	version_code,
	version_name,
	lang,
	source,
	os,
	area,
	model,
	brand,
	sdk_version,
	gmail,
	height_width,
	app_time,
	network,
	lng,
	lat,
	get_json_object(event_json,'$.kv.active_source') active_source,
	server_time
from "$APP".dwd_base_event_log 
where dt='$do_date' and event_name='active_background';


insert overwrite table "$APP".dwd_comment_log
PARTITION (dt='$do_date')
select 
	mid_id,
	user_id,
	version_code,
	version_name,
	lang,
	source,
	os,
	area,
	model,
	brand,
	sdk_version,
	gmail,
	height_width,
	app_time,
	network,
	lng,
	lat,
	get_json_object(event_json,'$.kv.comment_id') comment_id,
	get_json_object(event_json,'$.kv.userid') userid,
	get_json_object(event_json,'$.kv.p_comment_id') p_comment_id,
	get_json_object(event_json,'$.kv.content') content,
	get_json_object(event_json,'$.kv.addtime') addtime,
	get_json_object(event_json,'$.kv.other_id') other_id,
	get_json_object(event_json,'$.kv.praise_count') praise_count,
	get_json_object(event_json,'$.kv.reply_count') reply_count,
	server_time
from "$APP".dwd_base_event_log 
where dt='$do_date' and event_name='comment';


insert overwrite table "$APP".dwd_favorites_log
PARTITION (dt='$do_date')
select 
	mid_id,
	user_id,
	version_code,
	version_name,
	lang,
	source,
	os,
	area,
	model,
	brand,
	sdk_version,
	gmail,
	height_width,
	app_time,
	network,
	lng,
	lat,
	get_json_object(event_json,'$.kv.id') id,
	get_json_object(event_json,'$.kv.course_id') course_id,
	get_json_object(event_json,'$.kv.userid') userid,
	get_json_object(event_json,'$.kv.add_time') add_time,
	server_time
from "$APP".dwd_base_event_log 
where dt='$do_date' and event_name='favorites';


insert overwrite table "$APP".dwd_praise_log
PARTITION (dt='$do_date')
select 
	mid_id,
	user_id,
	version_code,
	version_name,
	lang,
	source,
	os,
	area,
	model,
	brand,
	sdk_version,
	gmail,
	height_width,
	app_time,
	network,
	lng,
	lat,
	get_json_object(event_json,'$.kv.id') id,
	get_json_object(event_json,'$.kv.userid') userid,
	get_json_object(event_json,'$.kv.target_id') target_id,
	get_json_object(event_json,'$.kv.type') type,
	get_json_object(event_json,'$.kv.add_time') add_time,
	server_time
from "$APP".dwd_base_event_log 
where dt='$do_date' and event_name='praise';


insert overwrite table "$APP".dwd_error_log
PARTITION (dt='$do_date')
select 
	mid_id,
	user_id,
	version_code,
	version_name,
	lang,
	source,
	os,
	area,
	model,
	brand,
	sdk_version,
	gmail,
	height_width,
	app_time,
	network,
	lng,
	lat,
	get_json_object(event_json,'$.kv.errorBrief') errorBrief,
	get_json_object(event_json,'$.kv.errorDetail') errorDetail,
	server_time
from "$APP".dwd_base_event_log 
where dt='$do_date' and event_name='error';
"

$hive -e "$sql"

```

2）增加脚本执行权限

```
[xzt@hadoop102 bin]$ chmod 777 dwd_event_log.sh
```

3）脚本使用

```
[xzt@hadoop102 module]$ dwd_event_log.sh 2019-02-11
```

4）查询导入结果

```
hive (gmall)> 
select * from dwd_comment_log where dt='2019-02-11' limit 2;
```

5）脚本执行时间

企业开发中一般在每日凌晨30分~1点



# 第5章 业务知识准备

## 5.1 业务术语

1.     用户

用户以设备为判断标准，在移动统计中，每个独立设备认为是一个独立用户。Android系统根据IMEI号，IOS系统根据OpenUDID来标识一个独立用户，每部手机一个用户。

2.     新增用户

首次联网使用应用的用户。如果一个用户首次打开某APP，那这个用户定义为新增用户；卸载再安装的设备，不会被算作一次新增。新增用户包括日新增用户、周新增用户、月新增用户。

3.     活跃用户

打开应用的用户即为活跃用户，不考虑用户的使用情况。每天一台设备打开多次会被计为一个活跃用户。

4.     周（月）活跃用户

某个自然周（月）内启动过应用的用户，该周（月）内的多次启动只记一个活跃用户。

5.     月活跃率

月活跃用户与截止到该月累计的用户总和之间的比例。

6.     沉默用户

用户仅在安装当天（次日）启动一次，后续时间无再启动行为。该指标可以反映新增用户质量和用户与APP的匹配程度。

7.     版本分布

不同版本的周内各天新增用户数，活跃用户数和启动次数。利于判断APP各个版本之间的优劣和用户行为习惯。

8.     本周回流用户

上周未启动过应用，本周启动了应用的用户。

9.     连续n周活跃用户

连续n周，每周至少启动一次。

10.  忠诚用户

连续活跃5周以上的用户

11.  连续活跃用户

连续2周及以上活跃的用户

12.  近期流失用户

连续n(2<= n <= 4)周没有启动应用的用户。（第n+1周没有启动过）

13.  留存用户

某段时间内的新增用户，经过一段时间后，仍然使用应用的被认作是留存用户；这部分用户占当时新增用户的比例即是留存率。

例如，5月份新增用户200，这200人在6月份启动过应用的有100人，7月份启动过应用的有80人，8月份启动过应用的有50人；则5月份新增用户一个月后的留存率是50%，二个月后的留存率是40%，三个月后的留存率是25%。

14.  用户新鲜度

每天启动应用的新老用户比例，即新增用户数占活跃用户数的比例。

15.  单次使用时长

每次启动使用的时间长度。

16.  日使用时长

累计一天内的使用时间长度。

17.  启动次数计算标准

IOS平台应用退到后台就算一次独立的启动；Android平台我们规定，两次启动之间的间隔小于30秒，被计算一次启动。用户在使用过程中，若因收发短信或接电话等退出应用30秒又再次返回应用中，那这两次行为应该是延续而非独立的，所以可以被算作一次使用行为，即一次启动。业内大多使用30秒这个标准，但用户还是可以自定义此时间间隔。

## 5.2 系统函数

### 5.2.1 collect_set函数

1）创建原数据表

```
hive (gmall)>

drop table if exists stud;

create table stud (name string, area string,course string, score int);

```



2）向原数据表中插入数据

```
hive (gmall)>

insert into table stud values('zhang3','bj','math',88);

insert into table stud values('li4','bj','math',99);

insert into table stud values('wang5','sh','chinese',92);

insert into table stud values('zhao6','sh','chinese',54);

insert into table stud values('tian7','bj','chinese',91);

```

3）查询表中数据

```
hive (gmall)> select * from stud;

stud.name       stud.area       stud.course     stud.score

zhang3 			bj     			math    		88

li4    			bj      		math    		99

wang5 			sh      		chinese 		92

zhao6  			sh      		chinese 		54

tian7  			bj      		chinese 		91

```

4）把同一分组的不同行的数据聚合成一个集合 

```
hive (gmall)> select course,collect_set(area), avg(score) from stud group by course;

chinese ["sh","bj"]     79.0

math    ["bj"]  93.5

```

5） 用下标可以取某一个

```
hive (gmall)> select course,collect_set(area)[0], avg(score) from stud group by course;

chinese sh      79.0

math    bj      93.5

```

### 5.2.2 日期处理函数

1）date_format函数（根据格式整理日期）

```
hive (gmall)> select date_format('2019-02-10','yyyy-MM');

2019-02

```

2）date_add函数（加减日期）

```
hive (gmall)> select date_add('2019-02-10',-1);

2019-02-09

hive (gmall)> select date_add('2019-02-10',1);

2019-02-11

```

3）next_day函数

（1）取当前天的下一个周一

```
hive (gmall)> select next_day('2019-02-12','MO')

2019-02-18

```

说明：星期一到星期日的英文（Monday，Tuesday、Wednesday、Thursday、Friday、Saturday、Sunday）

（2）取当前周的周一

```
hive (gmall)> select date_add(next_day('2019-02-12','MO'),-7);

2019-02-11

```

4）last_day函数（求当月最后一天日期）

```
hive (gmall)> select last_day('2019-02-10');

2019-02-28

```





# 第6章 需求一：用户活跃主题

## 6.1 DWS层

目标：统计当日、当周、当月活动的每个设备明细

### 6.1.1 每日活跃设备明细

1）建表语句

我们这里在dws创建一张表，将原始启动表的通用数据都创建为字段，可以方便我们后续开发别的类似需求时，不需要重复的去创建表。

比如：按手机品牌分类的日活，按渠道号分类的日活等等

```sql
drop table if exists dws_uv_detail_day;
create external table dws_uv_detail_day
(
    `mid_id` string COMMENT '设备唯一标识',
    `user_id` string COMMENT '用户标识', 
    `version_code` string COMMENT '程序版本号', 
    `version_name` string COMMENT '程序版本名', 
    `lang` string COMMENT '系统语言', 
    `source` string COMMENT '渠道号', 
    `os` string COMMENT '安卓系统版本', 
    `area` string COMMENT '区域', 
    `model` string COMMENT '手机型号', 
    `brand` string COMMENT '手机品牌', 
    `sdk_version` string COMMENT 'sdkVersion', 
    `gmail` string COMMENT 'gmail', 
    `height_width` string COMMENT '屏幕宽高',
    `app_time` string COMMENT '客户端日志产生时的时间',
    `network` string COMMENT '网络模式',
    `lng` string COMMENT '经度',
    `lat` string COMMENT '纬度'
)
partitioned by(dt string)
stored as parquet
location '/warehouse/gmall/dws/dws_uv_detail_day'
;

```

2）数据导入

​	以用户单日访问为key进行聚合，如果某个用户在一天中使用了两种操作系统、两个系统版本、多个地区，登录不同账号，只取其中之一

```sql
hive (gmall)>
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table dws_uv_detail_day 
partition(dt='2019-02-10')
select  
    mid_id,
    concat_ws('|', collect_set(user_id)) user_id,
    concat_ws('|', collect_set(version_code)) version_code,
    concat_ws('|', collect_set(version_name)) version_name,
    concat_ws('|', collect_set(lang))lang,
    concat_ws('|', collect_set(source)) source,
    concat_ws('|', collect_set(os)) os,
    concat_ws('|', collect_set(area)) area, 
    concat_ws('|', collect_set(model)) model,
    concat_ws('|', collect_set(brand)) brand,
    concat_ws('|', collect_set(sdk_version)) sdk_version,
    concat_ws('|', collect_set(gmail)) gmail,
    concat_ws('|', collect_set(height_width)) height_width,
    concat_ws('|', collect_set(app_time)) app_time,
    concat_ws('|', collect_set(network)) network,
    concat_ws('|', collect_set(lng)) lng,
    concat_ws('|', collect_set(lat)) lat
from dwd_start_log
where dt='2019-02-10'
group by mid_id;

```

3）查询导入结果

```sql
hive (gmall)> select * from dws_uv_detail_day limit 1;
hive (gmall)> select count(*) from dws_uv_detail_day;
```

### 6.1.2 每周活跃设备明细

根据日用户访问明细，获得周用户访问明细。

1）建表语句

```sql
hive (gmall)>
drop table if exists dws_uv_detail_wk;
create external table dws_uv_detail_wk( 
    `mid_id` string COMMENT '设备唯一标识',
    `user_id` string COMMENT '用户标识', 
    `version_code` string COMMENT '程序版本号', 
    `version_name` string COMMENT '程序版本名', 
    `lang` string COMMENT '系统语言', 
    `source` string COMMENT '渠道号', 
    `os` string COMMENT '安卓系统版本', 
    `area` string COMMENT '区域', 
    `model` string COMMENT '手机型号', 
    `brand` string COMMENT '手机品牌', 
    `sdk_version` string COMMENT 'sdkVersion', 
    `gmail` string COMMENT 'gmail', 
    `height_width` string COMMENT '屏幕宽高',
    `app_time` string COMMENT '客户端日志产生时的时间',
    `network` string COMMENT '网络模式',
    `lng` string COMMENT '经度',
    `lat` string COMMENT '纬度',
    `monday_date` string COMMENT '周一日期',
    `sunday_date` string COMMENT  '周日日期' 
) COMMENT '活跃用户按周明细'
PARTITIONED BY (`wk_dt` string)
stored as parquet
location '/warehouse/gmall/dws/dws_uv_detail_wk/'；

```

2）数据导入

```sql
hive (gmall)>
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table dws_uv_detail_wk partition(wk_dt)
select  
    mid_id,
    concat_ws('|', collect_set(user_id)) user_id,
    concat_ws('|', collect_set(version_code)) version_code,
    concat_ws('|', collect_set(version_name)) version_name,
    concat_ws('|', collect_set(lang)) lang,
    concat_ws('|', collect_set(source)) source,
    concat_ws('|', collect_set(os)) os,
    concat_ws('|', collect_set(area)) area, 
    concat_ws('|', collect_set(model)) model,
    concat_ws('|', collect_set(brand)) brand,
    concat_ws('|', collect_set(sdk_version)) sdk_version,
    concat_ws('|', collect_set(gmail)) gmail,
    concat_ws('|', collect_set(height_width)) height_width,
    concat_ws('|', collect_set(app_time)) app_time,
    concat_ws('|', collect_set(network)) network,
    concat_ws('|', collect_set(lng)) lng,
    concat_ws('|', collect_set(lat)) lat,
    date_add(next_day('2019-02-10','MO'),-7),
    date_add(next_day('2019-02-10','MO'),-1),
    concat(date_add( next_day('2019-02-10','MO'),-7), '_' , date_add(next_day('2019-02-10','MO'),-1) 
)
from dws_uv_detail_day 
where dt>=date_add(next_day('2019-02-10','MO'),-7) and dt<=date_add(next_day('2019-02-10','MO'),-1) 
group by mid_id;
```



### 6.1.3 每月活跃设备明细

1）建表语句

```sql
hive (gmall)>
drop table if exists dws_uv_detail_mn;

create external table dws_uv_detail_mn( 
    `mid_id` string COMMENT '设备唯一标识',
    `user_id` string COMMENT '用户标识', 
    `version_code` string COMMENT '程序版本号', 
    `version_name` string COMMENT '程序版本名', 
    `lang` string COMMENT '系统语言', 
    `source` string COMMENT '渠道号', 
    `os` string COMMENT '安卓系统版本', 
    `area` string COMMENT '区域', 
    `model` string COMMENT '手机型号', 
    `brand` string COMMENT '手机品牌', 
    `sdk_version` string COMMENT 'sdkVersion', 
    `gmail` string COMMENT 'gmail', 
    `height_width` string COMMENT '屏幕宽高',
    `app_time` string COMMENT '客户端日志产生时的时间',
    `network` string COMMENT '网络模式',
    `lng` string COMMENT '经度',
    `lat` string COMMENT '纬度'
) COMMENT '活跃用户按月明细'
PARTITIONED BY (`mn` string)
stored as parquet
location '/warehouse/gmall/dws/dws_uv_detail_mn/'
;

```

2）数据导入

```sql
hive (gmall)>
set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table dws_uv_detail_mn partition(mn)
select  
    mid_id,
    concat_ws('|', collect_set(user_id)) user_id,
    concat_ws('|', collect_set(version_code)) version_code,
    concat_ws('|', collect_set(version_name)) version_name,
    concat_ws('|', collect_set(lang)) lang,
    concat_ws('|', collect_set(source)) source,
    concat_ws('|', collect_set(os)) os,
    concat_ws('|', collect_set(area)) area, 
    concat_ws('|', collect_set(model)) model,
    concat_ws('|', collect_set(brand)) brand,
    concat_ws('|', collect_set(sdk_version)) sdk_version,
    concat_ws('|', collect_set(gmail)) gmail,
    concat_ws('|', collect_set(height_width)) height_width,
    concat_ws('|', collect_set(app_time)) app_time,
    concat_ws('|', collect_set(network)) network,
    concat_ws('|', collect_set(lng)) lng,
    concat_ws('|', collect_set(lat)) lat,
    date_format('2019-02-10','yyyy-MM')
from dws_uv_detail_day
where date_format(dt,'yyyy-MM') = date_format('2019-02-10','yyyy-MM')
group by mid_id;
```



### 6.1.4 DWS层加载数据脚本



1）在hadoop102的/home/xzt/bin目录下创建脚本

```
[xzt@hadoop102 bin]$ vim dws_uv_log.sh
```

在脚本中编写如下内容

```shell
#!/bin/bash

# 定义变量方便修改
APP=gmall
hive=/opt/module/hive/bin/hive

# 如果是输入的日期按照取输入日期；如果没输入日期取当前时间的前一天
if [ -n "$1" ] ;then
	do_date=$1
else 
	do_date=`date -d "-1 day" +%F`  
fi 


sql="
  set hive.exec.dynamic.partition.mode=nonstrict;

  insert overwrite table "$APP".dws_uv_detail_day partition(dt='$do_date')
  select  
    mid_id,
    concat_ws('|', collect_set(user_id)) user_id,
    concat_ws('|', collect_set(version_code)) version_code,
    concat_ws('|', collect_set(version_name)) version_name,
    concat_ws('|', collect_set(lang)) lang,
    concat_ws('|', collect_set(source)) source,
    concat_ws('|', collect_set(os)) os,
    concat_ws('|', collect_set(area)) area, 
    concat_ws('|', collect_set(model)) model,
    concat_ws('|', collect_set(brand)) brand,
    concat_ws('|', collect_set(sdk_version)) sdk_version,
    concat_ws('|', collect_set(gmail)) gmail,
    concat_ws('|', collect_set(height_width)) height_width,
    concat_ws('|', collect_set(app_time)) app_time,
    concat_ws('|', collect_set(network)) network,
    concat_ws('|', collect_set(lng)) lng,
    concat_ws('|', collect_set(lat)) lat
  from "$APP".dwd_start_log
  where dt='$do_date'  
  group by mid_id;


  insert overwrite table "$APP".dws_uv_detail_wk partition(wk_dt)
  select  
    mid_id,
    concat_ws('|', collect_set(user_id)) user_id,
    concat_ws('|', collect_set(version_code)) version_code,
    concat_ws('|', collect_set(version_name)) version_name,
    concat_ws('|', collect_set(lang)) lang,
    concat_ws('|', collect_set(source)) source,
    concat_ws('|', collect_set(os)) os,
    concat_ws('|', collect_set(area)) area, 
    concat_ws('|', collect_set(model)) model,
    concat_ws('|', collect_set(brand)) brand,
    concat_ws('|', collect_set(sdk_version)) sdk_version,
    concat_ws('|', collect_set(gmail)) gmail,
    concat_ws('|', collect_set(height_width)) height_width,
    concat_ws('|', collect_set(app_time)) app_time,
    concat_ws('|', collect_set(network)) network,
    concat_ws('|', collect_set(lng)) lng,
    concat_ws('|', collect_set(lat)) lat,
    date_add(next_day('$do_date','MO'),-7),
    date_add(next_day('$do_date','MO'),-1),
    concat(date_add( next_day('$do_date','MO'),-7), '_' , date_add(next_day('$do_date','MO'),-1) 
  )
  from "$APP".dws_uv_detail_day
  where dt>=date_add(next_day('$do_date','MO'),-7) and dt<=date_add(next_day('$do_date','MO'),-1) 
  group by mid_id; 


  insert overwrite table "$APP".dws_uv_detail_mn partition(mn)
  select
    mid_id,
    concat_ws('|', collect_set(user_id)) user_id,
    concat_ws('|', collect_set(version_code)) version_code,
    concat_ws('|', collect_set(version_name)) version_name,
    concat_ws('|', collect_set(lang))lang,
    concat_ws('|', collect_set(source)) source,
    concat_ws('|', collect_set(os)) os,
    concat_ws('|', collect_set(area)) area, 
    concat_ws('|', collect_set(model)) model,
    concat_ws('|', collect_set(brand)) brand,
    concat_ws('|', collect_set(sdk_version)) sdk_version,
    concat_ws('|', collect_set(gmail)) gmail,
    concat_ws('|', collect_set(height_width)) height_width,
    concat_ws('|', collect_set(app_time)) app_time,
    concat_ws('|', collect_set(network)) network,
    concat_ws('|', collect_set(lng)) lng,
    concat_ws('|', collect_set(lat)) lat,
    date_format('$do_date','yyyy-MM')
  from "$APP".dws_uv_detail_day
  where date_format(dt,'yyyy-MM') = date_format('$do_date','yyyy-MM')   
  group by mid_id;
"

$hive -e "$sql"

```

2）增加脚本执行权限

```
[xzt@hadoop102 bin]$ chmod 777 dws_uv_log.sh
```

3）脚本使用

```
[xzt@hadoop102 module]$ dws_uv_log.sh 2019-02-11
```

4）查询结果

```sql
hive (gmall)> select count(*) from dws_uv_detail_day where dt='2019-02-11';

hive (gmall)> select count(*) from dws_uv_detail_wk;

hive (gmall)> select count(*) from dws_uv_detail_mn ;

```

5）脚本执行时间

企业开发中一般在每日凌晨30分~1点



## 6.2 ADS层

目标：当日、当周、当月活跃设备数

### 6.2.1 活跃设备数

ADS将dws的活跃数据进一步聚合，集中在一张表直观的表现出来，方便后续报表展示使用。

1）建表语句

```sql
hive (gmall)>
drop table if exists ads_uv_count;
create external table ads_uv_count( 
    `dt` string COMMENT '统计日期',
    `day_count` bigint COMMENT '当日用户数量',
    `wk_count`  bigint COMMENT '当周用户数量',
    `mn_count`  bigint COMMENT '当月用户数量',
    `is_weekend` string COMMENT 'Y,N是否是周末,用于得到本周最终结果',
    `is_monthend` string COMMENT 'Y,N是否是月末,用于得到本月最终结果' 
) COMMENT '活跃设备数'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_uv_count/'
;

```

2）导入数据

```sql
hive (gmall)>
insert into table ads_uv_count 
select  
  '2019-02-10' dt,
   daycount.ct,
   wkcount.ct,
   mncount.ct,
   if(date_add(next_day('2019-02-10','MO'),-1)='2019-02-10','Y','N') ,
   if(last_day('2019-02-10')='2019-02-10','Y','N') 
from 
(
   select  
      '2019-02-10' dt,
       count(*) ct
   from dws_uv_detail_day
   where dt='2019-02-10'  
)daycount join 
( 
   select  
     '2019-02-10' dt,
     count (*) ct
   from dws_uv_detail_wk
   where wk_dt=concat(date_add(next_day('2019-02-10','MO'),-7),'_' ,date_add(next_day('2019-02-10','MO'),-1) )
) wkcount on daycount.dt=wkcount.dt
join 
( 
   select  
     '2019-02-10' dt,
     count (*) ct
   from dws_uv_detail_mn
   where mn=date_format('2019-02-10','yyyy-MM')  
)mncount on daycount.dt=mncount.dt
;

```

3）查询导入结果

```sql
hive (gmall)> select * from ads_uv_count ;
```

### 6.2.2 ADS层加载数据脚本

1）在hadoop102的/home/xzt/bin目录下创建脚本

```
[xzt@hadoop102 bin]$ vim ads_uv_log.sh
```

​        在脚本中编写如下内容

```shell
#!/bin/bash

# 定义变量方便修改
APP=gmall
hive=/opt/module/hive/bin/hive

# 如果是输入的日期按照取输入日期；如果没输入日期取当前时间的前一天
if [ -n "$1" ] ;then
	do_date=$1
else 
	do_date=`date -d "-1 day" +%F`  
fi 

sql="
  set hive.exec.dynamic.partition.mode=nonstrict;

insert into table "$APP".ads_uv_count 
select  
  '$do_date' dt,
   daycount.ct,
   wkcount.ct,
   mncount.ct,
   if(date_add(next_day('$do_date','MO'),-1)='$do_date','Y','N') ,
   if(last_day('$do_date')='$do_date','Y','N') 
from 
(
   select  
      '$do_date' dt,
       count(*) ct
   from "$APP".dws_uv_detail_day
   where dt='$do_date'  
)daycount   join 
( 
   select  
     '$do_date' dt,
     count (*) ct
   from "$APP".dws_uv_detail_wk
   where wk_dt=concat(date_add(next_day('$do_date','MO'),-7),'_' ,date_add(next_day('$do_date','MO'),-1) )
)  wkcount  on daycount.dt=wkcount.dt
join 
( 
   select  
     '$do_date' dt,
     count (*) ct
   from "$APP".dws_uv_detail_mn
   where mn=date_format('$do_date','yyyy-MM')  
)mncount on daycount.dt=mncount.dt;
"

$hive -e "$sql"

```

2）增加脚本执行权限

```
[xzt@hadoop102 bin]$ chmod 777 ads_uv_log.sh
```

3）脚本使用

```
[xzt@hadoop102 module]$ ads_uv_log.sh 2019-02-11
```

4）脚本执行时间

企业开发中一般在每日凌晨30分~1点



# 第7章 需求二：用户新增主题

​	首次联网使用应用的用户。如果一个用户首次打开某APP，那这个用户定义为新增用户；卸载再安装的设备，不会被算作一次新增。新增用户包括日新增用户、周新增用户、月新增用户。

## 7.1 DWS层（每日新增设备明细表）

1）建表语句

```sql
hive (gmall)>
drop table if exists dws_new_mid_day;
create external table dws_new_mid_day
(
    `mid_id` string COMMENT '设备唯一标识',
    `user_id` string COMMENT '用户标识', 
    `version_code` string COMMENT '程序版本号', 
    `version_name` string COMMENT '程序版本名', 
    `lang` string COMMENT '系统语言', 
    `source` string COMMENT '渠道号', 
    `os` string COMMENT '安卓系统版本', 
    `area` string COMMENT '区域', 
    `model` string COMMENT '手机型号', 
    `brand` string COMMENT '手机品牌', 
    `sdk_version` string COMMENT 'sdkVersion', 
    `gmail` string COMMENT 'gmail', 
    `height_width` string COMMENT '屏幕宽高',
    `app_time` string COMMENT '客户端日志产生时的时间',
    `network` string COMMENT '网络模式',
    `lng` string COMMENT '经度',
    `lat` string COMMENT '纬度',
    `create_date`  string  comment '创建时间' 
)  COMMENT '每日新增设备信息'
stored as parquet
location '/warehouse/gmall/dws/dws_new_mid_day/';

```

2）导入数据

​	用每日活跃用户表Left Join每日新增设备表，关联的条件是mid_id相等。如果是每日新增的设备，则在每日新增设备表中为null。

```sql
hive (gmall)>
insert into table dws_new_mid_day
select  
    ud.mid_id,
    ud.user_id , 
    ud.version_code , 
    ud.version_name , 
    ud.lang , 
    ud.source, 
    ud.os, 
    ud.area, 
    ud.model, 
    ud.brand, 
    ud.sdk_version, 
    ud.gmail, 
    ud.height_width,
    ud.app_time,
    ud.network,
    ud.lng,
    ud.lat,
    '2019-02-10'
from dws_uv_detail_day ud left join dws_new_mid_day nm on ud.mid_id=nm.mid_id
where ud.dt='2019-02-10' and nm.mid_id is null;

```

3)  需求分析

![每日新增设备分析](https://github.com/xzt1995/Data-Warehouse/blob/master/img/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E6%95%B0%E4%BB%93%E6%90%AD%E5%BB%BA/%E6%AF%8F%E6%97%A5%E6%96%B0%E5%A2%9E%E8%AE%BE%E5%A4%87%E5%88%86%E6%9E%90.png)





4）查询导入数据

```sql
hive (gmall)> select count(*) from dws_new_mid_day ;
```



## 7.2 ADS层（每日新增设备表）

1）建表语句

```sql
hive (gmall)>
drop table if exists ads_new_mid_count;
create external table ads_new_mid_count
(
    `create_date`     string comment '创建时间' ,
    `new_mid_count`   BIGINT comment '新增设备数量' 
)  COMMENT '每日新增设备信息数量'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_new_mid_count/';

```

2）导入数据

```sql
hive (gmall)>
insert into table ads_new_mid_count 
select
create_date,
count(*)
from dws_new_mid_day
where create_date='2019-02-10'
group by create_date;

```

3）查询导入数据

```sql
hive (gmall)> select * from ads_new_mid_count;
```

# 第8章 需求三：用户留存主题

## 8.1 需求目标

### 8.1.1 用户留存概念



![用户留存概念](https://github.com/xzt1995/Data-Warehouse/blob/master/img/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E6%95%B0%E4%BB%93%E6%90%AD%E5%BB%BA/%E7%94%A8%E6%88%B7%E7%95%99%E5%AD%98%E6%A6%82%E5%BF%B5.png)



### 8.1.2 需求描述

![用户留存率分析](https://github.com/xzt1995/Data-Warehouse/blob/master/img/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E6%95%B0%E4%BB%93%E6%90%AD%E5%BB%BA/%E7%94%A8%E6%88%B7%E7%95%99%E5%AD%98%E7%8E%87%E5%88%86%E6%9E%90.png)



## 8.2 DWS层

### 8.2.1 DWS层（每日留存用户明细表）



1）建表语句

```sql
hive (gmall)>
  drop table if exists dws_user_retention_day;
  create external table dws_user_retention_day 
  (
      `mid_id` string COMMENT '设备唯一标识',
      `user_id` string COMMENT '用户标识', 
      `version_code` string COMMENT '程序版本号', 
      `version_name` string COMMENT '程序版本名', 
  `lang` string COMMENT '系统语言', 
  `source` string COMMENT '渠道号', 
  `os` string COMMENT '安卓系统版本', 
  `area` string COMMENT '区域', 
  `model` string COMMENT '手机型号', 
  `brand` string COMMENT '手机品牌', 
  `sdk_version` string COMMENT 'sdkVersion', 
  `gmail` string COMMENT 'gmail', 
  `height_width` string COMMENT '屏幕宽高',
  `app_time` string COMMENT '客户端日志产生时的时间',
  `network` string COMMENT '网络模式',
  `lng` string COMMENT '经度',
  `lat` string COMMENT '纬度',
     `create_date`    string  comment '设备新增时间',
     `retention_day`  int comment '截止当前日期留存天数'
  )  COMMENT '每日用户留存情况'
  PARTITIONED BY (`dt` string)
  stored as parquet
  location '/warehouse/gmall/dws/dws_user_retention_day/'
  ;

```

2）导入数据（每天计算前1天的新用户访问留存明细）

```sql
hive (gmall)>
insert overwrite table dws_user_retention_day
partition(dt="2019-02-11")
select  
    nm.mid_id,
    nm.user_id , 
    nm.version_code , 
    nm.version_name , 
    nm.lang , 
    nm.source, 
    nm.os, 
    nm.area, 
    nm.model, 
    nm.brand, 
    nm.sdk_version, 
    nm.gmail, 
    nm.height_width,
    nm.app_time,
    nm.network,
    nm.lng,
    nm.lat,
nm.create_date,
1 retention_day 
from dws_uv_detail_day ud join dws_new_mid_day nm   on ud.mid_id =nm.mid_id 
where ud.dt='2019-02-11' and nm.create_date=date_add('2019-02-11',-1);

```

3）查询导入数据（每天计算前1天的新用户访问留存明细）

```sql
hive (gmall)> select count(*) from dws_user_retention_day;
```

### 8.2.2 DWS层（1,2,3,n天留存用户明细表）

1）导入数据（每天计算前1,2,3，n天的新用户访问留存明细）

```sql
hive (gmall)>
insert overwrite table dws_user_retention_day
partition(dt="2019-02-11")
select
    nm.mid_id,
    nm.user_id,
    nm.version_code,
    nm.version_name,
    nm.lang,
    nm.source,
    nm.os,
    nm.area,
    nm.model,
    nm.brand,
    nm.sdk_version,
    nm.gmail,
    nm.height_width,
    nm.app_time,
    nm.network,
    nm.lng,
    nm.lat,
    nm.create_date,
    1 retention_day 
from dws_uv_detail_day ud join dws_new_mid_day nm  on ud.mid_id =nm.mid_id 
where ud.dt='2019-02-11' and nm.create_date=date_add('2019-02-11',-1)

union all
select  
    nm.mid_id,
    nm.user_id , 
    nm.version_code , 
    nm.version_name , 
    nm.lang , 
    nm.source, 
    nm.os, 
    nm.area, 
    nm.model, 
    nm.brand, 
    nm.sdk_version, 
    nm.gmail, 
    nm.height_width,
    nm.app_time,
    nm.network,
    nm.lng,
    nm.lat,
    nm.create_date,
    2 retention_day 
from  dws_uv_detail_day ud join dws_new_mid_day nm   on ud.mid_id =nm.mid_id 
where ud.dt='2019-02-11' and nm.create_date=date_add('2019-02-11',-2)

union all
select  
    nm.mid_id,
    nm.user_id , 
    nm.version_code , 
    nm.version_name , 
    nm.lang , 
    nm.source, 
    nm.os, 
    nm.area, 
    nm.model, 
    nm.brand, 
    nm.sdk_version, 
    nm.gmail, 
    nm.height_width,
    nm.app_time,
    nm.network,
    nm.lng,
    nm.lat,
    nm.create_date,
    3 retention_day 
from  dws_uv_detail_day ud join dws_new_mid_day nm   on ud.mid_id =nm.mid_id 
where ud.dt='2019-02-11' and nm.create_date=date_add('2019-02-11',-3);

```

2）查询导入数据（每天计算前1,2,3天的新用户访问留存明细）

```sql
hive (gmall)> select retention_day , count(*) from dws_user_retention_day group by retention_day;
```

**注意：这边由于我们的数据只有两天，所以只有一日留存的数据**

## 8.3 ADS层

### 8.3.1 留存用户数

1）建表语句

```sql
drop table if exists ads_user_retention_day_count;
create external table ads_user_retention_day_count 
(
   `create_date`       string  comment '设备新增日期',
   `retention_day`     int comment '截止当前日期留存天数',
   `retention_count`    bigint comment  '留存数量'
)  COMMENT '每日用户留存情况'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_user_retention_day_count/';

```



2）导入数据

```sql
insert into table ads_user_retention_day_count 
select
    create_date,
    retention_day,
    count(*) retention_count
from dws_user_retention_day
where dt='2019-02-11' 
group by create_date,retention_day;

```

3）查询导入数据

```
hive (gmall)> select * from ads_user_retention_day_count
```



### 8.3.2 留存用户比率



1）建表语句

```sql
hive (gmall)>
drop table if exists ads_user_retention_day_rate;
create external table ads_user_retention_day_rate 
(
     `stat_date`          string comment '统计日期',
     `create_date`       string  comment '设备新增日期',
     `retention_day`     int comment '截止当前日期留存天数',
     `retention_count`    bigint comment  '留存数量',
     `new_mid_count`     bigint comment '当日设备新增数量',
     `retention_ratio`   decimal(10,2) comment '留存率'
)  COMMENT '每日用户留存情况'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_user_retention_day_rate/';

```

2）导入数据

```sql
hive (gmall)>
insert into table ads_user_retention_day_rate
select 
    '2019-02-11', 
    ur.create_date,
    ur.retention_day, 
    ur.retention_count, 
    nc.new_mid_count,
    ur.retention_count/nc.new_mid_count*100
from 
(
    select
        create_date,
        retention_day,
        count(*) retention_count
    from dws_user_retention_day
    where dt='2019-02-11' 
    group by create_date,retention_day
) ur join ads_new_mid_count nc on nc.create_date=ur.create_date;

```

3）查询导入数据

```sql
hive (gmall)>select * from ads_user_retention_day_rate;
```

# 第9章 新数据准备

​	为了分析沉默用户、本周回流用户数、流失用户、最近连续3周活跃用户、最近七天内连续三天活跃用户数，需要准备2019-02-12、2019-02-20日的数据。

1）2019-02-12数据准备

（1）修改日志时间

```
[xzt@hadoop102 ~]$ dt.sh 2019-02-12
```

（2）启动集群

```
[xzt@hadoop102 ~]$ cluster.sh start
```

（3）生成日志数据

```
[xzt@hadoop102 ~]$ lg.sh
```

（4）将HDFS数据导入到ODS层

```
[xzt@hadoop102 ~]$ ods_log.sh 2019-02-12
```

（5）将ODS数据导入到DWD层

```
[xzt@hadoop102 ~]$ dwd_start_log.sh 2019-02-12

[xzt@hadoop102 ~]$ dwd_base_log.sh 2019-02-12

[xzt@hadoop102 ~]$ dwd_event_log.sh 2019-02-12

```

（6）将DWD数据导入到DWS层

```
[xzt@hadoop102 ~]$ dws_uv_log.sh 2019-02-12
```

（7）验证

```
hive (gmall)> select * from dws_uv_detail_day where dt='2019-02-12' limit 2;
```

2）2019-02-20 数据准备

略

# 第10章 需求四：沉默用户数

沉默用户：指的是只在安装当天启动过，且启动时间是在一周前

## 10.1 DWS层

使用日活明细表dws_uv_detail_day作为DWS层数据

## 10.2 ADS层



![沉默用户](https://github.com/xzt1995/Data-Warehouse/blob/master/img/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E6%95%B0%E4%BB%93%E6%90%AD%E5%BB%BA/%E6%B2%89%E9%BB%98%E7%94%A8%E6%88%B7.png)



1）建表语句

```sql
hive (gmall)>
drop table if exists ads_slient_count;
create external table ads_slient_count( 
    `dt` string COMMENT '统计日期',
    `slient_count` bigint COMMENT '沉默设备数'
) 
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_slient_count';

```

2）导入2019-02-20数据

```sql
insert into table ads_slient_count
select 
    '2019-02-20' dt,
    count(*) slient_count
from 
(
    select mid_id
    from dws_uv_detail_day
    where dt<='2019-02-20'
    group by mid_id
    having count(*)=1 and min(dt)<date_add('2019-02-20',-7)
) t1;

```

3）查询导入数据

```sql
hive (gmall)> select * from ads_slient_count;
```

## 10.3 编写脚本

1）在hadoop102的/home/xzt/bin目录下创建脚本

```
[xzt@hadoop102 bin]$ vim ads_slient_log.sh
```

​        在脚本中编写如下内容

```shell
#!/bin/bash

hive=/opt/module/hive/bin/hive
APP=gmall

if [ -n "$1" ];then
	do_date=$1
else
	do_date=`date -d "-1 day" +%F`
fi

echo "-----------导入日期$do_date-----------"

sql="
insert into table "$APP".ads_slient_count
select 
    '$do_date' dt,
    count(*) slient_count
from 
(
    select 
        mid_id
    from "$APP".dws_uv_detail_day
    where dt<='$do_date'
    group by mid_id
    having count(*)=1 and min(dt)<=date_add('$do_date',-7)
)t1;"

$hive -e "$sql"


```



2）增加脚本执行权限

```
[xzt@hadoop102 bin]$ chmod 777 ads_slient_log.sh
```

3）脚本使用

```
[xzt@hadoop102 module]$ ads_slient_log.sh 2019-02-20
```

4）查询结果

```
hive (gmall)> select * from ads_slient_count;
```

5）脚本执行时间

企业开发中一般在每日凌晨30分~1点





# 第11章 需求五：本周回流用户数

本周回流=本周活跃-本周新增-上周活跃

## 11.1 DWS层

使用日活明细表dws_uv_detail_day作为DWS层数据

## 11.2 ADS层

![回流用户](https://github.com/xzt1995/Data-Warehouse/blob/master/img/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E6%95%B0%E4%BB%93%E6%90%AD%E5%BB%BA/%E5%9B%9E%E6%B5%81%E7%94%A8%E6%88%B7.png)



1）建表语句

```sql
hive (gmall)>
drop table if exists ads_back_count;
create external table ads_back_count( 
    `dt` string COMMENT '统计日期',
    `wk_dt` string COMMENT '统计日期所在周',
    `wastage_count` bigint COMMENT '回流设备数'
) 
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_back_count';

```

2）导入数据：

```sql
hive (gmall)> 
insert into table ads_back_count
select 
   '2019-02-20' dt,
   concat(date_add(next_day('2019-02-20','MO'),-7),'_',date_add(next_day('2019-02-20','MO'),-1)) wk_dt,
   count(*)
from 
(
    select t1.mid_id
    from 
    (
        select	mid_id
        from dws_uv_detail_wk
        where wk_dt=concat(date_add(next_day('2019-02-20','MO'),-7),'_',date_add(next_day('2019-02-20','MO'),-1))
    )t1
    left join
    (
        select mid_id
        from dws_new_mid_day
        where create_date<=date_add(next_day('2019-02-20','MO'),-1) and create_date>=date_add(next_day('2019-02-20','MO'),-7)
    )t2
    on t1.mid_id=t2.mid_id
    left join
    (
        select mid_id
        from dws_uv_detail_wk
        where wk_dt=concat(date_add(next_day('2019-02-20','MO'),-7*2),'_',date_add(next_day('2019-02-20','MO'),-7-1))
    )t3
    on t1.mid_id=t3.mid_id
    where t2.mid_id is null and t3.mid_id is null
)t4;

```



3）查询结果

```sql
hive (gmall)> select * from ads_back_count;
```

## 11.3 编写脚本

1）在hadoop102的/home/xzt/bin目录下创建脚本

```
[xzt@hadoop102 bin]$ vim ads_back_log.sh
```

​        在脚本中编写如下内容

```shell
#!/bin/bash

if [ -n "$1" ];then
	do_date=$1
else
	do_date=`date -d "-1 day" +%F`
fi

hive=/opt/module/hive/bin/hive
APP=gmall

echo "-----------导入日期$do_date-----------"

sql="
insert into table "$APP".ads_back_count
select 
       '$do_date' dt,
       concat(date_add(next_day('$do_date','MO'),-7),'_',date_add(next_day('$do_date','MO'),-1)) wk_dt,
       count(*)
from 
(
    select t1.mid_id
    from 
    (
        select mid_id
        from "$APP".dws_uv_detail_wk
        where wk_dt=concat(date_add(next_day('$do_date','MO'),-7),'_',date_add(next_day('$do_date','MO'),-1))
    )t1
    left join
    (
        select mid_id
        from "$APP".dws_new_mid_day
        where create_date<=date_add(next_day('$do_date','MO'),-1) and create_date>=date_add(next_day('$do_date','MO'),-7)
    )t2
    on t1.mid_id=t2.mid_id
    left join
    (
        select mid_id
        from "$APP".dws_uv_detail_wk
        where wk_dt=concat(date_add(next_day('$do_date','MO'),-7*2),'_',date_add(next_day('$do_date','MO'),-7-1))
    )t3
    on t1.mid_id=t3.mid_id
    where t2.mid_id is null and t3.mid_id is null
)t4;
"

$hive -e "$sql"

```

2）增加脚本执行权限

```
[xzt@hadoop102 bin]$ chmod 777 ads_back_log.sh
```

3）脚本使用

```
[xzt@hadoop102 module]$ ads_back_log.sh 2019-02-20
```

4）查询结果

```
hive (gmall)> select * from ads_back_count;
```

5）脚本执行时间

企业开发中一般在每周一凌晨30分~1点



# 第12章 需求六：流失用户数

流失用户：最近7天未登录我们称之为流失用户

## 12.1 DWS层

使用日活明细表dws_uv_detail_day作为DWS层数据

## 12.2 ADS层

1）建表语句

```sql
hive (gmall)>
drop table if exists ads_wastage_count;
create external table ads_wastage_count( 
    `dt` string COMMENT '统计日期',
    `wastage_count` bigint COMMENT '流失设备数'
) 
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_wastage_count';

```

2）导入2019-02-20数据

```sql
hive (gmall)>
insert into table ads_wastage_count
select
     '2019-02-20',
     count(*)
from 
(
    select mid_id
from dws_uv_detail_day
    group by mid_id
    having max(dt)<=date_add('2019-02-20',-7)
)t1;

```

## 12.3 编写脚本

1）在hadoop102的/home/xzt/bin目录下创建脚本

```
[xzt@hadoop102 bin]$ vim ads_wastage_log.sh
```

​        在脚本中编写如下内容

```shell
#!/bin/bash

if [ -n "$1" ];then
	do_date=$1
else
	do_date=`date -d "-1 day" +%F`
fi

hive=/opt/module/hive/bin/hive
APP=gmall

echo "-----------导入日期$do_date-----------"

sql="
insert into table "$APP".ads_wastage_count
select
     '$do_date',
     count(*)
from 
(
    select mid_id
    from "$APP".dws_uv_detail_day
    group by mid_id
    having max(dt)<=date_add('$do_date',-7)
)t1;
"

$hive -e "$sql"

```

2）增加脚本执行权限

```
[xzt@hadoop102 bin]$ chmod 777 ads_wastage_log.sh
```

3）脚本使用

```
[xzt@hadoop102 module]$ ads_wastage_log.sh 2019-02-20
```

4）查询结果

```
hive (gmall)> select * from ads_wastage_count;
```

5）脚本执行时间

企业开发中一般在每日凌晨30分~1点



# 第13章 需求七：最近连续3周活跃用户数

最近3周连续活跃的用户：通常是周一对前3周的数据做统计，该数据一周计算一次。

## 13.1 DWS层

使用周活明细表dws_uv_detail_wk作为DWS层数据

## 13.2 ADS层

1）建表语句

```sql
hive (gmall)>
drop table if exists ads_continuity_wk_count;
create external table ads_continuity_wk_count( 
    `dt` string COMMENT '统计日期,一般用结束周周日日期,如果每天计算一次,可用当天日期',
    `wk_dt` string COMMENT '持续时间',
    `continuity_count` bigint
) 
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_continuity_wk_count';

```

2）导入2019-02-20所在周的数据

```sql
hive (gmall)>
insert into table ads_continuity_wk_count
select 
     '2019-02-20',
     concat(date_add(next_day('2019-02-20','MO'),-7*3),'_',date_add(next_day('2019-02-20','MO'),-1)),
     count(*)
from 
(
    select mid_id
    from dws_uv_detail_wk
    where wk_dt>=concat(date_add(next_day('2019-02-20','MO'),-7*3),'_',date_add(next_day('2019-02-20','MO'),-7*2-1)) 
    and wk_dt<=concat(date_add(next_day('2019-02-20','MO'),-7),'_',date_add(next_day('2019-02-20','MO'),-1))
    group by mid_id
    having count(*)=3
)t1;

```

3）查询

```
hive (gmall)> select * from ads_continuity_wk_count;
```

## 13.3 编写脚本

1）在hadoop102的/home/xzt/bin目录下创建脚本

[xzt@hadoop102 bin]$ vim ads_continuity_wk_log.sh

​        在脚本中编写如下内容

```sql
#!/bin/bash

if [ -n "$1" ];then
	do_date=$1
else
	do_date=`date -d "-1 day" +%F`
fi

hive=/opt/module/hive/bin/hive
APP=gmall

echo "-----------导入日期$do_date-----------"

sql="
insert into table "$APP".ads_continuity_wk_count
select 
     '$do_date',
     concat(date_add(next_day('$do_date','MO'),-7*3),'_',date_add(next_day('$do_date','MO'),-1)),
     count(*)
from 
(
    select mid_id
    from "$APP".dws_uv_detail_wk
    where wk_dt>=concat(date_add(next_day('$do_date','MO'),-7*3),'_',date_add(next_day('$do_date','MO'),-7*2-1)) 
    and wk_dt<=concat(date_add(next_day('$do_date','MO'),-7),'_',date_add(next_day('$do_date','MO'),-1))
    group by mid_id
    having count(*)=3
)t1;"

$hive -e "$sql"

```

2）增加脚本执行权限

```
[xzt@hadoop102 bin]$ chmod 777 ads_continuity_wk_log.sh
```

3）脚本使用

```
[xzt@hadoop102 module]$ ads_continuity_wk_log.sh 2019-02-20
```

4）查询结果

```
hive (gmall)> select * from ads_continuity_wk_count;
```

5）脚本执行时间

企业开发中一般在每周一凌晨30分~1点



# 第14章 需求八：最近七天内连续三天活跃用户数

说明：最近7天内连续3天活跃用户数

## 14.1 DWS层

使用日活明细表dws_uv_detail_day作为DWS层数据

## 14.2 ADS层



![7天内连续三天活跃用户数](https://github.com/xzt1995/Data-Warehouse/blob/master/img/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E6%95%B0%E4%BB%93%E6%90%AD%E5%BB%BA/7%E5%A4%A9%E5%86%85%E8%BF%9E%E7%BB%AD%E4%B8%89%E5%A4%A9%E6%B4%BB%E8%B7%83%E7%94%A8%E6%88%B7%E6%95%B0.png)



​	1 首先用rank和窗口函数over来对日活中的活跃日期进行排序

​	2 利用等差数列的原理来计算活跃日期和排名之间的差值，如上图所示：如果用户是连续三天登录，那么求完差值之后的日期应该是相等的，如mid_id=1的用户在15，17，18，19这四天登录过，但由于16号没有登录，所以计算出的差值15日和其他三天不同，因此我们可以根据这个原理来求出每个用户连续登录的天数。

​	3 根据mid_id和差值date_diff来对数据分组，再求出count>=3的用户即为满足需求用户。如上图中mid_id=1的用户。

​	4 因为我们查询的是七天内连续三天登录用户，所以理论上可能出现123 登录，567登录的情况，这样会导致该用户被重复计算两次，所以我们最后再加一个子查询，group by mid_id 去重。



1）建表语句

```sql
hive (gmall)>
drop table if exists ads_continuity_uv_count;
create external table ads_continuity_uv_count( 
    `dt` string COMMENT '统计日期',
    `wk_dt` string COMMENT '最近7天日期',
    `continuity_count` bigint
) COMMENT '连续活跃设备数'
row format delimited fields terminated by '\t'
location '/warehouse/gmall/ads/ads_continuity_uv_count';

```

2）写出导入数据的SQL语句

```sql
hive (gmall)>
insert into table ads_continuity_uv_count
select
    '2019-02-12',
    concat(date_add('2019-02-12',-6),'_','2019-02-12'),
    count(*)
from
(
    select mid_id
    from
    (
        select mid_id      
        from
        (
            select 
                mid_id,
                date_sub(dt,rank) date_dif
            from
            (
                select 
                    mid_id,
                    dt,
                    rank() over(partition by mid_id order by dt) rank
                from dws_uv_detail_day
                where dt>=date_add('2019-02-12',-6) and dt<='2019-02-12'
            )t1
        )t2 
        group by mid_id,date_dif
        having count(*)>=3
    )t3 
    group by mid_id
)t4;

```



3）查询

```sql
hive (gmall)> select * from ads_continuity_uv_count;
```

## 14.3 编写脚本

1）在hadoop102的/home/xzt/bin目录下创建脚本

```
[xzt@hadoop102 bin]$ vim ads_continuity_log.sh
```

​        在脚本中编写如下内容

```shell
#!/bin/bash

if [ -n "$1" ];then
	do_date=$1
else
	do_date=`date -d "-1 day" +%F`
fi

hive=/opt/module/hive/bin/hive
APP=gmall

echo "-----------导入日期$do_date-----------"

sql="
insert into table "$APP".ads_continuity_uv_count
select 
     '$do_date',
     concat(date_add('$do_date',-6),'_','$do_date') dt,
     count(*) 
from 
(
    select mid_id
    from
    (
        select mid_id
        from 
        (
            select
                mid_id,
                date_sub(dt,rank) date_diff
            from 
            (
                select 
                    mid_id,
                    dt,
                    rank() over(partition by mid_id order by dt) rank
                from "$APP".dws_uv_detail_day
                where dt>=date_add('$do_date',-6) and dt<='$do_date'
            )t1
        )t2
        group by mid_id,date_diff
        having count(*)>=3
    )t3 
    group by mid_id
)t4;
"

$hive -e "$sql"

```

2）增加脚本执行权限

```
[xzt@hadoop102 bin]$ chmod 777 ads_continuity_log.sh
```

3）脚本使用

```
[xzt@hadoop102 module]$ ads_continuity_log.sh 2019-02-12
```

4）查询结果

```
hive (gmall)> select * from ads_continuity_uv_count;
```

5）脚本执行时间

企业开发中一般在每日凌晨30分~1点
