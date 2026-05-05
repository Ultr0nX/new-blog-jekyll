---
title: Shadow Mafia — On-Chain Social Deduction Game
description: A Mafia-style social deduction game on Solana where players stake SOL and roles are sealed inside cryptographic vaults — not even the server knows who is Mafia.
date: 2025-04-22
layout: default
---

**Live link 👉** -> [Shadow Mafia](https://shadow-mafia.vercel.app)

**Code :** [github](https://github.com/Ultr0nX/shadow-mafia)

An on-chain take on the classic Mafia party game. Players stake SOL to join a lobby, get assigned a hidden role (Mafia / Citizen / Doctor) via verifiable randomness, and play through night and day phases. The cryptographic twist: roles are sealed in a vault — the server itself can't see who is Mafia.

**How it works**
- Stake SOL to join a lobby
- Roles are dealt through verifiable randomness and kept private from the server
- Night phase → private votes & eliminations; Day phase → discussion & voting
- The SOL pot is paid out automatically on-chain to the winning faction

**Tech Stack used**
- Solana — L1
- Anchor — Rust framework for on-chain programs
- MagicBlock Ephemeral Rollups — privacy for hidden state
- Next.js + React — frontend
- Socket.io — real-time game events
- Phantom Wallet — wallet integration
