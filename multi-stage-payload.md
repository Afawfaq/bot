# Multi-Stage Payload Build Plan

## Overview
A PoC with multi-stage payload delivery, mimicking Hive's staged infection.

## Layout
- stage1_loader.py: Initial dropper
- stage2_payload.py: Main payload
- server.py: C2 server

## Config
- Stage 1 downloads and executes Stage 2
- Encrypted payload transfer

## Setup
1. Deploy stage1_loader.py on target
2. Stage 1 fetches stage2_payload.py from server
3. Server manages payloads

## Build Instructions
- Python scripts, optionally packed with pyinstaller

## Infrastructure
- C2 server, payload repository

## Example
```bash
python stage1_loader.py
```
