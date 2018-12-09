---
title: kibana(6.2.4)集成sentinl报警
tags:
  - ELK
toc: true
categories:
  - 运维
date: 2018-12-07 12:49:54
---

# 1. 安装
```sh
kibana_path/bin/kibana-plugin install https://github.com/sirensolutions/sentinl/releases/download/tag-6.2.4/sentinl-v6.2.4.zip
```
安装完成之后，重启kibana。在kibana页面可以看到相关的菜单
<img src="/image/elk/sentinl-dashborad.png">

# 2. 配置sentinl
## 2.1 配置邮箱
在`kibana.yml`中配置邮箱参数：
<!-- more -->
```yml
sentinl:
  settings:
    email:
      active: true
      user: xxx@xxx.com
      password: xxx
      host: smtp服务器
      ssl: true
      port: xxx
      timeout: 10000

```

配置完成之后重启服务
## 2.2 配置页面数据  

<img src="/image/elk/sentinl-new.png">
选择wizard或者advanced(最终都要转换成advanced，wizard更容易理解)  

<img src="/image/elk/sentinl-watcher.png">
参考watcher数据如下:
```json
{
  "actions": {
    "email_html_alarm_76b83c8f-0f4a-4db5-8a15-185933e17ca2": {
      "name": "测试日志异常",
      "throttle_period": "2m",
      "email_html": {
        "stateless": false,
        "subject": "日志异常邮件测试",
        "priority": "medium",
        "html": "<p>Hi {{watcher.username}}</p>\n<p>There are {{payload.hits.total}} results found by the watcher <i>{{watcher.title}}</i>.</p>\n\n<div style=\"color:grey;\">\n  <hr />\n  <p>This watcher sends alerts based on the following criteria:</p>\n  <ul><li>{{watcher.wizard.chart_query_params.queryType}} of {{watcher.wizard.chart_query_params.over.type}} over the last {{watcher.wizard.chart_query_params.last.n}} {{watcher.wizard.chart_query_params.last.unit}} {{watcher.wizard.chart_query_params.threshold.direction}} {{watcher.wizard.chart_query_params.threshold.n}} in index {{watcher.wizard.chart_query_params.index}}</li></ul>\n</div>\n\n<div>\n异常信息如下:\n{{#payload.hits.hits}} {{_source.message}} \n \n \n{{/payload.hits.hits}} \n</div>",
        "to": "xxx@sina.cn",
        "from": "ddd@qq.com"
      }
    },
    "Webhook_f3303006-a643-42f6-a2ff-8d4066d18c3a": {
      "name": "webhook告警",
      "throttle_period": "2m",
      "webhook": {
        "priority": "medium",
        "stateless": false,
        "method": "POST",
        "host": "oapi.dingtalk.com",
        "port": "443",
        "path": "/robot/send?access_token=xxxx",
        "body": "{\r\n    \"msgtype\": \"markdown\",\r\n    \"at\": {\r\n        \"isAtAll\": \"True\"\r\n    },\r\n    \"markdown\": {\r\n        \"title\": \"异常消息\",\r\n        \"text\": \" 异常日志: \\n {{#payload.hits.hits}} {{_source.message}} \r\n \r\n{{/payload.hits.hits}}\"\r\n    }\r\n}",
        "params": {
          "watcher": "{{watcher.title}}",
          "payload_count": "{{payload.hits.total}}"
        },
        "headers": {
          "Content-Type": "application/json"
        },
        "message": "生产环境异常",
        "use_https": true
      }
    }
  },
  "input": {
    "search": {
      "request": {
        "index": [
          "xxx*"
        ],
        "body": {
          "query": {
            "bool": {
              "must": {
                "match": {
                  "message": "ERROR"
                }
              },
              "filter": {
                "range": {
                  "@timestamp": {
                    "gte": "now-15m/m",
                    "lte": "now/m",
                    "format": "epoch_millis"
                  }
                }
              }
            }
          },
          "size": 2,
          "aggs": {
            "dateAgg": {
              "date_histogram": {
                "field": "@timestamp",
                "time_zone": "Asia/Shanghai",
                "interval": "1m",
                "min_doc_count": 1
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "script": {
      "script": "payload.hits.total >= 1"
    }
  },
  "trigger": {
    "schedule": {
      "later": "every 1 minutes"
    }
  },
  "disable": false,
  "report": false,
  "title": "测试告警",
  "wizard": {},
  "save_payload": false,
  "spy": false,
  "impersonate": false
}
```
如图选中开启日志报警
<img src="/image/elk/sentinl-test.png">
也可以点击测试
<img src="/image/elk/sentinl-test2.png">
