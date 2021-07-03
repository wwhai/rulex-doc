---
title: HTTP API
support: false
---
## RuleX 基础 HTTP API
RuleX 本身是个框架，不包含 HTTP 接口，但是支持插件扩展，下面这些 HTTP API 就是通过 `HttpApiServer` 插件实现的。

### 系统数据
-  curl 127.0.0.1:2580/api/v1/system
```json
{
  "arch": "amd64",
  "cpuPercent": 3.8053649407883903,
  "cpus": 16,
  "diskInfo": 7.107930010513884,
  "memInfo": 66.0473108291626,
  "os": "darwin"
}
```
### 统计数据
- curl 127.0.0.1:2580/api/v1/statistic
```json
{
  "statistics": {
    "inFailed": 0,
    "inSuccess": 0,
    "outFailed": 0,
    "outSuccess": 0
  }
}
```
### 数据入口
- curl 127.0.0.1:2580/api/v1/inends

```json
{
  "inends": {
    "INEND1": {
      "id": "INEND1",
      "state": 1,
      "type": "MQTT",
      "name": "MQTT Stream",
      "description": "MQTT Input Stream",
      "config": {
        "clientId": "test",
        "password": "test",
        "port": 1883,
        "server": "127.0.0.1",
        "username": "test"
      }
    },
    "INEND2": {
      "id": "INEND2",
      "state": 1,
      "type": "COAP",
      "name": "COAP Stream",
      "description": "COAP Input Stream",
      "config": {
        "port": 1883,
        "server": "127.0.0.1"
      }
    }
  }
}
```
### 数据出口
- curl 127.0.0.1:2580/api/v1/outends

```json
{
  "outends": {
    "MongoDB001": {
      "id": "MongoDB001",
      "type": "mongo",
      "state": 1,
      "name": "Data to mongodb",
      "description": "Save data to mongodb",
      "config": {
        "mongourl": "mongodb+srv://rulenginex:rulenginex@cluster0.rsdmb.mongodb.net/test"
      }
    }
  }
}
```
### 规则列表
- curl 127.0.0.1:2580/api/v1/rules
```json
{
  "rules": {
    "just_a_test_rule": {
      "id": "just_a_test_rule",
      "name": "just_a_test_rule",
      "description": "just_a_test_rule",
      "from": [
        "INEND1"
      ],
      "actions": "\nlocal json = require(\"json\")\nActions = {\n\tfunction(data)\n\t    dataToMongo(\"MongoDB001\", data)\n\t    print(\"[LUA Actions Callback]:dataToMongo Mqtt payload:\", data)\n\t\treturn true, data\n\tend\n}\n",
      "success": "\nfunction Success()\n  -- print(\"[LUA Callback] call success from lua\")\nend\n",
      "failed": "\nfunction Failed(error)\n  -- print(\"[LUA Callback] call failed from lua:\", error)\nend\n"
    }
  }
}
```
### 插件列表
- curl 127.0.0.1:2580/api/v1/plugins

```json
{
  "plugins": {
    "HttpApiServer": {
      "name": "HttpApiServer",
      "version": "0.0.1",
      "homepage": "www.ezlinker.cn",
      "helpLink": "www.ezlinker.cn",
      "author": "wwhai",
      "email": "cnwwhai@gmail.com",
      "license": "MIT"
    }
  }
}
```

> 因为接口不是本规则引擎的核心功能，所以仅仅做了几个基础数据查看接口，后期随着版本迭代会逐步完善起来。