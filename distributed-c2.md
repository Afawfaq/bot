# Distributed C2 Build Plan

## Overview
A PoC with distributed command and control nodes for redundancy and resilience.

## Layout
- nodes/: Multiple C2 node scripts
- client.py: Remote client
- sync/: Node synchronization modules

## Config
- Nodes sync commands and data
- Client connects to available node

## Setup
1. Deploy nodes on different servers
2. Configure sync modules
3. Run client.py

## Build Instructions
- Python scripts, network sync logic

## Infrastructure
- Multiple C2 nodes

## Example
```bash
python nodes/node1.py
python client.py
```
