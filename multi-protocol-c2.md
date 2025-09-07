# Multi-Protocol C2 Build Plan

## Overview
A PoC supporting multiple C2 protocols (HTTP, WebSocket, DNS, etc.).

## Layout
- server/: C2 server modules for each protocol
- client/: Client modules for each protocol
- config/: Protocol selection config

## Config
- Switch between protocols dynamically
- Fallback logic

## Setup
1. Configure supported protocols
2. Run server and client with selected protocol

## Build Instructions
- Python scripts, install requests/websockets/dnslib

## Infrastructure
- Multi-protocol servers

## Example
```bash
python server/http_server.py
python client/http_client.py
```
