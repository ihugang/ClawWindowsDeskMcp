# ClawDesk MCP Server

ClawDesk MCP Server 是一个 Windows 本地能力服务，遵循 Model Context Protocol (MCP) 标准协议，为 AI 助理（如 OpenClaw、Claude Desktop）提供安全、可控、可追溯的 Windows 系统操作能力。

## 功能特性

- **MCP 协议支持**: 实现 Model Context Protocol 标准协议
- **HTTP API**: 提供 HTTP 接口用于远程控制
- **实时监控**: Dashboard 窗口实时显示请求和处理过程
- **安全可控**: 基于白名单的权限管理和风险分级
- **审计追溯**: 完整的操作日志记录
- **系统托盘**: 便捷的图形化管理界面
- **灵活配置**: 支持 0.0.0.0 或 127.0.0.1 监听地址切换
- **文件操作**: 磁盘枚举、目录列表、文件读取（支持行范围）、内容搜索
- **剪贴板操作**: 通过 HTTP API 读写剪贴板内容（支持文本、图片、文件）
- **截图功能**: 捕获屏幕截图并返回 Base64 编码的图像数据
- **窗口管理**: 获取所有打开窗口的列表和详细信息
- **进程管理**: 获取所有运行进程的列表和详细信息
- **命令执行**: 执行系统命令并捕获输出

## 系统要求

- Windows 10/11 (x64, x86)
- 无需额外运行时依赖

## HTTP API

服务器启动后会在配置的端口（默认 35182）上监听 HTTP 请求。

**⚠️ 重要：从 v0.2.0 开始，所有 API 请求都需要 Bearer Token 认证。**

### 获取 Token

1. 启动服务器后，会自动生成 `config.json` 文件
2. 打开 `config.json`，找到 `auth_token` 字段
3. 复制 Token 值（64 字符的十六进制字符串）

示例 `config.json`：

```json
{
  "auth_token": "a1b2c3d4e5f6789012345678901234567890abcdefabcdefabcdefabcdefabcd",
  "server_port": 35182,
  "listen_address": "0.0.0.0",
  ...
}
```

### 使用 Token

在所有 HTTP 请求中添加 `Authorization` 头：

```bash
# 基本格式
curl -H "Authorization: Bearer YOUR_TOKEN" http://localhost:35182/status

# 实际示例
curl -H "Authorization: Bearer a1b2c3d4e5f6789012345678901234567890abcdefabcdefabcdefabcdefabcd" \
  http://192.168.31.3:35182/status
```

**注意**：

- Token 必须以 `Bearer ` 开头（注意空格）
- CORS 预检请求（OPTIONS）无需 Token
- 未授权请求返回 `401 Unauthorized`

详细说明请参考 [Authentication Guide](docs/Authentication.md)。

### 可用端点

#### 1. API 列表

```bash
GET http://localhost:35182/
```

返回所有可用的 API 端点列表。

#### 2. 健康检查

```bash
GET http://localhost:35182/health
```

响应：

```json
{ "status": "ok" }
```

#### 3. 获取状态

```bash
GET http://localhost:35182/status
```

响应：

```json
{
    "status": "running",
    "version": "0.2.0",
    "port": 35182,
    "listen_address": "0.0.0.0",
    "local_ip": "192.168.31.3",
    "license": "free",
    "uptime_seconds": 1234
}
```

#### 4. 获取磁盘列表

```bash
GET http://localhost:35182/disks
```

响应：

```json
[
    {
        "drive": "C:",
        "type": "fixed",
        "label": "Windows",
        "filesystem": "NTFS",
        "total_bytes": 500000000000,
        "free_bytes": 100000000000,
        "used_bytes": 400000000000
    }
]
```

#### 5. 列出目录内容

```bash
GET http://localhost:35182/list?path=C:\Users
```

响应：

```json
[
    {
        "name": "Documents",
        "type": "directory",
        "size": 0,
        "modified": "2026-02-03 10:30:00"
    },
    {
        "name": "file.txt",
        "type": "file",
        "size": 1024,
        "modified": "2026-02-03 12:00:00"
    }
]
```

#### 6. 读取文件内容

```bash
# 读取整个文件
GET http://localhost:35182/read?path=C:\test.txt

# 从第 10 行开始读取 20 行
GET http://localhost:35182/read?path=C:\test.txt&start=10&lines=20

# 读取最后 50 行
GET http://localhost:35182/read?path=C:\test.txt&tail=50

# 只获取行数
GET http://localhost:35182/read?path=C:\test.txt&count=true
```

响应：

```json
{
    "path": "C:\\test.txt",
    "total_lines": 100,
    "start_line": 0,
    "returned_lines": 100,
    "file_size": 5120,
    "content": "文件内容..."
}
```

#### 7. 搜索文件内容

