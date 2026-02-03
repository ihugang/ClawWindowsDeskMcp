# 需求文档：ClawDesk MCP Server

## 简介

ClawDesk MCP Server 是一个 Windows 本地能力服务，遵循 Model Context Protocol (MCP) 标准协议，为 AI 助理（如 OpenClaw、Claude Desktop）提供安全、可控、可追溯的 Windows 系统操作能力。

产品目标：让 AI 能安全、可控、可追溯地帮助用户操作 Windows。

## 术语表

- **MCP_Server**：实现 MCP 协议的本地服务器，提供工具调用能力
- **MCP_Client**：发起工具调用请求的客户端应用（如 OpenClaw、Claude Desktop）
- **Capability_Tool**：对外暴露的单一原子能力（如截图、读文件）
- **Policy_Guard**：权限校验和风险管控组件
- **Audit_Logger**：审计日志记录组件
- **Config_Manager**：配置文件管理组件
- **Risk_Level**：工具风险等级（Low/Medium/High/Critical）
- **Whitelist**：允许访问的目录、应用或命令的白名单
- **Auth_Token**：用于客户端鉴权的 Bearer Token

## 需求

### 需求 1：MCP 协议支持

**用户故事：** 作为 MCP Client，我希望通过标准 MCP 协议与服务器通信，以便能够发现和调用可用的工具能力。

#### 验收标准

1. WHEN MCP_Client 发送 initialize 请求，THEN MCP_Server SHALL 返回服务器信息和协议版本
2. WHEN MCP_Client 发送 tools/list 请求，THEN MCP_Server SHALL 返回所有可用工具的列表和描述
3. WHEN MCP_Client 发送 tools/call 请求，THEN MCP_Server SHALL 执行指定工具并返回结果
4. THE MCP_Server SHALL 使用 HTTP 作为传输协议
5. THE MCP_Server SHALL 监听 0.0.0.0 地址以允许网络访问

### 需求 2：身份认证

**用户故事：** 作为系统管理员，我希望只有授权的客户端才能调用服务器，以防止未授权访问。

#### 验收标准

1. WHEN MCP_Client 发送请求，THEN MCP_Server SHALL 验证请求中的 Bearer Token
2. IF Auth_Token 无效或缺失，THEN MCP_Server SHALL 拒绝请求并返回 401 错误
3. THE Config_Manager SHALL 在首次启动时自动生成 Auth_Token
4. IF Auth_Token 丢失或无效，THEN Config_Manager SHALL 自动重新生成并持久化

### 需求 3：截图工具

**用户故事：** 作为 AI 助理，我希望能够截取用户的屏幕，以便分析屏幕内容并提供帮助。

#### 验收标准

1. WHEN screenshot_full 工具被调用，THEN MCP_Server SHALL 截取主屏幕并保存为 PNG 文件
2. THE MCP_Server SHALL 将截图保存到程序目录下的 screenshots 子目录
3. WHEN 截图完成，THEN MCP_Server SHALL 返回文件路径、宽度、高度和创建时间
4. THE MCP_Server SHALL 在 300 毫秒内完成截图操作
5. WHEN screenshot_full 被调用，THEN Policy_Guard SHALL 要求用户确认操作

### 需求 4：文件搜索工具

**用户故事：** 作为 AI 助理，我希望能够搜索用户文件，以便帮助用户找到所需的文档。

#### 验收标准

1. WHEN find_files 工具被调用，THEN MCP_Server SHALL 在白名单目录中搜索匹配的文件
2. THE Policy_Guard SHALL 拒绝搜索不在白名单中的目录
3. WHEN 搜索参数包含文件扩展名过滤，THEN MCP_Server SHALL 仅返回匹配扩展名的文件
4. WHEN 搜索参数包含天数限制，THEN MCP_Server SHALL 仅返回指定天数内修改的文件
5. THE MCP_Server SHALL 限制返回结果数量不超过参数指定的最大值（默认 50）
6. WHEN 搜索完成，THEN MCP_Server SHALL 返回文件路径、大小、修改时间等基本信息

### 需求 5：文件读取工具

