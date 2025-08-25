---
layout: posts
title: 手写MCPserver
date: 2025-8-24 20:07:11
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "机器学习"
tags:
- "dify"
- "MCP"

---



# 安装uv/npm

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh # 安装uv

uv --version # uv 0.8.11
node -v # v22.16.0
npm -v # 10.9.2
npx -v # 10.9.2

# 如果npm相关的没有安装 用手动安装一下。
# 添加 Node.js 最新 LTS 版本仓库（比如 20.x）
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
# 安装 Node.js 和 npm
sudo apt-get install -y nodejs
```

![截屏2025-08-18 20.37.21](%E6%89%8B%E5%86%99MCPserver.assets/%E6%88%AA%E5%B1%8F2025-08-18%2020.37.21.png)

![截屏2025-08-18 20.42.22](%E6%89%8B%E5%86%99MCPserver.assets/%E6%88%AA%E5%B1%8F2025-08-18%2020.42.22.png)



# MCP 传输协议

## Stdio

标准输入输出流，进行双向通信，客户端写入输入，服务器输出相应；无须依赖网络；多用于本地开发调试，同一个设备中多用；

## SSE

http单向推送机制，客户端http get请求建立与服务器的长连接，服务器持续以温流的形式推送事件消息，客户端接收消息；

## Streamable Http

基于 **HTTP 请求-响应** 的标准通信方式，但响应体支持 **流式传输（chunked transfer / streaming）**。

- 客户端发起普通的 HTTP 请求（通常是 `POST`），
- 服务器立即返回响应，但响应体内容以流的形式逐步传输，直到完整结束；
- 相比 SSE：支持 **双向交互**，客户端可以通过请求体携带上下文，服务器端则以流式返回部分结果；



# 解释sse

clinet.py

```python
import requests

def sse_client(url):
    # stream=False 会立即下载整个响应内容到内存，再返回给你。
    # 而stream 建立连接后，不立即读取响应体，而是让你通过迭代器逐块获取数据。
    # 如果这里是 False 会让requests一直等服务端给一个终止信号再获取响应，然而服务端是个死循环。
    response = requests.get(url, stream=True)
    if response.status_code == 200:
        try:
            for line in response.iter_lines(): # iter_lines阻塞，直到收到新行。
                if line:
                    line = line.decode('utf-8')
                    if line.startswith('data:'):
                        data = line[5:].strip()
                        print(f"收到数据: {data}")
        except KeyboardInterrupt:
            print("手动关闭连接")
    else:
        print(f"连接失败，状态码: {response.status_code}")

if __name__ == "__main__":
    # 客户端向服务器建立 HTTP 连接，请求 /stream 资源。
    sse_client("http://localhost:5001/stream")
```

service.py

```python
from flask import Flask, Response,request
import time,json 

app = Flask(__name__)

@app.route('/stream')
def stream():
    question = request.args.get("question")
    def event_stream():
        num = 0
        while True:
            data = {
                "answer": f"回答片段 {num+1} for question: {question}"
            }
            yield f"data: {data}\n\n"
            num += 1
            time.sleep(1)
    # 将生成器包装为持续输出的 HTTP 响应。
    return Response(event_stream(), mimetype='text/event-stream')

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001, debug=True)

# 服务端视角
# 当客户端请求 /stream 时，Flask 调用 stream() 函数。
# stream() 返回一个生成器对象（event_stream() 的返回值）。
# Flask 将生成器包装在 Response 中，并开始迭代它：
# 每次迭代获取一个值（yield 的内容）。
# 将值发送给客户端。
# 等待下一次迭代。
# 生成器在客户端连接后立即开始执行（Flask 会调用它），但每次只执行到 yield 就暂停，暂停是为了把数据发送用的。


# 客户端视角
# requests.get(url, stream=True) 建立 HTTP 连接。
# response.iter_lines() 开始监听服务器数据：
# 初始时可能为空，等待服务器发送数据。
# 每次收到新行（\n），返回一行数据。

# 不会出现 服务端生成 1 2 3 4 5 客户端只收到 1 3 5 这样跳号的情况，因为 
# HTTP，具有以下特性：
# 有序传输：HTTP 是面向连接的协议，保证数据按发送顺序到达。
# 行分隔：SSE 消息以空行（\n\n）分隔，客户端按行解析。
# 自动重连：浏览器的 EventSource 内置重连机制，可恢复中断的连接。
# 服务端                网络                客户端
# ┌─────────┐      ┌─────────┐      ┌─────────┐
# │ 应用缓冲区│───→│ 网络缓冲区│───→│ 应用缓冲区│
# └─────────┘      └─────────┘      └─────────┘
```



# stdio模式

```python
mcp = FastMCP(name="weather")
# ....
if __name__ == "__main__":
    # 初始化并运行服务器 , 这里选传输方式。
    mcp.run(transport='stdio')
