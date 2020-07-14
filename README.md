Created: Jul 13, 2020 3:09 PM

## RESTful & Socket: market data connection and capture

Market data, the most important is real-time and effectiveness. Meanwhile, market conditions are changing rapidly, and the appropriate time interval for buying and selling may be only a few seconds or less. In high-frequency trading,  the millisecond level is basic unit for a good transaction.

## Market data

The exchange is a public matching platform between buyers and sellers. The buyer and seller submit the required/available quantity of goods and the price they are willing to offer/accept to the exchange.

![Page%202%2011b23d99c32244caa996ecaf681404f1/Untitled.png](Page%202%2011b23d99c32244caa996ecaf681404f1/Untitled.png)

### Websocket Presentation

The REST interface is essentially an HTTP interface, under which is a TCP/TSL socket (Socket) connection. Each REST request usually re-establishes a TCP/TSL handshake; then, after the request ends, the link is disconnected. This process is much slower than we thought.

```python
import requests
import timeit

def get_orderbook():
  orderbook = requests.get("https://api.gemini.com/v1/book/btcusd").json()

n = 10
latency = timeit.timeit('get_orderbook()', setup='from __main__ import get_orderbook', number=n) * 1.0 / n
print('Latency is {} ms'.format(latency * 1000))

###### output #######

Latency is 196.67642089999663 ms
```

Let me introduce WebSocket first. WebSocket is a protocol for full-duplex and bidirectional communication on a single TCP/TSL connection. WebSocket can make data exchange between the client and server easier and more efficient, and the server can also actively push data to the client. In the WebSocket API, the browser and the server only need to complete a handshake, and the two can directly create a persistent connection and perform two-way data transmission.

```python
import websocket
import thread

# 在接收到服务器发送消息时调用
def on_message(ws, message):
    print('Received: ' + message)

# 在和服务器建立完成连接时调用   
def on_open(ws):
    # 线程运行函数
    def gao():
        # 往服务器依次发送0-4，每次发送完休息0.01秒
        for i in range(5):
            time.sleep(0.01)
            msg="{0}".format(i)
            ws.send(msg)
            print('Sent: ' + msg)
        # sleep 1s in order to receive response from server
        time.sleep(1)
        
        # 关闭Websocket的连接 Close Websocket connection
        ws.close()
        print("Websocket closed")
    
    # run gao() function in another thread
    thread.start_new_thread(gao, ())

if __name__ == "__main__":
    ws = websocket.WebSocketApp("ws://echo.websocket.org/",
                              on_message = on_message,
                              on_open = on_open)
    
    ws.run_forever()

#### output #####
Sent: 0
Sent: 1
Received: 0
Sent: 2
Received: 1
Sent: 3
Received: 2
Sent: 4
Received: 3
Received: 4
Websocket closed
```

![Page%202%2011b23d99c32244caa996ecaf681404f1/Untitled%201.png](Page%202%2011b23d99c32244caa996ecaf681404f1/Untitled%201.png)

## Crawler module

![Page%202%2011b23d99c32244caa996ecaf681404f1/Untitled%202.png](Page%202%2011b23d99c32244caa996ecaf681404f1/Untitled%202.png)

