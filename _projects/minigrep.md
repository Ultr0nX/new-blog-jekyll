---
title: minigrep — A Tiny grep in Rust
description: A small command-line search tool written in Rust that prints lines of a file matching a query — built while working through The Rust Book to internalize ownership, error handling, and iterators.
date: 2025-03-20
layout: default
---

**Code :** [github](https://github.com/Ultr0nX/minigrep)

A minimal `grep` clone I built in Rust while working through *The Rust Book*. Searches a given file for a query string and prints the matching lines.

**Features**
- Case-sensitive and case-insensitive search
- Behavior configurable via environment variables
- A learning exercise — not a replacement for `ripgrep`

**Tech Stack used**
- Rust + Cargo