```bash
# 区分大小写搜索
GET http://localhost:35182/search?path=C:\test.txt&query=keyword

# 不区分大小写搜索
GET http://localhost:35182/search?path=C:\test.txt&query=keyword&case=i

# 限制结果数量
GET http://localhost:35182/search?path=C:\test.txt&query=keyword&max=50
```

响应：

```json
{
    "path": "C:\\test.txt",
    "query": "keyword",
    "total_lines": 100,
    "match_count": 5,
    "case_sensitive": true,
    "matches": [
        {
            "line_number": 10,
            "content": "This line contains keyword"
        }
    ]
}
```

#### 8. 读取剪贴板

**增强功能**：支持文本、图片和文件三种类型！

```bash
GET http://localhost:35182/clipboard
```

响应类型 1 - 文本：

```json
{
    "type": "text",
    "content": "剪贴板文本内容",
    "length": 15,
    "empty": false
}
```

响应类型 2 - 图片（截图或复制的图片）：

```json
{
    "type": "image",
    "format": "png",
    "url": "http://192.168.31.3:35182/clipboard/image/clipboard_images/clipboard_20260203_223015.png",
    "path": "/clipboard/image/clipboard_images/clipboard_20260203_223015.png"
}
```

响应类型 3 - 文件（从资源管理器复制的文件）：

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

**下载剪贴板图片或文件**：

```bash
# 下载图片
curl -H "Authorization: Bearer $TOKEN" \
  "http://192.168.31.3:35182/clipboard/image/clipboard_images/clipboard_20260203_223015.png" \
  -o clipboard_image.png

# 下载文件
curl -H "Authorization: Bearer $TOKEN" \
  "http://192.168.31.3:35182/clipboard/file/20260203_223015_0_document.pdf" \
  -o document.pdf
```

#### 9. 写入剪贴板

```bash
PUT http://localhost:35182/clipboard
Content-Type: application/json

{
  "content": "要写入的内容"
}
```

响应：

```json
{
    "success": true,
    "length": 18
}
```

#### 10. 截图

```bash
# PNG 格式（默认）
GET http://localhost:35182/screenshot

# JPEG 格式
GET http://localhost:35182/screenshot?format=jpg
```

响应：

```json
{
    "success": true,
    "format": "png",
    "width": 1920,
    "height": 1080,
    "data": "iVBORw0KGgoAAAANSUhEUgAA..."
}
```

#### 11. 获取窗口列表

```bash
GET http://localhost:35182/windows
```

响应：

```json
[
    {
        "hwnd": 123456,
        "title": "Google Chrome",
        "class": "Chrome_WidgetWin_1",
        "visible": true,
        "minimized": false,
        "maximized": false,
        "process_id": 1234,
        "x": 100,
        "y": 100,
        "width": 1280,
        "height": 720
    }
]
```

#### 12. 获取进程列表

```bash
GET http://localhost:35182/processes
```

响应：

```json
[
    {
        "pid": 1234,
        "name": "chrome.exe",
        "path": "C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe",
        "threads": 42,
        "parent_pid": 5678,
        "memory_kb": 102400
    }
]
```

#### 13. 执行命令

```bash
POST http://localhost:35182/execute
Content-Type: application/json

{
  "command": "dir C:\\"
}
```

响应：

```json
{
    "success": true,
    "exit_code": 0,
    "output": "Volume in drive C is Windows\\n..."
}
```

#### 14. MCP 协议

##### 初始化连接

```bash
POST http://localhost:35182/mcp/initialize
Content-Type: application/json

{
  "protocolVersion": "2024-11-05",
  "capabilities": {},
  "clientInfo": {
    "name": "test-client",
    "version": "1.0.0"
  }
}
```

响应：

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

##### 列出可用工具

```bash
POST http://localhost:35182/mcp/tools/list
Content-Type: application/json

{}
```

响应：

```json
{
    "tools": [
        {
            "name": "read_file",
            "description": "Read file content",
            "inputSchema": {
                "type": "object",
                "properties": {
                    "path": {
                        "type": "string"
                    }
                },
                "required": ["path"]
            }
        }
    ]
}
```

##### 调用工具

```bash
POST http://localhost:35182/mcp/tools/call
Content-Type: application/json

{
  "name": "list_windows",
  "arguments": {}
}
```

响应：

```json
{
    "content": [
        {
            "type": "text",
            "text": "[...]"
        }
    ],
    "isError": false
}
```

#### 15. 退出服务器

```bash
GET http://localhost:35182/exit
```

响应：

```json
{ "status": "shutting down" }
```

### 使用示例

**重要：所有示例都需要添加 Authorization 头！**

首先设置 Token 变量（从 config.json 获取）：

```bash
# 设置 Token 变量
TOKEN="your-auth-token-from-config-json"
```

使用 curl 测试：

