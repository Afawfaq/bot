# Docker Infrastructure Build Plan

## Overview
A PoC using Docker for isolated deployment and testing.

## Layout
- server/: Server code
- client/: Client code
- docker-compose.yml: Multi-container setup

## Config
- Docker images for server and client
- Network bridge for communication

## Setup
1. Install Docker and Docker Compose
2. Build images: `docker-compose build`
3. Start services: `docker-compose up`

## Build Instructions
- Use Dockerfile in each directory
- Use docker-compose for orchestration

## Infrastructure
- Docker network

## Example
```bash
# Build and deploy containerized C2 infrastructure using Docker Compose
# Creates isolated containers for server and client components
docker-compose up --build

# Alternative: Build individual components manually
# Build the C2 server container with security hardening
docker build -t c2-server ./server

# Build the client container with stealth configurations
docker build -t c2-client ./client

# Run containers with network isolation and resource limits
docker run --network c2-network --memory=512m --cpus=1.0 c2-server
```

## Enhanced Implementation with Modern Containerization

### Docker Compose Configuration (docker-compose.yml)
```yaml
# Docker Compose configuration for secure C2 infrastructure
# Implements network isolation, secrets management, and monitoring

version: '3.8'                    # Use modern Docker Compose version for latest features

# Define custom networks for traffic segregation
networks:
  c2-frontend:                    # Network for external communications
    driver: bridge
    driver_opts:
      com.docker.network.bridge.enable_icc: "false"  # Disable inter-container communication
    ipam:
      config:
        - subnet: 172.20.0.0/24   # Custom subnet for frontend network
  c2-backend:                     # Network for internal server communications
    driver: bridge
    driver_opts:
      com.docker.network.bridge.enable_icc: "true"   # Enable backend communication
    ipam:
      config:
        - subnet: 172.21.0.0/24   # Custom subnet for backend network
  monitoring:                     # Network for monitoring and logging
    driver: bridge
    ipam:
      config:
        - subnet: 172.22.0.0/24

# Define persistent volumes for data storage
volumes:
  c2-data:                        # Persistent storage for C2 server data
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ./data
  c2-logs:                        # Centralized logging volume
    driver: local
  redis-data:                     # Redis cache for session management
    driver: local
  postgres-data:                  # PostgreSQL database for persistent storage
    driver: local

# Define secrets for secure credential management
secrets:
  c2-tls-cert:                    # TLS certificate for HTTPS
    file: ./secrets/server.crt
  c2-tls-key:                     # TLS private key
    file: ./secrets/server.key
  c2-auth-key:                    # Authentication shared secret
    file: ./secrets/auth.key
  database-password:              # Database password
    file: ./secrets/db_password.txt

# Service definitions with security hardening
services:
  # Main C2 server with enhanced security
  c2-server:
    build:
      context: ./server            # Build context directory
      dockerfile: Dockerfile.hardened  # Security-hardened Dockerfile
      args:
        PYTHON_VERSION: 3.11      # Use latest stable Python version
        USER_ID: 1000             # Non-root user ID
        GROUP_ID: 1000            # Non-root group ID
    container_name: c2-server     # Fixed container name for networking
    hostname: c2-primary          # Internal hostname
    restart: unless-stopped       # Restart policy for reliability
    
    # Resource limits for security and stability
    deploy:
      resources:
        limits:
          cpus: '2.0'             # Maximum CPU cores
          memory: 1G              # Maximum memory allocation
        reservations:
          cpus: '0.5'             # Reserved CPU cores
          memory: 256M            # Reserved memory
      restart_policy:
        condition: on-failure     # Restart on failure
        delay: 10s                # Delay between restart attempts
        max_attempts: 3           # Maximum restart attempts
    
    # Environment variables for configuration
    environment:
      - PYTHONPATH=/app           # Python module search path
      - PYTHONUNBUFFERED=1        # Unbuffered output for logging
      - C2_DEBUG=false            # Disable debug mode in production
      - C2_LOG_LEVEL=INFO         # Set appropriate log level
      - C2_BIND_HOST=0.0.0.0      # Bind to all interfaces within container
      - C2_BIND_PORT=8443         # Internal HTTPS port
      - DATABASE_URL=postgresql://c2user:password@postgres:5432/c2db  # Database connection
      - REDIS_URL=redis://redis:6379/0  # Redis cache connection
      - TLS_CERT_PATH=/run/secrets/c2-tls-cert  # TLS certificate path
      - TLS_KEY_PATH=/run/secrets/c2-tls-key    # TLS key path
      - AUTH_KEY_PATH=/run/secrets/c2-auth-key  # Authentication key path
    
    # Port mappings (only expose necessary ports)
    ports:
      - "8443:8443"               # HTTPS port for C2 communications
      - "127.0.0.1:9090:9090"     # Metrics port (localhost only)
    
    # Volume mounts for persistent data
    volumes:
      - c2-data:/app/data:rw      # Application data (read-write)
      - c2-logs:/app/logs:rw      # Log files (read-write)
      - ./config:/app/config:ro   # Configuration files (read-only)
    
    # Secrets for secure credential access
    secrets:
      - c2-tls-cert               # TLS certificate
      - c2-tls-key                # TLS private key
      - c2-auth-key               # Authentication key
    
    # Network configuration
    networks:
      - c2-frontend               # Frontend network for client connections
      - c2-backend                # Backend network for database/cache
      - monitoring                # Monitoring network
    
    # Health check for container monitoring
    healthcheck:
      test: ["CMD", "python", "/app/healthcheck.py"]  # Custom health check script
      interval: 30s               # Check every 30 seconds
      timeout: 10s                # Timeout after 10 seconds
      retries: 3                  # Retry 3 times before marking unhealthy
      start_period: 60s           # Wait 60 seconds before first check
    
    # Security context (run as non-root user)
    user: "1000:1000"             # Non-privileged user
    read_only: true               # Read-only root filesystem
    tmpfs:
      - /tmp:size=100M            # Temporary filesystem in memory
      - /var/tmp:size=50M         # Variable temporary filesystem
    
    # Dependency management
    depends_on:
      postgres:
        condition: service_healthy  # Wait for database to be healthy
      redis:
        condition: service_healthy  # Wait for cache to be healthy
    
    # Security options
    security_opt:
      - no-new-privileges:true    # Prevent privilege escalation
      - apparmor:docker-default   # Use AppArmor security profile
    
    # Capabilities (drop all, add only necessary ones)
    cap_drop:
      - ALL                       # Drop all capabilities
    cap_add:
      - NET_BIND_SERVICE          # Allow binding to privileged ports
  
  # PostgreSQL database for persistent storage
  postgres:
    image: postgres:15-alpine     # Use lightweight Alpine Linux version
    container_name: c2-postgres   # Fixed container name
    restart: unless-stopped       # Restart policy
    
    # Environment variables for database configuration
    environment:
      - POSTGRES_DB=c2db          # Database name
      - POSTGRES_USER=c2user      # Database user
      - POSTGRES_PASSWORD_FILE=/run/secrets/database-password  # Password from secret
      - POSTGRES_INITDB_ARGS=--auth-host=md5  # Enable MD5 authentication
    
    # Volume for persistent database storage
    volumes:
      - postgres-data:/var/lib/postgresql/data  # Database files
      - ./db/init:/docker-entrypoint-initdb.d:ro  # Initialization scripts
    
    # Network configuration
    networks:
      - c2-backend                # Backend network only
    
    # Secrets for database password
    secrets:
      - database-password
    
    # Health check for database
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U c2user -d c2db"]  # PostgreSQL ready check
      interval: 10s               # Check every 10 seconds
      timeout: 5s                 # Timeout after 5 seconds
      retries: 5                  # Retry 5 times
    
    # Security configuration
    user: "999:999"               # PostgreSQL user
    read_only: false              # Database needs write access
    tmpfs:
      - /tmp:size=100M            # Temporary files in memory
    
    # Resource limits
    deploy:
      resources:
        limits:
          memory: 512M            # Maximum memory for database
        reservations:
          memory: 128M            # Reserved memory
  
  # Redis cache for session management
  redis:
    image: redis:7-alpine         # Latest Redis with Alpine Linux
    container_name: c2-redis      # Fixed container name
    restart: unless-stopped       # Restart policy
    
    # Redis configuration
    command: redis-server --requirepass $(cat /run/secrets/database-password) --maxmemory 256mb --maxmemory-policy allkeys-lru
    
    # Volume for Redis data
    volumes:
      - redis-data:/data          # Redis persistence
    
    # Network configuration
    networks:
      - c2-backend                # Backend network only
    
    # Secrets for Redis authentication
    secrets:
      - database-password
    
    # Health check for Redis
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]  # Redis ping check
      interval: 10s               # Check every 10 seconds
      timeout: 3s                 # Timeout after 3 seconds
      retries: 3                  # Retry 3 times
    
    # Security configuration
    user: "999:999"               # Redis user
    read_only: true               # Read-only filesystem
    tmpfs:
      - /tmp:size=50M             # Temporary files
  
  # Nginx reverse proxy for load balancing and SSL termination
  nginx:
    image: nginx:alpine           # Lightweight Nginx
    container_name: c2-nginx      # Fixed container name
    restart: unless-stopped       # Restart policy
    
    # Port mappings for external access
    ports:
      - "443:443"                 # HTTPS port
      - "80:80"                   # HTTP port (redirect to HTTPS)
    
    # Configuration files
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro  # Main configuration
      - ./nginx/conf.d:/etc/nginx/conf.d:ro           # Site configurations
      - c2-logs:/var/log/nginx:rw                     # Log files
    
    # TLS certificates
    secrets:
      - c2-tls-cert
      - c2-tls-key
    
    # Network configuration
    networks:
      - c2-frontend               # Frontend network
    
    # Health check
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    
    # Dependencies
    depends_on:
      - c2-server
    
    # Security configuration
    read_only: true
    tmpfs:
      - /var/cache/nginx:size=100M
      - /var/run:size=50M
  
  # Monitoring with Prometheus
  prometheus:
    image: prom/prometheus:latest  # Prometheus monitoring
    container_name: c2-prometheus # Fixed container name
    restart: unless-stopped       # Restart policy
    
    # Configuration
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./monitoring/rules:/etc/prometheus/rules:ro
    
    # Port for metrics access
    ports:
      - "127.0.0.1:9090:9090"     # Localhost only
    
    # Network configuration
    networks:
      - monitoring
    
    # Command line arguments
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'
    
    # Security configuration
    user: "65534:65534"           # Nobody user
    read_only: true
    tmpfs:
      - /tmp:size=100M
  
  # Grafana for visualization
  grafana:
    image: grafana/grafana:latest # Grafana dashboard
    container_name: c2-grafana    # Fixed container name
    restart: unless-stopped       # Restart policy
    
    # Environment variables
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin  # Change in production
      - GF_USERS_ALLOW_SIGN_UP=false      # Disable user registration
    
    # Port for dashboard access
    ports:
      - "127.0.0.1:3000:3000"     # Localhost only
    
    # Configuration and data
    volumes:
      - ./monitoring/grafana:/etc/grafana/provisioning:ro
    
    # Network configuration
    networks:
      - monitoring
    
    # Dependencies
    depends_on:
      - prometheus
    
    # Security configuration
    user: "472:472"               # Grafana user
```

