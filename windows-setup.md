# Windows Setup Build Plan

## Overview
A PoC setup for Windows environment, focusing on compatibility and deployment.

## Layout
- server.py: Server script
- client.py: Client script
- config/: Configuration files

## Config
- Use Windows paths and environment variables
- Optionally use pyinstaller for executable builds

## Setup
1. Install Python 3.x on Windows
2. Set up config files in config/
3. Run server.py and client.py

## Build Instructions
- To build executables: `pyinstaller server.py` and `pyinstaller client.py`

## Infrastructure
- Windows machines (local or remote)

## Example
```powershell
python server.py
python client.py
```
