# Proxy Relay Infrastructure Build Plan

## Overview
A PoC with proxy relays for covert C2 communication.

## Layout
- relay.py: Proxy relay node
- server.py: Main C2 server
- client.py: Remote client

## Config
- Relays forward encrypted traffic
- Dynamic relay selection

## Setup
1. Deploy relay.py on multiple nodes
2. Configure client to use relay chain
3. Server receives traffic via relays

## Build Instructions
- Python scripts, Docker optional

## Infrastructure
- Multiple relay nodes, C2 server

## Example
```bash
python relay.py
python client.py
```