**用户故事：** 作为 AI 助理，我希望能够读取文本文件内容，以便分析文件内容并回答用户问题。

#### 验收标准

1. WHEN read_text_file 工具被调用，THEN MCP_Server SHALL 读取指定文件的内容
2. THE Policy_Guard SHALL 拒绝读取不在白名单目录中的文件
3. THE Policy_Guard SHALL 仅允许读取 .txt、.log、.json、.yaml、.ini 格式的文件
4. IF 文件大小超过 200KB，THEN MCP_Server SHALL 拒绝读取并返回错误
5. WHEN 读取完成，THEN MCP_Server SHALL 返回文件内容字符串

### 需求 6：剪贴板读取工具

**用户故事：** 作为 AI 助理，我希望能够读取剪贴板内容，以便处理用户复制的文本。

#### 验收标准

1. WHEN clipboard_read 工具被调用，THEN MCP_Server SHALL 读取剪贴板中的纯文本内容
2. THE MCP_Server SHALL 仅读取纯文本格式的剪贴板内容
3. IF 剪贴板内容超过 10MB，THEN MCP_Server SHALL 截断内容并返回前 10MB
4. WHEN 读取完成，THEN MCP_Server SHALL 返回剪贴板文本字符串

### 需求 7：剪贴板写入工具

**用户故事：** 作为 AI 助理，我希望能够将文本写入剪贴板，以便用户可以粘贴使用。

#### 验收标准

1. WHEN clipboard_write 工具被调用，THEN MCP_Server SHALL 将指定文本写入剪贴板
2. THE MCP_Server SHALL 覆盖剪贴板中的现有内容
3. THE MCP_Server SHALL 仅支持纯文本格式
4. WHEN 写入完成，THEN MCP_Server SHALL 返回操作成功状态

### 需求 8：窗口列表工具

**用户故事：** 作为 AI 助理，我希望能够列出当前打开的窗口，以便了解用户正在使用的应用程序。

#### 验收标准

1. WHEN list_windows 工具被调用，THEN MCP_Server SHALL 列出所有可见窗口
2. THE MCP_Server SHALL 仅返回可见窗口，不包括隐藏或最小化的窗口
3. WHEN 列出窗口，THEN MCP_Server SHALL 返回每个窗口的标题和进程名称
4. THE MCP_Server SHALL 不返回系统级窗口信息

### 需求 9：应用启动工具

**用户故事：** 作为 AI 助理，我希望能够启动应用程序，以便帮助用户打开所需的软件。

#### 验收标准

1. WHEN launch_app 工具被调用，THEN MCP_Server SHALL 启动指定的应用程序
2. THE Policy_Guard SHALL 仅允许启动白名单中的应用程序
3. IF 应用程序不在白名单中，THEN MCP_Server SHALL 拒绝启动并返回错误
4. WHEN 启动完成，THEN MCP_Server SHALL 返回操作成功状态和进程 ID

### 需求 10：应用关闭工具

**用户故事：** 作为 AI 助理，我希望能够关闭应用程序，以便帮助用户管理运行的软件。

#### 验收标准

1. WHEN close_app 工具被调用，THEN MCP_Server SHALL 关闭指定的应用程序
2. WHEN close_app 被调用，THEN Policy_Guard SHALL 要求用户确认操作
3. IF 用户拒绝确认，THEN MCP_Server SHALL 取消操作并返回拒绝状态
4. WHEN 关闭完成，THEN MCP_Server SHALL 返回操作成功状态

### 需求 11：受限命令执行工具

**用户故事：** 作为开发者，我希望能够执行常用的命令行操作，以便完成开发任务。

#### 验收标准

1. WHEN run_command_restricted 工具被调用，THEN MCP_Server SHALL 执行指定的命令
2. THE Policy_Guard SHALL 仅允许执行白名单中的命令（npm、git、python）
3. THE Policy_Guard SHALL 禁止包含下载、注册表修改、提权操作的命令
4. WHEN run_command_restricted 被调用，THEN Policy_Guard SHALL 要求用户确认操作
5. THE MCP_Server SHALL 在 30 秒内完成命令执行或超时终止
6. WHEN 命令执行完成，THEN MCP_Server SHALL 返回标准输出、标准错误和退出码
7. THE MCP_Server SHALL 限制输出长度不超过 1MB

