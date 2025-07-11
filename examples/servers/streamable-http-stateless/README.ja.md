# MCP シンプルな StreamableHttp ステートレスサーバーの例

> [公式 Python MCP SDK サンプル](https://github.com/modelcontextprotocol/python-sdk/tree/main/examples/servers/simple-streamablehttp-stateless) を改変

セッション状態を維持せずに StreamableHttp トランスポートを実証するステートレス MCP サーバーの例です。この例は、リクエストを任意のインスタンスにルーティングできるマルチノード環境に MCP サーバーをデプロイする方法を理解するのに最適です。

## 機能

- StreamableHTTP トランスポートをステートレスモード (mcp_session_id=None) で使用します。
- 各リクエストごとに新しい一時的な接続が作成されます。
- リクエスト間でセッション状態は維持されません。
- タスクのライフサイクルは個々のリクエストに限定されます。
- マルチノード環境への導入に適しています。


## 使用方法

サーバーを起動します。

```bash
# Using default port 3000
uv run mcp-simple-streamablehttp-stateless

# Using custom port
uv run mcp-simple-streamablehttp-stateless --port 3000

# Custom logging level
uv run mcp-simple-streamablehttp-stateless --log-level DEBUG

# Enable JSON responses instead of SSE streams
uv run mcp-simple-streamablehttp-stateless --json-response
```

サーバーは「start-notification-stream」というツールを公開しており、以下の3つの引数を受け取ります。

- `interval`: 通知間隔（秒単位）（例：1.0）
- `count`: 送信する通知の数（例：5）
- `caller`: 呼び出し元の識別子文字列


## クライアント

HTTPクライアントを使用してこのサーバーに接続できます。現時点では、TypeScript SDKにのみストリーミング可能なHTTPクライアントサンプルが用意されています。また、テストには[Inspector](https://github.com/modelcontextprotocol/inspector)を使用することもできます。