```

配置json文件

```json
{
  "mcpServers": {
    "weather": {
      "command": "uv",
      "args": [
        "--directory", 
        "/home/cys/data/MCPserver/my_weather_mcp_server",
        "run",
        "weather_stdio.py"
      ]
    }
  }
}
```

# sse模式

```python
# 初始化FastMCP服务器,取名并开启sse网络。 
mcp = FastMCP(
  				name="weather", 	
  				host="0.0.0.0", 	
  				port=8005, 
  				sse_path="/sse")

# ....

if __name__ == "__main__": 
  	print('开启 服务 starting server') 
    # 初始化并运行服务器 , 这里选传输方式。 
    mcp.run(transport='sse')
```

开启服务 uv run weather_http.py

修改json 配置。

```json
{
  "mcpServers": {
    "weather": {
      "url": "http://127.0.0.1:8005/sse",
      "type": "sse"
    }
  }
}
```

# http模式

weather_http.py

```python
# 初始化FastMCP服务器,取名并开启网络。 
mcp = FastMCP(
  				name="weather", 	
  				host="0.0.0.0", 	
  				port=8005, 
  				sse_path="/mcp")

# ....

if __name__ == "__main__": 
  	print('开启 服务 starting server') 
    # 初始化并运行服务器 , 这里选传输方式。 
    mcp.run(transport='streamable-http')
```

开启服务 uv run weather_http.py

修改json 配置。

```json
{
  "mcpServers": {
    "weather": {
      "url": "http://172.17.0.1:8005/mcp",
      "type": "streamableHTTP"
    }
  }
}
```







# dify 中的 mcp 配置

dify 中 mcp 只支持 sse 模式，所以，我们可以看见 用的 mcp 配置 按照 一些mcp 的GitHub中的 指导 ，在 Cline 或者 Cherry Studio 中可以用，但是 在 DIfy 中 是不可用的， 可以遇到 这样类型的报错

```bash
Failed to transform agent message: req_id: 74bcdf4f71 PluginInvokeError: {"args":{},"error_type":"TypeError","message":"Invalid type for url. Expected str or httpx.URL, got \u003cclass 'NoneType'\u003e: None"}
```

 就是因为 dify 默认用了sse 访问形式，解析studio的命令会出现 返回值和预期不一致的情况。 

为了解决这个问题，我们 有两种方法，

方法一：改代码：

方法二：用转换工具：



# Python开发MCP Server

## 阅读：

用 openweather为模版 改一个自己的。

https://openweathermap.org/

进入它的API，搜free ，看到 有一个current weather data，进入API doc。

看它需要怎么用。默认就三个参数 经纬度和APIkey。

![截屏2025-08-21 20.18.53](%E6%89%8B%E5%86%99MCPserver.assets/%E6%88%AA%E5%B1%8F2025-08-21%2020.18.53.jpg)

从右边可见 还有基于别的 方法查询，比如城市名称。

```json
浏览器：
https://api.openweathermap.org/data/2.5/weather?q=beijing&appid=4cc5540e55d86d33858c1fdff4fda354

返回：
{"coord":{"lon":116.3972,"lat":39.9075},"weather":[{"id":501,"main":"Rain","description":"moderate rain","icon":"10n"}],"base":"stations","main":{"temp":299.35,"feels_like":299.35,"temp_min":299.35,"temp_max":299.35,"pressure":1003,"humidity":95,"sea_level":1003,"grnd_level":999},"visibility":10000,"wind":{"speed":1.34,"deg":14,"gust":1.94},"rain":{"1h":1.6},"clouds":{"all":90},"dt":1755778408,"sys":{"country":"CN","sunrise":1755725505,"sunset":1755774231},"timezone":28800,"id":1816670,"name":"Beijing","cod":200}
```

![截屏2025-08-21 20.55.57](%E6%89%8B%E5%86%99MCPserver.assets/%E6%88%AA%E5%B1%8F2025-08-21%2020.55.57.jpg)



## 建立文件夹环境：

```bash
uv init my_weather_mcp_server -p 3.10 # 用这个初始化出一个uv环节的文件夹，python用3.10的。
cd my_weather_mcp_server
uv venv                               # 创建一个虚拟的python环境。文件夹中的.venv 就是虚拟环境。
source .venv/bin/activate             # 激活这个环境，类似 conda。
uv add mcp[cli] httpx                 # 安装依赖
```



## 编码：

weather_stdio.py

```python
from typing import Any 
import httpx 
from mcp.server.fastmcp import FastMCP 

