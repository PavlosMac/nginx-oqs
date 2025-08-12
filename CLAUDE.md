# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a post-quantum cryptography (PQC) enabled nginx server with Node.js backend, containerized using Docker. The project integrates Open Quantum Safe (OQS) libraries for hybrid traditional + post-quantum TLS encryption.

**Architecture:**
- **Frontend**: OQS-enabled nginx reverse proxy (port 8080 HTTP, 8443 HTTPS)
- **Backend**: Node.js Express API server (internal port 3000)
- **Containerization**: Docker Compose orchestrates both services
- **Security**: Post-quantum certificates with hybrid cipher suites (Dilithium3, Kyber768)

## Development Commands

### Docker Operations
```bash
# Build and run the full stack
docker-compose up --build

# Run in background
docker-compose up -d

# View logs
docker-compose logs -f
docker-compose logs nginx
docker-compose logs node-app

# Stop the stack
docker-compose down

# Rebuild containers
docker-compose up --build --force-recreate

# Check container status
docker-compose ps

# Access container shells
docker-compose exec api-server sh
docker-compose exec reverse-proxy sh
```

### Node.js Development
```bash
# Local development (without Docker)
npm install
npm run dev

# Production start
npm start
```

### Testing Endpoints
```bash
# Test HTTP endpoints
curl http://localhost:8080/
curl http://localhost:8080/api/data
curl http://localhost:8080/health

# Test HTTPS with post-quantum crypto (requires oqs-curl)
curl -k https://localhost:8443/nginx-health
```

## Key Architecture Components

### OQS nginx Configuration
- Uses `openquantumsafe/nginx` Docker image for post-quantum TLS support
- Hybrid cipher configuration supporting both traditional and post-quantum algorithms
- Certificate management in `./certs/` directory with RSA and Dilithium3 certificates
- TLS 1.2/1.3 with post-quantum key exchange (Kyber768, X25519Kyber768)

### Node.js Backend
- Express.js API server with health checks and rate limiting
- Runs on internal Docker network, not directly exposed
- Provides REST endpoints: `/`, `/api/data`, `/api/users`, `/health`

### Reverse Proxy Setup
- nginx handles HTTP/HTTPS termination and static file serving
- Proxy passes requests to Node.js backend via `nodejs_backend` upstream
- Rate limiting configured for `/api/*` routes (10r/s with burst=20)
- Security headers and gzip compression enabled

### Certificate Infrastructure
- Traditional RSA certificates for backward compatibility
- Post-quantum Dilithium3 certificates for quantum-safe encryption
- Hybrid mode supports both client types seamlessly
- Certificate volume mount: `./certs:/etc/nginx/certs:ro`

## Post-Quantum Cryptography Integration

This project implements the OQS integration plan documented in `planner.md`. Key aspects:

- **Hybrid TLS**: Supports both traditional browsers and post-quantum clients
- **Cipher Suites**: Kyber768 for key exchange, Dilithium3 for signatures
- **Testing**: Use `oqs-curl` for post-quantum connectivity testing
- **Performance**: Benchmarking available for traditional vs post-quantum algorithms

## File Structure

- `app.js` - Main Node.js Express application
- `nginx.conf` - nginx reverse proxy configuration with OQS settings
- `docker-compose.yml` - Multi-container orchestration
- `Dockerfile` - Node.js application container build
- `healthcheck.js` - Container health check script
- `certs/` - SSL/TLS certificates (traditional and post-quantum)
- `static/` - Static files served by nginx
- `planner.md` - OQS implementation roadmap and phase tracking

## Development Notes

- The project is actively implementing post-quantum cryptography features
- Use the OQS nginx image, not standard nginx, for quantum-safe TLS
- Certificate generation requires OQS tools for post-quantum algorithms
- All API routes have rate limiting configured in nginx
- Health checks are implemented for both containers with proper timeouts