```bash
# 检查服务器状态
curl -H "Authorization: Bearer $TOKEN" http://localhost:35182/status

# 健康检查
curl -H "Authorization: Bearer $TOKEN" http://localhost:35182/health

# 获取磁盘列表
curl -H "Authorization: Bearer $TOKEN" http://localhost:35182/disks

# 列出目录
curl -H "Authorization: Bearer $TOKEN" "http://localhost:35182/list?path=C:\\"

# 读取文件
curl -H "Authorization: Bearer $TOKEN" "http://localhost:35182/read?path=C:\\test.txt"

# 搜索文件内容
curl -H "Authorization: Bearer $TOKEN" "http://localhost:35182/search?path=C:\\test.txt&query=keyword"

# 读取剪贴板
curl -H "Authorization: Bearer $TOKEN" http://localhost:35182/clipboard

# 写入剪贴板
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content":"Hello World"}' \
  http://localhost:35182/clipboard

# 截图（PNG）
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:35182/screenshot | jq -r '.data' | base64 -d > screenshot.png

# 截图（JPEG）
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:35182/screenshot?format=jpg" | jq -r '.data' | base64 -d > screenshot.jpg

# 获取窗口列表
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:35182/windows | jq '.[0:5]'

# 获取进程列表
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:35182/processes | jq '.[0:5]'

# 执行命令
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"command":"echo Hello"}' \
  http://localhost:35182/execute | jq .

# MCP 协议：初始化
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}' \
  http://localhost:35182/mcp/initialize | jq .

# MCP 协议：列出工具
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}' \
  http://localhost:35182/mcp/tools/list | jq '.tools | length'

# MCP 协议：调用工具
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"list_windows","arguments":{}}' \
  http://localhost:35182/mcp/tools/call | jq .

# 退出服务器
curl -H "Authorization: Bearer $TOKEN" http://localhost:35182/exit
```

使用 PowerShell：

```powershell
# 设置 Token（从 config.json 获取）
$TOKEN = "your-auth-token-from-config-json"
$headers = @{
    "Authorization" = "Bearer $TOKEN"
}

# 检查状态
Invoke-WebRequest -Uri "http://localhost:35182/status" -Headers $headers

# 读取剪贴板
Invoke-WebRequest -Uri "http://localhost:35182/clipboard" -Headers $headers

# 写入剪贴板
$body = @{ content = "Hello World" } | ConvertTo-Json
Invoke-WebRequest -Uri "http://localhost:35182/clipboard" `
  -Method PUT -ContentType "application/json" -Body $body -Headers $headers

# 截图
$response = Invoke-RestMethod -Uri "http://localhost:35182/screenshot" -Headers $headers
[System.Convert]::FromBase64String($response.data) | Set-Content -Path "screenshot.png" -Encoding Byte

# 获取窗口列表
Invoke-RestMethod -Uri "http://localhost:35182/windows" -Headers $headers

# 获取进程列表
Invoke-RestMethod -Uri "http://localhost:35182/processes" -Headers $headers

