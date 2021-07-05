---
title: RuleX-开发指南
support: false
---

# 开发者指南
这里简要出一个开发者文档，只罗列关键代码，相信有开发经验的朋友一眼懂。

## 源码结构
```txt
-> % tree
.
├── Dockerfile
├── Makefile
├── clocs.md
├── conf
│   └── banner.txt
├── go.mod
├── go.sum
├── main.go
├── pic
│   └── 1.png
├── plugin
│   ├── http_api_server.go
│   └── templates
│       └── dashboard.html
├── ppt.md
├── readme.md
├── statistics
│   └── statistic.go
├── test
│   ├── calllua_test.go
│   ├── input_data_select_test.go
│   ├── lua
│   │   ├── callback.lua
│   │   ├── data_to_mongo.lua
│   │   ├── mix_test.lua
│   │   └── sql_select.lua
│   ├── publish.sh
│   └── rulenginex_test.go
└── x
    ├── coap_resource.go
    ├── dblib.go
    ├── decodelib.go
    ├── encodelib.go
    ├── http_resource.go
    ├── in_end.go
    ├── interface.go
    ├── jqlib.go
    ├── kafka_target.go
    ├── mongo_target.go
    ├── mqtt_resource.go
    ├── mysql_target.go
    ├── out_end.go
    ├── protobuf_resource.go
    ├── rule.go
    ├── rule_cache.go
    ├── rulenginex.go
    ├── utils.go
    ├── xhook.go
    ├── xpipline.go
    └── xplugin.go

```
## 插件开发
插件接口

```go

type XPlugin interface {
	Load(*RuleEngine) *XPluginEnv
	Init(*XPluginEnv) error
	Install(*XPluginEnv) (*XPluginMetaInfo, error)
	Start(*RuleEngine, *XPluginEnv) error
	Uninstall(*XPluginEnv) error
	Clean()
}

```
### 开发实例
本实例展示了使用插件接口来实现一个 RestAPI Server。
#### 核心代码
```go
package plugin

import (
	"context"
	"net/http"
	"rulenginex/statistics"
	"rulenginex/x"
	"runtime"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/ngaut/log"
	"github.com/shirou/gopsutil/cpu"
	"github.com/shirou/gopsutil/disk"
	"github.com/shirou/gopsutil/mem"
)

func init() {
	gin.SetMode(gin.ReleaseMode)
}

const API_ROOT string = "/api/v1/"
const DASHBOARD_ROOT string = "/dashboard/v1/"

type HttpApiServer struct {
	ginEngine  *gin.Engine
	RuleEngine *x.RuleEngine
}

func (hh *HttpApiServer) Load(r *x.RuleEngine) *x.XPluginEnv {
	hh.ginEngine = gin.New()
	hh.ginEngine.LoadHTMLGlob("plugin/templates/*")
	hh.RuleEngine = r
	return x.NewXPluginEnv()
}

//
func (hh *HttpApiServer) Init(env *x.XPluginEnv) error {

	ctx := context.Background()
	go func(ctx context.Context) {
		hh.ginEngine.Run(":2580")
	}(ctx)
	return nil
}
func (hh *HttpApiServer) Install(env *x.XPluginEnv) (*x.XPluginMetaInfo, error) {
	return &x.XPluginMetaInfo{
		Name:     "HttpApiServer",
		Version:  "0.0.1",
		Homepage: "www.ezlinker.cn",
		HelpLink: "www.ezlinker.cn",
		Author:   "wwhai",
		Email:    "cnwwhai@gmail.com",
		License:  "MIT",
	}, nil
}

//
//
func (hh *HttpApiServer) Start(e *x.RuleEngine, env *x.XPluginEnv) error {
	hh.ginEngine.GET(API_ROOT+"system", func(c *gin.Context) {
		cros(c)
		//
		percent, _ := cpu.Percent(time.Second, false)
		memInfo, _ := mem.VirtualMemory()
		parts, _ := disk.Partitions(true)
		diskInfo, _ := disk.Usage(parts[0].Mountpoint)
		c.JSON(http.StatusOK, gin.H{
			"diskInfo":   diskInfo.UsedPercent,
			"memInfo":    memInfo.UsedPercent,
			"cpuPercent": percent[0],
			"os":         runtime.GOOS,
			"arch":       runtime.GOARCH,
			"cpus":       runtime.GOMAXPROCS(0)})
	})
	//
	log.Info("Http web dashboard started on:http://127.0.0.1:2580" + DASHBOARD_ROOT)
	return nil
}

func (hh *HttpApiServer) Uninstall(env *x.XPluginEnv) error {
	log.Info("HttpApiServer Uninstalled")
	return nil
}
func (hh *HttpApiServer) Clean() {
	log.Info("HttpApiServer Cleaned")
}

```
#### 加载

