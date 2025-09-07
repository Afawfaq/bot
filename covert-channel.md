# Covert Channel Build Plan

## Overview
A PoC using covert channels (e.g., DNS, HTTP headers) for C2 communication.

## Layout
- server.py: C2 server
- client.py: Remote client
- channels/: Covert channel modules

## Config
- DNS tunneling, HTTP header encoding
- Channel selection logic

## Setup
1. Configure DNS/HTTP infrastructure
2. Deploy channel modules
3. Run server and client

## Build Instructions
- Python scripts, install dnslib/requests

## Infrastructure
- DNS server, web server

## Example
```bash
python server.py
python client.py
```