### Hardened Server Dockerfile (server/Dockerfile.hardened)
```dockerfile
# Multi-stage build for security and efficiency
# Stage 1: Build dependencies and compile Python packages
FROM python:3.11-slim-bullseye AS builder

# Set build arguments for user configuration
ARG USER_ID=1000
ARG GROUP_ID=1000
ARG PYTHON_VERSION=3.11

# Install build dependencies in a single layer to reduce image size
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \                         # C compiler for Python extensions
    libc6-dev \                   # C library development files
    libffi-dev \                  # Foreign Function Interface library
    libssl-dev \                  # SSL/TLS library for cryptography
    cargo \                       # Rust compiler for cryptography packages
    && rm -rf /var/lib/apt/lists/* \  # Clean up package cache
    && apt-get clean               # Additional cleanup

# Create application directory with proper permissions
WORKDIR /build

# Copy requirements file first for Docker layer caching optimization
COPY requirements.txt .

# Install Python dependencies in virtual environment for isolation
RUN python -m venv /opt/venv && \
    /opt/venv/bin/pip install --no-cache-dir --upgrade pip setuptools wheel && \
    /opt/venv/bin/pip install --no-cache-dir -r requirements.txt

# Stage 2: Runtime image with minimal footprint
FROM python:3.11-slim-bullseye AS runtime

# Security labels for container identification
LABEL maintainer="security-team@example.com" \
      version="1.0" \
      description="Hardened C2 Server Container" \
      security.policy="restricted"

# Set build arguments
ARG USER_ID=1000
ARG GROUP_ID=1000

# Install only runtime dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \             # Certificate authorities for TLS
    curl \                        # HTTP client for health checks
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# Create non-root user and group for security
RUN groupadd -g ${GROUP_ID} c2user && \
    useradd -u ${USER_ID} -g ${GROUP_ID} -d /app -s /bin/bash -m c2user

# Copy Python virtual environment from builder stage
COPY --from=builder /opt/venv /opt/venv

# Set PATH to use virtual environment
ENV PATH="/opt/venv/bin:$PATH" \
    PYTHONPATH="/app" \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \    # Prevent .pyc file creation
    PIP_NO_CACHE_DIR=1 \           # Disable pip cache
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Create application directory structure
WORKDIR /app
RUN mkdir -p /app/data /app/logs /app/config && \
    chown -R c2user:c2user /app

# Copy application code with proper ownership
COPY --chown=c2user:c2user . /app/

# Create health check script
RUN echo '#!/bin/bash\n\
import requests\n\
import sys\n\
try:\n\
    response = requests.get("http://localhost:8443/health", timeout=5)\n\
    sys.exit(0 if response.status_code == 200 else 1)\n\
except:\n\
    sys.exit(1)' > /app/healthcheck.py && \
    chmod +x /app/healthcheck.py && \
    chown c2user:c2user /app/healthcheck.py

# Switch to non-root user
USER c2user

# Expose application port (documentation only, actual binding in docker-compose)
EXPOSE 8443

# Set security-focused entry point
ENTRYPOINT ["/opt/venv/bin/python", "-m", "c2_server"]

# Default command (can be overridden)
CMD ["--config", "/app/config/server.yml", "--log-level", "INFO"]
```

