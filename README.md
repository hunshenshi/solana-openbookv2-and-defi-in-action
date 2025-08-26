# Solana & Anchor 学习与实战

> 一个系统化的 Solana 开发学习仓库，涵盖基础知识、Anchor 框架、智能合约实战等内容，帮助开发者快速上手并深入掌握 Solana 生态的开发技能。

## 📚 项目简介

本仓库旨在为想学习 **Solana 智能合约开发** 的开发者提供从零到实战的完整路径。  
内容将从 **Solana 基础概念** 入手，逐步过渡到 **Anchor 框架使用**，并通过 **实战项目** 带你实现可运行的链上应用。  

适合人群：
- 对区块链开发感兴趣的初学者
- 有一定 Web2 开发经验，想转向 Web3 的工程师
- 想系统学习 Anchor 框架的开发者

## 🗂 内容规划

### 进行中 / 计划中
- **OpenBook-V2源码解析**


## 🚀 使用说明

### 环境准备
确保你已安装以下依赖：
- Rust（最新稳定版）
- Solana CLI（建议版本 >= 2.1.15）
- Anchor CLI（建议版本 >= 0.31.1）
- OpenBookV2 (v0.2.10)
- Node.js（用于前端或测试脚本）
- Yarn 或 NPM（包管理工具）

### 克隆项目
```bash
git clone git@github.com:hunshenshi/solana-openbookv2-and-defi-in-action.git
cd solana-openbookv2-and-defi-in-action
````

### 编译 & 测试

```bash
anchor build
anchor test
```

### 部署到本地验证网络

```bash
solana-test-validator
anchor deploy
```

## 🎯 学习目标

通过本项目，你将掌握：

- Solana 的账户与交易模型
    
- Anchor 框架的核心功能与使用方法
    
- 如何设计与实现安全的链上合约
    
- 与合约交互的客户端开发方法
    
- 开发过程中常见的调试与部署技巧

## 🤝 贡献

欢迎任何形式的贡献：

- 提交 PR 改进文档或代码
    
- 提交 Issue 报告 bug 或建议
    
- 分享你的学习笔记与案例

如果你觉得项目对你有帮助，请 **Star ⭐** 支持一下！