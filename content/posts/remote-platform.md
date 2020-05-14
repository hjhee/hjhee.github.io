+++
title = "Remote Monitor Platform Development"
date = 2016-06-29T01:25:20+00:00
tags = ["Go"]
draft = false
+++

# 运行环境

程序运行在阿里云云主机上，跑Gentoo Linux系统。服务器架构如论文和PPT描述。
服务器开放UDP端口5683与设备通信，目前采用自定义协议，以后可以尝试CoAP协议，但目前的GPRS模块通信质量比较差，使用CoAP协议需要考虑。
80端口开放http服务，用nginx反向代理，url结构如nginx配置文件所示。

<!--more-->

```nginx
server {
    listen 80;
    #listen 127.0.0.1;
    server_name localhost;

    access_log /var/log/nginx/localhost.access_log main;
    error_log /var/log/nginx/localhost.error_log info;

    root /var/www/localhost/htdocs;

    location /api {
        proxy_pass http://localhost:8080;
    }

    location /api/events {
        proxy_pass http://localhost:9952;
        proxy_set_header Connection '';
        proxy_http_version 1.1;
        chunked_transfer_encoding off;
        proxy_buffering off;
        proxy_cache off;
    }

    location /node {
        proxy_pass http://localhost:10242;
    }

    location /api/status {
        proxy_pass http://localhost:10242;
    }
}
```

sse-server.go编写了Sever-Sent Event的实现，通过/tmp/gasdata.sock从udp-server.go获取气体数据，nginx将/api/events路由给sse-server.go。

udp-server.go负责设备通信，它和其他服务器通过Unix Socket交换数据。

templateServer.go负责生成HTML模板。

上述服务器都通过go语言实现

# sse-server