## InEnd开发

InEnd接口
```go

type inEnd struct {
	Id          string                  `json:"id"`
	State       TargetState             `json:"state"`
	Type        string                  `json:"type"`
	Name        string                  `json:"name"`
	Description string                  `json:"description"`
	Binds       *map[string]rule        `json:"-"`
	Config      *map[string]interface{} `json:"config"`
}

```

### 开发实例
本实例演示了如何快速写一个资源输入端，下面的案例是一个串口数据输入端：

#### 核心代码

```go
package x

import (
	"time"

	"github.com/ngaut/log"
	"github.com/tarm/serial"
)

type SerialResource struct {
	*XStatus
	serialPort *serial.Port
}

func NewSerialResource(inEndId string) *SerialResource {
	s := SerialResource{}
	s.inEndId = inEndId
	return &s
}

func (s *SerialResource) Test(inEndId string) bool {
	return true
}

func (s *SerialResource) Register(inEndId string) error {
	return nil
}

func (s *SerialResource) Start(e *RuleEngine) error {
	config := e.GetInEnd(s.inEndId).Config
	name := (*config)["port"]
	baud := (*config)["baud"]
	readTimeout := (*config)["read_timeout"]
	size := (*config)["size"]
	parity := (*config)["parity"]
	stopbits := (*config)["stopbits"]

	serialPort, err := serial.OpenPort(&serial.Config{
		Name:        name.(string),
		Baud:        baud.(int),
		ReadTimeout: time.Duration(readTimeout.(int64)),
		Size:        size.(byte),
		Parity:      serial.Parity(parity.(int)),
		StopBits:    serial.StopBits(stopbits.(int)),
	})
	if err != nil {
		log.Error("SerialResource start failed:", err)
		return err
	} else {
		s.serialPort = serialPort
		log.Info("SerialResource start success:")
		return nil
	}
}

func (s *SerialResource) Enabled() bool {
	return true
}

func (s *SerialResource) Reload() {
}

func (s *SerialResource) Pause() {

}

func (s *SerialResource) Status(e *RuleEngine) State {
	return UP
}

func (s *SerialResource) Stop() {
}

```

#### 加载
```go
httpServer := plugin.HttpApiServer{}
if e := engine.LoadPlugin(&httpServer); e != nil {
    log.Fatal("rule load failed:", e)
}
```

```go
// 创建
in1 := x.NewInEnd("Serial", "串口输入", "这是个测试用的Lora模块", &map[string]interface{}{
	"name" : "/dev/ttyS1",
	"baud" : 115200,
	"readTimeout" : 5000,
	"size " : 1,
	"parity" : 1,
	"stopbits" : 0,
})
// 加载
if err := engine.LoadInEnd(in1); err != nil {
	log.Fatal("InEnd load failed:", err)
}
```


## OutEnd开发
OutEnd 接口

```go
type outEnd struct {
	Id          string                  `json:"id"`
	Type        string                  `json:"type"`
	State       TargetState             `json:"state"`
	Name        string                  `json:"name"`
	Description string                  `json:"description"`
	Config      *map[string]interface{} `json:"config"`
	Target      XTarget                 `json:"-"`
}

```

