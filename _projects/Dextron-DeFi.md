---
title: Dextron — Educational DEX (Uniswap V1 AMM)
description: An educational decentralized exchange implementing Uniswap V1's constant-product AMM (x * y = k) for ETH ↔ ERC-20 swaps and liquidity provision, deployed on Sepolia.
date: 2025-04-15
layout: default
---

**Live link 👉** -> [Dextron DeFi](https://dextron-defi.vercel.app/)

**Code :** [github](https://github.com/Ultr0nX/Dextron-DeFi)

A from-scratch Uniswap V1-style AMM I built to internalize how decentralized exchanges actually work under the hood — pricing, liquidity provisioning, and the math behind the constant-product curve.

**What it does**
- Swap ETH for an ERC-20 test token (and back)
- Add / remove liquidity to the pool and earn LP shares
- Pricing is fully driven by the on-chain reserves using `x * y = k`

**Tech Stack used**
- Foundry — Solidity smart contracts + tests
- React 19 + Vite — frontend
- wagmi + RainbowKit — wallet connection & contract interaction
- Sepolia testnet — deployment target