go语言的sse实现，很大程度参考了网上的例子[Writing a Server Sent Events server in Go](https://www.new-bamboo.co.uk/blog/2014/05/13/writing-a-server-sent-events-server-in-go/)。

Broker维护了客户端列表，数据都通过chan广播给客户端。

```go
// This happens in a goroutine somewhere else.
broker.Notifier <- eventBytes
```

Device结构定义了发送给浏览器的json数据格式，Data []float64是气体浓度数组。

```go
package main

import (
    "time"
    "net/http"
    "net"

    "github.com/justinas/alice"
    "os"
    "fmt"

    "github.com/op/go-logging"
    "encoding/binary"
    "encoding/json"
    "math"
)

var log = logging.MustGetLogger("sselog")
var logFormat = logging.MustStringFormatter(
    `%{color}%{time:15:04:05.000} %{shortfunc} | %{level:.4s} %{id:03x}%{color:reset} %{message}`,
)

var bchan = make(chan Device, 20)

type Device struct {
    Id int `json:"id"`
    Data []float64 `json:"data"`
}

// A MessageChan is a channel of channels
// Each connection sends a channel of bytes to a global MessageChan
// The main broker listen() loop listens on new connections on MessageChan
// New event messages are broadcast to all registered connection channels
type MessageChan chan []byte

// A Broker holds open client connections,
// listens for incoming events on its Notifier channel
// and broadcast event data to all registered connections
type Broker struct {

    // Events are pushed to this channel by the main events-gathering routine
    Notifier chan []byte

    // New client connections
    newClients chan chan []byte

    // Closed client connections
    closingClients chan chan []byte

    // Client connections registry
    clients map[chan []byte]bool
}

var broker = NewServer()

// Broker factory
func NewServer() (broker *Broker) {
  // Instantiate a broker
  broker = &Broker{
    Notifier:       make(chan []byte, 1),
    newClients:     make(chan chan []byte),
    closingClients: make(chan chan []byte),
    clients:    make(map[chan []byte]bool),
  }

  // Set it running - listening and broadcasting events
  go broker.listen()

  return
}

// Listen on different channels and act accordingly
func (broker *Broker) listen() {
    for {
        select {
        case s := <-broker.newClients:

            // A new client has connected.
            // Register their message channel
            broker.clients[s] = true
            log.Infof("Client added. %d registered clients", len(broker.clients))
        case s := <-broker.closingClients:

            // A client has dettached and we want to
            // stop sending them messages.
            delete(broker.clients, s)
            log.Infof("Removed client. %d registered clients", len(broker.clients))
        case event := <-broker.Notifier:

            // We got a new event from the outside!
            // Send event to all connected clients
            for clientMessageChan, _ := range broker.clients {
                clientMessageChan <- event
            }
        }
    }

}

func ByteToFloat64(b []byte) float64 {
    bits := binary.BigEndian.Uint64(b)
    return math.Float64frombits(bits)
}

func loggingHandler(next http.Handler) http.Handler {
    fn := func(w http.ResponseWriter, r *http.Request) {
        t1 := time.Now()
        next.ServeHTTP(w, r)
        t2 := time.Now()
        log.Infof("[%s] %q %v\n", r.Method, r.URL.String(), t2.Sub(t1))
    }

    return http.HandlerFunc(fn)
}

func (broker *Broker) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "Streaming unsupported!", http.StatusInternalServerError)
        return
    }
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    w.Header().Set("Access-Control-Allow-Origin", "*")

    // Each connection registers its own message channel with the Broker's connections registry
    messageChan := make(MessageChan)

    // Signal the broker that we have a new connection
    broker.newClients <- messageChan

    // Remove this client from the map of connected clients
    // when this handler exits.
    defer func() {
        broker.closingClients <- messageChan
    }()

    // Listen to connection close and un-register messageChan
    notify := w.(http.CloseNotifier).CloseNotify()

    go func() {
        <-notify
        broker.closingClients <- messageChan
    }()

    for {
        fmt.Fprintf(w, "data: %s\n\n", <-messageChan)
        flusher.Flush()
    }
}

func socketHandler() {
    var socket = "/tmp/gasdata.sock"
    os.Remove(socket)
    conn, err := net.ListenPacket("unixgram", socket)
    if err != nil {
        log.Fatalf("unix socket listen failed: %s\n", err.Error())
    }
    defer conn.Close()
    log.Infof("conn address: %s\n", conn.LocalAddr())

    var buff = make([]byte, 512)
    for {
        _, _, err := conn.ReadFrom(buff)
        if err != nil {
            log.Warningf("unix socket read failed: %s\n", err.Error())
            continue
        }
        var dev Device
        dev.Id = int(buff[0])
        dev.Data = make([]float64, 7)
        for i:=0; i<7; i++ {
            dev.Data[i] = ByteToFloat64(buff[1+4+8*i:])
        }
        //log.Info("new data: %+v\n", dev)
        jstr, err := json.Marshal(dev)
        if err != nil {
            log.Warningf("json marshal failed: %s\n", err.Error())
            continue
        }
        broker.Notifier <- jstr
    }
}

func main() {
    logging.SetFormatter(logFormat)

    go socketHandler()

    commonHandlers := alice.New(loggingHandler)
    http.Handle("/api/events/data.get", commonHandlers.Then(broker))
    http.ListenAndServe("localhost:9952", nil)
}
```


# udp-server

handleClient获取新的数据包，根据帧头分包，交给payloadHandler处理。payloadHandler根据devid维护设备的IP地址，sessionConn[devid]表示了与devid对应的对话。sessionHandler负责发送数据给设备，一旦设备发送的心跳包超时，则将设备的状态(client[id])更新为false。

```go
package main

import (
    "net"
    "bytes"
    "encoding/json"
    "encoding/binary"
    "math"
    "github.com/op/go-logging"
    "os"
    "time"
)

type Dev struct {
    Id int `json:"id"`
    Cmd string `json:"cmd"`
}

type Status struct {
    Id int `json:"id"`
    Status int `json:"status"`
    Type int `json:"type"`
}

var log = logging.MustGetLogger("gaslog")
var logFormat = logging.MustStringFormatter(
    `%{color}%{time:15:04:05.000} %{shortfunc} | %{level:.4s} %{id:03x}%{color:reset} %{message}`,
)
var socket net.Conn
var client = make(map[int]bool)
var clientAddr = make([]net.Addr, 512)
var sessionConn = make([]*net.UDPConn, 512)
var sessionChan [512]chan []byte

func ByteToFloat64(b []byte) float64 {
    bits := binary.BigEndian.Uint64(b)
    return math.Float64frombits(bits)
}

func payloadHandler(conn *net.UDPConn, addr *net.UDPAddr, buff []byte) {
    //log.Infof("new payload: %+v\n", buff)

    msgtype, devid := buff[0], int(buff[1])
    if client[devid] == false {
        client[devid] =true
        clientAddr[devid] = addr
        sessionConn[devid] = conn
        go sessionHandler(devid)
    } else if clientAddr[devid] != addr {
        clientAddr[devid] = addr
        sessionConn[devid] = conn
    }

    switch msgtype {
    case 0x0: // heartbeat
        var payload = make([]byte, 2)
        payload[0] = 0
        payload[1] = 1
        sessionChan[devid] <- payload
        //conn.WriteTo([]byte("OK"), addr)
    case 0x1: // gas data
        //log.Noticef("buff: %+v\n", buff[2:60])
        socket.Write(buff[1:])
        gasdata := make([]float64, 7)
        for i:=0; i<7; i++ {
            gasdata[i] = ByteToFloat64(buff[2+4+8*i:])
        }
        log.Infof("%d: %f %f %f %f %f %f %f\n", devid, gasdata[0], gasdata[1], gasdata[2], gasdata[3], gasdata[4], gasdata[5], gasdata[6])
        //conn.WriteToUDP([]byte("OK"), addr)
    case 0x2:
        status := buff[2]
        log.Infof("%d: status changed: %d\n", devid, status)
    }
}

func handleClient(conn *net.UDPConn) {
    var buff = make([]byte, 512)
    n, addr, err := conn.ReadFromUDP(buff[0:])
    if err != nil {
        return
    }

    payload := make([]byte, n)
    copy(payload, buff)

    //log.Infof("payload.length=%d\n", n)
    for i := bytes.Index(payload[0:], []byte("hjhee")); i != -1; i = bytes.Index(payload[0:], []byte("hjhee")) {
        //log.Infof("payload: %+v\n", payload[i+5:])
        go payloadHandler(conn, addr, payload[i+5:])
        payload = payload[i+5:]
    }
}

func sessionHandler(id int) {
    for {
        select {
        case b := <-sessionChan[id]:
            switch b[0] {
            case 0x0: // heartbeat
                log.Infof("new heartbeat[id=%d]: %s\n", id, clientAddr[id])
                conn := sessionConn[id]
                conn.WriteTo([]byte("OK"), clientAddr[id])
            case 0x1: // device status update
                conn := sessionConn[id]
                conn.WriteTo(b, clientAddr[id])
                log.Infof("status update[addr=%s]: %d\n", clientAddr[id], b[1])
            case 0x3: // status query
                conn := sessionConn[id]
                conn.WriteTo(b, clientAddr[id])
            default:
                log.Infof("invalid data[id=%d]\n", id)
            }
        case <-time.After(30 * time.Second):
            log.Noticef("heartbeat timeout: id[%d]\n", id)
            client[id] = false
            return
        }
    }
}

func devHandler() {
    var devSocket = "/tmp/devstatus.sock"
    os.Remove(devSocket)
    conn, err := net.ListenPacket("unixgram", devSocket)
    if err != nil {
        log.Fatalf("devSocket create failed: %s\n", err.Error())
    }
    defer conn.Close()
    log.Infof("devSocket local addr: %s\n", conn.LocalAddr())

    var buff = make([]byte, 512)
    for {
        n, _, err := conn.ReadFrom(buff)
        if err != nil {
            log.Warningf("devSocket read failed: %s\n", err.Error())
            continue
        }
        reader := bytes.NewReader(buff)
        decoder := json.NewDecoder(reader)
        var devCmd Dev
        err = decoder.Decode(&devCmd)
        if err != nil {
            log.Warningf("json decode failed: %s(error: %s)\n", string(buff[0:n]), err.Error())
            continue
        }
        log.Infof("new command: %+v\n", devCmd)
        switch devCmd.Cmd {
        case "devStart":
            var payload = make([]byte, 2)
            payload[0] = 1
            payload[1] = 0x3
            sessionChan[devCmd.Id] <- payload
        case "devStop":
            var payload = make([]byte, 2)
            payload[0] = 1
            payload[1] = 0x0
            sessionChan[devCmd.Id] <- payload
        case "devQuery":
            var payload = make([]byte, 1)
            payload[0] =2
            if client[devCmd.Id] {
                sessionChan[devCmd.Id] <- payload
            } else {
                status := Status{Id: devCmd.Id, Status: -1, Type: -1}
                statusStr, _ := json.Marshal(status)
                log.Infof("status[id=%d]=%s\n", status.Id, statusStr)
            }
            log.Infof("new Query[id=%d]\n", devCmd.Id)
        }
    }
}

func main() {
    logging.SetFormatter(logFormat)

    for i:=0; i<512; i++ {
        sessionChan[i] = make(chan []byte, 5)
    }

    go devHandler()

    var err error
    socket, err = net.Dial("unixgram", "/tmp/gasdata.sock")
    if err != nil {
        log.Fatal("Unix Domain Socket create failed")
    }
    defer socket.Close()
    udpAddr, _ := net.ResolveUDPAddr("udp4", ":5683")
    conn, _ := net.ListenUDP("udp", udpAddr)
    defer conn.Close()
    for {
        handleClient(conn)
    }
}
```


# templateServer

templateServer负责生成HTML模板，可以预处理加快生成速度。另一方面也提供REST API，/api/status/:id用于给设备发送命令变更工作状态，GET操作查询，POST操作更新。

```go
package main

import (
    "net"
    "net/http"
    "html/template"
    "github.com/julienschmidt/httprouter"
    "github.com/op/go-logging"
    "io/ioutil"
    "strconv"
    "encoding/json"
)

var log = logging.MustGetLogger("sselog")
var logFormat = logging.MustStringFormatter(
    `%{color}%{time:15:04:05.000} %{shortfunc} | %{level:.4s} %{id:03x}%{color:reset} %{message}`,
)

var devSocket net.Conn

type Dev struct {
    Id int `json:"id"`
    Cmd string `json:"cmd"`
}

func newTemplateHandler() (func(w http.ResponseWriter, r *http.Request, ps httprouter.Params), error) {
/*
    tmpl, err := template.ParseFiles("templates/layout.html")
    if err != nil {
        return nil, err
    }
*/
    return func(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
        id, _ := strconv.Atoi(ps.ByName("id"))
        tmpl, err := template.ParseFiles("templates/layout.html")
        if err != nil {
            log.Errorf("template Parse failed: %s\n", err.Error())
            return
        }
        err = tmpl.Execute(w, Dev{Id: id})
        if err != nil {
            log.Errorf("template execute failed: %s\n", err.Error())
        }
    }, nil
}

func statusHandler(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
    id, _ := strconv.Atoi(ps.ByName("id"))
    if r.Method == "POST" {
        result, _ := ioutil.ReadAll(r.Body)
        r.Body.Close()
        devSocket.Write([]byte(result))
        log.Infof("status.post: %s\n", result)
    }
    if r.Method == "GET" {
        devCmd := Dev{Id: id, Cmd: "devQuery"}
        cmd, _ := json.Marshal(devCmd)
        devSocket.Write([]byte(cmd))
    }
}

func main() {
    logging.SetFormatter(logFormat)

    var err error
    devSocket, err = net.Dial("unixgram", "/tmp/devstatus.sock")
    if err != nil {
        log.Fatalf("socket dial failed: %s\n", err.Error())
    }
    templateHandler, err := newTemplateHandler()
    if err != nil {
        log.Fatalf("template parse failed: %s\n", err.Error())
    }

    router := httprouter.New()
    router.GET("/node/:id", templateHandler)
    router.POST("/api/status/:id", statusHandler)
    router.GET("/api/status/:id", statusHandler)
    log.Fatal(http.ListenAndServe("localhost:10242", router))
}
```

HTML 模板

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>Dev{{.Id}}</title>
	<link rel="stylesheet" href="/css/chartist.min.css">
	<script src="/js/jquery-2.2.4.min.js"></script>
	<script src="/js/chartist.min.js"></script>
	<script src="/js/chartist-plugin-axistitle.js"></script>
	<script>
$(document).ready(function(){
	var data = {
		// A labels array that can contain any sort of values
		labels: [],
		// Our series array that contains series objects or in this case series data arrays
		series: [{
			name: "SO2",
			data: []
		}, {
			name: "NO2",
			data: []
		}, {
			name: "O3",
			data: []
		}, {
			name: "CO",
			data: []
		}, {
			name: "RH",
			data: []
		}, {
			name: "Temp",
			data: []
		}]
	};

	// Create a new line chart object where as first parameter we pass in a selector
	// that is resolving to our chart container element. The Second parameter
	// is the actual data object.
	chart = new Chartist.Line('.ct-chart', data, {
		chartPadding: {
			top: 30,
			right: 0,
			bottom: 30,
			left: 20
		},
		plugins: [
			Chartist.plugins.ctAxisTitle({
				axisX: {
					axisTitle: 'Time (mins)',
					axisClass: 'ct-axis-title',
					offset: {
						x: 0,
						y: 50
					},
					textAnchor: 'middle'
				},
				axisY: {
					axisTitle: 'Concentrations (ppb)',
					axisClass: 'ct-axis-title',
					offset: {
						x: 0,
						y: -1
					},
					flipTitle: false
				}
			})
		]
	});
	
	var source = new EventSource("/api/events/data.get");
	var indx = 0;
	var devid = window.location.pathname.split('/').pop();
	source.onmessage = function(evt) {
		var payload = JSON.parse(evt.data);
		if(payload.id != devid){
			return;
		}
		chart.data.labels.push(indx);
		indx++;
		if(chart.data.labels.length>10)
			chart.data.labels.shift();
		for(var i=0; i<chart.data.series.length; i++){
			var s = chart.data.series[i];
			s.data.push(Math.sin(Math.random()*2*Math.PI));
			if(s.data.length>10){
				s.data.shift();
			}
		}
		chart.update();
		
		var row = '';
		var row = '<tr><th>' + Math.round(payload.data[0]) + '</th>';
		for(var i=1; i<7; i++){
			row += '<th>' + payload.data[i].toPrecision(4) + '</th>';
		}
		$('#tbCon tbody').prepend(row);
		var rows = $('#tbCon tbody tr');
		if(rows.length > 10){
			rows.last().remove();
		}
	}
});
</script>
<script>
$(document).ready(function(){
	var devid = window.location.pathname.split('/').pop();
	
	$('#devCmd').submit(function(evt){
		evt.preventDefault();
		var cmd = $(document.activeElement).attr('name');
		var serize = JSON.stringify({id: parseInt(devid), cmd: cmd});
		$.post("/api/status/" + devid, serize);
		console.log('request sent:' + serize);
	});
	
	var devStatusSource = new EventSource("/api/event/status/" + devid);
	devStatusSource.onmessage = function(evt) {
		var payload = JSON.parse(evt.data);
		
		switch(payload.status) {
		case 0:
			$('#devStatus').text("停止采集");
			break;
		case 1:
			$('#devStatus').text("正在采集");
			break;
		default:
			$('#devStatus').text("无法获取");
			break;
		}
	}
	
	var source = new EventSource("/api/events/data.get");
	source.onmessage = function(evt) {
		var payload = JSON.parse(evt.data);
		console.log("[status]new event:" + payload);
	}
});
</script>
</head>
<body>
<header>
	<div class="hd-title" style="text-align: center;">检测节点{{.Id}}</div>
</header>
<main>
	<section>
		<div class="devStatus" style="text-align: center;">
			<form id="devCmd">
					<label>运行状态：</label>
					<label id="devStatus">无法获取设备信息</label>
					<input class="devBut" type="submit" name="devStart" value="Start" />
					<input class="devBut" type="submit" name="devStop" value="Stop" />
			</form>
		</div>
	</section>
	<section>
		<div class="ct-chart ct-major-eleventh" style="width: 80%; margin-left: auto; margin-right: auto;"></div>
	</section>
	<section>
		<table id="tbCon" style="width: 60%; margin-left: auto; margin-right: auto;">
			<colgroup>
				<col width="10%">
				<col width="15%">
				<col width="15%">
				<col width="15%">
				<col width="15%">
				<col width="15%">
				<col width="15%">
			</colgroup>
			<thead>
				<tr>
					<th>Time</th>
					<th>SO<sub>2</sub></th>
					<th>NO<sub>2</sub></th>
					<th>O<sub>3</sub></th>
					<th>CO</th>
					<th>RH</th>
					<th>Temp</th>
				</tr>
			</thead>
			<tbody>
			</tbody>
		</table>
	</section>
</main>
</body>
</html>
```