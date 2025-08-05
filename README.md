# Nginx + Node.js Docker Stack

A production-ready setup with Node.js Express application behind an Nginx reverse proxy, all containerized with Docker.

## Architecture

- **Node.js Application**: Express.js server running on port 3000
- **Nginx**: Reverse proxy handling HTTP requests, static files, and load balancing
- **Docker Compose**: Orchestrates both services with proper networking

## Features

- ✅ Reverse proxy configuration
- ✅ Static file serving with caching
- ✅ Rate limiting for API endpoints
- ✅ Security headers
- ✅ Health checks for both services
- ✅ Gzip compression
- ✅ Non-root user in containers
- ✅ Proper logging

## Quick Start

1. **Build and run the stack:**
   ```bash
   docker-compose up --build
   ```

2. **Run in background:**
   ```bash
   docker-compose up -d
   ```

3. **View logs:**
   ```bash
   docker-compose logs -f
   ```

4. **Stop the stack:**
   ```bash
   docker-compose down
   ```

## Testing the Setup

Once running, test these endpoints:

- **Main app**: http://localhost/
- **API endpoint**: http://localhost/api/data
- **Users API**: http://localhost/api/users
- **Health check**: http://localhost/health
- **Static files**: http://localhost/static/
- **Nginx health**: http://localhost/nginx-health

## Development

For local development without Docker:

```bash
npm install
npm run dev
```

## Production Considerations

- Configure SSL/TLS certificates for HTTPS
- Set up proper logging and monitoring
- Configure environment-specific variables
- Implement proper authentication for sensitive endpoints
- Consider using Docker Swarm or Kubernetes for scaling

## Troubleshooting

**Check container status:**
```bash
docker-compose ps
```

**View logs:**
```bash
docker-compose logs nginx
docker-compose logs node-app
```

**Access container shell:**
```bash
docker-compose exec node-app sh
docker-compose exec nginx sh
```

**Rebuild containers:**
```bash
docker-compose up --build --force-recreate
```
