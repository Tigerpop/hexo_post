---
layout: posts
title: 利用Dify+MCP开发定制功能Agent
date: 2025-6-13 20:27:21
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "机器学习"
tags:
- "dify"
- "MCP"

---



# 使用国内MCP市场

Dify 和 MCP 我们之前 都学习过了，现在我们 来 搞一个 Dify 的chatflow ，其中用到 MCP。 

MCP 市场 ，如 https://mcp.so/ 虽然很繁荣，但是都是 海外的，在大陆我们推荐还是使用 modelscope 社区的MCP广场 https://www.modelscope.cn/mcp 。

![截屏2025-06-11 09.44.21](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-11%2009.44.21.jpg)



# 以高德地图MCP为例

## 获取json配置

![截屏2025-06-11 09.48.57](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-11%2009.48.57.jpg)

点击 这个通用级 SSE 协议。

![截屏2025-06-11 10.01.51](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-11%2010.01.51.jpg)

![截屏2025-06-11 10.08.10](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-11%2010.08.10.jpg)

在配置 APIkey 时可见高德的 这个MCP不仅仅是地图功能，还能使用 天气 行政区 等等功能。

![截屏2025-06-11 10.50.14](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-11%2010.50.14.jpg)

填写APIkey

![截屏2025-06-11 10.21.04](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-11%2010.21.04.jpg)

```json
{
  "mcpServers": {
    "amap-maps": {
      "type": "sse",
      "url": "https://mcp.api-inference.modelscope.net/ce3a4f094e6d48/sse"
    }
  }
}
```

![截屏2025-06-11 10.24.34](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-11%2010.24.34.jpg)

**注意：**
这个 mcpserver 的 URL地址是 `mcp.api-inference.modelscope.net`，这说明什么，说明这个URL是专门用来给modelscope的试验场用的，不要在后续自己正式的在MCP host 或者 Dify 中配置的时候，使用这个 mcp的json 配置！！！亲。



## 去MCP试验场测试

添加刚才的 json配置，点击保存。

![截屏2025-06-11 10.27.35](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-11%2010.27.35.jpg)

点击试验场的小扳手🔧图标，确保默认的 高德地图插件处于关闭状态，而我们刚刚保存的MCP server处于开启。

![截屏2025-06-11 10.37.12](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-11%2010.37.12.jpg)

![截屏2025-06-11 10.46.48](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-11%2010.46.48.jpg)

它先查看了两地的天气，然后又规划了路线，由于我只在问题中说了潮汕，不是一个具体城市导致了 潮汕天气搜索失败，但是不影响大模型 思考了一下 用别的路线规划。



## 在Dify中添加MCP

搜索安装Agent策略（支持MCP工具）。

![截屏2025-06-11 15.17.30](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-11%2015.17.30.jpg)

![截屏2025-06-11 16.05.45](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-11%2016.05.45.jpg)

我这里新建了一个对话任务流，先对问题分类，然后接上了一个 agent，这个agent策略用的是 ReAct，模型还是本地的 70b的deepseek，工具添加了一个时间戳。

MCP服务配置中不能填写前面的 json配置文件"mcpServers": 包裹住的部分

```json
{
   "amap-maps": {
      "type": "sse",
      "url": "https://mcp.api-inference.modelscope.net/ce3a4f094e6d48/sse"
    }
}
```

而是要从高德MCP的文档中找到 快速使用的介绍

![截屏2025-06-13 11.11.59](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-13%2011.11.59.jpg)

用这个json配置

```json 
{
    "amap-amap-sse": {
      "url": "https://mcp.amap.com/sse?key=您在高德官网上申请的key"
    }
}
```

可见`https://mcp.amap.com/sse` 这个就是 高德开放的mcp访问地址。 

指令 写的是 `根据用户的问题选择 配置的MCP 工具amap-maps 查询结果来回答`

![微信图片_20250613111943_1837](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250613111943_1837.png)

迭代次数推荐 多一点。

我用一个 问题 来测试      `请帮我规划一下长春的旅游规划，注意查看未来三天天气情况，下不下雨温度等等情况，根据这个情况来规划旅游。`

