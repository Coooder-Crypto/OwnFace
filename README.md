# OwnFace

**Own Your Face Before You Own Your Data**

OwnFace delivers an end-to-end zero-knowledge biometric authentication stack: the frontend collects face embeddings, the backend wraps them in Pedersen commitments and Groth16 proofs, and on-chain smart contracts validate proofs while preserving user privacy.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Privacy & Cryptography](#privacy--cryptography)
  - [Pedersen Commitments](#pedersen-commitments)
  - [Poseidon Hashing](#poseidon-hashing)
  - [Groth16 Proofs](#groth16-proofs)
- [Components](#components)
- [Workflow](#workflow)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Smart Contracts](#smart-contracts)
- [Team & Contact](#team--contact)
- [Citation](#citation)

---

## Overview

- **Name**: OwnFace  
- **Tagline**: Prove you are you without handing over your face.  
- **Why it matters**: Centralized facial recognition exposes biometric data to breaches; Web3 still lacks a privacy-first KYC primitive.  
- **Who it's for**: Wallet providers, dApps needing face checks, DAO access gates, privacy-aware individuals.  
- **How it works**:  
  1. The Next.js frontend captures embeddings and orchestrates transactions.  
  2. The Node.js/TypeScript backend quantizes vectors, builds Pedersen commitments, and generates Groth16 proofs.  
  3. Solidity contracts verify the proof and store registration or verification snapshots.  
  4. Raw embeddings never leave the user device and ephemeral backend memory.

---

## Architecture

![OwnFace Architecture](ownface.png)

- **Frontend**: RainbowKit + wagmi UI for capture, wallet prompts, and proof feedback.  
- **Backend**: Embedding quantization, Pedersen commitments, Poseidon hashing, Groth16 proving, and transcript signing.  
- **Smart contracts**: `OwnFaceRegistry` stores commitments and verification records; `Groth16Verifier` checks distance proofs.  
- **Metrics**: `/metrics` exposes registration count, authentication attempts, and average proving latency for ops visibility.

---

## Privacy & Cryptography

### Pedersen Commitments

- **Formula**: `C = r·G + Σ vᵢ·Hᵢ`  
  - `vᵢ`: Quantized embedding limbs mapped into the babyjubjub/BLS12-381 field.  
  - `Hᵢ`: Independent generators sourced from circomlib.  
  - `r`: 32-byte blinding factor that keeps the vector hidden.  
  - `G`: babyjubjub base point.  
- **On-chain payload**: `commitmentPoint`, `commitmentHash`, `blinding`, `nonceHash`, `vectorHash`.  
- **Security properties**: Blinding prevents observers from reconstructing embeddings, and the `referenceHash` check blocks vector swaps.

### Poseidon Hashing

- `poseidonHashVector` collapses the quantized vector into a field element used as the circuit public signal `referenceHash` and the contract field `vectorHash`.  
- `poseidonHashTranscript` binds `referenceHash`, `candidateHash`, `threshold`, and `distance` into a single digest, ensuring the proof matches runtime parameters.

### Groth16 Proofs

![Distance Circuit](circuit.png)

- **Circuit objective**: Assert that the squared distance between the registered and candidate vectors stays below a threshold.  
- **Public signals**: `distance`, `threshold`, `within`, `referenceHash`, `candidateHash`, `transcriptHash`.  
- **Proving**: `groth16.fullProve` uses the Circom `distance.circom` circuit; artifacts (`.wasm`, `.zkey`, `verification_key.json`) live in `backend/circuits`.  
- **On-chain verification**: `OwnFaceRegistry.authenticate` calls `Groth16Verifier.verifyProof` and stores the resulting `VerificationRecord`.

---

## Components

| Module | Stack | Responsibility |
| --- | --- | --- |
| `frontend` | Next.js 15, React 19, RainbowKit, wagmi, viem, Tailwind | Registration/authentication UI, camera capture, wallet orchestration, proof display |
| `backend` | Node.js 20+, Express, TypeScript, snarkjs, circomlibjs, zod | Embedding quantization, Poseidon hashing, Pedersen commitments, Groth16 proving/verification, REST APIs |
| `contract` | Solidity 0.8.24, Hardhat, TypeScript scripts | `OwnFaceRegistry` storage and audit trail, `Groth16Verifier` circuit validation |
| `circuits` | Circom 2, snarkjs | `distance.circom`, compiled `.wasm`, proving `.zkey`, `verification_key.json` |

The repository tracks the three subprojects as Git submodules so each stack can evolve independently while sharing shared specs.

---

## Workflow

1. **Registration**  
   - The frontend captures an embedding and invokes `/register`.  
   - The backend responds with `commitmentHash`, `commitmentPoint`, `blinding`, `nonce`, `nonceHash`, `vectorHash`.  
   - The frontend submits a `register` transaction via wagmi; the contract persists the `Commitment` struct.  
2. **Authentication**  
   - A new embedding triggers `/authenticate`, producing a Groth16 proof plus public signals.  
   - The frontend relays an `authenticate` transaction; the contract verifies the proof and records a `VerificationRecord`.

---

## Getting Started

1. **Clone the repository**
   ```bash
   git clone --recurse-submodules https://github.com/Coooder-Crypto/OwnFace.git
   cd OwnFace
   ```
2. **Install dependencies**
   ```bash
   cd backend && npm install
   cd ../frontend && npm install
   cd ../contract && npm install
   ```
3. **Build circuits (optional)**
   ```bash
   cd backend
   npm run circuits:build
   ```
4. **Run services**
   ```bash
   # backend
   cd backend
   npm run dev   # http://localhost:4000

   # frontend
   cd ../frontend
   npm run dev   # http://localhost:3000
   ```
5. **Deploy contracts**
   ```bash
   cd contract
   npx hardhat compile
   npx hardhat run scripts/deploy-verifier.ts --network sepolia
   ```
6. **Try the full flow**  
   - Configure `frontend/.env.local` (see below) and connect a Sepolia wallet.  
   - During registration, upload a 16-dimensional vector (JSON/CSV) or capture via camera.  
   - During authentication, reuse the saved vector or capture a fresh sample to observe on-chain confirmations.

---

## Environment Variables

**Frontend (`frontend/.env.local`)**
```
NEXT_PUBLIC_SEPOLIA_RPC_URL=https://sepolia.infura.io/v3/<YOUR_KEY>
NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID=<walletconnect_id>
NEXT_PUBLIC_REGISTRY_ADDRESS=0x9F361ac7525F8c6f4306585F048Ebd7Bc75D4354
NEXT_PUBLIC_VERIFIER_ADDRESS=0xB28A3296ad9Add5BDB0C82b386229a34bAa17b1d
NEXT_PUBLIC_API_BASE_URL=http://localhost:4000
NEXT_PUBLIC_BLOCK_EXPLORER_BASE=https://sepolia.etherscan.io
```

**Backend (`backend/.env`, optional)**
```
PORT=4000
PROVER_THREADS=4
```

**Contract (`contract/.env`)**
```
SEPOLIA_RPC_URL="https://sepolia.infura.io/v3/<YOUR_KEY>"
DEPLOYER_PRIVATE_KEY="0x..."
ETHERSCAN_API_KEY=""
```

---

## Smart Contracts

- **Network**: Sepolia testnet  
- **Addresses**
  ```
  OwnFaceRegistry: 0x9F361ac7525F8c6f4306585F048Ebd7Bc75D4354
  Groth16Verifier: 0xB28A3296ad9Add5BDB0C82b386229a34bAa17b1d
  ```
- **Common commands**
  ```bash
  npx hardhat compile
  npx hardhat run scripts/deploy-verifier.ts --network sepolia
  ```
- **Minimal deploy script**
  ```bash
  # compile contracts
  npx hardhat compile

  # deploy verifier + registry (adapt as needed)
  npx hardhat run scripts/deploy-all.ts --network sepolia
  ```
- Etherscan verification and automation scripts will land post-hackathon (see `README1.md` in the submissions plan).

---

## Team & Contact

- **Team**: OwnFace Core  
- **Members**:  
  - Coooder — Full-stack engineer covering architecture, frontend, and contract integration  
- **Contact**:  
  - GitHub Issues: <https://github.com/Coooder-Crypto/OwnFace/issues>

---

Contributions, circuit co-design, audits, and integration talks are welcome. Reach out for collaboration or security reviews—let's build privacy-first biometric identity for Web3 together.

---

## Citation

If you reference OwnFace in research or publications, please cite:  
K. Guo, Z. Fu, Y. Liu, et al. **“BioID: Zero-Knowledge Proofs for Privacy-Preserving Biometric Authentication.”** arXiv:2409.17509, 2024. <https://arxiv.org/abs/2409.17509>

---

# OwnFace（中文版）

**在掌控数据之前，先掌控你的面孔**

OwnFace 是一套端到端的零知识生物识别认证栈：前端负责采集面部嵌入，后端将向量封装进 Pedersen 承诺与 Groth16 证明，链上智能合约在保持隐私的前提下验证证明并记录认证快照。

---

## 目录

- [项目概述](#项目概述)
- [系统架构](#系统架构)
- [隐私与密码学](#隐私与密码学)
  - [Pedersen 承诺](#pedersen-承诺)
  - [Poseidon 哈希](#poseidon-哈希)
  - [Groth16 证明](#groth16-证明)
- [组件划分](#组件划分)
- [端到端流程](#端到端流程)
- [快速开始](#快速开始)
- [环境变量](#环境变量)
- [合约部署](#合约部署)
- [团队与联系](#团队与联系)
- [引用](#引用)

---

## 项目概述

- **项目名称**：OwnFace  
- **一句话介绍**：用零知识证明“我就是我”，无需暴露脸部数据。  
- **项目动机**：传统云端人脸识别存在泄露隐患，而 Web3 缺少隐私友好的 KYC 基础设施。  
- **目标用户**：加密钱包、需要人脸校验的 dApp、DAO 访问控制、重视隐私的个人用户。  
- **解决思路**：  
  1. Next.js 前端采集嵌入并发起链上交易。  
  2. Node.js/TypeScript 后端量化向量、生成 Pedersen 承诺与 Groth16 证明。  
  3. Solidity 合约验证证明并写入注册或认证记录。  
  4. 原始嵌入仅在用户设备与后端内存中短暂存在。

---

## 系统架构

![OwnFace 架构](ownface.png)

- **前端**：集成 RainbowKit / wagmi，提供摄像头或文件上传、钱包交互与证明展示。  
- **后端**：负责嵌入量化、Pedersen 承诺、Poseidon 哈希、Groth16 生成与验签。  
- **智能合约**：`OwnFaceRegistry` 持久化承诺与认证记录，`Groth16Verifier` 校验距离电路证明。  
- **指标监控**：`/metrics` 路由输出注册量、认证次数与平均证明耗时，便于运营监控。

---

## 隐私与密码学

### Pedersen 承诺

- **公式**：`C = r·G + Σ vᵢ·Hᵢ`  
  - `vᵢ`：量化后的嵌入分量，映射到 babyjubjub/BLS12-381 有限域。  
  - `Hᵢ`：circomlib 提供的独立生成元。  
  - `r`：32 字节随机盲因子，保障 hiding 属性。  
  - `G`：babyjubjub 基点。  
- **链上字段**：`commitmentPoint`、`commitmentHash`、`blinding`、`nonceHash`、`vectorHash`。  
- **安全性**：盲因子阻止第三方复原嵌入，`referenceHash` 校验可阻止向量被调包。

### Poseidon 哈希

- `poseidonHashVector` 将量化向量压缩为 field 元素，用作电路公开信号 `referenceHash` 以及链上字段 `vectorHash`。  
- `poseidonHashTranscript` 将 `referenceHash`、`candidateHash`、`threshold`、`distance` 绑定为单一摘要，确保证明与实参一致。

### Groth16 证明

![距离电路](circuit.png)

- **电路目标**：验证注册向量与候选向量的平方距离不超过阈值。  
- **公开信号**：`distance`、`threshold`、`within`、`referenceHash`、`candidateHash`、`transcriptHash`。  
- **证明生成**：`groth16.fullProve` 基于 Circom `distance.circom` 输出证明；产物（`.wasm`、`.zkey`、`verification_key.json`）存放于 `backend/circuits`。  
- **链上验证**：`OwnFaceRegistry.authenticate` 调用 `Groth16Verifier.verifyProof`，并写入 `VerificationRecord`。

---

## 组件划分

| 模块 | 技术栈 | 职责 |
| --- | --- | --- |
| `frontend` | Next.js 15、React 19、RainbowKit、wagmi、viem、Tailwind | 注册/认证界面、摄像头采集、钱包交互、证明展示 |
| `backend` | Node.js 20+、Express、TypeScript、snarkjs、circomlibjs、zod | 嵌入量化、Poseidon 哈希、Pedersen 承诺、Groth16 生成与验签、REST API |
| `contract` | Solidity 0.8.24、Hardhat、TypeScript scripts | `OwnFaceRegistry` 存储承诺与认证记录，`Groth16Verifier` 验证距离电路 |
| `circuits` | Circom 2、snarkjs | `distance.circom`、编译 `.wasm`、证明 `.zkey`、`verification_key.json` |

仓库以 Git 子模块管理三个子项目，使各技术栈可以独立演进同时共享规范。

---

## 端到端流程

1. **注册**  
   - 前端采集 embedding 并调用 `/register`。  
   - 后端返回 `commitmentHash`、`commitmentPoint`、`blinding`、`nonce`、`nonceHash`、`vectorHash`。  
   - 前端通过 wagmi 发起 `register` 交易，合约写入 `Commitment` 结构体。  
2. **认证**  
   - 新的 embedding 调用 `/authenticate`，生成 Groth16 证明与公开信号。  
   - 前端提交 `authenticate` 交易，合约校验证明并记录 `VerificationRecord`。

---

## 快速开始

1. **克隆仓库**
   ```bash
   git clone --recurse-submodules https://github.com/Coooder-Crypto/OwnFace.git
   cd OwnFace
   ```
2. **安装依赖**
   ```bash
   cd backend && npm install
   cd ../frontend && npm install
   cd ../contract && npm install
   ```
3. **构建电路（可选）**
   ```bash
   cd backend
   npm run circuits:build
   ```
4. **启动服务**
   ```bash
   # backend
   cd backend
   npm run dev   # http://localhost:4000

   # frontend
   cd ../frontend
   npm run dev   # http://localhost:3000
   ```
5. **部署合约**
   ```bash
   cd contract
   npx hardhat compile
   npx hardhat run scripts/deploy-verifier.ts --network sepolia
   ```
6. **体验流程**  
   - 配置 `frontend/.env.local`（见下文），连接 Sepolia 测试钱包。  
   - 注册阶段可上传任意 16 维向量（JSON/CSV）或使用摄像头生成嵌入。  
   - 认证阶段复用原始向量或采集新样本，观察链上确认与 UI 反馈。

---

## 环境变量

**Frontend (`frontend/.env.local`)**
```
NEXT_PUBLIC_SEPOLIA_RPC_URL=https://sepolia.infura.io/v3/<YOUR_KEY>
NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID=<walletconnect_id>
NEXT_PUBLIC_REGISTRY_ADDRESS=0x9F361ac7525F8c6f4306585F048Ebd7Bc75D4354
NEXT_PUBLIC_VERIFIER_ADDRESS=0xB28A3296ad9Add5BDB0C82b386229a34bAa17b1d
NEXT_PUBLIC_API_BASE_URL=http://localhost:4000
NEXT_PUBLIC_BLOCK_EXPLORER_BASE=https://sepolia.etherscan.io
```

**Backend (`backend/.env` 可选)**
```
PORT=4000
PROVER_THREADS=4
```

**Contract (`contract/.env`)**
```
SEPOLIA_RPC_URL="https://sepolia.infura.io/v3/<YOUR_KEY>"
DEPLOYER_PRIVATE_KEY="0x..."
ETHERSCAN_API_KEY=""
```

---

## 合约部署

- **网络**：Sepolia 测试网  
- **核心地址**
  ```
  OwnFaceRegistry: 0x9F361ac7525F8c6f4306585F048Ebd7Bc75D4354
  Groth16Verifier: 0xB28A3296ad9Add5BDB0C82b386229a34bAa17b1d
  ```
- **常用命令**
  ```bash
  npx hardhat compile
  npx hardhat run scripts/deploy-verifier.ts --network sepolia
  ```
- **最小化部署示例**
  ```bash
  # 编译合约
  npx hardhat compile

  # 部署 verifier + registry（根据需要调整脚本）
  npx hardhat run scripts/deploy-all.ts --network sepolia
  ```
- Etherscan 验证与自动化部署脚本将在赛后补充，详见提交计划 `README1.md`。

---

## 团队与联系

- **团队**：OwnFace Core  
- **成员**：  
  - Coooder — 全栈工程师（架构、前端、合约集成）  
- **联系方式**：  
  - GitHub Issues：<https://github.com/Coooder-Crypto/OwnFace/issues>

---

欢迎提交 Issue / PR，参与电路共建、合约审计或集成合作。若需隐私身份解决方案与安全评估，欢迎随时联系。

---

## 引用

在研究或发表中引用 OwnFace 时请参考：  
K. Guo, Z. Fu, Y. Liu, 等. **《BioID: Zero-Knowledge Proofs for Privacy-Preserving Biometric Authentication》**. arXiv:2409.17509, 2024. <https://arxiv.org/abs/2409.17509>
