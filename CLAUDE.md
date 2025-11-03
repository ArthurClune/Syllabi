# Syllabi Development Notes

This document contains important information about the Syllabi project setup, common issues, and solutions discovered during development.

## Docker Setup

### Overview
The project uses Docker Compose to orchestrate multiple services:
- **Backend**: FastAPI application (port 8000)
- **Worker**: Celery worker for background tasks
- **Frontend**: Next.js application (port 3000)
- **Redis**: Message broker for Celery (port 6379)

### Important Docker Considerations

#### Volume Mounts Override Built Images
The docker-compose.yml uses volume mounts like `./backend:/app` and `./frontend:/app`. This means:
- Local files overlay the built container files
- Scripts must be executable on the host machine, not just in the container
- If builds succeed but containers fail at runtime, check file permissions

**Solution for permission issues:**
```bash
chmod +x backend/start.sh backend/start-worker.sh
```

#### .dockerignore Files Are Critical
Without proper .dockerignore files:
- `node_modules` gets copied to build context (can be 1GB+)
- Build times increase dramatically
- Builds may fail with OOM errors

**Required .dockerignore files:**
- `backend/.dockerignore` - Must allow `start.sh` and `start-worker.sh` with exceptions
- `frontend/.dockerignore` - Must exclude `node_modules`, `.next`, etc.

#### Backend .dockerignore Pattern
The backend uses a pattern that blocks all `.sh` files but makes exceptions:
```
*.sh
!start.sh
!start-worker.sh
```
This prevents accidentally copying shell scripts while allowing required startup scripts.

### Common Docker Errors

#### Error: "exec: ./start.sh: permission denied"
**Cause**: Volume mount overlays local non-executable files over container files
**Solution**: Make scripts executable on host: `chmod +x backend/start.sh backend/start-worker.sh`

#### Error: Build context is too large
**Cause**: Missing or incomplete .dockerignore file
**Solution**: Ensure .dockerignore excludes node_modules, .next, .git, etc.

## Node.js & Dependencies

### React 19 Compatibility
The project uses React 19, which has breaking changes:
- Old `react-query@3.x` is incompatible
- Use `@tanstack/react-query@5.x` instead
- Some peer dependencies require `--legacy-peer-deps`

### Memory Issues During Build
Next.js builds can exceed default Node.js heap size:

**Solution in Dockerfile:**
```dockerfile
ENV NODE_OPTIONS="--max-old-space-size=4096"
RUN npm run build
```

### TypeScript & React 19
React 19's type system is stricter. Common issue with Blob creation:

**Error:**
```typescript
const blob = new Blob([fileData], { type: mimeType });
// Type 'Uint8Array<ArrayBufferLike>' is not assignable to type 'BlobPart'
```

**Solution:**
```typescript
const blob = new Blob([new Uint8Array(fileData)], { type: mimeType });
```

### Dependency Management Best Practices
- Use `npm ci --legacy-peer-deps` in Dockerfile for consistent builds
- Update `package-lock.json` with `npm install --package-lock-only` to avoid local installs
- Never commit `node_modules` to git

## Supabase Database Setup

