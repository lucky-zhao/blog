# ELK之logstash学习

## logstash是什么？
logstash是开源的服务端数据处理管道，能够同时从多个来源采集数据，转换数据，然后将数据发送到你想要存的地方。
logstash的数据处理过程主要包括：inputs(数据源)，filters(处理数据格式转换)，outputs(输出)三部分；

# 环境：
centos7，JDK1.8，elasticsearch5.5，filebeats7.1，logstash7.1

1. **简单使用logstash**

采用标准输入和输出作为input和output，并且没有filter
* cd到logstash根目录，启动执行：
```
cd logstash-7.1.0
./bin/logstash -e 'input{stdin{} } output{stdout{}}'
```
当控制台出现：
`Successfully started Logstash API endpoint {:port=>9600}`说明已经启动成功

* `-e`表示允许直接通过命令行指定pipeline配置(pipeline 是input-filter-output三个阶段的处理流程，包含了队列管理、插件生命周期管理，简单的说就是配置从什么地方采集数据，怎么处理(filter)，输出到什么地方去的一个配置)
* 在控制台输入 hello logstash 可以看到如下输出：
```
{
    "@timestamp" => 2019-05-27T18:50:46.260Z,
    "host" => "localhost.localdomain",
    "message" => "hello logstash",
    "@version" => "1"
}

```
logstash会自动为数据加上`@version`,`host`,`@timestamp`等字段；
`@timestamp`表示事件发生时间

2. **日志采集**

下载filebeat插件，采集本地日志，然后将结果输出到标准输出;

