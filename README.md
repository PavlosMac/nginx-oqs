# Post-Quantum Cryptography Nginx Server

A production-ready post-quantum cryptography (PQC) enabled nginx server with Node.js backend, containerized using Docker. This setup integrates Open Quantum Safe (OQS) libraries for hybrid traditional + post-quantum TLS encryption.

## Architecture

- **Frontend**: OQS-enabled nginx reverse proxy (HTTP port 8080, HTTPS port 8443)
- **Backend**: Node.js Express API server (internal port 3000)
- **Containerization**: Docker Compose orchestrates both services
- **Security**: Post-quantum certificates with hybrid cipher suites (ML-DSA-44, X25519MLKEM768)

## Hybrid Cryptography Explained

### Two Types of "Hybrid" in This Setup

#### 1. **Hybrid Cryptographic Algorithms** (The Real Security)
- **X25519MLKEM768**: Single algorithm that runs **both** classical (X25519) and post-quantum (Kyber768) simultaneously
- **Dual Protection**: Connection is secure even if either algorithm is broken
- **How it works**: Both algorithms generate keys, combined result protects the session
- **Security**: Protected against classical computers (X25519) AND quantum computers (Kyber768)

#### 2. **Hybrid Server Configuration** (Client Compatibility)
- **Multiple Certificate Support**: Server can present both traditional RSA and post-quantum ML-DSA-44 certificates
- **Client Choice**: Different clients can negotiate different security levels based on their capabilities
- **Fallback Strategy**: Traditional clients use classical algorithms, OQS clients can request post-quantum

### Client Connection Examples

| Client Type | Request Method | Key Exchange | Signature | Security Level |
|-------------|---------------|--------------|-----------|----------------|
| **OQS Client (Explicit PQ)** | `--curves X25519MLKEM768` | X25519MLKEM768 | mldsa44 | **Full Quantum-Safe** |
| **OQS Client (Auto)** | No curve specified | X25519 | RSA-PSS | Classical Fallback |
| **Traditional Browser** | Standard HTTPS | X25519 | RSA-PSS | Classical Security |

## Post-Quantum Cryptography Features

### ✅ Implemented
- **Hybrid Key Exchange Algorithm**: X25519MLKEM768 (simultaneous classical + post-quantum protection)
- **Post-Quantum Signatures**: ML-DSA-44 (NIST standardized quantum-safe digital signatures)
- **Quantum-Safe Symmetric Encryption**: AES-256-GCM with TLS 1.3
- **Multi-Client Support**: Serves both traditional browsers and post-quantum clients
- **Certificate Flexibility**: Can present both RSA and ML-DSA-44 certificates based on client capabilities

### Security Status
- **Maximum Security**: Hybrid algorithms provide protection against both classical and quantum attacks
- **Client-Negotiated**: Clients can explicitly request post-quantum algorithms for full quantum safety
- **Backward Compatible**: Traditional clients automatically fall back to secure classical algorithms
- **Future-Proof**: Ready for widespread post-quantum adoption with smooth migration path

## Key Configuration Changes

### nginx.conf (Post-Quantum Enabled)
```nginx
server {
    listen 443 ssl;
    server_name localhost;

    # Post-quantum certificates (ML-DSA-44)
    ssl_certificate /etc/nginx/certs/server-pq.crt;
    ssl_certificate_key /etc/nginx/certs/server-pq.key;
    
    # Traditional fallback certificates (optional for hybrid mode)
    ssl_certificate /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;
    
    # Post-quantum key exchange configuration
    ssl_ecdh_curve X25519MLKEM768:mlkem768:X25519:P-256;
    
    # TLS 1.3 required for post-quantum algorithms
    ssl_protocols TLSv1.3;
    
    # Post-quantum and traditional cipher suites
    ssl_ciphers TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
}
```

### docker-compose.yml (OQS Container)
```yaml
reverse-proxy:
  image: openquantumsafe/nginx:latest  # Key difference: uses OQS nginx instead of standard nginx
  container_name: nginx-reverse-proxy
  ports:
    - "8080:80"
    - "8443:443"
  volumes:
    - ./nginx.conf:/opt/nginx/nginx-conf/nginx.conf:ro  # Different mount path for OQS nginx
    - ./certs:/etc/nginx/certs:ro
```

