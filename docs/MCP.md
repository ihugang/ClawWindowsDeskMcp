# ClawDesk MCP Server - MCP 协议文档

## 概述

ClawDesk MCP Server 实现了 Model Context Protocol (MCP) 标准协议，使 AI 助手能够通过标准化的方式调用 Windows 系统功能。

**⚠️ 重要：从 v0.2.0 开始，所有 MCP 请求都需要 Bearer Token 认证。**

## 认证

### 获取 Token

1. 启动 ClawDesk MCP Server
2. 打开 `config.json` 文件
3. 复制 `auth_token` 字段的值

示例：

```json
{
  "auth_token": "a1b2c3d4e5f6789012345678901234567890abcdefabcdefabcdefabcdefabcd",
  ...
}
```

### 使用 Token

所有 MCP 请求都必须在 HTTP 头中包含 Authorization：

```
Authorization: Bearer YOUR_AUTH_TOKEN
```

示例：

```bash
curl -X POST \
  -H "Authorization: Bearer a1b2c3d4..." \
  -H "Content-Type: application/json" \
  -d '{"protocolVersion":"2024-11-05",...}' \
  http://localhost:35182/mcp/initialize
```

详细说明请参考 [Authentication Guide](Authentication.md)。

## MCP 协议版本

- **协议版本**: 2024-11-05
- **实现状态**: 基础实现
- **支持的功能**: 工具调用

## 端点

### 1. 初始化连接

建立 MCP 连接并交换能力信息。

**端点**: `POST /mcp/initialize`

**请求**:

```json
{
    "protocolVersion": "2024-11-05",
    "capabilities": {},
    "clientInfo": {
        "name": "client-name",
        "version": "1.0.0"
    }
}
```

**响应**:

```json
{
    "protocolVersion": "2024-11-05",
    "capabilities": {
        "tools": {}
    },
    "serverInfo": {
        "name": "ClawDesk MCP Server",
        "version": "0.2.0"
    }
}
```

### 2. 列出可用工具

获取服务器提供的所有工具列表。

**端点**: `POST /mcp/tools/list`

**请求**:

```json
{}
```

**响应**:

```json
{
    "tools": [
        {
            "name": "read_file",
            "description": "Read file content with optional line range",
            "inputSchema": {
                "type": "object",
                "properties": {
                    "path": {
                        "type": "string",
                        "description": "File path"
                    },
                    "start": {
                        "type": "number",
                        "description": "Start line (optional)"
                    },
                    "lines": {
                        "type": "number",
                        "description": "Number of lines (optional)"
                    },
                    "tail": {
                        "type": "number",
                        "description": "Read last n lines (optional)"
                    }
                },
                "required": ["path"]
            }
        }
    ]
}
```

### 3. 调用工具

执行指定的工具。

**端点**: `POST /mcp/tools/call`

**请求**:

```json
{
    "name": "list_windows",
    "arguments": {}
}
```

**响应**:

```json
{
    "content": [
        {
            "type": "text",
            "text": "[{\"hwnd\":123456,\"title\":\"Chrome\",...}]"
        }
    ],
    "isError": false
}
```

## 可用工具

### 1. read_file

读取文件内容。

**参数**:

- `path` (string, 必需): 文件路径
- `start` (number, 可选): 起始行号
- `lines` (number, 可选): 读取行数
- `tail` (number, 可选): 从尾部读取 n 行

**示例**:

```json
{
    "name": "read_file",
    "arguments": {
        "path": "C:\\test.txt",
        "tail": 10
    }
}
```

### 2. search_file

在文件中搜索文本。

**参数**:

- `path` (string, 必需): 文件路径
- `query` (string, 必需): 搜索关键词
- `case_sensitive` (boolean, 可选): 是否区分大小写

**示例**:

```json
{
    "name": "search_file",
    "arguments": {
        "path": "C:\\log.txt",
        "query": "error",
        "case_sensitive": false
    }
}
```

### 3. list_directory

列出目录内容。

**参数**:

- `path` (string, 必需): 目录路径

**示例**:

```json
{
    "name": "list_directory",
    "arguments": {
        "path": "C:\\Users"
    }
}
```

### 4. get_clipboard

获取剪贴板内容。

**增强功能**：支持文本、图片和文件三种类型！

**参数**: 无

**返回类型**:

- `type: "text"` - 文本内容
- `type: "image"` - 图片（返回 URL）
- `type: "files"` - 文件列表（返回 URL 数组）

