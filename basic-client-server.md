# Basic Client-Server Build Plan

## Overview
A simple Python-based client-server remote access tool (RAT) PoC.

## Layout
- server.py: Command and control server
- client.py: Remote client

## Config
- Use TCP sockets
- Simple authentication (shared secret)

## Setup
1. Install Python 3.x
2. Place server.py and client.py in the same directory
3. Run server.py on your control machine
4. Run client.py on target machine

## Build Instructions
- No build required; run scripts directly

## Infrastructure
- Local network or internet (port forwarding required)

## Example
```bash
python server.py
python client.py
```
