# Layered Encryption Build Plan

## Overview
A PoC with layered encryption for all communications and payloads.

## Layout
- server.py: C2 server
- client.py: Remote client
- crypto/: Encryption modules (AES, RSA, etc.)

## Config
- Multiple encryption layers (e.g., AES over RSA)
- Key exchange protocol

## Setup
1. Generate keys for server and client
2. Configure crypto modules
3. Run server and client

## Build Instructions
- Python scripts, install cryptography package

## Infrastructure
- Secure key management

## Example
```bash
python server.py
python client.py
```
