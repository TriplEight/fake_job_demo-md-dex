# ⚠️ Security Analysis Report — "Fake Job Demo" DEX

**This is an active malware. Do not run this code.**

# Combined Attack Analysis
## AI Prompt-Injection × EJS Server-Side Template Injection (CVE-2022-29078)

**Original Repository:** `mike-dancey/dex` (original, now banned from GitHub)
**Reference:** Dependabot alert, `SECURITY_REPORT.md`
**Tools:** Dependabot, CodeQL, Claude (Sonnet 4.6)
**CVE:** CVE-2022-29078 — EJS `outputFunctionName` SSTI → RCE
**Analysis Date:** 2026-03-03
**Status:** ⚠️ HIGH-SEVERITY COMBINED EXPLOIT — Do not run `npm install` or any project commands

---

## 1. Why These Two Findings Belong Together

Read in isolation, each finding sounds contained:

- The `.cursorignore` issue looks like a privacy nuisance — the AI editor reads your `.env` files.
- The EJS vulnerability looks like a routine transitive-dependency CVE — one old package buried deep.

**Together they form a coherent, two-stage remote code execution chain.** The `.cursorignore` manipulation primes an AI assistant to become an unwitting attack delivery mechanism; the EJS vulnerability provides the actual execution primitive. A developer running this project as a "take-home interview task" can have their machine compromised without ever clicking a phishing link or running an obviously suspicious command.

All details in [SECURITY_REPORT.md](./SECURITY_REPORT.md).

---

<details>
<summary>Original description - don't trust it</summary>

# 🚀 DEX - 去中心化交易所

> 基于 UniswapV2 的企业级 DEX 实现，采用现代化技术栈，完全开源。