# 初始化FastMCP服务器,取名
mcp = FastMCP("weather")

# 常量
NWS_API_BASE = "https://api.openweathermap.org/data/2.5/weather"
USER_AGENT = "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/138.0.0.0 Safari/537.36"

# 开尔文转摄氏度
def kelvin_to_celsius(kelvin: float) -> float:
    return kelvin - 273.15

async def get_weather_from_cityname(cityname: str) -> dict[str,Any] | None:
    "发请求并处理错误 "
    # ​​Accept 告知服务器客户端期望接收的数据格式​​，application/geo+json 是地理常用的json格式。
    headers = {
        "User-Agent": USER_AGENT,
        "Accept": "application/geo+json"
    }
    params = {
        "q": cityname,
        "appid": "4cc5540e55d86d33858c1fdff4fda354"
    }
    async with httpx.AsyncClient()  as client:
        try:
            response = await client.get(NWS_API_BASE,headers=headers,params=params)
            response.raise_for_status() # 自检 HTTP 响应状态码，异常抛出。
            return response.json()
        except:
            return None
        
def format_alter(feature: dict) -> str:
    "接口返回数据格式化输出"
    if feature["cod"] == 404:
        return "参数异常，查看是否输入城市名称错误"
    elif feature["cod"] == 401:
        return "API key 异常"
    elif feature["cod"] == 200:
        return f"""
                city:{feature.get('name','unknowCity')}
                weather:{feature.get('weather',[{}])[0].get('description','unkonw_weather')}
                min_temperature:{kelvin_to_celsius(feature.get('main',{}).get('temp_min',0)):.2f}℃
                max_temperature:{kelvin_to_celsius(feature.get('main',{}).get('temp_max',0)):.2f}℃
                humidity:{feature.get('main',{}).get('humidity',0)}%
                wind:{feature.get('wind',{}).get('speed',0):.2f} m/s
                """

# 此装饰器将业务代码转为MCP服务，让大模型去调用。
@mcp.tool()
async def get_weather_from_cityname_tool(city: str) -> str:
    # 注意方法内的引号内内容，就是通过它让大模型读到这个def方法的用处，所以要写好。
    """
    Get weather information for a city.

    Args:
        city (str): City name. For Chinese cities, please use pinyin.

    Returns:
        str: Weather description.
    """
    data = await get_weather_from_cityname(city)
    return format_alter(data)

if __name__ == "__main__":
    # 初始化并运行服务器 , 这里选传输方式。
    mcp.run(transport='stdio')
```

weather_sse.py

```python
from typing import Any 
import httpx 
from mcp.server.fastmcp import FastMCP 

# 初始化FastMCP服务器,取名并开启sse网络。
mcp = FastMCP(name="weather",
              host="0.0.0.0",
              port=8005,
              sse_path="/sse")

# 常量
NWS_API_BASE = "https://api.openweathermap.org/data/2.5/weather"
USER_AGENT = "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/138.0.0.0 Safari/537.36"

# 开尔文转摄氏度
def kelvin_to_celsius(kelvin: float) -> float:
    return kelvin - 273.15

async def get_weather_from_cityname(cityname: str) -> dict[str,Any] | None:
    "发请求并处理错误 "
    # ​​Accept 告知服务器客户端期望接收的数据格式​​，application/geo+json 是地理常用的json格式。
    headers = {
        "User-Agent": USER_AGENT,
        "Accept": "application/geo+json"
    }
    params = {
        "q": cityname,
        "appid": "4cc5540e55d86d33858c1fdff4fda354"
    }
    async with httpx.AsyncClient()  as client:
        try:
            response = await client.get(NWS_API_BASE,headers=headers,params=params)
            response.raise_for_status() # 自检 HTTP 响应状态码，异常抛出。
            return response.json()
        except:
            return None
        
