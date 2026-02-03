# ClawDesk MCP Server - API 文档

## 概述

ClawDesk MCP Server 提供基于 HTTP 的 RESTful API，用于远程控制 Windows 系统操作。

**基础 URL**: `http://localhost:35182` (默认端口)

**网络访问**: `http://192.168.x.x:35182` (需要 listen_address 设置为 "0.0.0.0")

## 通用响应格式

所有 API 响应都使用 JSON 格式，并包含 CORS 头以支持跨域访问。

### 成功响应

```json
{
  "status": "ok",
  "data": { ... }
}
```

### 错误响应

```json
{
    "error": "错误描述"
}
```

## API 端点

### 1. 获取 API 列表

获取所有可用的 API 端点列表。

**请求**:

```
GET /
```

**响应**:

```json
{
  "name": "ClawDesk MCP Server",
  "version": "0.2.0",
  "status": "running",
  "endpoints": [
    {
      "path": "/",
      "method": "GET",
      "description": "API endpoint list"
    },
    ...
  ]
}
```

---

### 2. 健康检查

快速检查服务器是否运行正常。

**请求**:

```
GET /health
```

**响应**:

```json
{
    "status": "ok"
}
```

---

### 3. 获取服务器状态

获取服务器的详细状态信息。

**请求**:

```
GET /status
GET /sts  (别名)
```

**响应**:

```json
{
    "status": "running",
    "version": "0.2.0",
    "port": 35182,
    "listen_address": "0.0.0.0",
    "local_ip": "192.168.31.3",
    "license": "free",
    "uptime_seconds": 1234,
    "endpoints": ["/sts", "/status", "/health", "/exit"]
}
```

**字段说明**:

- `status`: 服务器状态 (running)
- `version`: 服务器版本号
- `port`: 监听端口
- `listen_address`: 监听地址 (0.0.0.0 或 127.0.0.1)
- `local_ip`: 本机 IP 地址
- `license`: 许可证类型 (free 或 professional)
- `uptime_seconds`: 运行时间（秒）

---

### 4. 获取磁盘列表

列出所有磁盘驱动器及其详细信息。

**请求**:

```
GET /disks
```

**响应**:

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
    },
    {
        "drive": "D:",
        "type": "removable",
        "label": "USB Drive",
        "filesystem": "FAT32",
        "total_bytes": 32000000000,
        "free_bytes": 16000000000,
        "used_bytes": 16000000000
    }
]
```

**字段说明**:

- `drive`: 驱动器盘符
- `type`: 驱动器类型
    - `fixed`: 固定磁盘
    - `removable`: 可移动磁盘
    - `network`: 网络驱动器
    - `cdrom`: 光驱
    - `ramdisk`: 内存盘
- `label`: 卷标
- `filesystem`: 文件系统类型 (NTFS, FAT32, exFAT 等)
- `total_bytes`: 总容量（字节）
- `free_bytes`: 可用空间（字节）
- `used_bytes`: 已用空间（字节）

---

### 5. 列出目录内容

列出指定目录下的所有文件和子目录。

**请求**:

```
GET /list?path=<目录路径>
```

**参数**:

- `path` (必需): 目录路径，支持 URL 编码

**示例**:

```bash
# Windows 路径
GET /list?path=C:\Users
GET /list?path=C:\Users\Documents

# URL 编码路径
GET /list?path=C%3A%5CUsers
```

**响应**:

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

**字段说明**:

- `name`: 文件或目录名
- `type`: 类型 (file 或 directory)
- `size`: 文件大小（字节），目录为 0
- `modified`: 最后修改时间

**错误响应**:

```json
{
    "error": "File not found or access denied"
}
```

---

### 6. 读取文件内容

读取文件内容，支持行范围、尾部读取和行数统计。

**请求**:

```
GET /read?path=<文件路径>&start=<起始行>&lines=<行数>&tail=<尾部行数>&count=<是否只统计>
```

**参数**:

- `path` (必需): 文件路径
- `start` (可选): 起始行号，默认 0
- `lines` (可选): 读取行数，默认全部
- `tail` (可选): 从尾部读取 n 行
- `count` (可选): 只返回行数，不返回内容 (true/1)

**示例**:

```bash
# 读取整个文件
GET /read?path=C:\test.txt

# 从第 10 行开始读取 20 行
GET /read?path=C:\test.txt&start=10&lines=20

# 读取最后 50 行
GET /read?path=C:\test.txt&tail=50

