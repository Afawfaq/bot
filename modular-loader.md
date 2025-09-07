# Modular Loader Build Plan

## Overview
A PoC with a modular loader capable of fetching and executing modules on demand.

## Layout
- loader.py: Main loader
- modules/: Module scripts
- server.py: Module repository server

## Config
- Loader fetches modules from server
- Modules executed in isolated environment

## Setup
1. Develop modules in modules/
2. Run server.py to serve modules
3. Run loader.py on target

## Build Instructions
- Python scripts, dynamic import

## Infrastructure
- Module repository server

## Example
```bash
python loader.py
```
