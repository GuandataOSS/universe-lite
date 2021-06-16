universe-lite （小宇宙）


# 下载地址

<https://github.com/GuandataOSS/universe-lite/releases>

其中 0.11.x，如果需要使用 python任务或者python插件任务，需要python中安装对应的 duckdb 0.2.7 版本

文件：

-   universe-lite-0.11.1.jar: works on Linux, Windows & Mac, depends on JDK 8 or above
-   universe-lite-0.11.1: native image on 64bit linux, don't depend on JDK


# 项目背景

目前数据处理ETL的一个趋势就是从 ETL （Extract Transform Load) 转为 ELT (Extract Load Transform).

对于ELT工具，我们往往会联想到 Hive、Spark、Snowflake等，但是这些都比较重量，其实对于“Transform”的SQL数据源，我们还可以有轻量的选择： <https://github.com/cwida/duckdb>

universe-lite 即是基于 duckdb & apache parquet文件的轻量级工具，其特点：

-   无外部依赖
-   可通过graalvm native-image编译为linux下原生应用，减少冷启动时间，适合于 serverless functions
-   通过typesafe config 配置文件来编排工作流，并配置其数据依赖
-   每个工作流中的节点（stage），其结果将被自动注册为duckdb中的view，方便后续节点直接引用数据
-   支持子工作流，支持循环
-   支持Python插件


# 关于是否开源

目前先只开放release下载和相关文档， 开放Python相关插件： <https://github.com/GuandataOSS/universe-lite-python-plugins>

universe-lite本身的代码还在内部完善过程中，后面会再决定何时开源。


# 版本匹配

目前0.11.x release 将依赖于 duckdb 0.2.7, duckdb 后续版本发布后，universe-lite 将做相应更新。 （主要是因为 python插件中，需要用 pip 安装官方的duckdb，这时jvm中内嵌的duckdb版本需要和python中的一致）


# 核心概念

开发平台(universe)的节点 （英文术语为 Stage） 有3种类型：

-   Source: 数据源， 本身只会有输出节点
-   Processor： 中间处理节点， 有输入节点也有输出
-   Target： 输出节点，本身只接受工作流中的输入

这些概念在universe-lite 中也通用


# 使用方法

下载 universe-lite-${version}.jar 后，放入本地的一个单独目录 （目录路径上请不要有中文，空格等特殊字符）

进入该目录： 编写一个 conf 文件，比如 test1.conf （该语法是 typesafe config 文件格式，类似于yaml）:

**注意：** config文件的语法可以直接参考其官方网站： <https://github.com/lightbend/config>

```yaml
stage = [
   {
     name=data_gen1
     type=sql
     sql="select '" ${dt} "' as dt"
   }
   {
     type=stdout
     input=data_gen1
   }
]
```

config中最重要的就是 stage 节点下的内容，会列出这个工作流中的所有节点， 然后 从上到下一个节点，一个节点的执行。

本例中，分别有2个节点。

第一个节点的名字是 “data\_gen1”, 这个是必须的，目前请尽量使用英文和数字等， 对于 Source 和 Processor 类型的节点，其节点名字将作为我们内嵌的数据库中的视图的名字，这样方便后续节点引用其结果

第一个节点的类型是： sql 类型，说明其是使用了我们内嵌的 duckdb 数据库 （类似于universe中强依赖的spark sql）

对于不同类型的节点，其配置参数是不同的（详细见后面说明）， 对于sql类型的节点，其有一个 “sql” 配置。

第二个节点其类型是 stdout，也就是把之前节点，选择一个节点的输出来打印成表格， 具体打印哪个节点由 input 参数来指定。 由于本例子中无后续节点依赖于stdout节点，所以这个节点可以不用明确的命名。（系统会默认分配名字类似于： unnamed\_1 ）