### Server Requirements (server/requirements.txt)
```txt
# Core dependencies with pinned versions for security
cryptography==41.0.7           # Cryptographic primitives and protocols
fastapi==0.104.1               # Modern web framework for APIs
uvicorn[standard]==0.24.0      # ASGI server with performance optimizations
pydantic==2.5.0                # Data validation and serialization
sqlalchemy==2.0.23             # Database ORM with type safety
alembic==1.13.1                # Database migration management
asyncpg==0.29.0                # Async PostgreSQL driver
redis==5.0.1                   # Redis client for caching
prometheus-client==0.19.0       # Metrics collection for monitoring
structlog==23.2.0              # Structured logging for security auditing
pyjwt==2.8.0                   # JSON Web Token implementation
httpx==0.25.2                  # Async HTTP client
psutil==5.9.6                  # System and process utilities
click==8.1.7                   # Command line interface creation
python-multipart==0.0.6        # Multipart form data parsing
websockets==12.0               # WebSocket protocol implementation

# Security-focused dependencies
pyotp==2.9.0                   # Time-based One-Time Password (TOTP)
bcrypt==4.1.2                  # Password hashing with salt
passlib[bcrypt]==1.7.4         # Password hashing library
python-jose[cryptography]==3.3.0  # JavaScript Object Signing and Encryption
secure==0.3.0                  # Security headers middleware

# Monitoring and observability
opentelemetry-api==1.21.0      # Distributed tracing API
opentelemetry-sdk==1.21.0      # Distributed tracing SDK
opentelemetry-instrumentation-fastapi==0.42b0  # FastAPI tracing
opentelemetry-exporter-prometheus==1.12.0rc1   # Prometheus metrics export

# Development and testing (only in dev)
pytest==7.4.3                 # Testing framework
pytest-asyncio==0.21.1        # Async testing support
pytest-cov==4.1.0             # Code coverage reporting
black==23.11.0                 # Code formatting
flake8==6.1.0                  # Code linting
mypy==1.7.1                    # Static type checking
bandit==1.7.5                  # Security linting
safety==2.3.5                  # Dependency vulnerability scanning
```
