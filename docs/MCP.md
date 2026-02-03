# ClawDesk MCP Server - MCP 协议文档

## 概述

ClawDesk MCP Server 实现了 Model Context Protocol (MCP) 标准协议，使 AI 助手能够通过标准化的方式调用 Windows 系统功能。

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

**参数**: 无

**示例**:

```json
{
    "name": "get_clipboard",
    "arguments": {}
}
```

### 5. set_clipboard

设置剪贴板内容。

**参数**:

- `content` (string, 必需): 要设置的内容

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

### curl 示例

```bash
# 初始化
curl -X POST http://localhost:35182/mcp/initialize \
  -H "Content-Type: application/json" \
  -d '{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}'

# 列出工具
curl -X POST http://localhost:35182/mcp/tools/list \
  -H "Content-Type: application/json" \
  -d '{}'

# 调用工具
curl -X POST http://localhost:35182/mcp/tools/call \
  -H "Content-Type: application/json" \
  -d '{"name":"list_windows","arguments":{}}'
```

### Python 示例

```python
import requests
import json

SERVER = "http://localhost:35182"

# 初始化
init_response = requests.post(
    f"{SERVER}/mcp/initialize",
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
    json={}
)
print("Tools:", tools_response.json())

# 调用工具
call_response = requests.post(
    f"{SERVER}/mcp/tools/call",
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

// 初始化
fetch(`${SERVER}/mcp/initialize`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
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
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({}),
})
    .then((res) => res.json())
    .then((data) => console.log("Tools:", data));

// 调用工具
fetch(`${SERVER}/mcp/tools/call`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
        name: "list_windows",
        arguments: {},
    }),
})
    .then((res) => res.json())
    .then((data) => console.log("Result:", data));
```

## 与 Claude Desktop 集成

### 配置文件

在 Claude Desktop 的配置文件中添加：

```json
{
    "mcpServers": {
        "clawdesk": {
            "command": "http",
            "args": ["http://localhost:35182/mcp"]
        }
    }
}
```

### 使用方法

1. 启动 ClawDesk MCP Server
2. 启动 Claude Desktop
3. Claude 将自动连接到服务器
4. 在对话中请求 Claude 使用工具

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

1. **本地访问**: 默认仅监听 127.0.0.1，仅本地访问
2. **无认证**: 当前版本未实现认证机制
3. **命令执行**: execute_command 工具具有高风险，请谨慎使用
4. **文件访问**: 建议配置 allowed_dirs 白名单限制访问范围

## 限制和已知问题

1. **简化实现**: 当前为基础实现，未完全遵循 MCP 规范
2. **工具参数**: 部分工具的参数解析较为简单
3. **错误处理**: 错误信息可能不够详细
4. **流式响应**: 不支持流式响应
5. **资源管理**: 不支持资源（resources）功能
6. **提示词**: 不支持提示词（prompts）功能

## 未来计划

- [ ] 完整的 MCP 规范实现
- [ ] Bearer Token 认证
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