[English](./README_EN.md) | 简体中文

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue.svg)](https://www.typescriptlang.org/)
[![React](https://img.shields.io/badge/React-18-61DAFB.svg)](https://reactjs.org/)
[![NestJS](https://img.shields.io/badge/NestJS-10-E0234E.svg)](https://nestjs.com/)
[![Solidity](https://img.shields.io/badge/Solidity-0.8.20-363636.svg)](https://soliditylang.org/)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](docs/CONTRIBUTING.md)

---

## 📖 项目简介

本项目是一个功能完整的去中心化交易所（DEX），实现了：

- ✅ **Swap** - 代币兑换（基于 AMM 算法）
- ✅ **Liquidity** - 流动性添加/移除
- ✅ **Pool** - 流动性池管理
- ✅ **Farms** - 流动性挖矿（MasterChef）
- ✅ **Price Oracle** - 价格预言机（Chainlink）
- ✅ **History** - 完整的交易历史记录
- ✅ **Analytics** - 数据分析和统计
- ✅ **Real-time** - WebSocket 实时数据推送
- 🔄 **The Graph** - 数据索引和快速查询（开发中）

### 🌟 特色

- **真正的去中心化** - 用户通过 MetaMask 自己管理私钥和签名交易
- **现代化架构** - 前端直接调用合约，后端提供只读数据服务
- **完整的数据分析** - 历史记录、统计数据、实时监控
- **生产级代码** - 经过多次重构和优化，代码清晰可维护

---

## 🏗️ 技术栈

### 智能合约
- **Solidity** - 合约语言
- **Hardhat** - 开发框架
- **UniswapV2** - AMM 协议

### 后端
- **NestJS** - Node.js 框架
- **TypeScript** - 类型安全
- **SQL** - ORM 框架
- **Redis** - 缓存
- **Socket.IO** - WebSocket 实时通信
- **Viem** - 以太坊库（只读查询）
- **The Graph** - 区块链数据索引（开发中）
- **GraphQL** - 数据查询语言

### 前端
- **React 18** - UI 框架
- **TypeScript** - 类型安全
- **Vite** - 构建工具
- **Ant Design** - UI 组件库
- **Wagmi** - React Hooks for Ethereum
- **Viem** - 以太坊库
- **Zustand** - 状态管理
- **React Query** - 数据请求

---

## 🚀 快速开始

### 前置要求

- Node.js >= 18
- npm（随 Node.js 安装）
- MetaMask 钱包

### 1. 克隆项目

```bash
git clone https://github.com/your-username/dex.git
cd dex
```

### 2. 一键启动

```bash
# 详细步骤请查看
cat START_ALL.md

# 或者查看快速开始指南
cat GETTING_STARTED.md
```

### 3. 访问应用

- **前端：** http://localhost:3000
- **后端 API：** http://localhost:3002
- **后端文档：** http://localhost:3002/api
- **钱包服务：** http://localhost:3001

---

## 🎯 核心概念

### 用户视角

```
用户 → MetaMask → 智能合约
         ↓
      签名交易
         ↓
    链上执行（去中心化）
```

### 系统架构

```
┌─────────────────────────────────────────────┐
│                   前端                       │
│   (React + Viem + MetaMask)                 │
│   - 直接调用合约执行交易                     │
│   - 调用后端 API 查询数据                    │
└──────────────┬──────────────────────────────┘
               │
               ├──────────────┐
               │              │
               ▼              ▼
      ┌────────────┐   ┌──────────────┐
      │ 智能合约    │   │  后端服务     │
      │ (Solidity) │   │  (NestJS)    │
      │            │   │  - 只读 API   │
      │ - Swap     │   │  - 数据分析   │
      │ - Pool     │   │  - 事件监听   │
      │ - Router   │   │  - 实时推送   │
      └────────────┘   └──────────────┘
            ▲                  │
            │                  │
            └──────────────────┘
              区块链事件监听
```

---

## 🔧 开发

### 项目结构

```
dex/
├── contracts/          # 智能合约
│   ├── contracts/      # Solidity 合约
│   │   ├── core/       # 核心合约（Factory, Pair, Router）
│   │   ├── farming/    # 挖矿合约（MasterChef）
│   │   └── oracle/     # 价格预言机（PriceOracle）
│   └── scripts/        # 部署脚本
│
├── backend/            # 后端服务
│   └── services/
│       ├── analytics-service/  # 数据分析服务
│       │   ├── modules/
│       │   │   ├── price/      # 价格服务
│       │   │   └── thegraph/   # The Graph 集成
│       │   └── ...
│       └── wallet-service/     # 钱包服务
│
├── frontend/           # 前端应用
│   └── web-app/        # React 应用
│       ├── src/
│       │   ├── pages/          # 页面（Swap, Pools, Farms...）
│       │   ├── hooks/          # React Hooks
│       │   └── components/     # 组件
│       └── ...
│
├── subgraph/           # The Graph Subgraph（新增）
│   ├── schema.graphql  # GraphQL Schema
│   ├── subgraph.yaml   # 配置文件
│   └── src/mappings/   # 事件处理器
│
├── scripts/            # 测试脚本
└── tests/              # 测试代码
```

### 开发流程

```bash
# 1. 启动本地链
cd contracts
npx hardhat node

# 2. 部署合约
npx hardhat run scripts/deploy.ts --network localhost

# 3. 启动后端
cd backend/services/analytics-service
npm run start:dev

# 4. 启动前端
cd frontend/web-app
npm run dev
```

### 常用命令

```bash
# Mint 代币
bash scripts/mint-tokens-simple.sh

# 同步池子数据
bash scripts/sync-all-pools.sh

# 测试 API
bash scripts/test-analytics-api.sh
```

---

## 🤝 贡献

### 开发规范

- **代码风格** - ESLint + Prettier
- **提交规范** - Conventional Commits
- **分支策略** - Git Flow
- **测试** - 单元测试 + 集成测试

---

## 📄 许可证

MIT License - 详见 [LICENSE](./LICENSE)

---

## 🌟 致谢

- [Uniswap V2](https://uniswap.org/) - AMM 协议
- [NestJS](https://nestjs.com/) - 后端框架
- [React](https://reactjs.org/) - 前端框架
- [Viem](https://viem.sh/) - 以太坊库
- [Wagmi](https://wagmi.sh/) - React Hooks

---

**最后更新：** 2025-10-30  
**维护者：** DEX Team

---

## 📈 项目亮点

### 已实现的核心功能

✅ **8 个智能合约** - Factory, Pair, Router, WETH, MasterChef, RewardToken, PriceOracle, Mock Aggregator  
✅ **62+ REST API 接口** - 完整的后端服务  
✅ **10+ GraphQL 查询** - The Graph 数据索引（开发中）  
✅ **7 个前端页面** - Swap, Liquidity, Pools, Farms, History...  
✅ **实时数据推送** - WebSocket 支持  
✅ **价格预言机** - Chainlink 集成，USD 价格显示  
✅ **流动性挖矿** - 完整的 Staking 和 Rewards 系统  

### 技术特色

🚀 **性能优化** - The Graph 索引，查询速度提升 10-100 倍  
🔒 **安全第一** - 完善的权限控制和输入验证  
🧪 **测试完整** - 单元测试 + 集成测试 + E2E 测试  
🎨 **现代化 UI** - Ant Design + 响应式设计

</details>