### Key File Changes from Traditional Server
1. **nginx.conf**:
   - `ssl_ecdh_curve X25519MLKEM768:mlkem768:X25519:P-256;` (enables post-quantum key exchange)
   - `ssl_protocols TLSv1.3;` (required for PQ algorithms)
   - `include /opt/nginx/conf/mime.types;` (OQS nginx path)
   - Gzip disabled (not available in OQS build)
   
2. **docker-compose.yml**:
   - `openquantumsafe/nginx:latest` instead of `nginx:alpine`
   - `/opt/nginx/nginx-conf/nginx.conf:ro` mount path instead of `/etc/nginx/nginx.conf`
   
3. **Certificate Files**:
   - `server-pq.crt` / `server-pq.key` (ML-DSA-44 post-quantum certificates)
   - Generated using OQS tools instead of standard OpenSSL

## Certificate Generation

### Post-Quantum Certificates (ML-DSA-44)
```bash
docker run --rm -v $(pwd)/certs:/certs -e OPENSSL_CONF=/opt/oqssa/ssl/openssl.cnf \
  openquantumsafe/curl openssl req -x509 -new -newkey mldsa44 \
  -keyout /certs/server-pq.key -out /certs/server-pq.crt -nodes \
  -subj "/CN=localhost" -days 365 -provider oqsprovider -provider default
```

### Verify Post-Quantum Certificate
```bash
openssl x509 -in certs/server-pq.crt -text -noout | grep "Public Key Algorithm"
# Output: Public Key Algorithm: 2.16.840.1.101.3.4.3.17 (ML-DSA-44)
```

## Testing Commands and Success Criteria

### 1. Basic Connectivity Test
```bash
curl -k https://localhost:8443/nginx-health
```
**Success Criteria**: Returns `healthy` with HTTP 200

### 2. Post-Quantum Client Test (KEY COMMAND)
```bash
docker run --rm --network host openquantumsafe/curl \
  curl -k -v --curves X25519MLKEM768 https://localhost:8443/nginx-health
```

**Success Criteria for Full Quantum-Safe Security**:
- **Output should contain**: `* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / X25519MLKEM768 / mldsa44`
- **Key Exchange**: `X25519MLKEM768` (hybrid algorithm = simultaneous X25519 + Kyber768 protection)
- **Signature Algorithm**: `mldsa44` (post-quantum ML-DSA-44 digital signatures)  
- **Cipher Suite**: `TLS_AES_256_GCM_SHA384` (already quantum-safe symmetric encryption)
- **Protocol**: `TLSv1.3` (required for post-quantum algorithm support)

**What This Means**: Your connection is protected against both classical AND quantum computer attacks.

### 3. Traditional Client Fallback Test
```bash
curl -k -v https://localhost:8443/nginx-health
```
**Success Criteria**: Traditional clients should connect successfully with fallback to classical algorithms

### 4. Detailed TLS Analysis
```bash
docker run --rm --network host openquantumsafe/curl \
  openssl s_client -connect localhost:8443 -servername localhost \
  -provider oqsprovider -provider default -groups X25519MLKEM768
```

**Success Criteria**:
- **Certificate shows**: `sigalg: mldsa44`
- **No "Server Temp Key" line** (indicates post-quantum key exchange)
- **Protocol**: `TLSv1.3`
- **Cipher**: `TLS_AES_256_GCM_SHA384`

## Quick Start

### 1. Start the Post-Quantum Stack
```bash
docker-compose up --build -d
```

### 2. Test Post-Quantum Connectivity
```bash
# Test with post-quantum client (MAXIMUM SECURITY - recommended)
docker run --rm --network host openquantumsafe/curl \
  curl -k --curves X25519MLKEM768 https://localhost:8443/nginx-health
# Expected: Uses X25519MLKEM768 hybrid key exchange + mldsa44 signatures

# Test with traditional client (CLASSICAL FALLBACK)
curl -k https://localhost:8443/nginx-health
# Expected: Falls back to traditional X25519 + RSA for compatibility
```