我们来分析一下 agent 的思考过程

![微信图片_20250613203730_1914](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20250613203730_1914.png)

round1 中有2个log ,`deepseek-r1-70b-AWQ Thought`  、`deepseek-r1-70b-AWQ Thought`。

`deepseek-r1-70b-AWQ Thought`  内容是

```json
{
  "action": "maps_weather",
  "action_input": {
    "city": "长春"
  },
  "thought": "<think>\n好，用户希望规划长春的旅游，并关心未来三天的天气情况。首先，我需要查询长春接下来的三天天气，包括天气情况和温度。\n\n使用maps_weather工具，传入城市名称“长春”获取天气数据。假设接下来的三天分别是晴天、多云和小雨天，温度在20°C左右。\n\n根据天气情况，制定适合的旅游计划。晴天适合户外活动，像伪满皇宫和净月潭公园都是不错的选择。多云适合文化景点，如长影世纪城。预计到第三天有小雨，应选择带有室内展区的景点，如长春电影博物馆。\n\n参考高德地图的信息，规划景点之间的路线和交通方式，建议合理安排时间，避免因天气原因造成的不便。\n\n因此，建议用户按天气情况分别安排光顾的景点，确保旅途顺利且乐趣多样。\n</think>"
}
```

`deepseek-r1-70b-AWQ Thought` 内容是

```json
{
  "output": "Tool execution result: [{'type': 'text', 'text': '{\"city\":\"长春市\",\"forecasts\":[{\"date\":\"2025-06-13\",\"week\":\"5\",\"dayweather\":\"多云\",\"nightweather\":\"阴\",\"daytemp\":\"29\",\"nighttemp\":\"18\",\"daywind\":\"西南\",\"nightwind\":\"西南\",\"daypower\":\"1-3\",\"nightpower\":\"1-3\",\"daytemp_float\":\"29.0\",\"nighttemp_float\":\"18.0\"},{\"date\":\"2025-06-14\",\"week\":\"6\",\"dayweather\":\"多云\",\"nightweather\":\"阴\",\"daytemp\":\"28\",\"nighttemp\":\"19\",\"daywind\":\"西南\",\"nightwind\":\"西南\",\"daypower\":\"1-3\",\"nightpower\":\"1-3\",\"daytemp_float\":\"28.0\",\"nighttemp_float\":\"19.0\"},{\"date\":\"2025-06-15\",\"week\":\"7\",\"dayweather\":\"雷阵雨\",\"nightweather\":\"雷阵雨\",\"daytemp\":\"25\",\"nighttemp\":\"19\",\"daywind\":\"东\",\"nightwind\":\"东\",\"daypower\":\"1-3\",\"nightpower\":\"1-3\",\"daytemp_float\":\"25.0\",\"nighttemp_float\":\"19.0\"},{\"date\":\"2025-06-16\",\"week\":\"1\",\"dayweather\":\"阴\",\"nightweather\":\"多云\",\"daytemp\":\"26\",\"nighttemp\":\"15\",\"daywind\":\"东南\",\"nightwind\":\"东南\",\"daypower\":\"1-3\",\"nightpower\":\"1-3\",\"daytemp_float\":\"26.0\",\"nighttemp_float\":\"15.0\"}]}'}]",
  "tool_call_args": {
    "city": "长春"
  },
  "tool_name": "maps_weather"
}
```

round2 中有1个log ,`deepseek-r1-70b-AWQ Thought`  内容是