* 下载一会要用到的日志文件[日志](https://download.elastic.co/demos/logstash/gettingstarted/logstash-tutorial.log.gz)
解压后放到一个地方，一会要用
* 解压完后到filebeat根目录下，有个`filebeat.yml`,修改他的配置文件
找到filebeat.inputs,改成下面配置：
```
filebeat.inputs:
    enabled:true
    paths:/home/logs/logstash-tutorial.log
```
`enabled:true`是开启filebeat.input的功能
`paths`后面是我在网上下载的日志路径。
继续往下找，由于我们暂时没有用es，直接用logstash标准输出，所以要把配置文件里默认的es的配置注释掉，然后把logstash的配置解开注释
```
output.logstash:
    hosts:"localhost:5044"
```
现在启动试试：
`./filebeat -e -c filebeat.yml -d "publish"`
没有报错
* 现在配置一下logstash
创建一个配置文件`first-pipeline.conf`文件内容如下:
```
input{
        beats{
                port=>"5044"
        }
}
output{
        stdout{
                codec=>rubydebug
        }
}

```
这里的`input`表示从beat里采集数据，`output stdout`在第一个例子已经用过，就是标准输出的意思，`code=>rubydebug`用于格式化输出，
在启动一下试试：
`./bin/logstash -f /home/logstash-config/first-pipeline.conf --config.reload.automatic`
`-f`后面跟着的是上面的pipeline配置文件路径，`--config.reload.automatic`是启用动态重载配置功能(配置文件改了，不需要重新启动)
可以看到输出：
```
{
           "log" => {
          "file" => {
            "path" => "/home/logs/logstash-tutorial.log"
        },
        "offset" => 16088
    },
           "ecs" => {
        "version" => "1.0.0"
    },
         "input" => {
        "type" => "log"
    },
          "host" => {
                   "id" => "e20692f79e4e409d8142632cec457b8f",
                 "name" => "localhost.localdomain",
             "hostname" => "localhost.localdomain",
                   "os" => {
            "codename" => "Core",
                "name" => "CentOS Linux",
              "family" => "redhat",
            "platform" => "centos",
             "version" => "7 (Core)",
              "kernel" => "3.10.0-862.el7.x86_64"
        },
         "architecture" => "x86_64",
        "containerized" => true
    },
         "agent" => {
                  "id" => "541451b1-ec8d-41a2-82cc-7287d8c3c42d",
            "hostname" => "localhost.localdomain",
                "type" => "filebeat",
        "ephemeral_id" => "693ebc23-c8ff-4d0e-9eda-cfd02dfb336a",
             "version" => "7.1.0"
    },
          "tags" => [
        [0] "beats_input_codec_plain_applied"
    ],
       "message" => "207.241.237.101 - - [04/Jan/2015:05:22:27 +0000] \"GET /blog/tags/regex HTTP/1.0\" 200 14888 \"http://www.semicomplete.com/blog/tags/C\" \"Mozilla/5.0 (compatible; archive.org_bot +http://www.archive.org/details/archive.org_bot)\"",
      "@version" => "1",
    "@timestamp" => 2019-05-27T19:44:22.764Z
}
```
这里使用filebeat input插件从日志中获取记录已经完成

3. **日志格式处理**
* 在配置文件`first-pipeline-conf`中增加`filter`配置：
```
  filter {
        grok {
            match => { "message" => "%{COMBINEDAPACHELOG}"}
        }
    }
```
* 到filebeat的根目录下删除之前上报的数据历史，以方便与重新上报数据
```
rm -rf ./data/registry
```
* 上面设置了logstash可以自动更新配置，所以就不用重启了，可以看到输出日志如下：
```
{
       "referrer" => "\"http://www.semicomplete.com/projects/xdotool/\"",
            "ecs" => {
        "version" => "1.0.0"
    },
          "input" => {
        "type" => "log"
    },
          "agent" => {
            "hostname" => "localhost.localdomain",
        "ephemeral_id" => "9053b90e-4dcb-4a9f-89e4-972cf7c06f92",
             "version" => "7.1.0",
                  "id" => "541451b1-ec8d-41a2-82cc-7287d8c3c42d",
                "type" => "filebeat"
    },
       "response" => "200",
           "host" => {
                 "name" => "localhost.localdomain",
                   "os" => {
              "family" => "redhat",
             "version" => "7 (Core)",
                "name" => "CentOS Linux",
            "codename" => "Core",
              "kernel" => "3.10.0-862.el7.x86_64",
            "platform" => "centos"
        },
             "hostname" => "localhost.localdomain",
         "architecture" => "x86_64",
                   "id" => "e20692f79e4e409d8142632cec457b8f",
        "containerized" => true
    },
           "auth" => "-",
          "ident" => "-",
            "log" => {
        "offset" => 24248,
          "file" => {
            "path" => "/home/logs/logstash-tutorial.log"
        }
    },
    "httpversion" => "1.1",
        "request" => "/style2.css",
       "clientip" => "86.1.76.62",
      "timestamp" => "04/Jan/2015:05:30:37 +0000",
        "message" => "86.1.76.62 - - [04/Jan/2015:05:30:37 +0000] \"GET /style2.css HTTP/1.1\" 200 4877 \"http://www.semicomplete.com/projects/xdotool/\" \"Mozilla/5.0 (X11; Linux x86_64; rv:24.0) Gecko/20140205 Firefox/24.0 Iceweasel/24.3.0\"",
     "@timestamp" => 2019-05-27T20:28:06.987Z,
       "@version" => "1",
           "tags" => [
        [0] "beats_input_codec_plain_applied"
    ],
           "verb" => "GET",
          "bytes" => "4877"
}

```
好像按照某种规则格式化了，但是不明显；把日志文件里换简单一点的。

* 新建一个日志文件`logstash-second-log` 内容为：`55.3.244.1 GET /index.html 15824 0.043`，看一下输出情况：
```
{
    "@timestamp" => 2019-05-27T20:31:27.885Z,
           "ecs" => {
        "version" => "1.0.0"
    },
      "@version" => "1",
          "tags" => [
        [0] "beats_input_codec_plain_applied",
        [1] "_grokparsefailure"
    ],
         "agent" => {
             "version" => "7.1.0",
        "ephemeral_id" => "f4289690-d3dc-4bf1-9e04-0fbc678de6c1",
                "type" => "filebeat",
                  "id" => "541451b1-ec8d-41a2-82cc-7287d8c3c42d",
            "hostname" => "localhost.localdomain"
    },
         "input" => {
        "type" => "log"
    },
          "host" => {
                   "os" => {
              "family" => "redhat",
             "version" => "7 (Core)",
                "name" => "CentOS Linux",
              "kernel" => "3.10.0-862.el7.x86_64",
            "codename" => "Core",
            "platform" => "centos"
        },
                 "name" => "localhost.localdomain",
         "architecture" => "x86_64",
        "containerized" => true,
                   "id" => "e20692f79e4e409d8142632cec457b8f",
             "hostname" => "localhost.localdomain"
    },
       "message" => "55.3.244.1 GET /index.html 15824 0.043",
           "log" => {
        "offset" => 0,
          "file" => {
            "path" => "/home/logs/logstash-second-log"
        }
    }
}

```
这条日志可以切分成5个部分：IP(55.3.244.1),方法(GET),请求文件路径(/index.html),字节数(15824),接口返回时间(0.043),对这条日志的解析，正则如下：
`%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:bytes} %{NUMBER:duration}`
把配置文件改成这个正则后，在看看输出什么：
```
{
    "@timestamp" => 2019-05-27T20:38:13.727Z,
         "bytes" => "15824",
          "tags" => [
        [0] "beats_input_codec_plain_applied"
    ],
           "ecs" => {
        "version" => "1.0.0"
    },
       "message" => "55.3.244.1 GET /index.html 15824 0.043",
         "agent" => {
        "ephemeral_id" => "4ed6962c-7325-4aeb-84c2-324af2b2e1c5",
                "type" => "filebeat",
             "version" => "7.1.0",
                  "id" => "541451b1-ec8d-41a2-82cc-7287d8c3c42d",
            "hostname" => "localhost.localdomain"
    },
         "input" => {
        "type" => "log"
    },
          "host" => {
         "architecture" => "x86_64",
                   "id" => "e20692f79e4e409d8142632cec457b8f",
             "hostname" => "localhost.localdomain",
        "containerized" => true,
                 "name" => "localhost.localdomain",
                   "os" => {
             "version" => "7 (Core)",
              "kernel" => "3.10.0-862.el7.x86_64",
            "codename" => "Core",
            "platform" => "centos",
              "family" => "redhat",
                "name" => "CentOS Linux"
        }
    },
       "request" => "/index.html",
      "duration" => "0.043",
      "@version" => "1",
        "client" => "55.3.244.1",
        "method" => "GET",
           "log" => {
          "file" => {
            "path" => "/home/logs/logstash-second-log"
        },
        "offset" => 0
    }
}

```
可以看到`message`被格式化成：
```
   "request" => "/index.html",
      "duration" => "0.043",
      "@version" => "1",
        "client" => "55.3.244.1",
        "method" => "GET",

```
4. 将数据输出到es
es用root帐号启动会报错，我新建了一个用户，并启动es还是报错：
```
ERROR: [2] bootstrap checks failed
[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]
[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

```
网上找的解决方法，实测可用：
编辑`/etc/security/limits.conf`在文件里添加如下两行
```
*   hard    nofile  65536
*   soft    nofile  65536
```
编辑 `/etc/sysctl.conf`，追加以下内容：
`vm.max_map_count=655360`
保存后，执行：`sysctl -p`
重新启动es，测试一下是否正常：
`curl http://192.168.1.44:9200`
输出：
```
{
  "name" : "cvSqDVG",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "VzVQjf8YRzSRaE4dXsFe9A",
  "version" : {
    "number" : "5.5.2",
    "build_hash" : "b2f0c09",
    "build_date" : "2017-08-14T12:33:14.154Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.0"
  },
  "tagline" : "You Know, for Search"
}

```
说明es正常运行。现在配置output到es。

* 在`first-pipeline.conf`文件中把output改成：
```
output {
        elasticsearch {
            hosts => [ "localhost:9200" ]
        }
    }
```
在清理filebeat的历史数据，并且重启。使用curl 查询是否把日志写入到es
`curl -XGET 'http://192.168.1.44:9200/logstash-2019.05.27/_search?pretty&q=response=200'
`

```
  {
        "_index" : "logstash-2019.05.27",
        "_type" : "doc",
        "_id" : "AWr7StoMWwUTief8sYwt",
        "_score" : 0.14107859,
        "_source" : {
          "agent" : {
            "type" : "filebeat",
            "version" : "7.1.0",
            "ephemeral_id" : "14babd54-352b-444f-ad12-bb183ae64823",
            "hostname" : "localhost.localdomain",
            "id" : "541451b1-ec8d-41a2-82cc-7287d8c3c42d"
          },
          "@version" : "1",
          "input" : {
            "type" : "log"
          },
          "@timestamp" : "2019-05-27T21:53:29.868Z",
          "tags" : [
            "beats_input_codec_plain_applied",
            "_grokparsefailure"
          ],
          "ecs" : {
            "version" : "1.0.0"
          },
          "host" : {
            "name" : "localhost.localdomain",
            "architecture" : "x86_64",
            "containerized" : true,
            "os" : {
              "platform" : "centos",
              "version" : "7 (Core)",
              "name" : "CentOS Linux",
              "codename" : "Core",
              "family" : "redhat",
              "kernel" : "3.10.0-862.el7.x86_64"
            },
            "hostname" : "localhost.localdomain",
            "id" : "e20692f79e4e409d8142632cec457b8f"
          },
          "log" : {
            "offset" : 12023,
            "file" : {
              "path" : "/home/logs/logstash-tutorial.log"
            }
          },
          "message" : "200.49.190.101 - - [04/Jan/2015:05:17:39 +0000] \"GET /reset.css HTTP/1.1\" 200 1015 \"-\" \"-\""
        }
```
现在已经用logstash+filebeat+es实现了一个简单的`input`,`filter`,`output`的过程。

* logstash配置文件
配置文件在logstash根目录下
`vim ./config/logstash.yml`
配置文件描述引用网上找的[https://segmentfault.com/a/1190000016591476?utm_source=tag-newest](https://segmentfault.com/a/1190000016591476?utm_source=tag-newest)
个人觉得比较常用的应该是：
* `node.name`节点名称，配置多节点的时候肯定要用到
* `config.reload.automatic`默认是`false`，如果设置为`true`就是自动加载配置文件
* `config.reload.interval` 配置文件对这条属性的解释：`How often to check if the pipeline configuration has changed (in seconds)
`,简单来说就是可以配置一个时间(秒)，每隔几秒重载配置文件
上面的2个配置如果配置了，那么在启动时候就不需要加`--config.reload.automatic`
* `log.level`根据实际情况配置输出的日志级别
* `log.format`日志输出的格式
* `path.logs`logstash的日志保存路径

对于日志文件可以进行简单的封装，从上面启动logstash的命令`./bin/logstash -f /home/logstash-config/first-pipeline.conf --config.reload.automatic`可以看到
`-f`后面可以指定配置文件或者配置文件目录比如启动的时候加载`/logstash/conf/*.conf`，`/logstash/conf/`目录下所有`.conf`的配置都会加载。这样我们可以把logstash.yml文件根据配置属性的不同封装成几个配置文件：
* `log`的可以封装到`log.conf`里面
* `pipeline`的可以封装到`pipeline.conf`里面
等等。

这样做的好处是：
* 配置文件抽象化，类似于java的封装特性：把相同类型或者同一个类型的属性封装到一个`class`里面，保证`class`的纯洁性
* 方便维护。如果我想改日志输出级别，那么不需要打开原来的logstash.yml里面很多我不需要的属性，我只需要打开`log.conf`修改即可
* 不容易出问题。如果我只是想改log输出级别，在原有的logstash.yml文件里修改，如果不小心改到其他什么地方，会造成莫名其妙的报错，并且难以定位问题。但是在`log.conf`里面改则不会有这样的问题，如果改了此文件发现启动报错，那么很明显就是`log.conf`里有地方写错了。

以上是初步学习的一点记录，如果有错误，后面遇到了在改回来吧。