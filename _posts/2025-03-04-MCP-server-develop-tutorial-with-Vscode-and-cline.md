---
layout: mypost
title: Building your own MCP server to use in Vscode and cline
categories: [MCP, LLM, Vscode, cline]
---

## 什么是MCP？

[MCP（Model Context Protocol）](https://modelcontextprotocol.io/introduction)是一个由anthropic 开源的开放协议，旨在标准化大型语言模型（LLMs）与外部数据源之间的连接。它提供了一种简单的一致的方法，使开发者能够快速构建和集成自己的服务器，并让AI与不同的数据源进行交互。


## 必备条件
- Python (Python 3.10 or higher installed.)
- LLMs like Google Gemini 2.0 

## 环境准备
let’s install `uv` and set up our Python project and environment:
```
#For MacOS/Linux 
curl -LsSf https://astral.sh/uv/install.sh | sh

# Ubuntu 24.04 LTS 
user@ubuntu2404LTS  ~/MCP  uv --version
uv 0.6.3
```
> ```[For Windows system environment]   powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex" ```
 

## 创建weather项目

```shell
# Create a new directory for our project
uv init weather
cd weather

# Create virtual environment and activate it
uv venv
source .venv/bin/activate

# Install dependencies
uv add "mcp[cli]" httpx

# Create our server file
touch weather.py

``` 
## 目录结构

```
 user@ubuntu2404LTS  ~/MCP  tree            
.
└── weather
    ├── main.py
    ├── pyproject.toml
    ├── README.md
    ├── uv.lock
    └── weather.py

```

## 编写weather.py
`weather.py`

```python
from typing import Any
import httpx
from mcp.server.fastmcp import FastMCP

# Initialize FastMCP server
mcp = FastMCP("weather")

# Constants
NWS_API_BASE = "https://api.weather.gov"
USER_AGENT = "weather-app/1.0"

async def make_nws_request(url: str) -> dict[str, Any] | None:
    """Make a request to the NWS API with proper error handling."""
    headers = {
        "User-Agent": USER_AGENT,
        "Accept": "application/geo+json"
    }
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(url, headers=headers, timeout=30.0)
            response.raise_for_status()
            return response.json()
        except Exception:
            return None

def format_alert(feature: dict) -> str:
    """Format an alert feature into a readable string."""
    props = feature["properties"]
    return f"""
Event: {props.get('event', 'Unknown')}
Area: {props.get('areaDesc', 'Unknown')}
Severity: {props.get('severity', 'Unknown')}
Description: {props.get('description', 'No description available')}
Instructions: {props.get('instruction', 'No specific instructions provided')}
"""

@mcp.tool()
async def get_alerts(state: str) -> str:
    """Get weather alerts for a US state.

    Args:
        state: Two-letter US state code (e.g. CA, NY)
    """
    url = f"{NWS_API_BASE}/alerts/active/area/{state}"
    data = await make_nws_request(url)

    if not data or "features" not in data:
        return "Unable to fetch alerts or no alerts found."

    if not data["features"]:
        return "No active alerts for this state."

    alerts = [format_alert(feature) for feature in data["features"]]
    return "\n---\n".join(alerts)

@mcp.tool()
async def get_forecast(latitude: float, longitude: float) -> str:
    """Get weather forecast for a location.

    Args:
        latitude: Latitude of the location
        longitude: Longitude of the location
    """
    # First get the forecast grid endpoint
    points_url = f"{NWS_API_BASE}/points/{latitude},{longitude}"
    points_data = await make_nws_request(points_url)

    if not points_data:
        return "Unable to fetch forecast data for this location."

    # Get the forecast URL from the points response
    forecast_url = points_data["properties"]["forecast"]
    forecast_data = await make_nws_request(forecast_url)

    if not forecast_data:
        return "Unable to fetch detailed forecast."

    # Format the periods into a readable forecast
    periods = forecast_data["properties"]["periods"]
    forecasts = []
    for period in periods[:5]:  # Only show next 5 periods
        forecast = f"""
{period['name']}:
Temperature: {period['temperature']}°{period['temperatureUnit']}
Wind: {period['windSpeed']} {period['windDirection']}
Forecast: {period['detailedForecast']}
"""
        forecasts.append(forecast)

    return "\n---\n".join(forecasts)

if __name__ == "__main__":
    # Initialize and run the server
    mcp.run(transport='stdio')
```


## 编辑cline_mcp_setting.json

```
{
  "mcpServers": {
    "weather": {
      "command": "uv",
      "args": [
          "--directory",
          "/ABSOLUTE/PATH/TO/PARENT/FOLDER/weather",
          "run",
          "weather.py"
      ]
  }
  }
}
```
## Vscode 安装Cline 

![vscode install cline ](https://sspark.ai/cfimages?u1=R6%2B425NYoczg07iYHsZPKdxSgMQiJDcC8NOa%2BuaXy05XWuQqHOOrelbF58yc1FVlj1K1zlZV6JzVLs3pi51TjELsv3i0V8g6nmmcXI%2BQ&u2=jIvNrVTC9bCAkjav&width=1024)

安装Cline插件的步骤相对简单，适合所有希望提升编码效率的开发者。以下是详细安装过程：

打开VSCode：首先启动你的Visual Studio Code。

访问插件市场：在左侧边栏中，点击“扩展”图标，或者你可以使用快捷键 Ctrl+Shift+X 直接打开扩展视图。

搜索Cline：在扩展市场的搜索框中输入"Cline"，然后按 Enter 键。

安装插件：在搜索结果中找到Cline插件，并点击"安装"按钮来进行安装。

重启VSCode：安装完成后，建议重启VSCode，以确保插件正常加载并使用其功能。

配置插件：重启后，你可以在左侧边栏看到一个小机器人图标，点击它开始配置Cline。在配置过程中，你需要设置API密钥，这通常可以从所需的API提供商那里获取。如果你已经有API密钥，可以在设置中输入它23。

开始使用：配置完成后，你就可以开始使用Cline进行编程了。Cline将会在你编码时提供代码补全、错误检测等智能辅助功能，提升你的开发效率



## 验证MCP运行

[![photo 2025 03 04 12 08 37](https://youjb.com/images/2025/03/04/photo_2025-03-04_12-08-37ddf065ab39113f03.md.jpg)](https://youjb.com/image/photo-2025-03-04-12-08-37.s7A)
[![photo 2025 03 04 12 08 18](https://youjb.com/images/2025/03/04/photo_2025-03-04_12-08-18420e7e296b451945.md.jpg)](https://youjb.com/image/photo-2025-03-04-12-08-18.s7D)