### 实例开发
本实例展示了如何编写一个数据持久化到 MongoDb 的出口。
#### 核心代码
```go
package x

import (
	"context"

	"github.com/ngaut/log"
	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

//
type MongoTarget struct {
	enabled    bool
	outEndId   string
	client     *mongo.Client
	collection *mongo.Collection
}

func NewMongoTarget() *MongoTarget {

	return &MongoTarget{
		enabled: false,
	}
}

func (m *MongoTarget) Register(outEndId string) error {
	m.outEndId = outEndId
	return nil
}

func (m *MongoTarget) Start(e *RuleEngine) error {
	config := e.GetOutEnd(m.outEndId).Config
	var clientOptions *options.ClientOptions
	if (*config)["mongourl"] != nil {
		clientOptions = options.Client().ApplyURI((*config)["mongourl"].(string))
	} else {
		clientOptions = options.Client().ApplyURI("mongodb://localhost:27017")
	}
	client, err0 := mongo.Connect(context.TODO(), clientOptions)
	if err0 != nil {
		return err0
	}

	if (*config)["database"] != nil {
		if (*config)["collection"] != nil {
			m.collection = client.Database((*config)["database"].(string)).Collection((*config)["collection"].(string))
		} else {
			m.collection = client.Database((*config)["mongourl"].(string)).Collection("rulex_data")

		}
	} else {
		m.collection = client.Database("rulex").Collection("rulex_data")
	}
	m.client = client
	m.enabled = true
	log.Info("Mongodb connect successfully")
	return nil

}

func (m *MongoTarget) Test(outEndId string) bool {
	err1 := m.client.Ping(context.Background(), nil)
	if err1 != nil {
		return false
	} else {
		return true
	}
}

func (m *MongoTarget) Enabled() bool {
	return m.enabled
}

func (m *MongoTarget) Reload() {
	log.Info("Mongotarget Reload success")
}

func (m *MongoTarget) Pause() {
	log.Info("Mongotarget Pause success")

}

func (m *MongoTarget) Status(e *RuleEngine) State {
	return e.GetOutEnd(m.outEndId).State
}

func (m *MongoTarget) Stop() {
	log.Info("Mongotarget Stop success")
}

func (m *MongoTarget) To(data interface{}) error {
	_, err := m.collection.InsertOne(context.TODO(), bson.D{{"data", data}})
	if err != nil {
		log.Error("Mongo To Failed:", err)
	}
	return err
}
 
```
#### 加载
```go
out1 := x.NewOutEnd("mongo", "Data to mongodb", "Save data to mongodb",
    &map[string]interface{}{
        "mongourl": "%%%%%%%%",
    })
if err1 := engine.LoadOutEnds(out1); err1 != nil {
    log.Fatal("OutEnd load failed:", err1)
}
```
---
## 完整案例
下面这个案例是一个相对完整的可以直接运行的网关：
```go
package main

import (
	"os"
	"os/signal"
	"rulenginex/plugin/http_server"
	"rulenginex/x"
	"syscall"

	"github.com/gin-gonic/gin"
	"github.com/ngaut/log"
)

//
var engine *x.RuleEngine

//
func main() {
	gin.SetMode(gin.ReleaseMode)
	c := make(chan os.Signal, 1)
	signal.Notify(c, syscall.SIGQUIT)
	engine = x.NewRuleEngine()

	in1 := x.NewInEnd("MQTT", "MQTT Stream", "MQTT Input Stream", &map[string]interface{}{
		"server":   "127.0.0.1",
		"port":     1883,
		"username": "test",
		"password": "test",
		"clientId": "test",
	})
	in1.Id = "INEND1"
	if err0 := engine.LoadInEnd(in1); err0 != nil {
		log.Fatal("InEnd load failed:", err0)

	}
	in2 := x.NewInEnd("COAP", "COAP Stream", "COAP Input Stream", &map[string]interface{}{
		"server": "127.0.0.1",
		"port":   1883,
	})
	in2.Id = "INEND2"
	if err := engine.LoadInEnd(in2); err != nil {
		log.Fatal("InEnd load failed:", err)
	}
	out1 := x.NewOutEnd("mongo", "Data to mongodb", "Save data to mongodb",
		&map[string]interface{}{
			"mongourl": "mongodb+srv://rulenginex:rulenginex@cluster0.rsdmb.mongodb.net/test",
		})
	out1.Id = "MongoDB001"
	if err1 := engine.LoadOutEnds(out1); err1 != nil {
		log.Fatal("OutEnd load failed:", err1)
	}
	actions := `
local json = require("json")
Actions = {
	function(data)
	    dataToMongo("MongoDB001", data)
	    print("[LUA Actions Callback]:dataToMongo Mqtt payload:", data)
		return true, data
	end
}
`
	from := []string{in1.Id}
	failed := `
function Failed(error)
  -- print("[LUA Callback] call failed from lua:", error)
end
`
	success := `
function Success()
  -- print("[LUA Callback] call success from lua")
end
`
	rule1 := x.NewRule(engine, "just_a_test_rule", "just_a_test_rule", from, success, actions, failed)
	rule1.Id = "just_a_test_rule"

	//
	if e := engine.LoadRule(rule1); e != nil {
		log.Fatal("rule load failed:", e)
	}
	httpServer := plugin.HttpApiServer{}
	if e := engine.LoadPlugin(&httpServer); e != nil {
		log.Fatal("rule load failed:", e)
	}
	engine.Start()
	<-c
	os.Exit(0)
}

```
> 注意：本页面涉及到的文档全部面向开发者群体而非终端用户，对于非研发用户，后期版本稳定以后会出详细的用户教程，敬请期待。