```python
import copy
import json
import ssl
import time
import websocket

class OrderBook(object):

    BIDS = 'bid'
    ASKS = 'ask'

    def __init__(self, limit=20):

        self.limit = limit

        # (price, amount)
        self.bids = {}
        self.asks = {}

        self.bids_sorted = []
        self.asks_sorted = []

    def insert(self, price, amount, direction):
        if direction == self.BIDS:
            if amount == 0:
                if price in self.bids:
                    del self.bids[price]
            else:
                self.bids[price] = amount
        elif direction == self.ASKS:
            if amount == 0:
                if price in self.asks:
                    del self.asks[price]
            else:
                self.asks[price] = amount
        else:
            print('WARNING: unknown direction {}'.format(direction))

    def sort_and_truncate(self):
        # sort
        self.bids_sorted = sorted([(price, amount) for price, amount in self.bids.items()], reverse=True)
        self.asks_sorted = sorted([(price, amount) for price, amount in self.asks.items()])

        # truncate
        self.bids_sorted = self.bids_sorted[:self.limit]
        self.asks_sorted = self.asks_sorted[:self.limit]

        # copy back to bids and asks
        self.bids = dict(self.bids_sorted)
        self.asks = dict(self.asks_sorted)

    def get_copy_of_bids_and_asks(self):
        return copy.deepcopy(self.bids_sorted), copy.deepcopy(self.asks_sorted)

class Crawler:
    def __init__(self, symbol, output_file):
        self.orderbook = OrderBook(limit=10)
        self.output_file = output_file

        self.ws = websocket.WebSocketApp('wss://api.gemini.com/v1/marketdata/{}'.format(symbol),
                                         on_message = lambda ws, message: self.on_message(message))
        self.ws.run_forever(sslopt={'cert_reqs': ssl.CERT_NONE})

    def on_message(self, message):
        # 对收到的信息进行处理，然后送给 orderbook
        data = json.loads(message)
        for event in data['events']:
            price, amount, direction = float(event['price']), float(event['remaining']), event['side']
            self.orderbook.insert(price, amount, direction)

        # 整理 orderbook，排序，只选取我们需要的前几个
        self.orderbook.sort_and_truncate()

        # 输出到文件
        with open(self.output_file, 'a+') as f:
            bids, asks = self.orderbook.get_copy_of_bids_and_asks()
            output = {
                'bids': bids,
                'asks': asks,
                'ts': int(time.time() * 1000)
            }
            f.write(json.dumps(output) + '\n')

if __name__ == '__main__':
    crawler = Crawler(symbol='BTCUSD', output_file='BTCUSD.txt')

###### 输出 #######

{"bids": [[11398.73, 0.96304843], [11398.72, 0.98914437], [11397.32, 1.0], [11396.13, 2.0], [11395.95, 2.0], [11395.87, 1.0], [11394.09, 0.11803397], [11394.08, 1.0], [11393.59, 0.1612581], [11392.96, 1.0]], "asks": [[11407.42, 1.30814001], [11407.92, 1.0], [11409.48, 2.0], [11409.66, 2.0], [11412.15, 0.525], [11412.42, 1.0], [11413.77, 0.11803397], [11413.99, 0.5], [11414.28, 1.0], [11414.72, 1.0]], "ts": 1562558996535}
{"bids": [[11398.73, 0.96304843], [11398.72, 0.98914437], [11397.32, 1.0], [11396.13, 2.0], [11395.95, 2.0], [11395.87, 1.0], [11394.09, 0.11803397], [11394.08, 1.0], [11393.59, 0.1612581], [11392.96, 1.0]], "asks": [[11407.42, 1.30814001], [11407.92, 1.0], [11409.48, 2.0], [11409.66, 2.0], [11412.15, 0.525], [11412.42, 1.0], [11413.77, 0.11803397], [11413.99, 0.5], [11414.28, 1.0], [11414.72, 1.0]], "ts": 1562558997377}
{"bids": [[11398.73, 0.96304843], [11398.72, 0.98914437], [11397.32, 1.0], [11396.13, 2.0], [11395.95, 2.0], [11395.87, 1.0], [11394.09, 0.11803397], [11394.08, 1.0], [11393.59, 0.1612581], [11392.96, 1.0]], "asks": [[11407.42, 1.30814001], [11409.48, 2.0], [11409.66, 2.0], [11412.15, 0.525], [11412.42, 1.0], [11413.77, 0.11803397], [11413.99, 0.5], [11414.28, 1.0], [11414.72, 1.0]], "ts": 1562558997765}
{"bids": [[11398.73, 0.96304843], [11398.72, 0.98914437], [11397.32, 1.0], [11396.13, 2.0], [11395.95, 2.0], [11395.87, 1.0], [11394.09, 0.11803397], [11394.08, 1.0], [11393.59, 0.1612581], [11392.96, 1.0]], "asks": [[11407.42, 1.30814001], [11409.48, 2.0], [11409.66, 2.0], [11412.15, 0.525], [11413.77, 0.11803397], [11413.99, 0.5], [11414.28, 1.0], [11414.72, 1.0]], "ts": 1562558998638}
{"bids": [[11398.73, 0.97131753], [11398.72, 0.98914437], [11397.32, 1.0], [11396.13, 2.0], [11395.95, 2.0], [11395.87, 1.0], [11394.09, 0.11803397], [11394.08, 1.0], [11393.59, 0.1612581], [11392.96, 1.0]], "asks": [[11407.42, 1.30814001], [11409.48, 2.0], [11409.66, 2.0], [11412.15, 0.525], [11413.77, 0.11803397], [11413.99, 0.5], [11414.28, 1.0], [11414.72, 1.0]], "ts": 1562558998645}
{"bids": [[11398.73, 0.97131753], [11398.72, 0.98914437], [11397.32, 1.0], [11396.13, 2.0], [11395.87, 1.0], [11394.09, 0.11803397], [11394.08, 1.0], [11393.59, 0.1612581], [11392.96, 1.0]], "asks": [[11407.42, 1.30814001], [11409.48, 2.0], [11409.66, 2.0], [11412.15, 0.525], [11413.77, 0.11803397], [11413.99, 0.5], [11414.28, 1.0], [11414.72, 1.0]], "ts": 1562558998748}
```