**示例**:

```json
{
    "name": "get_clipboard",
    "arguments": {}
}
```

**响应示例 1 - 文本**:

```json
{
    "type": "text",
    "content": "Hello World",
    "length": 11,
    "empty": false
}
```

**响应示例 2 - 图片**:

```json
{
    "type": "image",
    "format": "png",
    "url": "http://192.168.31.3:35182/clipboard/image/clipboard_images/clipboard_20260203_223015.png",
    "path": "/clipboard/image/clipboard_images/clipboard_20260203_223015.png"
}
```

**响应示例 3 - 文件**:

```json
{
    "type": "files",
    "files": [
        {
            "name": "20260203_223015_0_document.pdf",
            "url": "http://192.168.31.3:35182/clipboard/file/20260203_223015_0_document.pdf",
            "path": "/clipboard/file/20260203_223015_0_document.pdf"
        }
    ]
}
```

**使用场景**:

- 获取用户截图（Win+Shift+S）
- 获取用户复制的文件
- 获取用户复制的文本

### 5. set_clipboard

设置剪贴板内容（仅支持文本）。

**参数**:

- `content` (string, 必需): 要设置的文本内容

**示例**:

```json
{
    "name": "set_clipboard",
    "arguments": {
        "content": "Hello World"
    }
}
```

### 6. take_screenshot

捕获屏幕截图。

**参数**:

- `format` (string, 可选): 图像格式 (png 或 jpg)

**示例**:

```json
{
    "name": "take_screenshot",
    "arguments": {
        "format": "png"
    }
}
```

### 7. list_windows

列出所有打开的窗口。

**参数**: 无

**示例**:

```json
{
    "name": "list_windows",
    "arguments": {}
}
```

### 8. list_processes

列出所有运行中的进程。

**参数**: 无

**示例**:

```json
{
    "name": "list_processes",
    "arguments": {}
}
```

### 9. execute_command

执行系统命令。

**参数**:

- `command` (string, 必需): 要执行的命令

**示例**:

```json
{
    "name": "execute_command",
    "arguments": {
        "command": "dir C:\\"
    }
}
```

## 错误处理

当工具调用失败时，响应中的 `isError` 字段将为 `true`，并在 `content` 中包含错误信息。

**错误响应示例**:

```json
{
    "content": [
        {
            "type": "text",
            "text": "Error: File not found"
        }
    ],
    "isError": true
}
```

## 使用示例

**重要：所有示例都需要添加 Authorization 头！**

### curl 示例

```bash
# 设置 Token（从 config.json 获取）
TOKEN="your-auth-token-from-config-json"

# 初始化
curl -X POST http://localhost:35182/mcp/initialize \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}'

# 列出工具
curl -X POST http://localhost:35182/mcp/tools/list \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}'

# 调用工具
curl -X POST http://localhost:35182/mcp/tools/call \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"list_windows","arguments":{}}'
```

### Python 示例

```python
import requests
import json

SERVER = "http://localhost:35182"
TOKEN = "your-auth-token-from-config-json"  # 从 config.json 获取

# 设置认证头
headers = {
    "Authorization": f"Bearer {TOKEN}",
    "Content-Type": "application/json"
}

# 初始化
init_response = requests.post(
    f"{SERVER}/mcp/initialize",
    headers=headers,
    json={
        "protocolVersion": "2024-11-05",
        "capabilities": {},
        "clientInfo": {"name": "python-client", "version": "1.0"}
    }
)
print("Initialize:", init_response.json())

# 列出工具
tools_response = requests.post(
    f"{SERVER}/mcp/tools/list",
    headers=headers,
    json={}
)
print("Tools:", tools_response.json())

# 调用工具
call_response = requests.post(
    f"{SERVER}/mcp/tools/call",
    headers=headers,
    json={
        "name": "list_windows",
        "arguments": {}
    }
)
print("Result:", call_response.json())
```

### JavaScript 示例

