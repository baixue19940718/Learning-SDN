# Simple Switch with Restful

在 [Simple Switch](https://github.com/imac-cloud/SDN-tutorial/tree/master/Ryu/SimpleSwitch) 中，已經瞭解了在 Ryu 中，Switch 的基本運作模式。在此，將把這個基本的 Switch 進行擴充，讓它可以更實際、方便的運用。以下爲本章重點：

* 將 Switch 結合 Restful API 的機制。
* 本程式將會由兩個 class 完成。一個是 Switch 本身，另一個是 Controller 的部分。
* 分別使用到的 Restful 方法為```GET```（取得資料列表）及```PUT```（新增 Entry）。

接下來，就從程式碼開始說明。

## 開頭宣告

```python
import json

from ryu.app import simple_switch_13
from ryu.controller import ofp_event
from ryu.controller.handler import CONFIG_DISPATCHER
from ryu.controller.handler import set_ev_cls
from ryu.lib import dpid as dpid_lib
from webob import Response
from ryu.app.wsgi import ControllerBase, WSGIApplication, route

simple_switch_instance_name = 'simple_switch_api_app'
url = '/simpleswitch/mactable/{dpid}'
```

### import json
將統計數據轉換成```json```格式，作為 API 輸出時的資料格式，方便使用者運用。

### simple\_switch\_13

Ryu 中 Switch 的雛形。在此專案中，藉由繼承，實現 Switch 的基本功能。 

### ofp_event

引入 Openflow Protocol 的事件。

### ryu.controller 的事件類別名稱

* CONFIG_DISPATCHER：接收 SwitchFeatures
* MAIN_DISPATCHER：一般狀態 //在此沒有用到
* DEAD_DISPATCHER：連線中斷 //在此沒有用到
* HANDSHAKE_DISPATCHER：交換 HELLO 訊息 //在此沒有用到

### set\_ev\_cls

當裝飾器使用。因 Ryu 皆受到任何一個 OpenFlow 的訊息，都會需要產生一個對應的事件。為了達到這樣的目的，透過 set\_ev\_cls 當裝飾器，依接收到的參數（事件類別、 Switch 狀態），而進行反應。

### dpid as dpid_lib

在此運用其中對 Datapath ID 的規範```DPID_PATTERN```。當接收到 Restful 請求時，過濾 Datapath ID 是否符合格式。

### Response

回傳 Restful 請求。

### ryu.app.wsgi
WSGI 本身為 Python 中 Web Application 的框架。在 Ryu 中，透過它，建立 Web Service，提供 Restful API。

### simple\_switch\_instance\_name
以參數儲存程式中，字典用到的 ```key```。在這裡，用來代表專案中的 Switch。

### url

Restful API 對應的 URL。

## Switch Class
接下來說明，專案中 Switch 的部分。

### 初始化
繼承```simple_switch_13.SimpleSwitch13```，實現 Switch 的基本功能。

```python
class SimpleSwitchRest13(simple_switch_13.SimpleSwitch13):
	_CONTEXTS = {'wsgi':WSGIApplication}

	def __init__(self, *args, **kwargs):
		super(SimpleSwitchRest13, self).__init__(*args, **kwargs)
		self.switches = {}
		wsgi = kwargs['wsgi']
		wsgi.register(SimpleSwitchController, {simple_switch_instance_name : self})
```
#### \_CONTEXTS = {'wsgi':WSGIApplication}
變數```_CONTEXTS```為類別變數。在此將 Ryu 中的 WSGI 所運用的模式，定義成```WSGIApplication```。

> 在執行此專案的時候，注意一下 console output。可以看見：
> 
> ```shell
> #...
> creating context wsgi
> #...
> ```
> wsgi 將在此時被建立。在初始化時（init），由```kwargs```接收。

#### self.switches = {}
建立字典，用來存放 Datapath。```key```為 Datapath 的 ID。

#### wsgi = kwargs['wsgi']
取出 wsgi，並將 Controller（```SimpleSwitchController```） 註冊其中。

```python
wsgi.register(SimpleSwitchController, {simple_switch_instance_name : self})
```

> ```register```的第二個參數是一個字典，用意就是把 Switch 本身傳給 Controller。讓 Controller 可以自由取用 Switch。 

### SwitchFeatures 事件
覆寫```switch_features_handler```，目的是將接收到的 Datapath 存入 ```self.switches``` 供後續使用。
```python
@set_ev_cls(ofp_event.EventOFPSwitchFeatures, CONFIG_DISPATCHER)
	def switch_features_handler(self, ev):
		super(SimpleSwitchRest13, self).switch_features_handler(ev)
		datapath = ev.msg.datapath
		self.switches[datapath.id] = datapath
		self.mac_to_port.setdefault(datapath.id, {})
```

### 加入 新的設備資訊函式
接收到新的設備資訊（```entry```）後，將 Entry 新增入 MAC TABLE 中。
> 接收到 Restful ```PUT```時，控制函式將會呼叫此函式，並給予設備資訊（```entry```）和 Datapath ID，供此函式新增 Flow Entry 入指定的 Datapath 中。

```python
def set_mac_to_port(self, dpid, entry):
		mac_table = self.mac_to_port.setdefault(dpid, {})
		datapath = self.switches.get(dpid)

		entry_port = entry['port']
		entry_mac = entry['mac']
		if datapath is not None:
			parser = datapath.ofproto_parser
			if entry_port not in mac_table.values():
				for mac, port in mac_table.items():
					actions = [parser.OFPActionOutput(entry_port)]
					match = parser.OFPMatch(in_port=port, eth_dst=entry_mac)
					self.add_flow(datapath, 1, match, actions)
					
					actions = [parser.OFPActionOutput(port)]
					match = parser.OFPMatch(in_port=entry_port, eth_dst=mac)
					self.add_flow(datapath, 1, match, actions)
				mac_table.update({entry_mac : entry_port})

		return mac_table
```
#### 取得 MAC TABLE 及指定 Datapath 資訊
```python
mac_table = self.mac_to_port.setdefault(dpid, {})
datapath = self.switches.get(dpid)
```

#### 如果指定的 Datapath 存在，且此筆設備資訊（```entry```）是新的。則將它加入 MAC TABLE 中
加入新的設備資訊（```entry```）及原先 MAC TABLE 中的設備資訊之間的 Flow Entry。

```python
# from known device to new device
actions = [parser.OFPActionOutput(entry_port)]
match = parser.OFPMatch(in_port=port, eth_dst=entry_mac)
self.add_flow(datapath, 1, match, actions)

# from new device to known device
actions = [parser.OFPActionOutput(port)]
match = parser.OFPMatch(in_port=entry_port, eth_dst=mac)
self.add_flow(datapath, 1, match, actions)
```
#### 更新 MAC TABLE 並回傳
```python
    mac_table.update({entry_mac : entry_port})
return mac_table
```
## Controller Class
接下來說明，專案中 Controller 的部分。
## 參考

[Ryubook](https://osrg.github.io/ryu-book/zh_tw/html/)