# 执行命令
$body = @{ command = "dir C:\" } | ConvertTo-Json
Invoke-RestMethod -Uri "http://localhost:35182/execute" `
  -Method POST -ContentType "application/json" -Body $body -Headers $headers

# 退出服务器
Invoke-WebRequest -Uri "http://localhost:35182/exit" -Headers $headers
```

## 托盘菜单功能

右键点击系统托盘图标可以访问以下功能：

- **状态信息**: 显示服务器运行状态、端口、监听地址、许可证信息
- **Usage Statistics**: 查看今日使用统计和配额
- **View Logs**: 打开日志目录
- **Dashboard**: 打开实时监控窗口（显示请求和处理过程）
- **Toggle Listen Address**: 在 0.0.0.0 和 127.0.0.1 之间切换监听地址
- **Open Config**: 用记事本打开配置文件
- **About**: 显示关于信息
- **Exit**: 退出服务器

## Dashboard 实时监控

Dashboard 是一个置顶的浮动窗口，用于实时观察服务器的运行状态。

### 功能

- **实时日志**: 显示接收的请求、处理过程和执行结果
- **日志类型**:
    - `[REQ]` - 接收到的请求
    - `[PRO]` - 正在处理
    - `[OK ]` - 成功完成
    - `[ERR]` - 发生错误
- **操作按钮**:
    - **Clear Logs**: 清空所有日志
    - **Copy All**: 复制所有日志到剪贴板
- **自动滚动**: 新日志自动滚动到底部
- **日志限制**: 最多保留 1000 条日志

### 使用方法

1. 右键点击托盘图标
2. 选择 "Dashboard"
3. Dashboard 窗口会弹出并置顶显示
4. 再次点击 "Dashboard" 可以隐藏窗口

### 日志示例

```
[18:30:45.123] [REQ] HTTP - GET /status HTTP/1.1
[18:30:45.125] [PRO] status - Retrieving server status...
[18:30:45.128] [OK ] status - Status returned: port=35182, license=free

[18:31:02.456] [REQ] HTTP - POST /exit HTTP/1.1
[18:31:02.458] [PRO] exit - Shutting down server...
[18:31:02.460] [OK ] exit - Shutdown command sent
```

详细文档请参考 [Dashboard.md](docs/Dashboard.md)。

## 配置说明

首次运行时，程序会自动生成 `config.json` 配置文件：

```json
{
    "auth_token": "自动生成的认证令牌",
    "server_port": 35182,
    "auto_port": true,
    "listen_address": "0.0.0.0",
    "allowed_dirs": ["C:/Users", "C:/Temp", "C:/Windows/Temp"],
    "allowed_apps": {
        "notepad": "C:/Windows/System32/notepad.exe",
        "calc": "C:/Windows/System32/calc.exe"
    },
    "allowed_commands": ["npm", "git", "python", "node"],
    "license_key": ""
}
```

### 配置项说明

- **auth_token**: Bearer Token，用于客户端认证（自动生成）
- **server_port**: 服务器监听端口（默认 35182）
- **auto_port**: 端口被占用时是否自动选择随机端口
- **listen_address**: 监听地址
    - `"0.0.0.0"`: 允许网络访问（局域网内其他设备可访问）
    - `"127.0.0.1"`: 仅本地访问
- **allowed_dirs**: 允许访问的目录白名单
- **allowed_apps**: 允许启动的应用程序白名单
- **allowed_commands**: 允许执行的命令白名单
- **license_key**: 许可证密钥（可选）

## 构建说明

### 开发环境

- macOS (Apple Silicon 或 Intel)
- CMake 3.20+
- MinGW-w64 交叉编译工具链

### 安装依赖

```bash
# 安装 MinGW-w64
brew install mingw-w64

# 安装 CMake
brew install cmake
```

### 编译

```bash
# 编译所有架构（x64, x86）
./build.sh

# 或手动编译单个架构
mkdir -p build-x64
cd build-x64
cmake .. -DCMAKE_TOOLCHAIN_FILE=../toolchain-mingw-x64.cmake \
         -DCMAKE_BUILD_TYPE=Release
make -j$(sysctl -n hw.ncpu)
```

### 构建产物

- `build-x64/ClawDeskMCP.exe` - Windows x64 版本
- `build-x86/ClawDeskMCP.exe` - Windows x86 版本

编译完成后会自动复制到 `/Volumes/Test/ClawDeskMCP/` 目录。

## 使用说明

1. **运行程序**：
    - 双击 `ClawDeskMCP-x64.exe`（64位系统）或 `ClawDeskMCP-x86.exe`（32位系统）
    - 程序会在系统托盘显示图标

2. **查看托盘图标**：
    - 图标会出现在任务栏右下角（系统托盘区）
    - 右键点击图标可以看到完整菜单

3. **自动生成的文件**：
    - `config.json` - 配置文件（首次运行自动生成）
    - `logs/audit-YYYY-MM-DD.log` - 审计日志

4. **网络访问**：
    - 本地访问: `http://localhost:35182`
    - 局域网访问: `http://192.168.x.x:35182`（需要 listen_address 设置为 "0.0.0.0"）

## 安全注意事项

- **监听地址**: 如果设置为 `0.0.0.0`，局域网内的其他设备可以访问服务器
- **防火墙**: Windows 防火墙可能会阻止网络访问，需要添加例外
- **认证**: 目前 HTTP API 未实现认证，建议仅在可信网络中使用
- **白名单**: 通过配置文件限制可访问的目录、应用和命令

## 许可证

Copyright © 2026 ClawDesk

## 版本历史

- v0.2.0 - 当前版本
    - ✅ 基础项目结构
    - ✅ 交叉编译环境配置
    - ✅ ConfigManager（配置管理）
    - ✅ AuditLogger（审计日志）
    - ✅ 系统托盘界面
    - ✅ HTTP API（健康检查、状态、退出）
    - ✅ 监听地址切换（0.0.0.0 / 127.0.0.1）
    - ✅ Dashboard 实时监控窗口
    - ✅ 自动防火墙配置
    - ✅ 磁盘枚举和目录列表
    - ✅ 文件读取（支持行范围、尾部读取、行数统计）
    - ✅ 文件内容搜索（支持大小写敏感/不敏感）
    - ✅ 剪贴板读写操作
    - ✅ 屏幕截图（PNG/JPEG 格式，Base64 编码）
    - ✅ 窗口列表（标题、类名、位置、状态）
    - ✅ 进程列表（PID、名称、路径、内存使用）
    - ✅ 命令执行（捕获输出和退出代码）
    - ✅ MCP 协议（initialize、tools/list、tools/call）
    - ⏳ 工具功能实现（待完善）
