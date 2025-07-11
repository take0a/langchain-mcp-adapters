# LangChain MCP アダプター

このライブラリは、[Anthropic Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction) ツールを [LangChain](https://github.com/langchain-ai/langchain) および [LangGraph](https://github.com/langchain-ai/langgraph) と互換性を持たせる軽量ラッパーを提供します。

![MCP](static/img/mcp.png)

## 機能

- 🛠️ MCPツールを[LangChainツール](https://python.langchain.com/docs/concepts/tools/)に変換し、[LangGraph](https://github.com/langchain-ai/langgraph)エージェントで使用できるようにします。
- 📦 複数のMCPサーバーに接続し、そこからツールをロードできるクライアント実装

## インストール

```bash
pip install langchain-mcp-adapters
```

## クイックスタート

LangGraphエージェントでMCPツールを使用する簡単な例を示します。

```bash
pip install langchain-mcp-adapters langgraph "langchain[openai]"

export OPENAI_API_KEY=<your_api_key>
```

### サーバー

まず、数字の加算と乗算ができるMCPサーバーを作成しましょう。

```python
# math_server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Math")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b

@mcp.tool()
def multiply(a: int, b: int) -> int:
    """Multiply two numbers"""
    return a * b

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### クライアント

```python
# stdio接続用のサーバーパラメータを作成する
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

from langchain_mcp_adapters.tools import load_mcp_tools
from langgraph.prebuilt import create_react_agent

server_params = StdioServerParameters(
    command="python",
    # math_server.pyファイルへの完全な絶対パスに更新してください。
    args=["/path/to/math_server.py"],
)

async with stdio_client(server_params) as (read, write):
    async with ClientSession(read, write) as session:
        # 接続を初期化する
        await session.initialize()

        # Get tools
        tools = await load_mcp_tools(session)

        # Create and run the agent
        agent = create_react_agent("openai:gpt-4.1", tools)
        agent_response = await agent.ainvoke({"messages": "what's (3 + 5) x 12?"})
```

## 複数のMCPサーバー

このライブラリでは、複数のMCPサーバーに接続し、そこからツールを読み込むこともできます:

### サーバ

```python
# math_server.py
...

# weather_server.py
from typing import List
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Weather")

@mcp.tool()
async def get_weather(location: str) -> str:
    """Get weather for location."""
    return "It's always sunny in New York"

if __name__ == "__main__":
    mcp.run(transport="streamable-http")
```

```bash
python weather_server.py
```

### Client

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.prebuilt import create_react_agent

client = MultiServerMCPClient(
    {
        "math": {
            "command": "python",
            # math_server.pyファイルへの完全な絶対パスに更新してください。
            "args": ["/path/to/math_server.py"],
            "transport": "stdio",
        },
        "weather": {
            # 天気予報サーバーをポート8000​​で起動するようにしてください
            "url": "http://localhost:8000/mcp/",
            "transport": "streamable_http",
        }
    }
)
tools = await client.get_tools()
agent = create_react_agent("openai:gpt-4.1", tools)
math_response = await agent.ainvoke({"messages": "what's (3 + 5) x 12?"})
weather_response = await agent.ainvoke({"messages": "what is the weather in nyc?"})
```

> [!note]
> 上記の例では、ツールの呼び出しごとに新しいMCP `ClientSession`が開始されます。特定のサーバーに対して明示的にセッションを開始したい場合は、次のようにします:
>
>    ```python
>    from langchain_mcp_adapters.tools import load_mcp_tools
>
>    client = MultiServerMCPClient({...})
>    async with client.session("math") as session:
>        tools = await load_mcp_tools(session)
>    ```

## Streamable HTTP

MCP は、[ストリーミング可能な HTTP](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports#streamable-http) トランスポートをサポートするようになりました。

[例](examples/servers/streamable-http-stateless/) ストリーミング可能な HTTP サーバーを起動するには、次のコマンドを実行します。

```bash
cd examples/servers/streamable-http-stateless/
uv run mcp-simple-streamablehttp-stateless --port 3000
```

あるいは、上記の例のように FastMCP を直接使用することもできます。

Python MCP SDK `streamablehttp_client` で使用するには、次のようにします:

```python
# examples/servers/streamable-http-stateless/ からサーバーを使用する

from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

from langgraph.prebuilt import create_react_agent
from langchain_mcp_adapters.tools import load_mcp_tools

async with streamablehttp_client("http://localhost:3000/mcp/") as (read, write, _):
    async with ClientSession(read, write) as session:
        # Initialize the connection
        await session.initialize()

        # Get tools
        tools = await load_mcp_tools(session)
        agent = create_react_agent("openai:gpt-4.1", tools)
        math_response = await agent.ainvoke({"messages": "what's (3 + 5) x 12?"})
```

`MultiServerMCPClient` と一緒に使用します:

```python
# examples/servers/streamable-http-stateless/ からサーバーを使用する
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.prebuilt import create_react_agent

client = MultiServerMCPClient(
    {
        "math": {
            "transport": "streamable_http",
            "url": "http://localhost:3000/mcp/"
        },
    }
)
tools = await client.get_tools()
agent = create_react_agent("openai:gpt-4.1", tools)
math_response = await agent.ainvoke({"messages": "what's (3 + 5) x 12?"})
```

## ランタイムヘッダーの受け渡し

MCPサーバーに接続する際に、接続設定の`headers`フィールドを使用して、カスタムヘッダー（認証用やトレース用など）を含めることができます。これは以下のトランスポートでサポートされています。

* `sse`
* `streamable_http`

### 例: `MultiServerMCPClient` でヘッダーを渡す

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.prebuilt import create_react_agent

client = MultiServerMCPClient(
    {
        "weather": {
            "transport": "streamable_http",
            "url": "http://localhost:8000/mcp",
            "headers": {
                "Authorization": "Bearer YOUR_TOKEN",
                "X-Custom-Header": "custom-value"
            },
        }
    }
)
tools = await client.get_tools()
agent = create_react_agent("openai:gpt-4.1", tools)
response = await agent.ainvoke({"messages": "what is the weather in nyc?"})
```

> `sse` および `streamable_http` トランスポートのみがランタイムヘッダーをサポートします。これらのヘッダーは、MCP サーバーへのすべての HTTP リクエストとともに渡されます。


## LangGraph StateGraph と併用する

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.graph import StateGraph, MessagesState, START
from langgraph.prebuilt import ToolNode, tools_condition

from langchain.chat_models import init_chat_model
model = init_chat_model("openai:gpt-4.1")

client = MultiServerMCPClient(
    {
        "math": {
            "command": "python",
            # math_server.pyファイルへの完全な絶対パスに更新してください。
            "args": ["./examples/math_server.py"],
            "transport": "stdio",
        },
        "weather": {
            # 天気予報サーバーをポート8000​​で起動するようにしてください
            "url": "http://localhost:8000/mcp/",
            "transport": "streamable_http",
        }
    }
)
tools = await client.get_tools()

def call_model(state: MessagesState):
    response = model.bind_tools(tools).invoke(state["messages"])
    return {"messages": response}

builder = StateGraph(MessagesState)
builder.add_node(call_model)
builder.add_node(ToolNode(tools))
builder.add_edge(START, "call_model")
builder.add_conditional_edges(
    "call_model",
    tools_condition,
)
builder.add_edge("tools", "call_model")
graph = builder.compile()
math_response = await graph.ainvoke({"messages": "what's (3 + 5) x 12?"})
weather_response = await graph.ainvoke({"messages": "what is the weather in nyc?"})
```

## LangGraph APIサーバーと併用

> [!TIP]
> LangGraph APIサーバーを使い始めるには、[こちらのガイド](https://langchain-ai.github.io/langgraph/tutorials/langgraph-platform/local-server/)をご覧ください。

LangGraph APIサーバーでMCPツールを使用するLangGraphエージェントを実行する場合は、以下の設定を行ってください。

```python
# graph.py
from contextlib import asynccontextmanager
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.prebuilt import create_react_agent

async def make_graph():
    client = MultiServerMCPClient(
        {
            "math": {
                "command": "python",
                # Make sure to update to the full absolute path to your math_server.py file
                "args": ["/path/to/math_server.py"],
                "transport": "stdio",
            },
            "weather": {
                # make sure you start your weather server on port 8000
                "url": "http://localhost:8000/mcp/",
                "transport": "streamable_http",
            }
        }
    )
    tools = await client.get_tools()
    agent = create_react_agent("openai:gpt-4.1", tools)
    return agent
```

[`langgraph.json`](https://langchain-ai.github.io/langgraph/cloud/reference/cli/#configuration-file) で、グラフのエントリポイントとして `make_graph` を指定してください:

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./graph.py:make_graph"
  }
}
```

## LangChain ツールを FastMCP サーバーに追加する

`to_fastmcp` を使用して LangChain ツールを FastMCP に変換し、初期化子を使用して `FastMCP` サーバーに追加します。

> [!NOTE]
> `tools` 引数は `mcp >= 1.9.1` 以降の FastMCP でのみ利用可能です。

```python
from langchain_core.tools import tool
from langchain_mcp_adapters.tools import to_fastmcp
from mcp.server.fastmcp import FastMCP


@tool
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b


fastmcp_tool = to_fastmcp(add)

mcp = FastMCP("Math", tools=[fastmcp_tool])
mcp.run(transport="stdio")
```
