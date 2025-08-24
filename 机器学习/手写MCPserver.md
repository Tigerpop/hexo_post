---
layout: posts
title: 手写MCPserver
date: 2025-8-18 20:07:11
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



# http模式

weather_http.py

```python
# 初始化FastMCP服务器,取名并开启sse网络。 
mcp = FastMCP(
  				name="weather", 	
  				host="0.0.0.0", 	
  				port=8005, 
  				sse_path="/mcp")

# 。。。。。

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