### 需求 12：风险分级管理

**用户故事：** 作为系统管理员，我希望根据工具的风险等级进行不同的管控，以平衡安全性和易用性。

#### 验收标准

1. THE Policy_Guard SHALL 为每个工具分配风险等级（Low/Medium/High/Critical）
2. WHEN 工具风险等级为 High 或 Critical，THEN Policy_Guard SHALL 要求用户确认
3. WHEN 工具风险等级为 Low 或 Medium，THEN Policy_Guard SHALL 允许自动执行
4. THE Policy_Guard SHALL 通过 Win32 MessageBox 显示确认对话框
5. WHEN 显示确认对话框，THEN Policy_Guard SHALL 包含工具名称、参数摘要和风险提示

### 需求 13：审计日志

**用户故事：** 作为系统管理员，我希望记录所有工具调用，以便审计和追溯操作历史。

#### 验收标准

1. WHEN 任何工具被调用，THEN Audit_Logger SHALL 记录操作日志
2. THE Audit_Logger SHALL 使用 JSON Lines 格式存储日志
3. WHEN 记录日志，THEN Audit_Logger SHALL 包含时间、工具名称、风险等级、结果状态和执行耗时
4. THE Audit_Logger SHALL 使用 UTC ISO 8601 格式记录时间戳
5. THE Audit_Logger SHALL 将日志持久化到本地文件

### 需求 14：配置管理

**用户故事：** 作为系统管理员，我希望通过配置文件管理白名单和权限，以便灵活控制服务器行为。

#### 验收标准

1. THE Config_Manager SHALL 从 config.json 文件加载配置
2. THE Config_Manager SHALL 支持配置 Auth_Token、允许的目录、允许的应用和允许的命令
3. WHEN 配置文件不存在，THEN Config_Manager SHALL 创建默认配置文件
4. WHEN 配置文件被修改，THEN Config_Manager SHALL 在服务重启后加载新配置
5. THE Config_Manager SHALL 验证配置文件的格式和内容有效性

### 需求 15：错误处理

**用户故事：** 作为 MCP Client，我希望收到清晰的错误信息，以便了解操作失败的原因。

#### 验收标准

1. WHEN 工具执行失败，THEN MCP_Server SHALL 返回包含错误码和错误信息的错误响应
2. WHEN 用户拒绝确认，THEN MCP_Server SHALL 返回特定的拒绝错误码
3. WHEN 操作超时，THEN MCP_Server SHALL 终止操作并返回超时错误
4. IF 异常发生，THEN MCP_Server SHALL 记录错误日志但不退出主进程

### 需求 16：系统托盘界面

**用户故事：** 作为用户，我希望通过系统托盘图标管理服务器，以便方便地启动、停止和查看状态。

#### 验收标准

1. THE MCP_Server SHALL 在 Windows 系统托盘显示图标
2. WHEN 用户右键点击托盘图标，THEN MCP_Server SHALL 显示上下文菜单
3. THE 上下文菜单 SHALL 包含退出选项
4. WHEN 用户选择退出，THEN MCP_Server SHALL 优雅地关闭服务并退出进程

### 需求 17：性能要求

**用户故事：** 作为用户，我希望服务器占用最少的系统资源，以便不影响其他应用程序的运行。

#### 验收标准

1. WHILE MCP_Server 空闲运行，THE 内存占用 SHALL 不超过 100MB
2. WHILE MCP_Server 空闲运行，THE CPU 占用 SHALL 接近 0%
3. WHEN 执行截图操作，THEN MCP_Server SHALL 在 300 毫秒内完成
4. WHEN 执行任何工具，THEN MCP_Server SHALL 在 30 秒内完成或超时

### 需求 18：稳定性要求

**用户故事：** 作为用户，我希望服务器能够稳定运行，不会因为单个操作失败而崩溃。

#### 验收标准