# 只获取行数
GET /read?path=C:\test.txt&count=true
```

**响应（完整内容）**:

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

**响应（仅行数）**:

```json
{
    "path": "C:\\test.txt",
    "total_lines": 100,
    "file_size": 5120
}
```

**限制**:

- 最大文件大小: 10MB
- 超过限制返回 413 错误

**错误响应**:

```json
{
    "error": "File not found or access denied"
}
```

```json
{
    "error": "File too large",
    "max_size": "10MB"
}
```

---

### 7. 搜索文件内容

在文件中搜索指定关键词。

**请求**:

```
GET /search?path=<文件路径>&query=<搜索词>&case=<大小写>&max=<最大结果数>
```

**参数**:

- `path` (必需): 文件路径
- `query` (必需): 搜索关键词
- `case` (可选): 大小写敏感性
    - `sensitive` (默认): 区分大小写
    - `i` 或 `insensitive`: 不区分大小写
- `max` (可选): 最大结果数，默认 100，最大 1000

**示例**:

```bash
# 区分大小写搜索
GET /search?path=C:\test.txt&query=keyword

# 不区分大小写搜索
GET /search?path=C:\test.txt&query=keyword&case=i

# 限制结果数量
GET /search?path=C:\test.txt&query=keyword&max=50
```

**响应**:

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
        },
        {
            "line_number": 25,
            "content": "Another line with keyword"
        }
    ]
}
```

**字段说明**:

- `path`: 文件路径
- `query`: 搜索关键词
- `total_lines`: 文件总行数
- `match_count`: 匹配数量
- `case_sensitive`: 是否区分大小写
- `matches`: 匹配结果数组
    - `line_number`: 行号（从 0 开始）
    - `content`: 行内容

**限制**:

- 最大文件大小: 10MB
- 最大结果数: 1000

**错误响应**:

```json
{
    "error": "Missing required parameter: query"
}
```

---

### 8. 读取剪贴板

获取当前剪贴板的文本内容。

**请求**:

```
GET /clipboard
```

**响应（有内容）**:

```json
{
    "content": "剪贴板内容",
    "length": 15,
    "empty": false
}
```

**响应（空剪贴板）**:

```json
{
    "content": "",
    "empty": true
}
```

**字段说明**:

- `content`: 剪贴板文本内容
- `length`: 内容长度（字符数）
- `empty`: 是否为空

**错误响应**:

```json
{
    "error": "Failed to open clipboard"
}
```

---

### 9. 写入剪贴板

设置剪贴板的文本内容。

**请求**:

```
PUT /clipboard
Content-Type: application/json

{
  "content": "要写入的内容"
}
```

**请求体**:

```json
{
    "content": "要写入的内容"
}
```

**字段说明**:

- `content` (必需): 要写入剪贴板的文本内容

**响应**:

```json
{
    "success": true,
    "length": 18
}
```

**字段说明**:

- `success`: 是否成功
- `length`: 写入的内容长度

**示例**:

```bash
# 写入简单文本
curl -X PUT -H "Content-Type: application/json" \
  -d '{"content":"Hello World"}' \
  http://localhost:35182/clipboard

# 写入多行文本
curl -X PUT -H "Content-Type: application/json" \
  -d '{"content":"Line 1\nLine 2\nLine 3"}' \
  http://localhost:35182/clipboard

# 写入特殊字符
curl -X PUT -H "Content-Type: application/json" \
  -d '{"content":"Special: \"quotes\", \ttabs, and \\backslashes\\"}' \
  http://localhost:35182/clipboard
```

**错误响应**:

```json
{
    "error": "Missing request body"
}
```

```json
{
    "error": "Missing 'content' field in JSON"
}
```

```json
{
    "error": "Failed to open clipboard"
}
```

---

### 10. 截图

捕获屏幕截图并返回 Base64 编码的图像数据。

**请求**:

```
GET /screenshot?format=<格式>
```

**参数**:

- `format` (可选): 图像格式
    - `png` (默认): PNG 格式
    - `jpg` 或 `jpeg`: JPEG 格式

**示例**:

```bash
# PNG 格式（默认）
GET /screenshot

# JPEG 格式
GET /screenshot?format=jpg
```

**响应**:

```json
{
    "success": true,
    "format": "png",
    "width": 1920,
    "height": 1080,
    "data": "iVBORw0KGgoAAAANSUhEUgAA..."
}
```

**字段说明**:

- `success`: 是否成功
- `format`: 图像格式 (png 或 jpg)
- `width`: 屏幕宽度（像素）
- `height`: 屏幕高度（像素）
- `data`: Base64 编码的图像数据

**使用示例**:

保存为文件（bash）:

```bash
# PNG
curl http://localhost:35182/screenshot | jq -r '.data' | base64 -d > screenshot.png

# JPEG
curl "http://localhost:35182/screenshot?format=jpg" | jq -r '.data' | base64 -d > screenshot.jpg
```

保存为文件（PowerShell）:

```powershell
$response = Invoke-RestMethod -Uri "http://localhost:35182/screenshot"
[System.Convert]::FromBase64String($response.data) | Set-Content -Path "screenshot.png" -Encoding Byte
```

在浏览器中显示（JavaScript）:

```javascript
fetch("http://localhost:35182/screenshot")
    .then((response) => response.json())
    .then((data) => {
        const img = document.createElement("img");
        img.src = `data:image/${data.format};base64,${data.data}`;
        document.body.appendChild(img);
    });
```

**错误响应**:

```json
{
    "error": "Failed to capture screenshot"
}
```

---

### 11. 退出服务器

优雅地关闭服务器。

**请求**:

```
GET /exit
```

**响应**:

```json
{
    "status": "shutting down"
}
```

**说明**:

- 服务器会先关闭 Dashboard 窗口
- 然后关闭主窗口和托盘图标
- 最后停止 HTTP 服务器线程

---

## CORS 支持

所有 API 端点都支持 CORS（跨域资源共享），允许从浏览器直接调用。

**CORS 头**:

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, PUT, OPTIONS
Access-Control-Allow-Headers: Content-Type
```

**OPTIONS 预检请求**:

```
OPTIONS /list?path=C:\
```

响应:

```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, OPTIONS
Access-Control-Allow-Headers: Content-Type
```

---

## 错误代码

| HTTP 状态码 | 说明           |
| ----------- | -------------- |
| 200         | 成功           |
| 400         | 请求参数错误   |
| 404         | 资源未找到     |
| 413         | 文件过大       |
| 500         | 服务器内部错误 |

---

## 使用示例

### curl 示例

```bash
# 获取状态
curl http://localhost:35182/status

# 列出目录
curl "http://localhost:35182/list?path=C:\\"

# 读取文件
curl "http://localhost:35182/read?path=C:\\test.txt&tail=10"

# 搜索文件
curl "http://localhost:35182/search?path=C:\\test.txt&query=error&case=i"

# 读取剪贴板
curl http://localhost:35182/clipboard

# 写入剪贴板
curl -X PUT -H "Content-Type: application/json" \
  -d '{"content":"Hello from API"}' \
  http://localhost:35182/clipboard

# 截图（PNG）
curl http://localhost:35182/screenshot | jq -r '.data' | base64 -d > screenshot.png

# 截图（JPEG）
curl "http://localhost:35182/screenshot?format=jpg" | jq -r '.data' | base64 -d > screenshot.jpg
```

### PowerShell 示例

```powershell
# 获取状态
Invoke-RestMethod -Uri "http://localhost:35182/status"

# 列出目录
Invoke-RestMethod -Uri "http://localhost:35182/list?path=C:\"

# 读取文件
Invoke-RestMethod -Uri "http://localhost:35182/read?path=C:\test.txt"

# 读取剪贴板
Invoke-RestMethod -Uri "http://localhost:35182/clipboard"

# 写入剪贴板
$body = @{ content = "Hello World" } | ConvertTo-Json
Invoke-RestMethod -Uri "http://localhost:35182/clipboard" `
  -Method PUT -ContentType "application/json" -Body $body

# 截图
$response = Invoke-RestMethod -Uri "http://localhost:35182/screenshot"
[System.Convert]::FromBase64String($response.data) | Set-Content -Path "screenshot.png" -Encoding Byte
```

### JavaScript 示例

```javascript
// 获取状态
fetch("http://localhost:35182/status")
    .then((response) => response.json())
    .then((data) => console.log(data));

// 读取剪贴板
fetch("http://localhost:35182/clipboard")
    .then((response) => response.json())
    .then((data) => console.log("Clipboard:", data.content));

// 写入剪贴板
fetch("http://localhost:35182/clipboard", {
    method: "PUT",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ content: "Hello from JavaScript" }),
})
    .then((response) => response.json())
    .then((data) => console.log("Success:", data));

// 截图并显示
fetch("http://localhost:35182/screenshot")
    .then((response) => response.json())
    .then((data) => {
        const img = document.createElement("img");
        img.src = `data:image/${data.format};base64,${data.data}`;
        document.body.appendChild(img);
    });
```

---

## 安全注意事项

1. **认证**: 当前版本未实现认证机制，建议仅在可信网络中使用
2. **监听地址**:
    - `127.0.0.1`: 仅本地访问，最安全
    - `0.0.0.0`: 允许网络访问，需要防火墙保护
3. **文件访问**: 建议在配置文件中设置 `allowed_dirs` 白名单
4. **防火墙**: 使用 `0.0.0.0` 时需要添加 Windows 防火墙规则

---

## 版本历史

- v0.2.0 (2026-02-03)
    - ✅ 基础 HTTP API
    - ✅ 磁盘和文件操作
    - ✅ 剪贴板操作
    - ✅ CORS 支持