有了这个config文件，可以到命令行下运行如下命令 (需要安装 jdk/jre 8 以上版本）:

```sh
java -jar universe-lite-${version}.jar -c test1.conf
```

得到如下输出

```text
[INFO] 2021-01-07 10:45:44.353 c.g.u.lite.UniverseLite$:[81] - --------- start to run stage 'data_gen1' (1/2) -------->
[INFO] 2021-01-07 10:45:45.897 c.g.u.lite.UniverseLite$:[83] - <-------- finish stage 'data_gen1' (1/2) ---------------
[INFO] 2021-01-07 10:45:45.897 c.g.u.lite.UniverseLite$:[81] - --------- start to run stage 'print' (2/2) -------->
[INFO] 2021-01-07 10:45:46.132 c.g.u.l.p.PluginRunner:[100] -

+--------+
|      dt|
+--------+
|20210107|
+--------+

[INFO] 2021-01-07 10:45:46.133 c.g.u.lite.UniverseLite$:[83] - <-------- finish stage 'print' (2/2) ---------------
```


# 怎么build项目

```sh
sbt universal:packageBin  
```


# 支持的原子任务类型


## jdbc

jdbc是最常见的操作，类似于开发平台的 “sql节点”的“获取数据”和“插入数据”节点。 可以参考 <examples/mysql_task_status_to_postgresql.conf> 样例

| name            | type                 | required | default | comments                                                                                         |
|--------------- |-------------------- |-------- |------- |------------------------------------------------------------------------------------------------ |
| type            | string               | true     | jdbc    | 决定了是jdbc类型任务                                                                             |
| driver          | string               | true     |         | jdbc driver class full name                                                                      |
| url             | string               | true     |         | 连接connection string                                                                            |
| user            | string               | false    |         |                                                                                                  |
| password        | string               | false    |         |                                                                                                  |
| pre\_statement  | StringList or string | false    | []      |                                                                                                  |
| query           | string               | ?        |         | 当作为“获取数据”节点时，必须，而且是SELECT SQL                                                 |
| jdbc.fetchsize  | int                  | false    |         | 当作为“获取数据”节点时，可选。一般为 int 类型，但是对于MySQL等，支持特殊字符串 Integer.MIN\_VALUE |
| jdbc.autocommit | boolean              | false    | true    | 是否把所有的statements或query都放在一个transaction中，比如对于Postgresql如果我们要使得 fetchsize生效，比如设置 autocommit 为false |
| statement       | string               | ?        |         | 当作为“插入数据”节点时，必须，一般为非查询的 CREATE / UPDATE/ INSERT 等                        |
| jdbc.batchsize  | int                  | false    | 1000    | 当作为“插入数据”节点时，如果有 input，则会转为每个batch插入目标表的行数                        |
| input           | StringList or string | false    |         | 只有当作为“插入数据”节点，并且需要把input 表的数据一行一行进行bind并插入目标时才需要           |

具体的例子请参考 config/mysql\_task\_status\_to\_postgresql.conf.template 样例， 该例子中把 mysql 中的一张表，批量插入到 postgresql中

注意： 打包的 universe-lite-${version}.jar 只打包了 sqlite 的jdbc driver。

如果需要其它类型的jdbc driver， 比如oracle、clickhouse等，需要手工下载该jar包，比如 ojdbc6.jar

然后运行

```sh
java -cp universe-lite-${version}.jar:ojdbc6.jar com.guandata.universe.lite.UniverseLite -c test1.conf
```

```yaml
stage = [
  {
    name = mysql_task_status
    type = jdbc
    driver = "com.mysql.jdbc.Driver"
    url = "jdbc:mysql://127.0.0.1:3306/guandata?zeroDateTimeBehavior=convertToNull&tinyInt1isBit=false&useUnicode=true&characterEncoding=UTF8&socketTimeout=60000&verifyServerCertificate=false"
    query = """
select `id`,
`task_id`,
`dom_id`,
`task_type`,
`task_name`,
`task_result`,
`task_state`,
`user_id`,
`user_name`,
`obj_id`,
`obj_name`,
`content`,
`task_param`,
`task_status_history`,
`submit_time`,
`running_time`,
`finished_time`,
`utime`,
`is_del`
from task_status
where submit_time >= '2021-01-05' and submit_time < '2021-01-06'

"""
    user = "****"
    password = "******"
    jdbc.fetchsize = "Integer.MIN_VALUE"
  }

  {
    name=insert_into_postgresql
    type = jdbc
    driver = "org.postgresql.Driver"
    url = "jdbc:postgresql://127.0.0.1:5432/postgres?socketTimeout=60000&stringtype=unspecified"
    statement = """
insert into pg_task_status ("id",
"task_id",
"dom_id",
"task_type",
"task_name",
"task_result",
"task_state",
"user_id",
"user_name",
"obj_id",
"obj_name",
"content",
"task_param",
"task_status_history",
"submit_time",
"running_time",
"finished_time",
"utime",
"is_del") values (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, cast(? as boolean)) ON CONFLICT (submit_time, task_id) DO NOTHING
"""
    user = "*****"
    password = "******"

    input = "mysql_task_status"
  }
]
```


## stdout

调试节点，打印上游的某些表到屏幕， 可以参考 <examples/simple_sql_demo.conf> 样例

| name     | type    | required | default | comments                       |
|-------- |------- |-------- |------- |------------------------------ |
| type     | string  | true     | stdout  | 决定了是stdout类型任务         |
| limit    | int     | false    | 30      | 打印行数                       |
| truncate | int     | false    | 20      | 如果cell中的长度超过 20，则只显示前面部分 |
| vertical | boolean | false    | false   | 默认是表格形式，如果为 true，则一行打印一个cell内容 |

```yaml
stage =[
   {
     name=data_gen1
     type=sql
     sql="select now() as a, current_date as b, 'test' as c, 123 as d"
   },

   {
     name=add_col
     type=sql
     sql="select *, c || d as e from data_gen1"
   }

   {
     type=stdout
     input=add_col
   }
]
```


## sql

universe-lite 中内嵌了一个 duckdb 数据库， 每一步的结果，无论是 jdbc还是python等，都将结果注册为同名的view到duckdb中，这样，sql节点可以做很多sql中间操作

可以参考 <examples/simple_sql_demo.conf> 样例

| name    | type    | required | default | comments                                       |
|------- |------- |-------- |------- |---------------------------------------------- |
| type    | string  | true     | sql     | 决定了是sql类型任务                            |
| sql     | string  | true     |         | 执行的sql语句，可以直接访问之前节点的输出      |
| persist | boolean | false    | false   | 默认结果将注册为 view， 如果 persist是true的话，则注册为物理表 table |

```yaml
stage = [
   {
     name=data_gen1
     type=sql
     sql="select now() as a, current_date as b, 'test' as c, 123 as d"
   }

   {
     name=add_col
     type=sql
     sql="select *, c || d as e from data_gen1"
   }

   {
     type=stdout
     input=add_col
   }
]
```


## shell

执行shell script， 注意：目前shell任务是 无输入 无输出概念的。 可以参考 <examples/test_shell.conf> 样例

| name   | type   | required | default | comments      |
|------ |------ |-------- |------- |------------- |
| type   | string | true     | shell   | 决定了是shell类型任务 |
| script | string | true     |         | 脚本内容      |

```yaml
stage = [
  {
    type = shell
    script = """
echo $PWD
ls ./
echo "OK, finsihed!"
"""
  }
]
```

注意在 config 文件中， 可以使用 """ """ 来写多行的string


## python

执行python任务，可以和 pandas dataframe 很好结合。具体用法和 universe开发平台一样，支持其它节点的输入和输出 可以参考 <examples/python_test.conf> 样例

注意：

-   要使用python任务，需要python 3 (python 2.7不支持)
-   需要python中安装了 duckdb 0.23及以后版本。 可以通过 pip install duckdb 进行安装
-   如果python的名字不是python 而是 python3, 或者python在另外的目录中，可以通过设置环境变量 PYTHON\_EXE 解决, 比如:

```sh
export PYTHON_EXE=~/.virtualenv/python3/bin/python3
```

| name   | type       | required | default | comments                                                                                             |
|------ |---------- |-------- |------- |---------------------------------------------------------------------------------------------------- |
| type   | string     | true     | python  | 决定了是python类型任务                                                                               |
| script | string     | true     |         | python 脚本内容                                                                                      |
| input  | StringList | false    | []      | 可选，如果python中需要引用其它节点内容,则可以列出来, 然后python中可以通过 load\_input1(), load\_input2() 等来使用 |
| output | StringList | false    | []      | 可选，当python节点有输出时，可以把输出个数列到这里，比如   output=[output1, output2],  注意：目前输出列表中只支持 output1、output2 等，而且需要有序 |

注意： 当 output 只有一个时，这个python节点的结果将注册在 duckdb 中表名为 节点名。

而当output为多个时，则分别注册为： 节点名\_output1, 节点名\_output2 &#x2026;

```yaml
stage = [
   {
     name=sql_input
     type=sql
     sql = """
     select 1 as a, 2 as b, 3 as c
     """
   }

   {
     name=test_pandas
     type=python
     script = """

df = load_input1()
df['col_new']='newcolumn'
save_output1(df)

    """
     input=sql_input
     output=[output1]
   }

   {
     type = stdout
     input = test_pandas
   }
]
```

注意python script中的行的缩进！ （顶格写）


## file

支持将数据写入到文件， 目前支持 parquet 和 csv 两种类型的文件（文件名必须以 .parquet 和 .csv 结尾）。 可以参考 <examples/file_write.conf> 样例

| name  | type   | required | default | comments     |
|----- |------ |-------- |------- |------------ |
| type  | string | true     | file    | 决定了是file类型任务 |
| path  | string | true     |         | 文件位置     |
| input | string | true     |         | 要存储的 input |

```yaml
stage = [
   {
      name=input1
      type=sql
      sql="""
      select 1 as a, 2 as b
      """
   }

   {
     name=file1
     type=file
     path=~/data/file_write.parquet
     input=input1
   }
]
```

注意: 这个file任务，只能用于输出内容到文件。 如果想读取已经存在的parquet文件，则可以直接使用 sql 任务, 其sql内容为:

```sql
select * from parquet_scan('~/data/file_write.parquet')
```

另外，对于parquet文件，支持一种特殊的用法，用来增量的append到同一个parquet文件中, 设置 append=true

```yaml
stage = [
   {
      name=input1
      type=sql
      sql="""
      select 1 as a, 2 as b
      """
   }

   {
     name=file1
     type=file
     path=~/data/file_write.parquet
     append=true
     input=input1
   }
]

```

上面的例子中，每次运行这个脚本，则会行数增加一行


## jq

使用 jq 命令行工具来处理数据中某个列的json数据，并根据不同的 jq 表达式来生成不同的新的列。 可以参考 <examples/jq_process.conf> 样例

注意： 需要提前安装 jq 工具 （从 <https://github.com/stedolan/jq/releases> 下载 1.6 以后版本）

| name            | type   | required | default | comments                                                                         |
|--------------- |------ |-------- |------- |-------------------------------------------------------------------------------- |
| type            | string | true     | jq      | 决定了是jq类型任务                                                               |
| base            | string | true     |         | 在源数据集中的 json 文本的的列                                                   |
| input           | string | true     |         | 单个输入数据集                                                                   |
| simple\_expr.\* | string | false    |         | single\_expr 或 generator 至少存在一个, single\_expr 是指普通的expr （非 generator），一行输入对应一行输出 |
| generator.\*    | string | false    |         | generator 是一种特殊的jq expression， 一行输入可能对应于多行输出， 多个 generator 的输出是 cross join 的关系 |

```yaml
stage = [
  {
    name=read_pg
    type=sql
    sql ="""
    select task_status_history
    from parquet_scan('~/data/task_status_20210118_limit_10000.parquet')
    limit 1
    """
  }

  {
    name=jq1
    type=jq
    base=task_status_history
    simple_expr.first_status=".[0].state"
    generator.each=".[]|.state"
    simple_expr.last_status=".[-1].state"
    input=read_pg
  }

  {
    type=stdout
    input=jq1
    vertical=true
    limit=10
    truncate=80
  }
]
```

上面例子的输出为:

```text
-RECORD 0-----------------------------------------------------------------------------------------------
 task_status_history | [{"ts":"2021-01-18 00:00:00","state":"QUEUEING"},{"ts":"2021-01-18 00:00:00",... 
 first_status        | "QUEUEING"                                                                       
 last_status         | "FINISHED"                                                                       
 each                | "QUEUEING"                                                                       
-RECORD 1-----------------------------------------------------------------------------------------------
 task_status_history | [{"ts":"2021-01-18 00:00:00","state":"QUEUEING"},{"ts":"2021-01-18 00:00:00",... 
 first_status        | "QUEUEING"                                                                       
 last_status         | "FINISHED"                                                                       
 each                | "RUNNING"                                                                        
-RECORD 2-----------------------------------------------------------------------------------------------
 task_status_history | [{"ts":"2021-01-18 00:00:00","state":"QUEUEING"},{"ts":"2021-01-18 00:00:00",... 
 first_status        | "QUEUEING"                                                                       
 last_status         | "FINISHED"                                                                       
 each                | "FINISHED"                                                                       
```


## universe java plugin

如果已经下载了开发平台的插件， 则可以把这些插件放在 universe-lite-${version}.jar 同目录的 plugins 目录下 （需要手工创建）

插件的使用请参考 <examples/sample_data_to_csv.conf>

```yaml

stage = [
  {
    name=raw_json_input1
    type="com_guandata_plugin_dev_source_RawDataSource"
    param {
       rawDataJson = """
       [
         {"f1": "1", "f2": "2", "f3": "3"},
         {"f1": "11", "f2": "22", "f3": "33"}
       ]
       """
    }
  }

  {
    name=add_column
    type = sql
    sql = """
    select *, f1 || f2 || f3  as f4
    from raw_json_input1
    """
  }

  {
      name=output_csv
      type = "CsvDumpTarget"
      param {
         limit = 1
      }
      input = ["add_column"]
  }
]
```

这里

| name  | type                 | required | default | comments                                                                                               |
|----- |-------------------- |-------- |------- |------------------------------------------------------------------------------------------------------ |
| type  | string               | true     |         | java plugin的全名， 比如： com\_guandata\_plugin\_dev\_source\_RawDataSource，  也可以是全名的最后部分，比如   CsvDumpTarget |
| param | object               | true     |         | 根据不同的插件需要的参数不同，根据插件本身的定义填写                                                   |
| input | StringList or string | false    |         | 如果插件类型是 Processor，或Target，则需要把依赖的之前节点的名字写入                                   |


## python plugin

使用“python插件”前，请确保系统能执行普通的 “python”任务。

可以先git clone <https://github.com/GuandataOSS/universe-lite-python-plugins> 项目， 比如: 放到 ~/universe-lite-python-plugins 目录下， 设置如下环境变量：

```sh
export UL_PYTHON_PLUGIN_DIR=~/universe-lite-python-plugins
```

| name          | type   | required | default        | comments                          |
|------------- |------ |-------- |-------------- |--------------------------------- |
| type          | string | true     | python\_plugin | 决定了是python\_plugin类型任务    |
| library\_name | string | true     |                | 在python插件根目录下的子目录名    |
| plugin\_name  | string | true     |                | 具体插件名，需要和python脚本中的入口函数名一致 |
| param         | object | true     |                | 根据不同的插件需要的参数不同，根据插件本身的定义填写 |
| input         | list   | false    |                | 要处理的 input                    |
| output        | list   | false    |                | 输出的个数，比如：  ["output1", "output2"] |

比如： 我想使用 guandata\_plugin 插件库中的 “upload\_bi\_dataset” 插件来上传数据到Guandata BI服务器，那我可以先到其插件的参数定义需求如下： <https://github.com/GuandataOSS/universe-lite-python-plugins/blob/main/guandata_plugin/library.conf>

```yaml
plugin {
   upload_bi_dataset {
       url = "https://app.guandata.com"
       url=${?GUANDATA_BI_URL}
       domain=${?GUANDATA_BI_DOMAIN}
       email=${?GUANDATA_BI_EMAIL}
       # note password need to encode in base64
       password=${?GUANDATA_BI_PASSWORD}

       # table_name 或者 ds_id 至少设置一个
       table_name='uploaded dataset'

       # when replace is true, it will overwrite existing data in that table!
       replace=false
   }
}
```

我们可以使用时把这些参数设置到 任务conf 文件中，也可以设置为环境变量（对于机密信息，建议放到环境变量中）

那我们可以编写如下的任务文件来上传数据

```yaml

stage = [
  {
     name=input1
     type=sql
     sql="""
     select 1 as a, 2 as b
     """
  }

  {
     name=upload_to_bi
     type=python_plugin
     library_name=guandata_plugin
     plugin_name=upload_bi_dataset
     param {
       url = "https://demo.guandata.com"
       table_name="my test upload data"
       replace=true
     }
     input=[input1]
  }
]
```


# 支持参数

部分支持通过命令行传递参数， 具体样例可以参考 <examples/variable-substitution.conf>

```yaml
######
###### This config file is a demonstration of using variables substitution
######
######   the "dt" parameter can be passed in by command line option:    -i dt=20200101
######

dt = "20210106"

stage =[
   {
     name=data_gen1
     type=sql
     sql="select '"  ${dt}  "' as dt"
   }

   {
     type=stdout
     input=data_gen1
   }
]
```


# 复合任务类型： 子流程  (sub\_stage)


## 简单子流程 (sub\_stage)

```yaml
stage = [
   {
     name=sample
     type = sql
     sql = "select 1, 2, 3"
   }
   {
     name=sub_stage1
     type=sub_stage
     sub_stage = {
         stage = [
           {
             name=add_col
             type=sql
             sql="select *, 4 as new_col from "  ${?SCHEMA_PREFIX}"input1"
           }
           {
             name=do_union
             type=sql
             sql="select * from " ${?SCHEMA_PREFIX}"add_col union all select * from " ${?SCHEMA_PREFIX}"add_col"
           }
         ]
     }
     input=sample
     output=[do_union]
   }
   {
    type=stdout
    input=sub_stage1
  }
]

```

请注意： sub\_stage 节点中又可以嵌入其它 stage

输出为：

| 1 | 2 | 3 | new\_col |
|--- |--- |--- |-------- |
| 1 | 2 | 3 | 4        |
| 1 | 2 | 3 | 4        |


## 将子流程放入到不同文件中

接上面例子，对于子流程(sub\_stage)，我们可以把其放入到其它文件中，方便组装和抽象不同的子逻辑

file sub\_stage1.conf

```yaml
stage = [
  {
    name=add_col
    type=sql
    sql="select *, 4 as new_col from "  ${?SCHEMA_PREFIX}"input1"
  }
  {
    name=do_union
    type=sql
    sql="select * from " ${?SCHEMA_PREFIX}"add_col union all select * from " ${?SCHEMA_PREFIX}"add_col"
  }
]
```

在主任务文件 main\_stage.conf 中

```yaml
stage = [
   {
     name=sample
     type = sql
     sql = "select 1, 2, 3"
   }
   {
     name=sub_stage1
     type=sub_stage
     sub_stage = { include "sub_stage1.conf" }
     input=sample
     output=[do_union]
   }
   {
    type=stdout
    input=sub_stage1
  }
]
```


## 支持循环的子流程  (sub\_stage)

sub\_stage 支持根据某个前置节点的输出来做循环，其循环次数为该前置节点的总行数，并且该前置节点的每行数据将作为 “参数” 传入子流程

```yaml
stage = [
  {
     name=index
     type=sql
     sql="select range as current_index from range(0, 3)"
  },
  {
     name=sub_stage1
     type=sub_stage
     loop_input=index
     sub_stage = {
       stage=[
         {
            name=sql1
            type=sql
            sql = "select " ${current_index} " as idx"
         }
         {
            type=stdout
            input=sql1
         }
       ]
     }
  }
]
```

子流程将被执行3次，输出为：

| idx |
|--- |
| 0   |

| idx |
|--- |
| 1   |

| idx |
|--- |
| 2   |


# 怎么跳过某些节点

有时在排查脚本时，可能会临时暂停一些节点的执行，这个时候不用完全注释掉这个节点的所有行，而可以设置 skip = true, 别忘记同样skip掉其它依赖这个节点的节点

```yaml
stage = [
  {
    name=debug
    type=stdout
    input=other_node
    skipt=true
  }
]
```


# 致谢

universe-lite 使用或参考了很多开源的软件库， thanks to them！

| project                   | home url                                               | license            | license url                                                             |
|------------------------- |------------------------------------------------------ |------------------ |----------------------------------------------------------------------- |
| duckdb                    | <https://github.com/cwida/duckdb>                      | MIT License        | <https://github.com/cwida/duckdb/blob/master/LICENSE>                   |
| config                    | <https://github.com/lightbend/config>                  | Apache-2.0 License | <https://github.com/lightbend/config/blob/master/LICENSE-2.0.txt>       |
| Apache Parquet            | <https://github.com/apache/parquet-mr>                 | Apache-2.0 License | <https://github.com/apache/parquet-mr/blob/master/LICENSE>              |
| Apache DolphinScheduler   | <https://github.com/apache/incubator-dolphinscheduler> | Apache-2.0 License | <https://github.com/apache/incubator-dolphinscheduler/blob/dev/LICENSE> |
| waterdrop                 | <https://github.com/InterestingLab/waterdrop>          | Apache-2.0 License | <https://github.com/InterestingLab/waterdrop/blob/master/LICENSE>       |
| StreamSets Data Collector | <https://github.com/streamsets/datacollector>          | Apache-2.0 License | <https://github.com/streamsets/datacollector/blob/master/LICENSE.txt>   |


# 已知问题

-   目前duckdb存储timestamp，和查询timestamp都是按照UTC时区，所以，需要额外注意这一点