```javascript
const SERVER = "http://localhost:35182";
const TOKEN = "your-auth-token-from-config-json"; // 从 config.json 获取

// 设置认证头
const headers = {
    Authorization: `Bearer ${TOKEN}`,
    "Content-Type": "application/json",
};

// 初始化
fetch(`${SERVER}/mcp/initialize`, {
    method: "POST",
    headers: headers,
    body: JSON.stringify({
        protocolVersion: "2024-11-05",
        capabilities: {},
        clientInfo: { name: "js-client", version: "1.0" },
    }),
})
    .then((res) => res.json())
    .then((data) => console.log("Initialize:", data));

// 列出工具
fetch(`${SERVER}/mcp/tools/list`, {
    method: "POST",
    headers: headers,
    body: JSON.stringify({}),
})
    .then((res) => res.json())
    .then((data) => console.log("Tools:", data));

// 调用工具
fetch(`${SERVER}/mcp/tools/call`, {
    method: "POST",
    headers: headers,
    body: JSON.stringify({
        name: "list_windows",
        arguments: {},
    }),
})
    .then((res) => res.json())
    .then((data) => console.log("Result:", data));
```

## 与 Claude Desktop 集成

**重要：需要配置 Token 认证！**

### 配置方法

由于 ClawDesk Server 需要 Bearer Token 认证，你需要创建一个 MCP 客户端脚本来桥接 Claude Desktop 和 ClawDesk Server。

**步骤 1：获取 Token**

从 ClawDesk Server 的 `config.json` 文件中获取 `auth_token`。

**步骤 2：创建 MCP 客户端**

创建 `clawdesk-mcp-client.js` 文件：

```javascript
// clawdesk-mcp-client.js
const fetch = require("node-fetch");

const SERVER_URL = process.env.CLAWDESK_URL || "http://localhost:35182";
const AUTH_TOKEN = process.env.CLAWDESK_TOKEN;

if (!AUTH_TOKEN) {
    console.error("Error: CLAWDESK_TOKEN environment variable not set");
    process.exit(1);
}

const headers = {
    Authorization: `Bearer ${AUTH_TOKEN}`,
    "Content-Type": "application/json",
};

// 实现 MCP 客户端逻辑
// 处理 stdin/stdout 通信
// 转发请求到 ClawDesk Server
// ...
```

**步骤 3：配置 Claude Desktop**

编辑 Claude Desktop 配置文件：

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

添加配置：

```json
{
    "mcpServers": {
        "clawdesk": {
            "command": "node",
            "args": ["/path/to/clawdesk-mcp-client.js"],
            "env": {
                "CLAWDESK_URL": "http://localhost:35182",
                "CLAWDESK_TOKEN": "your-auth-token-from-config-json"
            }
        }
    }
}
```

### 使用方法

1. 启动 ClawDesk MCP Server
2. 从 `config.json` 获取 `auth_token`
3. 在 Claude Desktop 配置中设置 `CLAWDESK_TOKEN`
4. 启动 Claude Desktop
5. Claude 将通过客户端连接到服务器

**示例对话**:

```
User: 请列出当前打开的所有窗口

Claude: 我来帮你查看当前打开的窗口...
[调用 list_windows 工具]
当前打开的窗口有：
1. Google Chrome - ...
2. Visual Studio Code - ...
...
```

## 安全注意事项

1. **Token 认证**: v0.2.0 开始所有请求都需要 Bearer Token
2. **Token 安全**: 不要将 Token 提交到版本控制或公开分享
3. **网络访问**: 可配置监听 0.0.0.0（网络）或 127.0.0.1（本地）
4. **命令执行**: execute_command 工具具有高风险，请谨慎使用
5. **文件访问**: 建议配置 allowed_dirs 白名单限制访问范围
6. **防火墙**: 网络访问时建议配置防火墙规则

## 限制和已知问题

1. **简化实现**: 当前为基础实现，未完全遵循 MCP 规范
2. **工具参数**: 部分工具的参数解析较为简单
3. **错误处理**: 错误信息可能不够详细
4. **流式响应**: 不支持流式响应
5. **资源管理**: 不支持资源（resources）功能
6. **提示词**: 不支持提示词（prompts）功能

## 未来计划

- [ ] 完整的 MCP 规范实现
- [x] Bearer Token 认证（v0.2.0 已实现）
- [ ] 资源（resources）支持
- [ ] 提示词（prompts）支持
- [ ] 流式响应
- [ ] 更详细的错误信息
- [ ] 工具调用日志和审计
- [ ] 用户确认机制

## 参考资料

- [Model Context Protocol 规范](https://modelcontextprotocol.io/)
- [MCP GitHub](https://github.com/modelcontextprotocol)
- [Claude Desktop MCP 文档](https://docs.anthropic.com/claude/docs/mcp)

## 版本历史

- v0.2.0 (2026-02-03)
    - ✅ 基础 MCP 协议实现
    - ✅ 9 个工具支持
    - ✅ initialize、tools/list、tools/call 端点