```json
{
  "action": "Final Answer",
  "action_input": "根据长春未来三天的天气情况，天气主要以多云和雷阵雨为主，温度在15°C到29°C之间。建议您做好雨具准备，以下是为您规划的旅游行程：\n  第 1 天：游览净月潭公园和长影世纪城，适合户外活动。\n  第 2 天：探访伪满皇宫和长春世界雕塑公园。\n  第 3 天：（雷阵雨）建议安排室内活动，如长春电影城市和博物馆。\n  第 4 天：（阴天）适合步行游览南湖公园和致远市场。\n  希望这能帮助您顺利进行旅游!",
  "thought": "<think>\n好，我先看看用户的需求。用户想要规划长春的旅游，特别关注未来三天的天气情况，包括是否下雨和温度，这样可以根据这些信息来安排行程。\n\n首先，我需要调用高德地图的天气API，查询长春市接下来的三天的天气情况。确保获取的信息准确，包括每天的天气状况、温度和风力等。\n\n接下来，分析天气数据。第一个两天是多云，气温适中，适合外出游览。第三天有雷阵雨，最好避免在户外活动。第四天则是阴天，仍然适合游览。\n\n然后，考虑长春的主要景点，如净月潭公园、长影世纪城、伪满皇宫等，确保在晴天安排户外活动，在可能有雨的天气安排室内景点。\n\n最后，综合天气情况，制定一个详细的每日行程，确保游客能合理安排时间，享受旅游行程而不受到天气影响。\n</think>"
}
```

我在预览中运行一下，发现 和 前面 agent 试运行的时候是不一样的回答，说明 deepseek 思考过程每一次都有不一样。这次就没有 调用温度查询的 内容。

![c8f2b1f5-c3e7-4015-8b5a-0a6239c7e482](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/c8f2b1f5-c3e7-4015-8b5a-0a6239c7e482.png)





# 用自己的MCP

下载插件 MCP SSE

![截屏2025-06-11 22.32.36](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-11%2022.32.36.jpg)

在MCP SSE 插件的授权中，写上 我们自己写的 MCP server的 服务器名和 开放连接的IP:port 。
这里是可以添加多个 自己写的 MCP server。

![截屏2025-06-11 22.36.24](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-11%2022.36.24.jpg)

注意一下 这个 json配置 中最好不要有空格。



# 遇到的若干问题

## 1、Dify 中 不能在应用市场中一键安装插件

解决办法：先下载，再手动安装。

![截屏2025-06-11 15.58.14](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-11%2015.58.14.jpg)

![截屏2025-06-11 16.00.58](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-11%2016.00.58.jpg)

可见在我的插件中已经有了 这个新下载的插件。

![截屏2025-06-11 16.02.44](%E5%88%A9%E7%94%A8Dify+MCP%E5%BC%80%E5%8F%91%E5%AE%9A%E5%88%B6%E5%8A%9F%E8%83%BDAgent/%E6%88%AA%E5%B1%8F2025-06-11%2016.02.44.jpg)





# 一些概念的理解

## 1、uv uvx 等 本地部署MCP server

uv 和 uvx 都是全局的，uvx 带运行的意思，uv 有点 pip 的意思，但是以文件夹为单位进行了隔离，每个 运行的文件夹会生成一个 .venv 文件夹，里面存各种下载的包。
**本地部署MCP server**

```json
"fetch": {
  "command": "/Users/chenyushao/.local/bin/uvx",
  "args": ["mcp-server-fetch"]
}
```

在配置文件中如果没有写具体在哪个文件夹中运行 ，那 `.venv` 会在哪里？

> 答案是：取决于 **运行这个命令时的“当前工作目录”**（working directory）。

```json
"weather": {
  "command": "/Users/chenyushao/.local/bin/uv",
  "args": [
     "--directory",
     "/Users/chenyushao/mcp_server/weather-mcp",
     "run",
     "-m",
     "weather_mcp_server.main",
     "--api-key",
     "xxx"
  ]
}
```

而配置文件 写明了 --directory，.venv 自然会在 这个 指定的文件夹中。

## 2、sse 调用第三方在线服务 MCP server

**调用第三方在线服务 MCP server**

```json 
"amap-amap-sse": {
  "url": "https://mcp.amap.com/sse?key=您在高德官网上申请的key"
}
```

SSE 全称是 **Server-Sent Events**，意思是：

> **服务器主动向客户端持续推送数据**的一种 Web 通信方式。

客户端只发一次请求，服务器就可以不断发送更新（比如天气、消息、进度），常用于实时数据流。