### Database Initialization Process
Supabase uses a two-part schema system:
1. **schema.sql** - Base schema with tables, types, functions
2. **migrations/** - Incremental changes and policies

**Critical**: Migrations assume tables exist. Schema must be applied first.

### Setup Steps for New Database

1. **Apply base schema** (one-time setup):
   ```bash
   cd frontend/supabase
   psql "postgresql://postgres.PROJECT:PASSWORD@aws-0-REGION.pooler.supabase.com:6543/postgres" -f schema.sql
   ```

2. **Apply migrations**:
   ```bash
   cd frontend/supabase
   supabase db push
   ```

### Common Supabase Errors

#### Error: "relation public.chatbots does not exist"
**Cause**: Migrations run before base schema is applied
**Solution**: Apply schema.sql first, then run migrations

#### Error: SQL syntax error with "//"
**Cause**: JavaScript-style comments in SQL files
**Solution**: Use SQL comments `--` instead of `//`

### Supabase Commands Reference

```bash
# Link to remote project (one-time)
supabase link --project-ref PROJECT_REF

# Dry-run to see pending migrations
supabase db push --dry-run

# Apply pending migrations (use this after schema.sql is applied)
supabase db push

# Reset remote database (WARNING: drops all data, applies schema + migrations)
supabase db reset --linked
```

**Important**: `supabase db reset --linked` on remote databases:
- Does NOT automatically apply schema.sql
- Only runs migrations
- Drops all existing data
- Use `db push` for incremental updates instead

## Environment Variables

### Variable Naming Inconsistency
The project has an environment variable mapping issue:
- `.env` file uses `SUPABASE_SERVICE_ROLE_KEY`
- Backend `config.py` expects `SUPABASE_KEY`

**Solution in docker-compose.yml:**
```yaml
environment:
  - SUPABASE_KEY=${SUPABASE_SERVICE_ROLE_KEY}
```

### Required Environment Variables
See `.env.example` for all required variables. Key ones:
- `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY`
- `OPENAI_API_KEY`
- `BACKEND_API_KEY`
- `REDIS_URL` and `CELERY_BROKER_URL`

### Docker Compose Environment Mapping
Services need proper environment variable mapping:
```yaml
backend:
  environment:
    - SUPABASE_KEY=${SUPABASE_SERVICE_ROLE_KEY}  # Note the mapping
    - CELERY_BROKER_URL=redis://redis:6379/0     # Required, was missing
```

## Git Workflow

### Branch Strategy
Create feature branches for related changes:
```bash
git checkout -b container-fixes
git add .
git commit -m "Fix Docker configuration and build issues"
```

### Testing Before Committing
Always test changes before committing:
```bash
# Test Docker builds
docker-compose build

# Test services start correctly
docker-compose up -d

# Check service health
docker-compose ps
docker logs syllabi-backend-1
curl http://localhost:8000/health
```

## Common Issues & Solutions

### Issue: Changes not reflected in container
**Cause**: Docker layer caching
**Solution**: Build with `--no-cache` flag:
```bash
docker-compose build --no-cache SERVICE_NAME
```

### Issue: Service fails health check
**Cause**: Service not ready or misconfigured
**Debug**:
```bash
docker-compose ps  # Check status
docker logs SERVICE_NAME  # View logs
```

### Issue: Database connection fails
**Cause**: Wrong environment variable names or values
**Solution**: Check docker-compose.yml environment mapping matches backend expectations

## Next.js Configuration

### Invalid Configuration Warning
The project shows warnings about deprecated `eslint` config in next.config.js:
```
⚠ `eslint` configuration in next.config.js is no longer supported
⚠ Unrecognized key(s) in object: 'eslint'
```

This doesn't break builds but should be cleaned up by moving eslint config to `.eslintrc.json`.

### Middleware Deprecation
```
⚠ The "middleware" file convention is deprecated. Please use "proxy" instead.
```

Consider renaming middleware files to proxy for Next.js 16 compatibility.

## Quick Reference

### Start Development Environment
```bash
# Build all services
docker-compose build

# Start all services in background
docker-compose up -d

# View logs
docker-compose logs -f

# Stop all services
docker-compose down
```

### Rebuild Single Service
```bash
docker-compose build SERVICE_NAME
docker-compose up -d SERVICE_NAME
```

### Database Management
```bash
cd frontend/supabase

# See what would be applied
supabase db push --dry-run

# Apply migrations
supabase db push
```

### Check Service Health
```bash
# Backend health
curl http://localhost:8000/health

# Check all containers
docker-compose ps

# View specific service logs
docker logs syllabi-backend-1
docker logs syllabi-frontend-1
docker logs syllabi-worker-1
```

## Documentation Links
- Local Setup: https://www.syllabi-ai.com/docs/getting-started/local-setup
- Docker Compose Deployment: https://www.syllabi-ai.com/docs/deployment/docker-compose
- Supabase Dashboard: https://vlvrbofltokrkgltdtzr.supabase.co

## Notes for Future Development

1. **Always check .dockerignore** when adding new required files to containers
2. **Test locally before committing** to catch issues early
3. **Use --legacy-peer-deps** for npm operations due to React 19
4. **Schema must exist before migrations** - remember the two-step process
5. **Volume mounts override builds** - file permissions matter on host machine