### 3. View Logs
```bash
docker-compose logs reverse-proxy
docker-compose ps
```

## Key Findings

### 1. Client-Side Curve Negotiation is Critical
- **Issue**: OQS nginx was properly configured, but clients needed explicit post-quantum curve requests
- **Solution**: Use `--curves X25519MLKEM768` parameter for full post-quantum security
- **Impact**: Without explicit curves, clients fall back to traditional algorithms

### 2. Certificate Order Matters
- **Finding**: First certificate in nginx.conf is preferred
- **Best Practice**: Place post-quantum certificates first for preference
- **Fallback**: Traditional certificates provide backward compatibility

### 3. OQS nginx Configuration Differences
- **Mount Path**: `/opt/nginx/nginx-conf/nginx.conf` (not `/etc/nginx/nginx.conf`)
- **MIME Types**: `/opt/nginx/conf/mime.types` (not `/etc/nginx/mime.types`)
- **Missing Modules**: gzip and http2 modules not available in OQS build
- **Directive**: `ssl_ecdh_curve` works correctly with OQS curve names

### 4. Hybrid Algorithms Provide Best Security and Compatibility
- **Hybrid Algorithm Security**: X25519MLKEM768 runs both classical and quantum-safe algorithms simultaneously
- **Dual Protection**: Connection stays secure even if either X25519 OR Kyber768 is compromised
- **Client Compatibility**: Traditional clients use classical fallbacks, OQS clients can request full quantum-safety
- **Migration Path**: Smooth transition strategy for post-quantum adoption

### 5. Algorithm Naming
- **Standard**: Use NIST standardized names (`mldsa44` not `dilithium3`)
- **Curves**: `X25519MLKEM768` provides hybrid key exchange
- **Compatibility**: Both pure PQ (`mlkem768`) and hybrid curves supported

## Container Management

```bash
# Start containers
docker-compose up --build -d

# Stop containers
docker-compose down

# View nginx logs
docker-compose logs reverse-proxy

# Check container status
docker-compose ps

# Access nginx container
docker-compose exec reverse-proxy sh

# Restart with changes
docker-compose restart reverse-proxy
```

## Troubleshooting

### Common Issues

1. **"SSL_CTX_set1_curves_list failed"**
   - **Cause**: Unsupported curve names in `ssl_ecdh_curve`
   - **Solution**: Use supported OQS curve names: `X25519MLKEM768:mlkem768:X25519:P-256`

2. **Traditional Certificate Used Instead of Post-Quantum**
   - **Cause**: Certificate order or client curve negotiation
   - **Solution**: Place PQ certificate first, use `--curves` parameter on client

3. **"Connection Refused"**
   - **Cause**: nginx configuration error or container restart loop
   - **Solution**: Check logs with `docker-compose logs reverse-proxy`

### Health Checks

```bash
# Container status
docker-compose ps

# nginx configuration test (if needed)
docker-compose exec reverse-proxy nginx -t

# Port connectivity
curl -v http://localhost:8080/nginx-health
curl -k -v https://localhost:8443/nginx-health
```

## Security Implications

### Quantum-Safe Protection
- **Key Exchange**: Protected against quantum attacks via Kyber768
- **Digital Signatures**: Protected against quantum attacks via ML-DSA-44
- **Symmetric Encryption**: Already quantum-safe with AES-256

### Migration Strategy
- **Phase 1**: Hybrid server configuration (supports both traditional and PQ clients) ✅ Implemented
- **Phase 2**: Hybrid algorithm deployment (X25519MLKEM768 provides dual protection) ✅ Available  
- **Phase 3**: Client adoption (explicit `--curves` requests unlock maximum security) ✅ Ready
- **Phase 4**: Full post-quantum-only mode (optional future step for maximum security)

This setup provides a production-ready foundation for post-quantum cryptography adoption with full backward compatibility.