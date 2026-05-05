---
title: Privacy Shield — ZK Proof-of-Personhood
description: A zero-knowledge proof-of-personhood system that ties a unique human to an Ethereum wallet without revealing their face or storing biometric data anywhere.
date: 2025-05-01
layout: default
---

**Code :** [github](https://github.com/Ultr0nX/Privacy-Shield)

A privacy-preserving proof-of-personhood prototype. The idea: prove you are a real, unique human linked to your wallet — without your face ever leaving your device and without biometric data being stored anywhere.

**How it works**
- MediaPipe extracts facial landmarks locally in the browser
- BCH error-correcting codes turn those landmarks into a stable, recoverable secret
- A Groth16 zk-SNARK proves possession of that secret on-chain — the face data itself is never revealed
- Cross-device recovery works via on-chain helper data
- A Rust relayer abstracts gas so end users don't need ETH to register

**Tech Stack used**
- Frontend — React + ethers.js
- Backend — Rust (Axum)
- Smart Contracts — Solidity + Foundry
- ZK Layer — Circom circuits compiled to WASM, snarkjs for in-browser proof generation (BN128 curve)
