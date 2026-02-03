# ClawDesk MCP Server

ClawDesk MCP Server 是一个 Windows 本地能力服务，遵循 Model Context Protocol (MCP) 标准协议，为 AI 助理（如 OpenClaw、Claude Desktop）提供安全、可控、可追溯的 Windows 系统操作能力。

## 下载

请从 [Releases](../../releases) 页面下载最新版本。

### 版本选择

- **ClawDeskMCP-x64.exe**: 适用于 64 位 Windows 系统（推荐）
- **ClawDeskMCP-x86.exe**: 适用于 32 位 Windows 系统
- **ClawDeskMCP-arm64.exe**: 适用于 ARM64 Windows 系统（如 Surface Pro X）

## 快速开始

1. 下载对应版本的可执行文件
2. 双击运行，程序会在系统托盘显示图标
3. 首次运行会自动生成 `config.json` 配置文件
4. 右键点击托盘图标可以查看状态和配置

详细使用说明请参考 [README.md](README.md)。

## 文档

- [README.md](README.md) - 完整使用指南
- [docs/API.md](docs/API.md) - HTTP API 文档
- [docs/Dashboard.md](docs/Dashboard.md) - Dashboard 使用指南
- [docs/MCP-Requirements.md](docs/MCP-Requirements.md) - MCP 协议需求说明
- [FIREWALL.md](FIREWALL.md) - 防火墙配置指南

## 系统要求

- Windows 10/11 (x64, x86, ARM64)
- 无需额外运行时依赖

## 许可证

Copyright © 2026 ClawDesk

详见 [LICENSE](LICENSE) 文件。

## 支持

如有问题或建议，请提交 [Issue](../../issues)。