1. IF 工具执行抛出异常，THEN MCP_Server SHALL 捕获异常并继续运行
2. THE MCP_Server SHALL 为每个工具调用设置超时限制
3. WHEN 子进程超时，THEN MCP_Server SHALL 终止子进程并释放资源
4. THE MCP_Server SHALL 在启动失败时记录错误日志并提示用户

### 需求 19：许可证管理

**用户故事：** 作为产品提供者，我希望管理用户的许可证状态，以便实现付费功能和使用限制。

#### 验收标准

1. THE MCP_Server SHALL 在启动时验证许可证状态
2. THE Config_Manager SHALL 支持配置许可证密钥（license_key）
3. WHEN 许可证密钥不存在或无效，THEN MCP_Server SHALL 以免费模式运行
4. WHEN 许可证密钥有效，THEN MCP_Server SHALL 以付费模式运行
5. THE MCP_Server SHALL 定期验证许可证的有效性（每 24 小时）

### 需求 20：免费版功能限制

**用户故事：** 作为产品提供者，我希望限制免费版的功能，以鼓励用户升级到付费版本。

#### 验收标准

1. WHILE MCP_Server 运行在免费模式，THE 每日工具调用次数 SHALL 不超过 100 次
2. WHILE MCP_Server 运行在免费模式，THE run_command_restricted 工具 SHALL 不可用
3. WHILE MCP_Server 运行在免费模式，THE screenshot_full 工具每日调用次数 SHALL 不超过 20 次
4. WHEN 达到免费版限制，THEN MCP_Server SHALL 返回限制错误并提示升级
5. THE MCP_Server SHALL 在每日 00:00 UTC 重置免费版计数器

### 需求 21：付费版功能

**用户故事：** 作为付费用户，我希望享受无限制的功能使用，以便充分利用服务器能力。

#### 验收标准

1. WHILE MCP_Server 运行在付费模式，THE 工具调用次数 SHALL 无限制
2. WHILE MCP_Server 运行在付费模式，THE 所有工具 SHALL 可用
3. WHILE MCP_Server 运行在付费模式，THE 审计日志保留时间 SHALL 为 90 天
4. WHILE MCP_Server 运行在免费模式，THE 审计日志保留时间 SHALL 为 7 天

### 需求 22：使用统计

**用户故事：** 作为用户，我希望查看我的使用统计，以便了解我的使用情况和剩余配额。

#### 验收标准

1. THE MCP_Server SHALL 记录每日工具调用次数
2. THE MCP_Server SHALL 提供 get_usage_stats 工具查询使用统计
3. WHEN get_usage_stats 被调用，THEN MCP_Server SHALL 返回当日调用次数、剩余配额和许可证状态
4. THE 使用统计 SHALL 包含每个工具的调用次数明细
5. THE MCP_Server SHALL 在系统托盘菜单中显示当前许可证状态

### 需求 23：许可证激活

**用户故事：** 作为用户，我希望能够激活我购买的许可证，以便使用付费功能。

#### 验收标准

1. THE MCP_Server SHALL 提供 activate_license 工具用于激活许可证
2. WHEN activate_license 被调用，THEN MCP_Server SHALL 验证许可证密钥的有效性
3. IF 许可证密钥有效，THEN MCP_Server SHALL 保存许可证信息到配置文件
4. IF 许可证密钥无效，THEN MCP_Server SHALL 返回错误信息
5. WHEN 许可证激活成功，THEN MCP_Server SHALL 立即切换到付费模式

### 需求 24：许可证过期处理

**用户故事：** 作为产品提供者，我希望处理许可证过期情况，以确保订阅模式的正常运作。

#### 验收标准

1. THE MCP_Server SHALL 检查许可证的过期时间
2. WHEN 许可证过期，THEN MCP_Server SHALL 自动切换到免费模式
3. WHEN 许可证即将过期（7 天内），THEN MCP_Server SHALL 在系统托盘显示提醒通知
4. THE MCP_Server SHALL 在许可证过期后保留许可证信息 30 天
5. WHEN 许可证续费，THEN MCP_Server SHALL 自动恢复付费模式