def format_alter(feature: dict) -> str:
    "接口返回数据格式化输出"
    if feature["cod"] == 404:
        return "参数异常，查看是否输入城市名称错误"
    elif feature["cod"] == 401:
        return "API key 异常"
    elif feature["cod"] == 200:
        return f"""
                city:{feature.get('name','unknowCity')}
                weather:{feature.get('weather',[{}])[0].get('description','unkonw_weather')}
                min_temperature:{kelvin_to_celsius(feature.get('main',{}).get('temp_min',0)):.2f}℃
                max_temperature:{kelvin_to_celsius(feature.get('main',{}).get('temp_max',0)):.2f}℃
                humidity:{feature.get('main',{}).get('humidity',0)}%
                wind:{feature.get('wind',{}).get('speed',0):.2f} m/s
                """

# 此装饰器将业务代码转为MCP服务，让大模型去调用。
@mcp.tool()
async def get_weather_from_cityname_tool(city: str) -> str:
    # 注意方法内的引号内内容，就是通过它让大模型读到这个def方法的用处，所以要写好。
    """
    Get weather information for a city.

    Args:
        city (str): City name. For Chinese cities, please use pinyin.

    Returns:
        str: Weather description.
    """
    data = await get_weather_from_cityname(city)
    return format_alter(data)

if __name__ == "__main__":
    print('开启 服务 starting server')
    # 初始化并运行服务器 , 这里选传输方式。
    mcp.run(transport='sse')
```



## 写MCP的json配置：

### stdio 传输方式

配置json文件

```json
{
  "mcpServers": {
    "weather": {
      "command": "uv",
      "args": [
        "--directory", 
        "/home/cys/data/MCPserver/my_weather_mcp_server",
        "run",
        "weather_stdio.py"
      ]
    }
  }
}
```

### sse 传输方式

先开启服务

```bash
uv run weather_sse.py
```

![截屏2025-08-22 14.08.57](%E6%89%8B%E5%86%99MCPserver.assets/%E6%88%AA%E5%B1%8F2025-08-22%2014.08.57.jpg)

再配置json文件

```json
{
  "mcpServers": {
    "weather": {
      "url": "http://127.0.0.1:8005/sse",
      "type": "sse"
    }
  }
}
```

**对于运行在docker容器中的dify而言，还需要如下配置。**

宿主机运行命令，查出docker容器访问宿主机的IP。

```bash
ip addr show docker0

7: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 3a:60:3b:3b:d5:f4 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever

```

172.17.0.1 就是默认的 容器访问宿主机的地址。

上面的json配置改写

```json
{
  "mcpServers": {
    "weather": {
      "url": "http://172.17.0.1:8005/sse",
      "type": "sse"
    }
  }
}
```

![截屏2025-08-22 14.30.25](%E6%89%8B%E5%86%99MCPserver.assets/%E6%88%AA%E5%B1%8F2025-08-22%2014.30.25.jpg)

![截屏2025-08-22 14.29.59](%E6%89%8B%E5%86%99MCPserver.assets/%E6%88%AA%E5%B1%8F2025-08-22%2014.29.59.jpg)









# 手写 MCP 遇到的一个常见问题

如果在 自己写的 MCP服务的 后台已经看见了 200 的日志，说明 Dify 已经能成功连到你的  服务了。

但是遇到：

```bash
Run failed: Failed to transform agent message: req_id: 52df8d2480 PluginInvokeError: {"args":{},"error_type":"AttributeError","message":"'NoneType' object has no attribute 'parameters'"}
```

这个意思是 PluginInvokeError MCP插件没调用起来，

`FastMCP` 的 `@mcp.tool()` 装饰器会自动从 **函数签名**和**docstring**生成 JSON Schema，返回给 Dify。
 但是 docstring 写法有点“模糊”，导致 schema 生成失败：

比如这种风格的 

```python
"""
Get weather information for a city.
Args:
    city: city name. for chinese cities,please use pinyin.
"""
```

`fastmcp` 并不解析这种 `Args:` 风格，它期望 **Google / NumPy 风格** 。

```python
    """
    Get weather information for a city.

    Args:
        city (str): City name. For Chinese cities, please use pinyin.

    Returns:
        str: Weather description.
    """
```

可以直接手写你的原始doctoring，然后要大模型给你 改写成 **Google / NumPy 风格** ，再 贴进代码中。

另外，还有一个问题是 ，一个 agent 只要被 大模型调用，就会进入 MCP 的业务中，如果这个 MCP 业务是处理 单个城市信息的，你问了一堆 城市信息，如果大模型不懂的拆分问题，一个一个去调用 MCP业务查询，那就会 一股脑丢进 MCP 业务中，然后导致 查不到东西。
除非让大模型懂拆分，或者在 MCP 中 写好 业务逻辑，或者在 docstring 中解释清楚看看大模型能不能解决这个拆分查询的问题。



